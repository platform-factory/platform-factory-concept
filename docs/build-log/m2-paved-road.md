# M2: Paved road — 2026-09-17 (opened 2026-09-02; built 2026-09-16; tested 2026-09-16 and 2026-09-17)

> **Status: CLOSED 2026-09-17**, on the evidence below. Five claims are
> graded ADJUSTED or HELD, one stays UNTESTED with the reason, and the design
> changes the build forced are decided in
> [ADR-0016](../adr/0016-what-the-m2-build-changed.md). The short version: the
> paved road works — a tenant from one eleven-line file in about five minutes,
> a database an application logs into with no password, a database that
> survives its claim and its cluster and is adopted back in about a minute, a
> rebuild from parked with zero manual steps — **and three of the four
> pre-build decisions had to change before that was true.** The entry records
> the changes as plainly as the results.
>
> **Carried forward, not closed by this entry:**
>
> - The shared-grant hazard (surprise 14): decided in ADR-0016 §2, **not yet
>   built**; pre-registered as C-24 for M3.
> - The upstream provider bug that costs one manual command per new database
>   (surprise 6); pre-registered as C-25 for the M4 dependency-bump class.
> - The external-account IAM question — the analyzer says the nested group
>   grants access, the runtime refuses it, ~19 hours after the membership was
>   added. **UNRESOLVED**, which is why C-06's identity check rests on RBAC
>   impersonation plus a real owner token rather than a real non-owner login.
> - C-08 (stretch) was not attempted: the org-level grant it needs is an
>   undecided governance question.
> - ADR-0015 §6 asked for both rebuild modes at M2. The parked rebuild ran;
>   **the rebuild after a deliberate delete (fresh creation) did not.** The
>   `EXPECT_FRESH` path in `cycle.sh` is therefore still unexercised.
> - Housekeeping: codify or disable the three APIs enabled by hand
>   (`cloudidentity`, `policytroubleshooter`, `cloudasset`); remove the
>   creator's direct membership of `gke-security-groups@` (Google requires
>   groups only); a Tailscale route for the Private Services Access range —
>   the private address range Cloud SQL lives on (manual, on the jump box);
>   the human database path and the stale break-glass password (ADR-0016 §4).
>
> **Cost note:** the cluster and the Cloud SQL instance ran ~19 hours
> unattended overnight 2026-09-16→17 because the session's cloud credentials
> expired before teardown (surprise 15). At the end of 2026-09-17 the cluster
> was destroyed and the instance parked.
>
> Everything below this banner down to "Built" is the **pre-registered**
> test-readiness walk (2026-09-02) and the 2026-09-14 decisions note. It is
> left exactly as written, predictions included — it is the thing the build is
> compared against, and correcting it after the fact would destroy the only
> property that makes the comparison worth reading.

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

**Decided 2026-09-14, twelve days after the walk, before any build
command.** The four ADRs are 0012 through 0015, in the order above:

1. **ADR-0012** — a System is one deployable unit with its own namespace;
   the team is `spec.owner.team`, resolved to `<team>@<org-domain>` by
   convention. C-06 is a one-line edit; the predicted "re-created" entry is
   the registry IAM member. The GKE authentication hop goes to
   `gke-security-groups` once, in layer 0 (the smaller team-group IAM
   decision, settled the layer-0 way).
2. **ADR-0013** — the application authenticates as its own Google service
   account through Cloud SQL IAM database authentication; no password is
   handed to it. The bootstrap `postgres` password is platform-generated,
   team-held, and used only by the Composition's `GRANT` job. "Usable"
   is the pod serving a request from its own table with no Secret mounted.
   Private IP over PSA is assumed (the smaller network-path decision,
   settled the 1-network way).
3. **ADR-0014** — the XRD schema denies first (`enum`, closed fields, CEL
   with developer-facing messages); Kyverno is installed validate-only for
   the raw-managed-resource reality gate and per-System budgets. Messages
   are recorded from CI, the API server, and Argo CD.
4. **ADR-0015** — instances, databases and registry repositories are
   durable (no `Delete` policy, both deletion-protection flags), adopted
   on rebuild by deterministic external names, and parked with
   `activationPolicy: NEVER` by an explicit `cycle.sh park` that is not part
   of `down`. C-02's meaning changes from M2 on and the results file gains
   an adopted-vs-recreated check.

The remaining small decision — how the `svc-hello` image gets into the
registry — is settled the way the walk recommended: pushed by hand once for
M2, recorded as a manual step, with a CI identity deferred to the M4 change
class that needs it.

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

Everything here was authored on 2026-09-16 and, except where a PR is still
open, merged the same day. Times are UTC.

**`platform-factory` (this repo, the design seed).** PR #5 — the
test-readiness walk above. PR #6 — ADR-0012..0015, stacked on #5. Both still
**open**: the build ran against the branches, which is the same practice the
earlier layers used and is recorded rather than tidied.

**`platform-bootstrap`.** PR #11, **open**, applied from the branch before
merge. One bundled change per layer — layers 0, 1 and 2 as the walk
pre-declared, plus two the walk's own table does not list: a layer-3 change
and the `cycle.sh` work. (The fourth thing the walk pre-declared, the
org-level apply for C-08, never happened, because C-08 was not attempted.)
Layer 0 gains the provider identity (Google service account, Workload
Identity bindings, roles, the two APIs, and the umbrella-group grant);
layer 1 gains Private Services Access; layer 2 gains
`authenticator_groups_config`; layer 3 gains Argo CD health checks for the
two new XR kinds. `scripts/cycle.sh` gains ADR-0015's `park` command and an
adoption check on `up`. The PR also merges in the PR #10 branch, so the
already-destroyed `xpkg.upbound.io` remote is not recreated. Follow-up
commit `88129e7` forces UTC on the adoption check's timestamps (surprise 11)
and records the bring-up.

**`platform-config`.** PR #2 (merged 16:31:17) is the M2 surface:
`DeploymentRuntimeConfig`s pinning each provider's service account, the
`ClusterProviderConfig`, an `EnvironmentConfig`, three Crossplane functions,
the aggregated `ClusterRole`, the `System` XRD + Composition, the `Database`
XRD + Composition, a vendored Kyverno chart 3.9.1 with two policies, and five
new Argo Applications. Then **five fix PRs, every one of them found by the
live run and not by review**:

- PR #3 — the namespace ordering gate (merged ~16:58).
- PR #4 — Kyverno concrete provider groups and spec-level `Enforce` (17:01:24).
- PR #5 — `ServerSideDiff` on both Kyverno Applications (~17:07).
- PR #6 — team-bearing IAM members carry the team in their names (17:18:30).
- PR #7 — Kyverno zero drift: explicit defaulted rule fields, and two CRD
  label/annotation pointers ignored (~17:24).

**`systems`.** PR #1, the first tenant `svc-hello`/`payments` (16:31:20).
PR #2, C-05's second tenant `svc-ledger`/`checkout` (17:02:13). PR #3,
C-06's move of `svc-hello` from `payments` to `checkout` (17:14:26).

**`svc-hello`.** PR #1, a Go application, Dockerfile, `k8s/` manifests and a
`Database` claim (16:31:24). PR #2, the Dockerfile rebuilt as a native
builder stage that cross-compiles, plus the `docs/c07-denials` fixtures
(~17:00).

**`svc-ledger`.** A new repo, created 2026-09-16, running a placeholder
unprivileged nginx pulled through the `docker-hub` remote — enough for the
second tenant's Application to have something to sync.

**Google Workspace (manual prerequisite, done 2026-09-16).**
`gke-security-groups@`, `payments@` and `checkout@thecloudgeek.io` created
with `gcloud identity groups create`; `payments` and `checkout` nested in the
umbrella group; the project owner in `payments`; a non-owner external account
added to `payments` as the C-06 test identity. Creating the umbrella group
made its creator a direct OWNER/MEMBER, which is precisely what Google's
groups-only rule for `gke-security-groups` forbids — not yet cleaned up.

### How it was authored, and what the gates caught

One multi-agent workflow produced the whole surface above: 4 research spikes,
9 authoring streams, 2 adversarial reviewers per stream, 9 fixers and 1
integration pass — **41 agents, 0 errors, ~72 minutes, ~7.8M subagent
tokens.** The reviewers raised **84 findings, 13 of them blockers**, and all
84 were fixed before anything touched the cloud.

The uncomfortable half is the other column. The live run then found **8
further defects**, among surprises 3–13 below (the live record counts them
without naming which eight, so no mapping is asserted here). Two of them are
worse than "review missed something". One is the Kyverno kind selector: the reviewers
had recorded the wildcard provider group as **VERIFIED by reading Kyverno's
source code** ("resolved through discovery to concrete GVRs"), and a live
probe showed it is written verbatim into the webhook and matches nothing — so
the reality gate was open, and a raw `DatabaseInstance` applied by hand was
admitted and created a real Cloud SQL instance. The other is the container
image: the authoring agent **reported that `docker build` succeeded**, and it
had not been run in a form that could work.

ADR-0006's reasoning — an agent's blast radius is everything it can do, so
agent-produced instructions get a heavier gate — is why this surface got two
adversarial reviewers per stream. (The analogy is the reasoning, not the ADR:
ADR-0006's actual gate is CODEOWNERS on `platform-knowledge` plus a pinned
release, and nothing here exercised it.) What this run adds is
where such a gate's ceiling is: both escapes were *claims about reality*
— what a webhook matches, what a build produces — and neither reading the
source nor an agent's own report settled them; only execution did. That
argues for keeping the gate and for making execution part of it, not that the
gate is sufficient.

## Data

Collected live on 2026-09-16, organised by the claim that asked for it. The
register asks for: C-05 lines of YAML per tenant and merge→usable wall-clock;
C-06 files touched and anything re-created; C-07 minutes PR→ready, denial
messages verbatim, state after claim deletion; C-08 leaked schema fields.
C-01's apply count continues from one. C-02 and C-03 appear because M2's
first rebuild and M2's Compositions are where their remaining halves live.

Where a test was not run, it says so in place rather than being left out.

### C-05 — One YAML per tenant

**Lines of YAML per tenant: 11 non-comment lines** (91 with the explanatory
header). The PR itself changed **1 file** — a rename from `docs/` into
`tenants/`.

| Time (UTC) | Event |
|---|---|
| 17:02:13 | `systems` PR #2 merged |
| 17:05:10 | `System` object created — Argo CD's ~3-minute repo poll |
| 17:06:21 | `System` Ready — **merge→Ready 4m08s** |
| ~17:07 | Placeholder `Deployment` Running in namespace `svc-ledger` under AppProject `svc-ledger` (pod age 17s when checked) — **merge→usable ≈ 5 minutes** |

What appeared from that one file: the namespace; a `ResourceQuota` (pods
1/20, `requests.cpu` 20m/2, …); two `RoleBinding`s (admin, crossplane-edit);
the `svc-ledger` Kubernetes service account; an `AppProject` and an
`Application` (Synced/Healthy); a Google service account and its Workload
Identity binding; four project IAM members; an Artifact Registry repository
and its writer member.

The first tenant, for comparison: created 16:49:08, Ready 16:55:04
(**5m56s**) — and it absorbed both the CRD-not-yet-installed race and the
namespace ordering lottery (surprise 3). The second tenant ran through the
ordering gate that came out of that and showed no compose error at all.

**Against the walk's prediction.** The walk's verdict was "runnable at M2.
One apply (the floor); everything else is PR-shaped." Right on both counts
for C-05: the tenant surface arrived by PR, and the identity floor it needed
was a single layer-0 apply. What the walk did not predict is that the first
tenant would only compose by luck: it died on a different namespaced object
every reconcile until the `Namespace` happened to go first (Ready 16:55:04),
and the ordering gate that made it deterministic merged after that
(`platform-config` PR #3, ~16:58) — in time for the second tenant, not the
first.

### C-06 — Ownership moves without re-plumbing

**Files touched: 1** — `systems/tenants/svc-hello.yaml`, 1 insertion, 1
deletion (`team: payments` → `team: checkout`). Merged 17:14:26.

**In-cluster carriers converged by 17:15:41** (~75 seconds after merge; Argo's
poll happened to be quick): the namespace `team` label, both `RoleBinding`
subjects (the same objects — `creationTimestamp` unchanged at 16:53:06, so
updated in place), the `AppProject` role groups, the `platform-system`
ConfigMap's team/group keys, and the registry `labels.team`. All `checkout`.

**RBAC, by impersonation** (which tests RBAC only, not the IdP): before the
move, `payments` could list pods in `svc-hello` and `checkout` could not;
after, the reverse.

**Cloud IAM did not move.** The three team-bearing IAM members — registry
writer, `roles/cloudsql.client`, `roles/cloudsql.instanceUser` — kept their
object names, so upjet — the layer that wraps the Terraform GCP provider into
this Crossplane provider, which is why it inherits Terraform's
replace-don't-update semantics — was asked to change `member` in place and
refused:

```
async update failed: refuse to update the external resource because the
following update requires replacing it
```

External names still said `payments`; the registry still granted writer to
`payments` only. **And the System reported `Ready=True` throughout** — each
member's `Ready` condition stayed True from its original creation and only
`Synced` went False — so the failure was invisible from the XR and from Argo
CD (surprise 7).

**Re-created: 0. Stuck: 3.**

The fix (`platform-config` PR #6) puts the team in those three objects' names
and composition-resource-names, so a move composes new members and
garbage-collects the old. After it synced, the three `-checkout` members
reported `Ready=True`. The three old ones have been **stuck DELETING since
17:23:33**:

```
delete failed … Create IAM Members group:checkout@… for project ""
```

— the refused in-place update had already rewritten their spec, so the delete
path now runs with an empty project. Cloud still grants `payments` the two
Cloud SQL roles and registry writer. As of the end of 2026-09-16 this needed
manual cleanup and then one clean re-run of the move under the fixed
Composition. **Both happened on 2026-09-17 — see "Clean re-run" below, which
also records what the stuck objects did when nobody was watching.**

**Against the predictions, of which there were two.** ADR-0012's consequences
predicted files touched one and **one** re-created resource, the Artifact
Registry IAM member, on Terraform's IAM-member replacement semantics. The
Composition's own header, written before the run, refined that to files
touched one and **three** re-created — the registry member plus the two
project IAM members ADR-0013 §5 added. The measured result: **right about
which three, wrong about how.** Nothing was re-created; three objects were
asked to update, refused, and stayed wrong while every signal the platform
exposes said Ready.

**The identity half of the test is not settled either.** `kubectl auth
whoami` with the project owner's real token shows Groups
[`gke-security-groups@`, `payments@`, `checkout@`, `system:authenticated`], so
GKE resolves nested Google Groups from a real token [C, 2026-09-16]. But a
project owner cannot be the test subject — IAM grants it everything
regardless of RBAC — and the non-owner external account added for the test is
refused with HTTP 403 at the DNS endpoint front door, with `gcloud container
clusters describe` reporting `Required "container.clusters.get"`, a day's
propagation later. Cloud Asset `analyze-iam-policy --expand-groups` lists that
same account as holding `container.clusters.get` through
`group:gke-security-groups@` on `roles/container.clusterViewer`; Policy
Troubleshooter answers `MEMBERSHIP_UNKNOWN_INFO_DENIED`. **UNRESOLVED:** the
analyzer says yes and the runtime says no, for an external consumer account
nested two groups deep. On 2026-09-17, roughly 19 hours after the membership
was added, the same account was still refused with HTTP 403 — so this is not
propagation delay. It stays UNRESOLVED.

**Clean re-run under the fixed Composition (2026-09-17).** The move was run
again in the other direction (`team: checkout` → `team: payments`), systems
PR #4: one file, one insertion, one deletion, merged 11:42:01.

| Time (UTC) | What happened |
|---|---|
| 11:42:01 | PR merged |
| 11:43:26 | The three `-checkout` member objects received deletion timestamps; three `-payments` objects were created |
| 11:43:56 | All three `-payments` members Ready; all three `-checkout` objects gone, no stuck deletes; the cloud policy shows `payments` on both Cloud SQL roles and on the registry writer |

About 30 seconds after the change reached the cluster, 1m55s after the merge.
Held against the prediction written into the Composition before the first
run — files touched one, re-created three — **the re-run matches it. The
first run did not (re-created zero, stuck three); the design had to change
before its own prediction could come true.**

**The shared-grant hazard, found by what the stuck objects did overnight.**
The three members stuck deleting on 2026-09-16 eventually completed their
deletes unattended. Because the refused in-place update had already rewritten
their spec to `member: checkout`, what they deleted was the *checkout* grants:
on the morning of 2026-09-17 the project carried no `roles/cloudsql.client` or
`roles/cloudsql.instanceUser` binding for `group:checkout@` and the svc-hello
registry's policy was empty — while every `-checkout` member object in both
tenant namespaces reported Ready. The mechanism is general, not an artefact of
the stuck objects: a project-level IAM binding is identified by (role,
member), `svc-hello` and `svc-ledger` were both owned by `checkout`, and each
System composes its own managed resource for the *same* cloud binding. Deleting
any one removes the grant for all; the others notice only at their next poll.
The clean re-run reproduced it on demand: from ~11:43:56 the project listed
only `payments` on the two Cloud SQL roles although `svc-ledger` was still
owned by `checkout`, and the provider's own drift correction restored
`checkout` by 11:48:52 — about five minutes, no intervention, the affected
objects Ready throughout. (An earlier watch the same morning saw the registry
writer restored 504 seconds in, consistent with a roughly ten-minute poll.)
Surprise 14; the candidate fix is in the errata.

Manual cleanup on 2026-09-17: the stale `payments` bindings left by the first
run were removed with `gcloud`. No finalizer surgery was needed in the end.

### C-07 — Guardrails replace review for databases

**(a) PR → usable database.**

The `svc-hello` PR merged at 16:31:24 while the cluster was still down, so the
honest clock starts when Argo CD applied the claim.

| Time (UTC) | Event |
|---|---|
| 16:54:06 | `Database` claim created |
| ~17:08 | Cloud SQL instance `svc-hello-main` RUNNABLE (~14 min) |
| 17:08:17 → 17:10:44 | The IAM `User` managed resource sits failing |
| 17:10:44 | **MANUAL INTERVENTION** — `gcloud sql users create` out of band |
| 17:11:22 | User adopted, Ready; GRANT Job `main-grant-b1f59c270b` starts |
| 17:11:42 | GRANT Job succeeded |
| 17:11:51 | Application pod Ready — **claim→usable 17m45s, including one manual step** |
| 17:12:41 | `Database` XR Ready |
| 17:13:28 | Proof: `POST /notes` then `GET /notes` from a probe pod returned the row (`1  2026-09-16T17:13:28Z  hello from C-07 at 17:13:28Z`); `/healthz` 200 |

The instance as built: `db-f1-micro`, private IP `10.60.0.3`, `ipv4Enabled`
false, `cloudsql.iam_authentication=on`, `deletionProtectionEnabled` true,
`activationPolicy ALWAYS`. **No Secret is mounted in the application**; login
is IAM through the Auth Proxy sidecar (`--private-ip --auto-iam-authn`),
which is ADR-0013's definition of "usable", met.

The manual step is an upstream provider bug, not a design choice. The `User`
managed resource failed to create with:

```
async create failed: recovered from panic: not a string
```

That is `crossplane-contrib/provider-upjet-gcp` issue **#1000** (open;
root-caused by a maintainer on 2026-09-14 — v3.0.0 strips `password_wo` from
the runtime schema, so *every* passwordless `sql.User` create panics). The
workaround from the issue is to create the user out of band and let Observe
adopt it. Before the user existed, the application log read:

```
FATAL: password authentication failed for user "svc-hello@platform-factory-ref.iam"
```

Whether the Composition could sidestep the bug by supplying a password was
probed, and it cannot — Cloud SQL answers:

```
HTTPError 400: Invalid request: Cloud IAM password cannot be set in the database.
```

So there is no Composition-side workaround; the fix is a provider release.

One unplanned data point on the tenant surface: the `ResourceQuota` works and
bites. A probe pod with no resource requests was rejected —

```
pods "c07-probe" is forbidden: failed quota: svc-hello: must specify limits.cpu … requests.memory
```

**(b) Denied at admission.** ADR-0014 §5 names three *surfaces* — the CLI,
the API server, and Argo CD's sync status — and Kyverno is a *mechanism*
whose denial arrives at one of them (the API server, through the webhook).
Two of the three surfaces were run; Argo CD's was not.

API server (`kubectl apply`), verbatim:

```
The Database "denied-region" is invalid: * spec.region: Unsupported value: "europe-west1": supported values: "us-central1", "us-east1"

The Database "denied-size" is invalid: * spec.size: Unsupported value: "XL": supported values: "S", "M", "L"

The Database "denied-cel" is invalid: spec: Invalid value: size L is only available to a critical-tier database. Set tier: critical if this database really is business critical, otherwise use size M.
```

The two enum denials also print:

```
* <nil>: Invalid value: null: some validation rules were not checked because the object was invalid; correct the existing errors to complete validation
```

CLI, offline (`crossplane resource validate` v2.5.0 against the XRD): the same
three messages, prefixed `[x] schema validation error …` and `[x] CEL
validation error …`, ending `Total 3 resources: 0 missing schemas, 0 success
cases, 3 failure cases`.

Kyverno reality gate — a raw `DatabaseInstance` applied by hand in the tenant
namespace, **after** the fix:

```
admission webhook "validate.kyverno.svc-fail" denied the request: resource DatabaseInstance/svc-hello/raw-observe-only was blocked due to the following policies

deny-raw-managed-resources:
  no-raw-managed-resources-in-tenant-namespaces: Cloud resources are not created by hand here. Ask for what you need with a platform.thecloudgeek.io claim — a Database, for example — …
```

**Before** the fix, the identical test was **ADMITTED and created a real Cloud
SQL instance** (surprise 4).

**Argo CD surface — a bad claim merged to `main`: RUN 2026-09-17.** svc-hello
PR #5 put the wrong-region claim into `k8s/` on `main` (merged 11:39:32),
deliberately skipping the CI gate to see what a developer meets when CI was
skipped. At 11:41:25 the `svc-hello` Application read sync **OutOfSync**, health
**Healthy**, operation **Running** (retrying), with this message:

```
one or more synchronization tasks completed unsuccessfully, reason: Database.platform.thecloudgeek.io "denied-region" is invalid: [spec.region: Unsupported value: "europe-west1": supported values: "us-central1", "us-east1", <nil>: Invalid value: null: some validation rules were not checked because the object was invalid; correct the existing errors to complete validation]. Retrying attempt #1 at 11:41AM.
```

and per resource: `Database/denied-region: SyncFailed` with the same text. The
running service and the real claim were untouched. Reverted by PR #6 (merged
11:41:40). For the wrong-region claim all three of ADR-0014 §5's surfaces are
now recorded, and the field name and the allowed values survive intact through
every one. **§5 asked for more than was run:** the oversized claim and the CEL
rule were recorded at the CLI and the API server only, and Kyverno's denial at
the API server only. Argo CD relays the API server's message verbatim, so
there is little doubt what it would show — but that is an inference, not a
recording.

**(c) Delete the claim, database survives: RUN 2026-09-17, and it held.**

| Time (UTC) | What happened |
|---|---|
| 11:33:39 | svc-hello PR #3 (removes `k8s/database.yaml`) merged |
| 11:34:02 | Claim pruned by Argo CD. Every composed object left the namespace: the instance, database and user managed resources, the GRANT Job, the admin Secret, the connection ConfigMap |
| after | Cloud SQL: `svc-hello-main` RUNNABLE, createTime `2026-09-16T16:54:08.673Z` unchanged, deletion protection still on, database `app` present, the IAM user present (it is composed with `deletionPolicy: ABANDON`). The application pod stayed 2/2 Running and kept serving — nothing it depends on had changed |
| 11:35:05 | svc-hello PR #4 (restores the file) merged |
| 11:37:34 | Claim object re-created by Argo CD |
| 11:37:42 (the recorder's first check) | Instance managed resource already Ready and Synced — adopted by external name, not created. The IAM user adopted too (`LastAsyncOperation: Success`), so the provider bug was never touched: no create was needed |
| 11:38:40 | Claim Ready — 66 seconds after it appeared, against ~14 minutes for a fresh instance on 2026-09-16 |

`gcloud sql instances list` afterwards: still exactly one instance, same
createTime. The row written on 2026-09-16T17:13:28Z read back through the
service, `/healthz` 200. ADR-0015 §2 — adoption by deterministic external name
— is confirmed [C, 2026-09-17].

Deletion protection was exercised by accident rather than by test. The raw
instance applied by hand carried both protection flags, copied from the
rendered spec; clearing them needed the object's `deletionProtection` patched
to false **and** `gcloud sql instances patch --no-deletion-protection`, after
which the provider deleted it (90s + 90s). Both locks held until deliberately
removed. That is evidence about the locks, not about C-07(c), which asks what
happens to the database when the *claim* goes away.

### C-08 — Schema survives the Composition swap (stretch)

**Not attempted.** The org-level grant it stands on — project creator plus
billing user for the provider identity — is an undecided governance question,
which is exactly the condition the walk's own verdict put on it ("runnable at
M2 only if the org-level grant is accepted… a 'no' is also a valid C-08
outcome").

Partial evidence exists and is the part the walk said was the real data point:
**the `System` schema carries no mechanism fields.** What a developer writes
is a size class, a tier, a security tier, `owner.team` and `owner.repo` —
nothing that names a Kubernetes or GCP resource. Leaked fields so far: none.
That is a review of the schema, not the Composition swap the test asks for.

### C-01 — Terraform applies after M1

The counter continues from one. Every row is a real `terraform apply` against
the live project.

| # | Date | Layer | Result | Why it could not be a PR |
|---|---|---|---|---|
| 1 | 2026-09-02 | 0-foundation | `xpkg.upbound.io` remote removed (`platform-bootstrap` PR #10) | Cleanup of a layer-0 resource; layer 0 is Terraform's by the repo's own rule |
| 2 | 2026-09-16 | 0-foundation | 14 added, 0 changed, 0 destroyed: service account `crossplane-provider-gcp`; 5 `workloadIdentityUser` bindings, one per pinned provider service account; project roles `artifactregistry.admin`, `cloudsql.admin`, `iam.serviceAccountAdmin`, `compute.viewer`, `resourcemanager.projectIamAdmin` (IAM Condition `only-cloudsql-connect-roles`); `roles/container.clusterViewer` for `group:gke-security-groups@`; APIs `sqladmin` + `servicenetworking` | The identity for the thing that applies PRs cannot itself arrive by PR — surprise 17's "the paved road can pave everything except its own on-ramp", one milestone on. API enablement and project IAM are layer 0 by the repo's rule |
| 3 | 2026-09-16 | 1-network | 2 added: `google_compute_global_address` `psa` 10.60.0.0/16 (VPC_PEERING) + `google_service_networking_connection` | PSA is reachability, and reachability persists — the layer boundary rule |
| 3b | 2026-09-16 | 1-network | 0 added, 0 changed, 0 destroyed — outputs only (`+gke_security_group`) | **Not a design crossing: an operator error.** The layer-1 plan was generated *before* layer 0 was applied, so the passthrough output read null, and Terraform omits null outputs from state (surprise 10) |

Two things are not in the table and belong in the grade.

- **Layers 2 and 3 carried their M2 changes on the normal `cycle.sh up`** —
  `authenticator_groups_config` on the cluster, and Argo CD health
  customizations for `platform.thecloudgeek.io_System` and `_Database`. Those
  layers are disposable and are applied by every rebuild anyway, so they added
  no crossing that C-02 was not already paying for. Whether they count is a
  decision the grade has to make out loud rather than assume. (Argo CD 3.4.6
  already ships a built-in `*.upbound.io` health check, so none was added for
  managed resources.)
- **Three APIs were enabled by hand, outside Terraform**: `cloudidentity`
  (needed as a quota project to manage groups — the default quota project
  resolves to a project this identity cannot use), plus `policytroubleshooter`
  and `cloudasset` as diagnostics for the external-account IAM question.
  These should be codified in layer 0 or disabled; **NOT YET DONE**, and they
  are the honest counterweight to the apply count looking small.

**Against the walk's prediction.** The walk pre-declared four applies — layer
0, layer 1, layer 2, and an org-level one for C-08. Measured so far: layer 0
once (as predicted), layer 1 twice (the second an operator error, not a
design crossing), layer 2 folded into the rebuild, and the org-level apply
never attempted because C-08 was not. The shape of the prediction held; the
count is not final until the remaining M2 work is done.

### C-02 — the M2 bring-up (cycle 4 `up`), explicitly not a clean cycle

| Phase | Time |
|---|---|
| `2-cluster` apply | 774s |
| `3-argocd` apply | 89s |
| verify (10/10 Applications Synced/Healthy; deadline 2400s) | 2289s |
| **TOTAL** | **3157s (52m37s)** |

The durable-resource check that ADR-0015 added to `cycle.sh` exited 1:
**adopted 0, recreated 0, new 1** (`cloudsql/svc-hello-main@2026-09-16T16:54:08.673Z`),
**unknown 2** (`registry/svc-hello`, `registry/svc-ledger` — the timestamp bug
in surprise 11, since fixed).

**This is a bring-up, not a C-02 measurement**, and the interventions are why:

1. The planned one-time image push (the walk's own accepted manual step).
2. One out-of-band `gcloud sql users create` — the provider bug above.
3. Five fix PRs merged while `verify` was still waiting.
4. A hard refresh of two Applications.

What the run does establish:

- **Sync waves behaved**: crossplane (0) → providers (1; went Degraded once
  and the retry backstop recovered it, same as M1) → crossplane-platform +
  kyverno (2) → compositions + kyverno-policies (3) → systems (4) → the tenant
  Applications composed by the `System`.
- **The provider identity path works on the first reconcile** [C]: the
  per-System Google service account and all four `ProjectIAMMember`s reported
  `Synced=True` on the first try — Google service account + Workload Identity
  + a `DeploymentRuntimeConfig`-pinned Kubernetes service account +
  `ClusterProviderConfig` `InjectedIdentity`, end to end.
- `verify` passed with **111 seconds to spare** against its 2400s deadline,
  and only after the two Kyverno drift fixes (surprise 5); before them,
  `cycle.sh` would have failed the rebuild on Applications that were
  `OutOfSync` with nothing actually different.

### C-02 — cycle 5, the parked rebuild (2026-09-17): the clean cycle

The bring-up above is not a measurement. This is: `down`, `park`, `up`, with
**zero manual interventions**, run after every test had finished.

| Phase | Seconds | Note |
|---|---|---|
| `down` — 3-argocd destroy | 46 | |
| `down` — 2-cluster destroy | 594 | `down` total 642s (10m42s). `svc-hello-main` untouched, as ADR-0015 §3 requires |
| `park` | 55 | found the instance by its `system` label, set `activationPolicy: NEVER`, "stopped in 54s". Verified `STOPPED` / `NEVER`, createTime unchanged |
| `up` — 2-cluster apply | 801 | |
| `up` — 3-argocd apply | 89 | |
| `up` — verify | 1235 | 10/10 Applications Synced/Healthy |
| `up` — durable | 1 | exit 0: `adopted: 3 [cloudsql/svc-hello-main@2026-09-16T16:54:08Z registry/svc-hello@2026-09-16T16:53:10Z registry/svc-ledger@2026-09-16T17:05:14Z], recreated: 0` |
| `up` total | **2131 (35m31s)** | |

What happened inside the verify wait, from a read-only recorder:

| Time (UTC) | |
|---|---|
| 12:22:50 | Tenant Applications appear on the new cluster |
| 12:23:12 | Instance goes `STOPPED/NEVER` → `MAINTENANCE/ALWAYS` — **Crossplane restored the activation policy by itself, about 20 seconds after the claim synced. No unpark command exists or was needed** |
| 12:24:40 | Both Systems Ready; `svc-ledger` Healthy |
| 12:31:55 | Instance `RUNNABLE` — 8m43s to restart from parked |
| 12:34:55 | Application container ready |
| 12:36:24 | Database claim Ready; verify passes at 12:36:32 |

No provider bug on this path — the IAM user already existed and was adopted —
and no image push, because the registry and the image are durable. Afterwards
the row written on 2026-09-16T17:13:28Z read back through the service,
`/healthz` 200, one Cloud SQL instance, createTime unchanged: **the same data
has now survived a claim deletion, a cluster teardown, and a park.**

Against M1, as ADR-0015 predicted: M1's clean `up`s were about seventeen
minutes with nothing durable behind them. This one is 35m31s, and about
twenty of those minutes (1235s) are the verify wait — dominated by a database
restarting and a tenant that is not Healthy until its database is. The
rebuild is no longer "from empty"; it is from persisted identity,
reachability and data, and the number says so. The adoption row's timestamp
fix (surprise 11) was exercised here for the first time and held.

### C-03 — the hands-on half, deferred here from M1

Every kind the two Compositions use reconciled at **namespaced** scope on
provider-upjet-gcp v3.0.0: `RegistryRepository`,
`RegistryRepositoryIAMMember`, `ServiceAccount`, `ServiceAccountIAMMember`,
`ProjectIAMMember`, `DatabaseInstance`, `Database` — **except `sql.User` of
an IAM type, whose create path panics** (issue #1000 above). The kind exists
and is served; the create path for passwordless users is broken at this
version. That distinction is the whole of what C-03's grade has to decide.

## Claims graded

Graded on 2026-09-17, on the evidence above. A provisional reading stood in
this section between the two test days and said, explicitly, that it was not a
set of grades; these replace it. ADJUSTED grades link the superseding ADR that
ADR-0008 requires.

### C-01 — Terraform ends at layer 0 → **ADJUSTED** ([ADR-0016 §7](../adr/0016-what-the-m2-build-changed.md))

The test counts `terraform apply` runs after M1, target zero. The count is
**four applies on the persistent layers**: the 2026-09-02 cleanup, a bundled layer-0 apply (the provider's
cloud identity, its roles, two APIs, the group grant), a layer-1 apply
(Private Services Access), and an outputs-only re-apply that was an operator
error — a layer planned before the one beneath it was applied (surprise 10).
Three APIs were also enabled by hand. **And two more Terraform changes are
counted here rather than hidden:** layer 2's Google Groups setting and layer
3's health checks rode the ordinary `cycle.sh up` that C-02 already runs
every session. They added no apply *run*, which is why the count above is
four; they are still Terraform crossings, which makes six in all. Zero was
the wrong target in a predictable way: M1's surprise 17 said each new
platform capability needs one identity-or-reachability crossing, and the
readiness walk pre-declared three of these before the build (layers 0, 1 and
2) — the only reason they arrived bundled. The cleanup, the operator-error
re-apply and the layer-3 change were not pre-declared. What the
claim protects did hold: **no tenant, service, database, policy or ownership
change in M2 needed Terraform** — onboarding, a team move, and creating,
deleting and re-adopting a database were all pull requests. ADR-0016 §7
restates the boundary to match.

### C-03 — provider-upjet-gcp kind coverage → **HELD for the kinds M2 composes, with one recorded gap**

The test asks for a hands-on create of each kind and "gaps and workarounds."
At v3.0.0, namespaced scope: `RegistryRepository`,
`RegistryRepositoryIAMMember`, `ServiceAccount`, `ServiceAccountIAMMember`,
`ProjectIAMMember`, `DatabaseInstance` and `Database` all created and
reconciled. **Gap:** `sql.User` of an IAM type cannot be *created* — the
create path panics (upstream issue #1000, open); it can be observed and
adopted. **Workaround:** create the user out of band, one command, and let
the provider adopt it (38 seconds, measured). Not hands-on tested because no
M2 Composition uses them: GCS bucket and Cloud DNS records — carried to M3,
whose edge and DNS work needs them. This is graded HELD rather than ADJUSTED
because nothing in the design changed; a pinned version has a bug with a
known workaround and a known fix path, which is exactly the data the claim
asked for. It is the judgment call in this list, and is named as one.

### C-05 — One YAML per tenant → **HELD**

The second tenant was one file of eleven non-comment lines. Merge to System
Ready 4m08s, of which about three minutes is Argo CD's repository poll;
merge to a workload Running under the tenant's own AppProject about five
minutes. The full surface appeared: namespace, quota, two RoleBindings,
service account, AppProject and Application, Google service account and its
Workload Identity binding, four project IAM members, registry and writer.
Two caveats, stated rather than buried. The first tenant found the ordering
defect (surprise 3) that the second tenant, the actual test, then ran clean
through — so the claim held on a Composition one fix *newer* than the one
first merged. And the provisional reading said an honest HELD wanted a tenant
onboarded with no fix PRs in flight; that did not happen — three more fixes
merged to `platform-config` later the same session — so the five minutes is
a build-session number, not a steady-state one. The claim's test, as
registered, was run and passed; a steady-state number is cheap to collect the
next time a tenant is added.

### C-06 — Ownership moves without re-plumbing → **ADJUSTED** ([ADR-0016 §1–2](../adr/0016-what-the-m2-build-changed.md))

Files touched: one, one line, both runs. **First run: re-created zero, stuck
three** — the in-cluster half moved in about 75 seconds and the cloud half did
not move at all, while every signal the platform exposes said Ready. After the
design change (team in the IAM members' names): **one file, three re-created,
1m55s from merge, no stuck objects** — which is the prediction written into
the Composition before the first run, true only on the second. Two things
keep this from HELD beyond the design change itself: the shared-grant hazard
(a sibling System loses its team's Cloud SQL grant for about five minutes;
decided in ADR-0016 §2, not yet built, pre-registered as C-24), and the
identity check. RBAC flipped exactly as designed under impersonation, and a
real owner token shows GKE resolving the nested groups [C] — but the real
non-owner login the walk asked for could not be made, because that account is
refused for a reason still UNRESOLVED.

### C-07 — Guardrails replace review for databases → **ADJUSTED** ([ADR-0016 §3–4](../adr/0016-what-the-m2-build-changed.md))

**(a)** Claim to usable in 17m45s, about fourteen minutes of it Cloud SQL
creating the instance, with **one manual command** forced by the provider bug.
The application logs in as its own Google identity, no password, no Secret
mounted, and wrote and read a row. **(b)** The schema denies first and the
message names the field and the allowed values: recorded at the offline CLI
and the API server for all three bad claims, and through Argo CD for the
wrong-region one (the rest of ADR-0014 §5's matrix was not run). But the reality gate
**admitted the exact thing it exists to deny** on its first live test, and a
real Cloud SQL instance was created by hand before the gate was fixed the same
hour; it has held since, and Crossplane's own resources pass it. **(c)** Held
cleanly: the claim was deleted, the instance, its database, its user and its
data stayed and the application kept serving; the claim came back and adopted
the instance in 66 seconds. The design changed in one place (the gate
enumerates provider groups) and depends on a provider fix in another, hence
ADJUSTED rather than HELD.

### C-08 — Schema survives the Composition swap (stretch) → **UNTESTED, not attempted**

The test needs an alternate Composition that mints a GCP project, which needs
the provider identity to hold project-creation power at the organization —
the governance question the readiness walk said this claim would come down
to. It was not decided, so the test was not run. Partial evidence only: the
`System` schema carries intent fields only — team, repo, tier, security tier,
a size class — and nothing that names a mechanism. Re-scheduled to M3 pending that decision; a recorded "no" would
also be a legitimate outcome.

### C-02 — not regraded; redefined, with its first M2 number

C-02 was graded HELD at M1. ADR-0015 changed what it measures from M2 on, and
cycle 5 is the first clean cycle under the new meaning: zero manual
interventions, 35m31s up, three durable resources adopted and none
re-created. The grade stands; the M1 and M2 numbers are not comparable, and
the data section says why.

### What the grades add up to

Three ADJUSTED, two HELD, one not attempted, none WRONG — and the ADJUSTED
ones are the interesting result. Two of the three (C-06, C-07) trace to
decisions made on 2026-09-14, two days before the build, from primary
sources; the third (C-01) to the design's founding thesis. A careful
pre-build design was wrong in four places that only a live run could find,
and one of those (the gate) had been *verified* by independent reviewers
reading source code. That is the argument for ADR-0008's method, made by the
method's own output.

## Research verified

- **[C] Crossplane XRD schemas accept `x-kubernetes-validations` CEL rules
  with a `message`, and `crossplane resource validate` evaluates them
  offline** (Crossplane CLI reference, v2.4/v2.5, read 2026-09-14). The
  basis for ADR-0014's gate order.
- **[C] Crossplane adopts an existing external resource by
  `crossplane.io/external-name`;** the `Create` policy applies only "if the
  external resource doesn't exist" (managed-resources doc and
  import-existing-resources guide, v2.3/v2.4, 2026-09-14). The basis for
  ADR-0015's rebuild-adopts rule.
- **[C] Cloud SQL IAM database authentication** (Google docs, 2026-09-14):
  principals are user accounts, service accounts, or groups; login is by
  temporary token over required SSL; `roles/cloudsql.instanceUser` to log
  in plus `roles/cloudsql.client` for the Auth Proxy or connectors; the
  instance flag `cloudsql.iam_authentication=on`; the Postgres username for
  a service account drops the `.gserviceaccount.com` suffix. Workforce
  Identity Federation has its own login path; *workload* identity
  federation principals are not listed, so ADR-0013 assumes a Google
  service account per System [I].
- **[C] provider-gcp-sql v3.0.0 `User.spec.forProvider.type`** accepts
  `BUILT_IN`, `CLOUD_IAM_USER`, `CLOUD_IAM_SERVICE_ACCOUNT`,
  `CLOUD_IAM_GROUP` and the group-member variants; `DatabaseInstance`
  carries `settings.activationPolicy` (`ALWAYS` / `NEVER` / `ON_DEMAND`),
  `deletionProtection`, and `settings.deletionProtectionEnabled` (CRDs at
  tag v3.0.0, 2026-09-14).
- **[C] A stopped Cloud SQL instance (`activationPolicy: NEVER`) "suspends
  instance charges"; storage and IP address charges continue** (Cloud SQL
  start/stop doc, 2026-09-14). The basis for ADR-0015's park-not-delete.
- **[C] IAM Conditions can bound a project-IAM-admin grant to specific
  roles** via `api.getAttribute('iam.googleapis.com/modifiedGrantsByRole',
  []).hasOnly([...])`; the attribute lists only the roles a request
  modifies and is empty otherwise (IAM conditions attribute reference,
  2026-09-14). The basis for ADR-0013's bounded provider grant.
- **[C] The root Application carries no `resources-finalizer`**
  (`3-argocd/charts/root-app`, read 2026-09-14), so `cycle.sh down` orphans
  child Applications rather than cascading deletes — which is why nothing
  Crossplane created has ever been deleted by a teardown, and why that is
  luck rather than policy until ADR-0015's management policies land.
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

### Verified live by the run (2026-09-16)

These are hands-on results, not document checks — each says what verified it.

- **[C] The provider identity path works end to end** (2026-09-16, cycle 4
  `up`): a Google service account, a `roles/iam.workloadIdentityUser` binding
  for a `DeploymentRuntimeConfig`-pinned Kubernetes service account, and a
  `ClusterProviderConfig` with `credentials.source: InjectedIdentity`.
  Verified by the per-System Google service account and all four
  `ProjectIAMMember`s reporting `Synced=True` on the first reconcile.
  Namespaced managed resources default to `ClusterProviderConfig/default`
  — read off provider-upjet-gcp v3.0.0's CRDs on 2026-09-16 (the citation is
  in the header of
  `platform-config/crossplane/compositions/system/composition.yaml`), and
  borne out by those composed resources reconciling with
  `spec.providerConfigRef` omitted.
- **[C] Crossplane 2.3.5 composes native Kubernetes objects directly**
  (2026-09-16) — `Namespace`, `ResourceQuota`, `RoleBinding`,
  `ServiceAccount`, `ConfigMap`, `Secret`, `Job`, Argo CD `Application` and
  `AppProject` — given the aggregated `ClusterRole` carrying `bind` on the
  bound roles. Verified in two pieces, because the two Compositions emit
  different kinds: the two tenants materialised the `System` Composition's set
  (`Namespace`, `ResourceQuota`, `RoleBinding`, `ServiceAccount`, `ConfigMap`,
  `AppProject`, `Application`), and the `Secret` and the GRANT `Job` were
  verified once, by the `svc-hello` `Database` claim — `svc-ledger` is a
  placeholder with no database. This does
  *not* settle what Crossplane composes by *default*; see below.
- **[C] Argo CD 3.4.6 per-kind health keys for
  `platform.thecloudgeek.io_System` and `_Database` work** (2026-09-16) — the
  layer-3 health customisations were installed on the rebuild and the two
  per-kind keys work. (What was *not* recorded is a trace of an Application
  going Healthy only after its XR went Ready, so that is not claimed here.)
  An XRD-schema denial and a Kyverno denial both reach the
  developer with the messages recorded under C-07(b) — from the API server and
  the webhook. The Argo CD sync-status surface was run on 2026-09-17 and carries the same message (see Data, C-07(b)).
- **[C] Kyverno 1.19.x / chart 3.9.1: a kind selector's *group* wildcard is
  not expanded** (2026-09-16), verified by live probe — a wildcard group is
  written verbatim into the webhook's `apiGroups`, which the API server does
  not glob, so the rule matches nothing; a concrete group with wildcard
  version and kind (`g/*/*`) does expand. Rule-level `failureAction: Enforce`
  is honoured once the webhook actually matches. See surprise 4.
- **[C] Cloud SQL IAM service-account login through the Auth Proxy with
  `--auto-iam-authn` works with zero passwords handed to the application**
  (2026-09-16), verified by the pod serving a row from its own table with no
  Secret mounted. The converse is also confirmed: a password on an IAM user is
  rejected by the API (`Cloud IAM password cannot be set in the database`), so
  ADR-0013's no-password path is not merely preferred, it is the only one.
- **[C] GKE Google Groups RBAC resolves *nested* groups from a real user
  token** (2026-09-16), verified by `kubectl auth whoami` with the owner's
  real token listing the umbrella group and both team groups.
- **[C] Cloud Asset `analyze-iam-policy --expand-groups` reports
  nested-group access** (2026-09-16) — and the runtime disagreed for an
  external consumer account nested two groups deep. Recorded as confirmed for
  the analyzer's behaviour and **open** for the contradiction; see C-06's data.

### Verified live on the second day (2026-09-17)

- **[C] Adoption by deterministic external name** (ADR-0015 §2): a re-created
  claim adopted the surviving Cloud SQL instance, its database and its IAM
  user — instance createTime unchanged, one instance listed, data intact — and
  a rebuilt cluster adopted the instance and both registries
  (`adopted: 3, recreated: 0`).
- **[C] Drift correction from `activationPolicy: NEVER` back to `ALWAYS`**
  (ADR-0015 §4, [I] at decision time): the provider restored it about twenty
  seconds after the claim synced on the rebuilt cluster. No unpark command.
- **[C] `deletionPolicy: ABANDON` on the `User`** leaves the IAM database user
  in place when the claim is deleted, which is also why re-adding a claim does
  not meet the create bug.
- **[C] Argo CD 3.4.6 surfaces an XRD schema denial verbatim** in the
  Application's `operationState.message` and per-resource `SyncFailed`, with
  the Application `OutOfSync` but `Healthy` and the sync retrying.
- **[C] Two managed resources for one unconditioned (role, member) project
  binding remove each other's grant** — observed twice on 2026-09-17.
- **[C] Google documents a condition on `roles/cloudsql.client` scoped by
  `resource.name`** "to grant permission to just the named instance" (Cloud
  SQL IAM Conditions doc, read 2026-09-17). **[I]** that giving each System's
  grant its own condition makes them separate cloud objects the provider
  manages cleanly — the basis of ADR-0016 §2, and exactly what C-24 tests.
- **[C] The provider's drift correction restores a deleted IAM binding
  unaided** within its poll: about five minutes on one run, 504 seconds on
  another.
- **Still unverified:** why an external consumer account nested two groups
  deep is refused (HTTP 403) when Cloud Asset's analyzer lists it as holding
  `container.clusters.get`; whether Cloud SQL evaluates the Auth Proxy's calls
  against a `resource.name` condition (ADR-0016 §2's precondition).

### What the walk's "Unverified at walk time" list looks like now

- **Whether any Google Groups exist in the Workspace — settled.** The blocker
  was the Cloud Identity API on a usable quota project; it was enabled by hand
  with `--billing-project` and the three groups were created on 2026-09-16
  (see Built). The answer was "none existed"; they exist now. One residue: the
  creator became a direct member of `gke-security-groups@`, which Google's
  groups-only rule forbids, and it is not yet cleaned up.
- **Whether `InjectedIdentity` works with the direct federated-principal
  form — still unverified.** The build took the documented Google
  service-account form and it works (above), so the shortcut that would have
  shrunk the floor apply to IAM bindings alone was never tested. It stays on
  the list.
- **Crossplane's default composable-kinds list — still unverified.** M2
  granted the aggregated `ClusterRole` up front, so what Crossplane composes
  *without* it remains untested. What is now known is that the aggregated role
  with `bind` is sufficient for the full tenant surface.
- **Project-creation quota on the billing account — still unverified.** It is
  C-08's prerequisite and C-08 was not attempted.

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

*Surprises 3 onward were met on 2026-09-16, in the order they are numbered.*

3. **A Composition that emits a namespace and sixteen things inside it is
   a lottery, and Crossplane stops at the first loser.** Composed resources
   are applied in map order and the pipeline halts at the first apply error.
   A new `System` therefore died on a *different* namespaced object every
   reconcile — `namespaces svc-hello not found` on `gsa`, then the same on
   `registry-writer` — until the `Namespace` happened to be applied first and
   the whole set went through. Cost: the first tenant absorbed it (created
   16:49:08, Ready 16:55:04), and nothing about the failure named the real
   cause. Fix: emit only the `Namespace` until it is observed, then
   everything — two deterministic passes, the same gating the `Database`
   Composition already uses for its GRANT Job. `crossplane render` shows 2
   objects before and 17 after. **The general shape:** a Composition that
   creates a container *and* its contents carries an ordering requirement
   that nothing in the XR, the XRD or Crossplane's own model expresses — it
   has to be written by hand, and it is invisible until a fresh namespace is
   involved, which is exactly the case a tenant Composition exists for.

4. **The reality gate was open, and the code review had marked it
   verified.** ADR-0014's raw-managed-resource gate names its kinds with a
   wildcard *group* (`*.gcp.m.upbound.io`). Kyverno writes that string
   verbatim into the webhook's `apiGroups`, and the API server does not glob
   apiGroups — so the rule matched nothing at all. Cost: a raw
   `DatabaseInstance` applied by hand in a tenant namespace was **admitted,
   and created a real Cloud SQL instance** (deleted ~15 minutes later). Fix:
   enumerate the five installed provider groups in both families; the webhook
   then lists ten concrete groups and the identical test is denied with the
   message recorded under C-07(b). Crossplane's own composed resources still
   pass, confirmed by the second tenant composing after the fix. **The part
   worth carrying:** the adversarial reviewers recorded this selector as
   VERIFIED *by reading Kyverno's source* ("resolved through discovery to
   concrete GVRs"). They had read the right code; the question was what the
   API server does with what that code emits, and only a live probe answered
   it. M1's evidence hierarchy (validate < plan < apply) applies to review
   too: **source read is not behaviour observed.**

5. **Two Applications sat OutOfSync with nothing different, and the rebuild
   gate is what made that expensive.** Kyverno defaults the deprecated
   spec-level `validationFailureAction` to `Audit` and displays it in
   `kubectl get clusterpolicy` next to rules that say `Enforce`; it defaults
   `skipBackgroundRequests`, `allowExistingViolations` and `apiCall.method`
   *inside* `rules[]`, where even Argo CD's `ServerSideDiff` cannot attribute
   the defaults to the API server; and its chart renders empty
   `labels`/`annotations` maps on CRDs. Cost: not cosmetic — `cycle.sh` gates
   a rebuild on every Application being Synced, so this would have failed the
   rebuild rather than annoying someone. Fix: state the defaulted fields
   explicitly, ignore two CRD pointers, and turn on `ServerSideDiff`
   (`platform-config` PRs #5 and #7); verify then passed with 111 seconds to
   spare. **General form:** a defaulting admission controller and a GitOps
   differ disagree by construction, and the disagreement is silent — the
   Application is red with an empty diff.

6. **The credential path ADR-0013 chose rests on one provider kind, and that
   kind's create path panics.** `provider-upjet-gcp` v3.0.0 cannot create a
   passwordless `sql.User` (issue #1000: v3.0.0 strips `password_wo` from the
   runtime schema, so every such create panics). ADR-0013's whole design —
   the application logs in as itself, no password anywhere — needs exactly
   that object. Cost: one manual `gcloud sql users create` per database, and
   2m27s of C-07(a)'s 17m45s. Fix: none available inside this repo; Cloud SQL
   refuses a password on an IAM user, so the Composition cannot route around
   it. The out-of-band create plus Observe adoption is the upstream
   workaround until a fixed provider ships — **which is precisely the
   dependency-bump change class M4 is built around.** The first time this
   build has wanted the factory's own machinery for its own sake.

7. **An XR can report Ready while the cloud grant it represents is stale, and
   every signal the platform exposes agrees with it.** Two mechanisms
   compound. upjet refuses an update that would require replacing the
   external resource, permanently rather than retrying; and a managed
   resource's `Ready` condition is not re-evaluated by a failed update, so it
   keeps saying True from its original creation while only `Synced` goes
   False. `function-auto-ready` judges the XR on `Ready`, not `Synced`. Cost:
   C-06's first run "succeeded" — one-line edit, namespace and RBAC moved in
   75 seconds, System Ready — with three cloud bindings still granting the
   old team. Fix: put the team in those objects' names so a move composes new
   ones (`platform-config` PR #6). **General form:** `Ready` answers "did
   this ever work", `Synced` answers "does it match what you asked for", and
   a status surface built on the first cannot see drift. Same family as M1's
   surprises 12, 14 and 15 — a confident, well-formed answer pointing the
   wrong way — and the most expensive version of it, because here the wrong
   answer is the green one.

8. **"docker build succeeded" was reported by the authoring agent and was
   false.** The legacy builder cannot cross-build — it loses the platform at
   the first intermediate layer — and with buildx the amd64 Go toolchain
   panicked under CPU emulation in `go mod tidy`. Cost: found only when the
   image was needed by a running cluster. Fix: a builder stage on
   `$BUILDPLATFORM` that cross-compiles with `TARGETOS`/`TARGETARCH`
   (`svc-hello` PR #2). **Same family as M1 surprises 8 and 12, with one
   addition:** exit 0 is not a result, *and an agent's report of a validation
   is not the validation*. In M1 the misleading evidence came from a tool; in
   M2 it came from the author, and the author's report was the only evidence
   anyone had.

9. **The image push is a genuine circular dependency, and kubelet dissolves
   it.** The registry the image goes to is created by the `System` the
   `Deployment` belongs to, so at first bring-up neither can go first. It
   resolved itself inside the verify window with no machinery: push once the
   registry managed resource is Ready, kubelet's `ImagePullBackOff` retries,
   the pod recovers. First bring-up only — afterwards the registry is durable
   (ADR-0015) and the cycle does not recur. Worth recording before someone
   designs an ordering mechanism for a problem a retry loop already solves.

10. **A plan generated before the previous layer's apply silently dropped a
    passthrough output.** Terraform omits null outputs from state, so a
    layer-1 plan made before layer 0 was applied read null for the
    passthrough and stored nothing — no error, no diff, just a missing
    output. Cost: one outputs-only re-apply, recorded as #3b on C-01's
    counter rather than folded into #3. Fix and rule: plan each layer only
    after the previous layer's apply.

11. **A guard written against drift fired on its own first real run, because
    `gcloud` rewrote a timestamp.** `gcloud artifacts repositories list`
    rewrites `createTime` into local time with no zone, even under
    `--format=value()`, so the adoption check's Zulu guard tripped and both
    registries came back `unknown` in the durable row. Fix: force UTC (commit
    `88129e7`). Small, and the reason it is here is that the check was
    ADR-0015's new safety net on its first outing — **a guard's first run is
    a test of the guard, not of the thing it guards.**

12. **M1's quota-project confusion came back wearing different clothes.**
    Group management and both IAM diagnostics resolved to a project this
    identity cannot use as a quota project; every call needed
    `--billing-project`. Cost: three APIs enabled by hand, which are now open
    C-01 debt (codify in layer 0 or disable). Recurrence is the finding —
    the identity that builds this platform is not the identity Google's
    tooling assumes it is talking to, and that shows up once per milestone in
    a new place.

13. **Crossplane's watch circuit breaker opened on the project IAM
    members.** `Too many watch events from ProjectIAMMember … Allowing events
    periodically`, with `Responsive=False`. Transient and self-healed, and
    recorded only because M2 has just made `ProjectIAMMember` the resource a
    team move creates and deletes in threes — a churn signal on the object
    C-06's fix multiplies.

14. **A team's project-level grant is one cloud object, and every System that
    team owns composes its own copy of it.** Found on 2026-09-17 by reading
    what the stuck C-06 objects had done overnight, then reproduced on demand.
    An IAM binding on a project is identified by (role, member) — nothing
    else. `svc-hello` and `svc-ledger` were both owned by `checkout`, so each
    System's Composition produced its own managed resource for the *same*
    binding of `group:checkout@` to `roles/cloudsql.client` (and again for
    `roles/cloudsql.instanceUser`). Two objects, one grant. When one System
    moved to another team, its object was deleted, the provider removed the
    binding, and the other System's grant went with it — while that System's
    object went on reporting Ready. The cost is bounded and measured: the
    provider's drift correction put the grant back unaided in about five
    minutes on the clean re-run, and within one ~ten-minute poll on the
    earlier watch. The generalisation is the uncomfortable part. ADR-0012
    decided a team is a *field*, never a thing; but a project-level grant to a
    team is a fact about the team, and modelling it as a per-System object
    means N Systems race over one cloud object. It is the same family as
    surprise 7: the platform's readiness signal is about the Kubernetes
    object, and says nothing about whether the cloud agrees. The registry
    writer is immune by construction — it is a grant on the System's *own*
    repository, so no other System shares it. Candidate fix in the errata.

15. **Teardown depends on a credential that expires overnight.** The cluster
    and the Cloud SQL instance ran for about nineteen hours unattended between
    the two sessions, because the Workspace reauthentication policy expired
    both the CLI credential and the application-default credential before
    `cycle.sh down` could be run, and re-authenticating needs a human at a
    browser. M1 recorded the same policy as a per-session manual cost. M2
    adds the asymmetry: an expired credential cannot *start* anything, so it
    fails safe for builds, but it also cannot *stop* anything, so it fails
    expensive for teardown. The rhythm C-02 relies on — tear down at the end
    of the session — has to happen before the credential lapses, not after.
    No fix built; the honest mitigation is procedural (tear down first, write
    up second), and the real one is a teardown path that does not run on a
    human's interactive credential.

### ADR errata found by the build

ADRs are superseded, never edited, so where the build found an ADR's text
wrong the correction lives here. Each says whether it looks like it needs a
superseding ADR at M2 close; that call is made at close, not now.

- **ADR-0012 §4 — four carriers, not six.** §4 says changing
  `spec.owner.team` changes "exactly four things"; with ADR-0013 §5's two
  team project IAM members (`roles/cloudsql.client`,
  `roles/cloudsql.instanceUser`) it is **six**, plus a seventh that is not a
  grant — the `platform-system` ConfigMap's `group` key, a derived
  convenience copy. §5's consequence "a team move therefore touches no cloud
  IAM" is wrong for the same reason, and it is wrong in the Decision rather
  than in an example. The implementation documents six. **Likely needs a
  superseding ADR at close**, because C-06's grade rests on the count and on
  that consequence.
- **ADR-0014 §3 — a wildcard provider group cannot be expressed in a Kyverno
  kind selector.** §3 writes the reality gate's kinds as
  `*.gcp.m.upbound.io` / `*.gcp.upbound.io`; a group wildcard is passed
  through to the webhook verbatim and matches nothing (surprise 4). The
  working form enumerates the installed provider groups — ten concrete groups
  across the two families — and must be extended whenever a provider is
  added. Also erratum in the same ADR: its consequence "the `ServerSideDiff`
  change is not needed yet" did not hold, though not for the reason it
  names — no mutating policy was installed, and Kyverno's *own* field
  defaulting was enough to force it (surprise 5). **Probably does not need a
  superseding ADR**: the decision — schema denies first, Kyverno validate-only
  for the rest — is unaffected. Recorded here so nobody implements §3
  literally and reopens the gate.
- **ADR-0013 — three gaps between the decision and what M2 built.** (i) §2's
  IAM `User` depends on a provider fix: at v3.0.0 the create path panics, so
  the object at the centre of the credential path arrives by one manual
  `gcloud` command per database (surprise 6). (ii) §5's human path — a
  `CLOUD_IAM_GROUP` user plus `psql` from the tailnet — **is deferred and not
  built in M2**; the `Database` Composition's header says so explicitly and,
  importantly, says the cross-XR team lookup that would enable it *is*
  possible (roughly eight lines plus one RBAC rule), so the deferral must not
  be recorded as a Crossplane limitation. In the meantime a human reaches the
  database with the break-glass superuser password, which is the shared
  credential ADR-0013 set out to avoid. (iii) **That break-glass password is
  stale after a rebuild.** The `<claim>-admin` Secret dies with the cluster,
  the Composition mints a new one, and Cloud SQL never learns it — the root
  password is pushed exactly once, at instance create (traced 2026-09-16
  through the provider, upjet and `terraform-provider-google` sources; the
  citations are in the header of
  `platform-config/crossplane/compositions/database/composition.yaml`).
  Recovery is one `gcloud sql users set-password postgres` command, and
  C-02's zero-intervention bar is unaffected because the GRANT Job tries the
  application's IAM login first. **(i) is upstream — record and re-test on a
  fixed release. (ii) and (iii) need either the build to catch up or a
  superseding ADR at close stating the M2 position honestly.**
- **ADR-0012's "re-created" mechanism is not what the provider does.** The
  Consequences predict one re-created resource on Terraform's IAM-member
  replacement semantics ("changing `member` forces replacement"); the
  Composition's own header refined that to three before the run. What
  happened is neither: upjet **refuses** to replace, so zero were re-created
  and three are permanently stuck (surprise 7). The fix — the team in the
  object's name, so a move composes new objects and garbage-collects the old
  — makes "re-created" literal and keeps the count at three, but it is a
  design change to the Composition's naming. **If C-06 grades ADJUSTED at
  close, this is the change the superseding ADR has to describe.**

- **ADR-0013 §1 and §5 — grants at project scope are shared between Systems
  (surprise 14).** ADR-0013 places the team group's two Cloud SQL roles on the
  System, as project-level IAM members (ADR-0012 §5 had said a team move
  "touches no cloud IAM"; ADR-0013 §5, written the same day, made that
  untrue). Every System a team
  owns therefore manages the same cloud binding, and removing one removes it
  for all until drift correction restores it (about five minutes, measured).
  Candidate fix, not built: give each System's grant its own identity with an
  IAM Condition scoped to that System's instances — for example
  `resource.name.startsWith("projects/<project>/instances/<system>-")` — which
  makes the bindings distinct *and* narrows `roles/cloudsql.instanceUser`,
  today project-wide, to the instances the System owns. The alternative is to
  say out loud that a team is a thing after all, with its grants composed once
  per team. Either is a design decision, so this **will need a superseding
  ADR at close**; it is recorded here rather than patched in passing.
