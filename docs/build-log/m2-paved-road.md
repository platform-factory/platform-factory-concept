# M2: Paved road — in progress (opened 2026-09-02)

> **Status: OPEN.** No build command has run yet. This entry opens with the
> test-readiness walk the M1 close asked for — "what must exist for this
> test to run, and does it exist by this milestone?" — applied to C-05..C-08
> *before* anything is built. Claims are graded at the bottom when M2
> closes, including C-01 and C-03, deferred here from M1.

## Test-readiness walk (2026-09-02, before the first build command)

**Why this exists.** M1 closed with two of its five claims ungradeable — not
because the build failed, but because the register had scheduled their
tests into a milestone that could not run them (`m1-spine.md`, "Two claims
the register mis-assigned"). The fix named there was a question to ask of
every remaining claim *before* its milestone starts rather than at its
close. This section is that question, asked of M2.

**How to read it.** Each claim's test is taken literally, and every
prerequisite the test needs is listed with one of five tags:

- **exists** — present and verified on 2026-09-02.
- **PR** — GitOps work in the shape M2 is supposed to produce anyway.
- **apply** — a `terraform apply`. C-01's test counts these, so each one is
  named here up front rather than discovered later.
- **decide** — a design decision the build cannot proceed without, which
  by this repo's convention means an ADR.
- **unverified** — checked, and could not be confirmed either way.

**The answer in one line:** every M2 test can run at M2 — no C-01/C-03
repeat — but all four stand on the same missing floor (a cloud identity for
the Crossplane providers), and two of them cannot be *defined*, let alone
run, until four decisions are made that the register silently assumed.

### Starting state (verified 2026-09-02)

- `platform-config` holds Crossplane core and five GCP provider packages
  (storage, sql, cloudplatform, artifact, dns), all Healthy on the last
  rebuild. No XRD, no Composition, no `ProviderConfig` — its README says
  so. Nothing in the running platform can create a cloud resource yet. [C]
- `systems` and `svc-hello` are README stubs. [C]
- `platform-factory-ref` has 32 APIs enabled; `sqladmin`,
  `servicenetworking`, `secretmanager` and `cloudidentity` are not among
  them. [C]
- The cluster is regional, private-node, Workload Identity on, with no
  Google Groups for RBAC setting; the VPC has no Private Services Access.
  [C, from `platform-bootstrap` layers 1 and 2]
- The project lives directly under a Google Workspace org. At the org
  level, project creation and billing-account creation are granted
  domain-wide to humans; no workload identity holds either. [C,
  `gcloud organizations get-iam-policy`]
- The cluster is down (C-02 rhythm). Every test below starts with a
  ~17-minute `cycle.sh up`. [C]

### The floor every test stands on

Three prerequisites are shared by all four claims, so they are listed once.

1. **A GCP identity for the providers — apply.** A Crossplane provider
   creates cloud resources with whatever identity its `ProviderConfig`
   names, and none exists. At v3.0.0 the provider family documents one
   keyless path (`docs/family/Configuration.md`, read 2026-09-02): a
   Google service account, a `roles/iam.workloadIdentityUser` binding from
   the provider pod's Kubernetes service account, an annotation on that
   Kubernetes service account, and `credentials.source: InjectedIdentity`.
   [C] Two consequences. The Google service account and its roles are
   layer-0 Terraform — the "one structural crossing per component" that
   surprise 17 predicted, arriving on schedule. And the provider's
   Kubernetes service account is controller-managed and named after the
   `ProviderRevision`, so a `DeploymentRuntimeConfig` pinning a stable
   name is part of the PR, or the binding breaks on every provider bump.
   [I] The roles it needs accrete per claim below: Artifact Registry admin
   (C-05), Cloud SQL admin (C-07), project creator at the org (C-08).
2. **`svc-hello` as a running service — PR.** C-06 moves it and C-07 gives
   it a database; today it is a README. Minimum: a container image, a
   `Deployment` in `k8s/`, and an Argo Application syncing that directory
   into its tenant's namespace. The image needs somewhere to live —
   `platform-factory-ref` has only pull-through remotes, no standard
   Artifact Registry repository — which is exactly what C-05's "Artifact
   Registry prefix" is for, so the tenant Composition lands first. A CI
   identity to push images (GitHub Actions → Workload Identity Federation
   pool) would be another layer-0 apply; for M2 the image can be built and
   pushed once by hand, and CI can wait for the M4 change class that
   actually needs it. **decide** (small): hand-push now vs. CI identity now.
3. **The cluster up, and `cycle.sh` still honest — see surprise 1.** M2 is
   the first milestone in which things Crossplane creates outlive the
   cluster, and that changes what "rebuild from empty" means.

### C-05 — One YAML per tenant

Test: onboard a second tenant with one file. Data: lines of YAML;
wall-clock from PR merge to usable namespace.

- **`System` XRD + Composition — PR.** Cluster-scoped, per the research
  doc: a Cluster XR may compose cluster-scoped objects *and* namespaced
  ones in any namespace, which is what lets one recipe put an `AppProject`
  into `argocd` and a `RoleBinding` into the tenant namespace. [C]
- **Crossplane's own Kubernetes permissions — PR, with a spike.**
  Crossplane composes "some" built-in kinds out of the box; the research
  doc says to assume an aggregated `ClusterRole` covering `Namespace`,
  `ResourceQuota`, `RoleBinding`, `AppProject` is required until tested.
  `RoleBinding` is the interesting one: Kubernetes forbids granting a
  permission the granter doesn't hold, so Crossplane must itself hold
  every permission the tenant role confers, or hold the `bind` verb.
  [C mechanics / I recipe]
- **`RegistryRepository` + `RegistryRepositoryIAMMember` — exists.** Both
  kinds ship in provider-gcp-artifact v3.0.0 at both scopes (CRD list
  checked 2026-09-02). [C]
- **An Argo Application for the `systems` repo — PR.** Same shape as M1's
  providers Application: the `System` kind doesn't exist until the XRD
  Application is Healthy, so it sits in a later wave with the retry
  backstop. Argo CD ≥ 3.1 carries built-in health for `*.crossplane.io`
  kinds and this cluster runs 3.4.6, so "Healthy" on a `System` should
  mean Ready without another Lua check. [C from research; verify on the
  first sync]
- **A first tenant — PR to `systems`.** The test is the *second* file;
  `svc-hello`'s team is the first.
- **"Usable" needs a definition — decide (cheap).** Proposed: the XR
  reports Ready *and* a Deployment applied to the namespace is admitted
  under the tenant's AppProject. Merge time from the GitHub API, Ready
  time from the XR's condition timestamp.

Verdict: runnable at M2. One apply (the floor); everything else is
PR-shaped.

### C-06 — Ownership moves without re-plumbing

Test: move `svc-hello` between teams. Data: files touched (target 1);
anything that had to be re-created.

- **The tenant model — decide, and decide first.** The claim says
  "identity binds at one point," but nothing in the design says whether a
  *team* is a *System*. If it is, moving `svc-hello` to another team means
  moving it to another namespace — a redeploy, not a YAML edit, and the
  test would measure the wrong thing. If a System carries a `team` field
  that can change while the namespace stays (or owns services with a team
  each), the edit is real. This decision shapes the C-05 XRD's schema, so
  it precedes C-05's *build*, not just C-06's test.
- **Identity groups — unverified.** The design chain is IdP group → cloud
  IAM → namespace RBAC. The org is Google Workspace, so Google Groups are
  the IdP groups, and the test needs at least two team groups. Whether
  any exist could not be checked: the Cloud Identity API is not enabled on
  any project this identity can use as a quota project — itself a small
  prerequisite.
- **Google Groups for RBAC on the cluster — apply (2-cluster).** GKE only
  resolves group membership if the cluster is created with a security
  group. Google's setup doc (read 2026-09-02) requires a domain group
  named exactly `gke-security-groups`, with the team groups nested inside
  it and no individual users, and the cluster flag pointing at it. [C]
  `2-cluster/gke.tf` has no `authenticator_groups_config`. Because
  2-cluster is disposable the setting lands on the next rebuild, but it is
  still a Terraform change and C-01 counts it.
- **The IAM hop — decide.** GKE requires `container.clusters.get` (in
  `roles/container.clusterViewer`) just to *authenticate* to a cluster;
  RBAC applies only after that. [C, GKE RBAC doc, 2026-09-02] So each team
  group also needs a project-level IAM binding. Either the Composition
  creates it (`ProjectIAMMember` exists; Crossplane's identity then needs
  project IAM admin, a strong grant) or layer 0 grants it once to
  `gke-security-groups`. The choice decides whether "files touched" can
  honestly be one.
- **A second identity to prove the move — manual.** `kubectl auth can-i
  --as-group` skips the IdP and tests RBAC alone. The real test needs a
  human in the destination group and not the source one, or one human
  whose group membership changes between the two checks.
- **`svc-hello` running in the source team's namespace — the floor.**

Verdict: runnable at M2 *after* the tenant-model decision. Two applies
(floor + groups setting), one Workspace-admin task, one design decision.

### C-07 — Guardrails replace review for databases

Test: (a) PR → usable database, timed; (b) an oversized instance and a
wrong region denied at admission; (c) delete the claim, database survives.

**(a) PR → usable database**

- **`Database` XRD + Composition — PR.** Namespaced XR composing
  `DatabaseInstance`, `Database`, `User`. All three verified at both
  scopes in M1 (C-03's doc half). [C]
- **`sqladmin.googleapis.com` — apply, or decide.** Not enabled. Layer 0
  owns the API list, so the honest path is one more entry there. The
  alternative — a `ProjectService` MR from Crossplane (kind exists [C]) —
  moves it to a PR but requires the provider identity to hold
  service-usage admin, and it would be the first API the platform enables
  for itself. Recommend layer 0, bundled with the floor apply.
- **Cloud SQL admin on the provider identity — apply (same run).**
- **A network path from pods to the database — apply (1-network).**
  ADR-0009's corp-real posture means private IP. Private-IP Cloud SQL
  needs Private Services Access: a reserved range on the VPC and a
  service-networking peering, plus the `servicenetworking` API. None
  exists. The Crossplane kinds do (`compute.GlobalAddress`,
  `servicenetworking.Connection`, both scopes [C]) but live in
  `provider-gcp-compute` and `provider-gcp-servicenetworking`, neither
  installed. The peering is reachability, and reachability persists
  (the layer boundary rule), so it belongs in 1-network as Terraform
  regardless. **decide** an alternative only if PSA is unwanted: Private
  Service Connect, or the Auth Proxy over a public IP, which is not
  corp-real.
- **The credential path into the app namespace — decide, ADR.** ADR-0003
  says managed rotation and no credentials in git. The research doc says
  Cloud SQL has no equivalent of the AWS managed-master-password path and
  the GCP secret flow "needs its own design." Three candidates: (i) the
  `User` MR writes a Crossplane-generated password to a connection Secret
  in the namespace — simplest, no rotation; (ii) External Secrets Operator
  + Secret Manager — matches the design doc, but ESO is not installed and
  the API is off; (iii) Cloud SQL IAM database authentication with the
  connector — no password exists at all, closest to ADR-0003's spirit.
  "Usable" in test (a) means the chosen path works end to end, so the
  test cannot be defined until this is.
- **Timing — exists.** Merge timestamp from GitHub; Ready from the
  `DatabaseInstance` condition. Cloud SQL creation takes minutes, not
  seconds, so the number is easy to read. [I]

**(b) Denied at admission**

- **Something that denies — decide.** The XRD's own OpenAPI schema can
  deny a wrong region (`enum`) and an oversized tier (`enum` or a CEL
  rule) at admission with no Kyverno at all. Kyverno is on M2's build
  list but not installed (its images are on ghcr.io, so the existing
  remote is expected to cover it). The claim text accepts either
  ("Kyverno/Composition bounds"), but the data point is "denial messages
  as seen by the developer," and the two mechanisms produce different
  messages. Recommend the schema first, because it is free, and Kyverno
  for what a schema cannot express (cross-field rules, cluster-wide
  budgets); record both messages.

**(c) Database survives claim deletion**

- **`managementPolicies` without `Delete` on `DatabaseInstance` — PR.**
  Set at authoring time; functions don't run at delete. [C]
- **The interaction with C-02 — decide.** See surprise 1.

Verdict: runnable at M2 after two decisions (credential path, admission
mechanism). Three applies (floor, API + role, PSA), bundleable into one
layer-0 run and one 1-network run.

### C-08 — Schema survives the Composition swap (stretch)

Test: an alternate Composition for the same XRD, no schema change. Data:
schema fields that leaked implementation detail.

- **The C-05 XRD and first Composition — M2.**
- **A second Composition mapping a System to a GCP project — PR.**
  `Project`, `ProjectIAMMember`, `RegistryRepository` in the new project.
  All present at v3.0.0, at both scopes. [C]
- **Project-creation power for the provider identity — apply above
  layer 0, and a governance question.**
  `roles/resourcemanager.projectCreator` on the org and
  `roles/billing.user` on the billing account. Today the first is granted
  domain-wide to humans; no workload holds either. No Terraform layer
  manages the org — layer 0 *creates a project under* it — so this is
  either a new layer above 0 or a manual grant, and either way it is a
  decision about whether the paved road may mint projects at all.
- **Quota headroom — check.** Billing accounts carry a project-creation
  quota; confirm before the test, not during. [I]
- **Composition selection without a schema change — exists.**
  `spec.crossplane.compositionRef` is reserved plumbing, not developer
  schema, so pointing an XR at the alternate is legal by construction. [C]
- **The real data point is a review of the C-05 XRD.** A field like
  `quota.cpu` is a `ResourceQuota` wearing a costume; a project has no
  such thing. Writing C-05's schema in terms of intent ("size class")
  rather than mechanism costs nothing now and is the whole of C-08's
  evidence.

Verdict: runnable at M2 only if the org-level grant is accepted. The
register already marks it stretch; the one thing to decide *early* is the
governance question, because a "no" is also a valid C-08 outcome.

### What the walk does to C-01

C-01's test counts `terraform apply` runs after M1. The walk finds four,
all of the "identity and reachability" shape surprise 17 predicted:

| Layer | Change | Needed by |
|---|---|---|
| 0-foundation | provider service account, WI binding, roles; Cloud SQL + Service Networking APIs | C-05, C-07 |
| 1-network | Private Services Access (reserved range + peering) | C-07 |
| 2-cluster | Google Groups for RBAC (`authenticator_groups_config`) | C-06 |
| org (above layer 0) | project creator + billing user for the provider identity | C-08 only |

Plus the one already on the counter: the `xpkg-upbound-io` remote removal
(platform-bootstrap PR #10, applied 2026-09-02). Bundling each layer's
changes into a single apply at M2 start keeps the count at one crossing
per layer rather than one per discovery — the honest number, since the
register's target is zero and the grade will hinge on *why* each one could
not be a PR.

### Decisions that gate the build (ADRs pending)

In the order the build needs them:

1. **Tenant model** — is a team a System, or a field on one? Shapes the
   C-05 schema; decides what C-06 measures.
2. **Database credential path** — Crossplane-written Secret, ESO + Secret
   Manager, or IAM database authentication. Defines "usable" in C-07(a).
3. **Admission mechanism for denials** — XRD schema/CEL, Kyverno, or
   both. Defines the messages C-07(b) records.
4. **Resources that outlive the cluster** — what `cycle.sh down` does
   about Cloud SQL instances and Artifact Registry repositories Crossplane
   created, and what `up` expects to find. Surprise 1.

Smaller, decided in passing: hand-push the `svc-hello` image vs. a CI
identity now; Composition-managed vs. layer-0 IAM for team groups; PSA
vs. an alternative for the database network path.

### Build order the walk implies

1. The four ADRs above.
2. One apply per layer (0, 1, 2) with every M2 change bundled; rebuild.
3. `System` XRD + Composition, the `systems` Application, first tenant.
4. `svc-hello` deployed into it.
5. `Database` XRD + Composition; Kyverno if decision 3 says so.
6. Run C-05 (second tenant), then C-07, then C-06.
7. C-08 if the org-level grant is accepted and time remains.

### Unverified at walk time

- Whether any Google Groups exist in the Workspace (needs the Cloud
  Identity API on a usable quota project).
- Whether `InjectedIdentity` works with the *direct* federated-principal
  form `0-foundation/iam.tf` already uses for Crossplane's package pulls
  (`principal://…/subject/ns/…/sa/…`, no Google service account). The
  v3.0.0 docs describe only the service-account-impersonation form; if
  the direct form works, the floor apply shrinks to IAM bindings alone.
- Crossplane's default composable-kinds list (research watch item; still
  undocumented).
- Project-creation quota on the billing account.

## Built

Nothing yet. This entry opens before the first build command.

## Data

Collected as the build runs. The register asks for: C-05 lines of YAML
per tenant and merge→usable wall-clock; C-06 files touched and anything
re-created; C-07 minutes PR→ready, denial messages verbatim, state after
claim deletion; C-08 leaked schema fields. C-01's apply count continues
from one.

## Claims graded

At M2 close. C-01 and C-03 (deferred from M1) are graded here alongside
C-05..C-08.

## Research verified

- **[C] provider-upjet-gcp v3.0.0 kinds beyond the C-03 list (checked
  2026-09-02** against the shipped CRDs under `package/crds/` at tag
  v3.0.0): `artifact.RegistryRepositoryIAMMember`, `cloudplatform.Project`
  / `ProjectService` / `ProjectIAMMember` / `Folder`,
  `servicenetworking.Connection`, `compute.GlobalAddress` — every one at
  both scopes. The last two are in provider packages `platform-config`
  does not install.
- **[C] provider-upjet-gcp v3.0.0 Workload Identity path**
  (`docs/family/Configuration.md` at v3.0.0): Google service account +
  `roles/iam.workloadIdentityUser` on the provider's Kubernetes service
  account + annotation + `credentials.source: InjectedIdentity`. The
  Kubernetes service account is controller-managed unless pinned.
- **[C] GKE Google Groups for RBAC** (Google's setup doc, 2026-09-02): a
  domain group named exactly `gke-security-groups`, team groups nested in
  it, no direct user members, cluster created with `--security-group`.
- **[C] GKE authentication requires IAM before RBAC** (GKE RBAC doc,
  2026-09-02): `container.clusters.get`, included in
  `roles/container.clusterViewer`, is required to authenticate to any
  cluster in the project and authorizes nothing inside it.
- **[C] Cloud SQL instance names are reusable immediately after
  deletion** (Cloud SQL delete-instance doc, 2026-09-02). Recorded
  because the opposite — a week-long reservation — is widely repeated
  and was this walk's own first assumption; it would have been a false
  constraint on the C-02 rhythm at M2.

## Surprises (running list)

1. **C-07(c) quietly rewrites C-02's test.** A database whose claim
   deletion cannot delete it also survives `cycle.sh down`, because
   Crossplane dies with the cluster and the Cloud SQL instance does not.
   On `up`, Argo recreates the claim, Crossplane recreates the managed
   resource, and the provider must *adopt* the existing instance rather
   than fail on it. "Rebuild from empty" stops being from empty the
   moment the paved road creates something durable — and that is the
   paved road's whole purpose. Two consequences the M1 script has no
   notion of: the rebuild now has an adoption step to verify, and an idle
   instance bills between sessions, so `down` needs an explicit decision
   about whether to delete what C-07(c) says must not be deleted
   automatically. (The name-reservation worry that first came with this
   was checked and dismissed — see Research verified.) Same family as
   surprise 9: the claim's test is a requirement on the build — and here
   one claim's test is a requirement on *another claim's* build.
2. **The register's C-06 test presumes a tenant model the design never
   states.** "Move `svc-hello` between teams" is a one-file edit only if
   a team is a *field* on a System; if a team *is* a System, the move is
   a namespace migration and the test measures deployment mechanics
   instead of identity binding. Found by asking what the test needs to
   exist — the answer was "a decision," and it turned out to gate C-05's
   schema too. Pre-registration caught vague success criteria (ADR-0008)
   and un-runnable tests (surprise 9); this is a third kind: a test that
   is runnable under either reading and means something different under
   each.
