# ADR-0022: What the M2b build changed in the engine-swap decisions

Status: Accepted 2026-10-31, at M2b's close · supersedes
ADR-0017 decision 7's list of identities its update rule lets through, and
the sentence in ADR-0017's Consequences that a Kyverno outage "now blocks every
write to the gated kinds"; amends ADR-0019's headline to what its guards cover;
states the `allowExistingViolations` rule ADR-0017 left implicit; decides the
`/status` gap that `62b9451` left for this close · from the M2b build
(2026-10-06 rehearsal, Session A 2026-10-19, the C-33 rehearsal and S1 on
2026-10-27, S3 on 2026-10-29)

## Context

ADR-0017, ADR-0018 and ADR-0019 were written before any M2b build command ran.
The build's evidence is in [`docs/build-log/m2b-engine-swap.md`](../build-log/m2b-engine-swap.md). ADR-0008 says a
claim that works only after a design change is graded ADJUSTED and needs a
superseding ADR, linked. ADR-0016 did this for M2, and this ADR does it for M2b.
It changes only what the build showed to be wrong or incomplete, and names the
evidence for each change. The findings, one line each:

1. **The reality gate admitted every edit.** Both of `deny-raw-cloud-resources`'
   rules said `allowExistingViolations: true`. On an update Kyverno judges the
   new object and the old one, and a deny with no conditions refused both, so
   every update counted as "already refused before" and was let through. The
   2026-10-06 rehearsal found it with a team member's tier change and label
   (build log, C-28; `62b9451`).
2. **The gate then refused the garbage collector.** Argo CD prunes in the
   foreground, and the garbage collector removes the `foregroundDeletion`
   finalizer with an UPDATE. ADR-0017 decision 7 let the garbage collector
   through on DELETE only, so four pruned IAMPolicyMembers stayed Terminating
   after their cloud grants were gone (Session A, first team move; `5da7d4a`).
3. **"A Kyverno outage blocks every write" is true of only one kind of
   outage.** If the admission controller crashes or is unready, its webhook
   configurations stay registered, and C-33 (e) observed the prediction: writes
   refused, the namespace finishing its teardown 41 s after Kyverno returned,
   and one manual sync per failed Application. If its Deployment is scaled to
   zero or deleted, Kyverno deletes its own webhook configurations as it shuts
   down, and nothing is refused until it returns [C, kyverno v1.19.2
   `pkg/webhooks/server.go` `Stop` → `cleanup`, gated by
   `pkg/utils/runtime/utils.go` `IsGoingDown`: Deployment deleted, not found,
   or `spec.replicas == 0`; kyverno#6554, closed]. C-33's registered step was
   the second kind (build log, C-33 (e), Surprises).
4. **ADR-0019's guards cover a file's presence and name, not the name inside
   it.** An in-place edit of `metadata.name` in a tenant file, with the file not
   renamed, made the old tenant's own Application prune the old tenant's
   Applications, AppProject and Config Connector objects with no person's step.
   The Namespace and the workloads were left behind, owned by nothing (C-33
   extra case (x), outside the registered cases; platform-config#34).
   `systems`' `[name-matches-file]` check would have failed on that PR. It is
   not a required status check, and the rehearsal did not run it.
5. **A write to a cloud object's `/status` is not sent to Kyverno.** The webhook
   is registered for resources, not subresources (`62b9451`'s body, which left
   the decision to this close).

## Decision

### 1. The update rule lets the garbage collector through — supersedes ADR-0017 decision 7's list for UPDATE

`deny-raw-cloud-resources`' create-and-update rule lets through Argo CD's
application controller, `cnrm-controller-manager`, `cnrm-deletiondefender` and
`kube-system/generic-garbage-collector`. The last is the name GKE presents
[C, Session A audit; `5da7d4a`]. The garbage collector only edits metadata.
Letting it through admits no change to a cloud object's spec, because the
engine reconciles spec and the garbage collector does not write it. The
same exemption applies to every later rule that refuses a kind Argo CD
prunes. M3's `deny-edge-objects` is the first such rule (the M3 readiness
note's correction of 2026-10-27).

### 2. Every deny with no conditions says `allowExistingViolations: false`

This is the rule ADR-0017 decision 7 needed and never stated: without it, a
deny with no conditions refuses nothing on UPDATE. `scripts/check_charts.py`
enforces it for every rule in `kyverno/policies/`, and fails on the policy as it
was before `62b9451`.

### 3. What a Kyverno outage blocks — supersedes ADR-0017's Consequences sentence

A Kyverno outage blocks every write to the gated kinds only while Kyverno's
webhook configurations are registered: a crash, an unready pod, or a node lost
with the pod on it. In that case Argo CD's syncs, the engine's finalizer
updates and a namespace's teardown wait, and recovery is one manual sync per
Application whose retries ran out (C-33 (e), attempt 2). A Deployment scaled
to zero or deleted opens the gates instead, by Kyverno's own code, until the
admission controller returns. A rolling update or node drain keeps
`spec.replicas` above zero and should not trigger the deletion [I, from
`IsGoingDown`; not run]. Only someone who can change the Deployment's desired
state can open the gates this way, which in this design means the platform's
own Argo CD from `platform-config`, or a cluster admin. This ADR does not
choose a remedy, because the gate is C-11's subject in M3. M3's readiness note
gets a dated entry, before C-11's first cluster case, that either adopts a
remedy or accepts this window with the fact stated.

### 4. ADR-0019's headline, read as what its guards cover

ADR-0019's guards cover these cases: a file removed, renamed or misnamed;
`tenants/` renamed; the ApplicationSet removed, renamed or deleted; a claims
file emptied or misnamed; a tenant file that will not render; a chart that
stops rendering the Namespace. They do not cover a tenant file whose
`metadata.name` changes in place. That deletes the old tenant's Kubernetes
objects through its own Application. Cloud objects that carry the abandon
annotation (ADR-0015) survive in the cloud, but the old tenant is gone.
The headline "a removed, renamed or broken file deletes no tenant by itself"
is read with this exception until a guard exists. The candidates are:
- making `systems`' `[name-matches-file]` a required status check (it fails
  on exactly this edit);
- naming generated Applications from the file's path, not from the name
  inside it;
- a `charts/system` guard comparing the rendered name with the Application's.

This ADR decides one of them: `systems`' `[name-matches-file]` becomes a
required status check once it has failed once and passed once on a real PR,
as ADR-0019 §4 does for every check. That trigger has not been recorded for it
in the M2b evidence. The other two candidates are left to platform-config#34.

### 5. The `/status` gap is accepted, and stated

A team member can write a cloud object's `/status` in the team's own
namespace. RBAC admits it, because the aggregated `admin` role covers the
subresource, and Kyverno never sees it, because the webhook is registered for
resources, not subresources. S3 showed both on 2026-10-29: `kubectl auth can-i
patch sqlinstances --subresource=status` answered `yes` for a member of
`payments`, and a server-side dry run of a status patch on `svc-hello-main`
returned `patched` (build log, C-28 (b)).

What a status write can and cannot reach, read in source before this ADR
was accepted:

- **It cannot change anything but `status`.** On a `/status` update the API
  server copies the stored object and keeps only the request's `status` and
  `managedFields`. Spec, labels, annotations (the abandon annotation
  included) and finalizers are reset to the stored values [C,
  apiextensions-apiserver v0.35.0
  `pkg/registry/customresource/status_strategy.go` `PrepareForUpdate`,
  L66–90].
- **It does not choose the cloud resource the engine acts on.** For all six
  kinds the charts render, the engine derives the target from spec or
  metadata (`spec.resourceID` or `metadata.name`) and the project annotation
  or namespace, never from status [C, k8s-config-connector v1.156.0]:
  - `SQLInstance`, direct controller: `sqlinstance_controller.go` L526–563;
    its status type has no `externalRef`.
  - `SQLDatabase` and `SQLUser`, Terraform controller: import IDs from
    `config/servicemappings/sql.yaml` L27–32 and L124–130. Their
    `instanceRef` resolves through `pkg/krmtotf/references.go`
    `resolveDefaultTargetFieldValue` to the referenced instance's
    `GetResourceID()` (`resource.go` L306–328). The status-reading branch for
    direct kinds is commented out at this tag (L236–245).
  - `IAMServiceAccount` and `ArtifactRegistryRepository`, Terraform: ID
    templates `iam.yaml` L77 and `artifactregistry.yaml` L39, filled from
    spec.
  - `IAMPolicyMember`: `pkg/controller/iam/iamclient/tfiamclient.go`
    `getResourceID` L526–547 builds the referenced object's import ID from
    the same templates.
- **One reference type does read status.** `memberFrom.serviceAccountRef` on
  an IAM member takes the member's email from the referenced
  `IAMServiceAccount`'s `status.email` (`tfiamclient.go` L658–670, L730–744;
  `iam.yaml` L74–76). A tenant who forged that field could change who is
  granted. No chart on `main` or on the M3 branches uses `memberFrom`,
  `IAMPartialPolicy` or `IAMPolicy` (`git grep`, 2026-10-29).
  **This ADR makes that a rule:** a platform chart names IAM members
  literally, and uses neither `IAMPartialPolicy` nor `IAMPolicy`.
  `scripts/check_charts.py` enforces it from platform-config#40 (`b65aa43`),
  which merges with M3's charts. The check renders every real tenant, scans
  the charts' text, and is shown refusing a `memberFrom` member and an
  `IAMPartialPolicy`.
- **What a status write can do:**
  - Forge `conditions` (Ready), `observedGeneration` or `observedState`.
    Argo CD's health check and `cycle.sh`'s verify read those. The engine
    does not reconcile on a status-only change. Its periodic requeue rewrites
    status after 5 to 15 minutes, mean 10, unless the object's
    reconcile-interval annotation says otherwise [C, `directbase_controller.go`
    L115–123 and L279–284; `pkg/controller/jitter/jitter.go` L66–74;
    `pkg/k8s/constants.go` L35–36].
  - A forged `Ready=False` on a referenced object stops its dependents
    reconciling. Those are the tenant's own `SQLDatabase`, `SQLUser` and IAM
    members, which wait on the Ready gate (`references.go` L192;
    `tfiamclient.go` L619–621). That is a stall of the tenant's own objects,
    not a reach into another tenant's.

So the gap is a false health reading, bounded by one reconcile period, plus
a self-inflicted stall. It is not a door to another tenant's resources while
no chart uses `memberFrom`. Closing it would mean one of:
- a Kyverno match on `*/status` for the gated kinds, which puts Kyverno in
  the path of every status write the engine makes (the engine's identity is
  already exempt, so the cost is latency and a new fail-closed dependency on
  the engine's own writes);
- an RBAC carve-out from `admin`.

Neither is worth that cost for a reading the engine corrects within 15
minutes.

## Consequences

- C-28's ADJUSTED links §1 and §2. C-33's scope note links §3 and §4.
- From §5: platform charts name IAM members literally, and use neither
  `IAMPartialPolicy` nor `IAMPolicy`. `scripts/check_charts.py` enforces it
  (platform-config#40, `b65aa43`).
- Not decided here: C-11's Kyverno remedy (M3); repairing the billing export
  (C-32); the other two guards for case (x) (platform-config#34); anything the
  build has not measured;
  - Argo CD's retry-pinning: a failed sync retries at the revision it started
    with, so a fix or a restore is not read until the operation is cleared
    (Session A's move 2; S3's `svc-scratch` onboarding; build log Surprise 7);
  - `down` not parking the instances (build log Surprise 16);
  - the release path for a stuck delete in a tenant namespace. S3 (b) ran in a
    namespace with no System label, where the operator's annotate was
    admitted. In a tenant namespace `deny-raw-cloud-resources` refuses it, and
    S3 showed the operator holds no exemption.
- No grade of C-05..C-07 changes, and no M3 ceiling moves. C-33's register text
  is not reworded. Its (e) step is recorded in the build log as testing the
  wrong kind of outage.
