# Claims register

Pre-registered 2026-07-31, before the first build command (ADR-0008). Each
claim is a falsifiable prediction made by the design docs, with the test that
grades it and the data to collect. Append-only: claims are added with a date,
never reworded or deleted. Grades change only via a linked build-log entry.

Grades: **UNTESTED** → **HELD** | **ADJUSTED** (superseding ADR linked) |
**WRONG** (ADR linked).

*2026-10-27:* from M3's readiness note on, grades use the six words of
[ADR-0020](../adr/0020-a-failed-test-is-graded-without-abandoning-the-design.md)
(HELD, ADJUSTED, PARTIAL, UNSUPPORTED, WRONG, NOT REACHED) and may carry a
pre-committed ceiling.

## Scoreboard

| ID | Claim (short) | Milestone | Grade |
|----|---------------|-----------|-------|
| C-01 | Terraform ends at layer 0 | M1 | **ADJUSTED** (2026-09-17, ADR-0016 §7) — deferred from M1 |
| C-02 | Tear-down/rebuild is cheap enough to run between sessions | M1 | **HELD** on wall-clock and interventions (2026-08-28); the pre-registered spend figure exists only from 2026-08-27, so it is unrecoverable for all but M1's last two days (scope recorded 2026-09-21) |
| C-03 | provider-upjet-gcp covers the kinds the Compositions need | M1 | **HELD** for the kinds M2 composes: seven created and reconciled, one create-path gap (IAM `sql.User`); two kinds C-03 names that M2 does not compose (GCS bucket, Cloud DNS records) never hands-on created (2026-09-17; scope recorded 2026-09-21) — deferred from M1 |
| C-04 | One Argo app-of-apps can own the whole cluster surface | M1 | **HELD** for the M1 surface (2026-08-28) |
| C-05 | One YAML per tenant materializes the full tenant surface | M2 | **HELD** (2026-09-17) |
| C-06 | Ownership moves with a YAML edit, no re-plumbing | M2 | **ADJUSTED** (2026-09-17, ADR-0016 §1–2) |
| C-07 | Database claims: guardrails replace review | M2 | **ADJUSTED** (2026-09-17, ADR-0016 §3–4) |
| C-08 | The System schema survives a Composition swap | M2 (stretch) | UNTESTED — not attempted; deferred to M3 pending a governance decision (2026-09-17) |
| C-09 | internal→external is a `git mv` that forces security review | M3 | UNTESTED |
| C-10 | CI can hold folder ↔ field consistency | M3 | UNTESTED |
| C-11 | Kyverno reality gates back every intent gate | M3 | UNTESTED |
| C-12 | The metadata spine joins alert→team→repo→PRs structurally | M3 | UNTESTED |
| C-13 | The edge model works on GKE as designed | M3 | UNTESTED |
| C-14 | FQDN egress policy is available where the design assumes it | M3 | UNTESTED |
| C-15 | A rung is enforceable by branch protection + CODEOWNERS alone | M4 | UNTESTED |
| C-16 | The approval packet keeps L1 review real, not ritual | M4 | UNTESTED |
| C-17 | The scorecard derives from PR metadata + verification alone | M4 | UNTESTED |
| C-18 | Escape detection is automatic enough for automatic demotion | M4 | UNTESTED |
| C-19 | A change class can earn L2 on accrued track record | M4 | UNTESTED |
| C-20 | Gates can be three-outcome in practice | M4 | UNTESTED |
| C-21 | Agents operate PR-only with their own identity, fully attributable | M4 | UNTESTED |
| C-22 | Knowledge freshness and the question bank work as CI | M4 | UNTESTED |
| C-23 | Image pulls ride the Google-API path; internet egress reduces to git (added 2026-08-06) | M1 | **HELD** (2026-08-28) |
| C-24 | A per-System IAM Condition makes a team's Cloud SQL grants independent between Systems (added 2026-09-17, ADR-0016 §2) | M3 | UNTESTED |
| C-25 | A provider release with the sql.User fix removes the manual database-user step (added 2026-09-17, ADR-0016 §4) | M4 | UNTESTED |
| C-26 | The existing tenant files drive the new engine unchanged (added 2026-09-19, ADR-0017 §3) | M2b | UNTESTED |
| C-27 | Ownership still moves with a YAML edit on the new engine, and a broken naming rule fails loudly (added 2026-09-19, ADR-0017 §10) | M2b | UNTESTED |
| C-28 | Guardrails replace review without a custom API: a values file only the platform renders (added 2026-09-19, ADR-0017 §4–7 and §9) | M2b | UNTESTED |
| C-29 | The engine cutover is a rebuild: nothing durable is re-created, and rollback is a rebuild too (added 2026-09-19, ADR-0017 §12) | M2b | UNTESTED |
| C-30 | The new stack is smaller, measured rather than asserted (added 2026-09-19, ADR-0017) | M2b | UNTESTED |
| C-31 | The engine runs on narrow custom roles, and what they leave out is the third deletion lock (added 2026-09-21, ADR-0018) | M2b | UNTESTED |
| C-32 | Rebuild cost is measured, not estimated (added 2026-09-22, ADR-0009 and ADR-0015) | M2b | UNTESTED |
| C-33 | A removed, renamed or broken file deletes no tenant by itself, and a Kyverno outage recovers with one manual sync per failed Application (added 2026-09-24, ADR-0019) | M2b | UNTESTED |
| C-34 | Tenant namespaces are isolated by default (added 2026-10-27, ADR-0021 §15) | M3 | UNTESTED |

*2026-10-27:* ADR-0020's vocabulary applies from M3's readiness note on; every grade above was given under ADR-0008's three words and stands.

## M1 — Spine

- **C-01 — Terraform ends at layer 0.** (`platform-pattern.md` thesis)
  After bootstrap (project, VPC, GKE, workload identity, Argo CD install),
  every platform change lands as a PR; no recurring Terraform.
  **Test:** count `terraform apply` runs after M1 completes, across the entire
  build. Target: zero. **Data:** every exception and why it couldn't be a PR.
- **C-02 — Tear-down/rebuild is cheap.** (CLAUDE cost model; `platform-pattern.md`)
  The cluster can be destroyed between sessions and rebuilt from
  `platform-bootstrap` + Argo sync without manual steps.
  **Test:** scripted down/up cycle, run at least 3 times. **Data:** wall-clock
  minutes down and up; manual interventions (target zero); actual monthly GCP
  spend.
- **C-03 — provider-upjet-gcp kind coverage.** (`research/crossplane-v2-2026.md`,
  currently [unverified] — the research verified the AWS family)
  The GCP provider family covers every kind the Compositions need: Cloud SQL
  instance/database/user, GCS bucket, service account + IAM bindings, Artifact
  Registry, Cloud DNS records.
  **Test:** primary-source check of the provider docs, then hands-on create of
  each kind. **Data:** kind-by-kind checklist with provider versions; gaps and
  workarounds.
- **C-04 — One app-of-apps owns the cluster surface.** (`platform-pattern.md`)
  Argo CD from `platform-config` manages Crossplane, Kyverno, ESO,
  external-dns, gateway, and the XRDs/Compositions with sync status as the
  single truth.
  **Test:** fresh cluster reaches all-synced from empty with no manual
  ordering. **Data:** CRD-race / sync-wave workarounds required (a known
  friction the design doesn't currently acknowledge).
- **C-23 — Image plane on the Google-API path.** (ADR-0010; added 2026-08-06,
  during M1 — the register allows dated additions.)
  With private nodes + Private Google Access, every cluster image pull is
  served by Artifact Registry remote repositories (`*.pkg.dev`); the only
  internet egress the cluster generates is Argo CD's git traffic to GitHub
  through Cloud NAT.
  **Test:** with the cluster up and Argo CD running, inspect NAT logs / VPC
  flow logs: no egress to public registries; image pulls resolve to the
  Google-API path. **Data:** the list of upstream registries proxied; any
  image that could not be served through a remote repo and what was done
  about it.

### M1 grading note (2026-08-28)

Graded at M1 close; full evidence and reasoning in
[`m1-spine.md`](m1-spine.md) under "Claims graded".

**HELD:** C-02 (3 scripted cycles; 0 manual interventions on cycles 2 and 3,
the single cycle-1 intervention caused by a mechanism ADR-0011 has since
deleted — actual monthly spend still open, see the entry), C-04 (all-synced
from empty, twice, at the cost of five sync-wave workarounds; graded for the
Crossplane surface M1 built, not the full addon list the claim names), C-23
(two independent pullers, digest-matched, zero registry egress).

**Deferred, not graded:** C-01 and C-03. Neither claim's test can run at M1 —
C-01 counts terraform applies *after* M1 completes, and C-03's hands-on half
needs Compositions that are M2 work. Both are re-scheduled for M2 close. The
claims themselves are unchanged per the append-only rule; only the grade
column carries the deferral.

**The pattern worth naming:** two of five M1 claims turned out to be assigned
to a milestone that cannot execute their test. Surprise 9 recorded that a
claim's test is a requirement on the *build*; this adds that it is also a
constraint on the *schedule*. Pre-registration should ask, at write time,
"what must exist for this test to run, and does that exist by this
milestone?" — a question worth applying now to every remaining claim rather
than rediscovering it at each close.

## M2 — Paved road

- **C-05 — One YAML per tenant.** (`platform-pattern.md`, System resource)
  A single System XR materializes namespace, Argo AppProject, RBAC bindings,
  quotas, and an Artifact Registry prefix.
  **Test:** onboard a second tenant with one file. **Data:** lines of YAML per
  tenant; wall-clock from PR merge to usable namespace.
- **C-06 — Ownership moves without re-plumbing.** (`platform-pattern.md`)
  Identity binds at one point (IdP group → IAM → RBAC), so a reorg is a YAML
  edit.
  **Test:** move `svc-hello` between teams. **Data:** files touched (target:
  1), anything that had to be re-created.
- **C-07 — Guardrails replace review for databases.** (ADR-0003)
  A database claim in the service repo provisions on the golden path;
  Kyverno/Composition bounds block out-of-range requests; deletion protection
  holds.
  **Test:** (a) PR → usable database, timed; (b) claim requesting an oversized
  instance and a wrong region is denied at admission; (c) delete the claim —
  the database survives (`managementPolicies` omit Delete).
  **Data:** minutes PR→ready; denial messages as seen by the developer; state
  after claim deletion.
- **C-08 — Schema survives the Composition swap.** (`platform-pattern.md`;
  stretch) Whether a "system" maps to a namespace or a project is absorbed by
  the Composition, not the developer-facing schema.
  **Test:** implement an alternate Composition for the same XRD without a
  schema change. **Data:** schema fields that leaked implementation detail.

### M2 readiness note (2026-09-02)

The question the M1 grading note asked for — "what must exist for this
test to run, and does it exist by this milestone?" — was applied to
C-05..C-08 before M2's first build command. Result, with the full
prerequisite list in [`m2-paved-road.md`](m2-paved-road.md): all four
tests are runnable at M2, so no claim is re-scheduled. Two of them cannot
be *defined* until decisions the register silently assumed are made — the
tenant model behind "move `svc-hello` between teams" (C-06), and the
database credential path behind "usable database" (C-07) — and those go to
ADRs before the `System` XRD is written. Four `terraform apply` runs are
required by the tests and are pre-declared there for C-01's count. Claims
unchanged, per the append-only rule.

**2026-09-14 — the gating decisions landed as ADR-0012..0015** (tenant
model; database credential path; admission mechanism; durable resources and
the rebuild). Two consequences for this register, stated before the build:
C-06's "move `svc-hello` between teams" is a one-line edit of
`spec.owner.team` with one predicted re-creation (the registry IAM member);
and from M2 on, C-02's rebuild is no longer "from empty" — durable resources
are adopted on `up`, and the cycle results record adopted-vs-recreated so
C-07(c) and C-02 share evidence. Claims unchanged.

**2026-09-16 — M2 was built and first exercised.** One bundled `terraform
apply` for layer 0 and one for layer 1 (layer 1 twice — the second an
outputs-only re-apply after an operator error), with layers 2 and 3 carried on
the normal rebuild; both XRDs and Compositions live on a rebuilt cluster, two
tenants, one database an application logs into with no password, after one
manual out-of-band user create forced by an upstream provider bug; and Kyverno
installed. First data exists for C-05, C-06 and C-07(a)(b), and for C-01's
apply count, C-02's bring-up and C-03's hands-on half; C-07(c), the Argo CD
denial surface, a clean C-06 re-run and a parked rebuild have not been run,
and C-08 was not attempted. **Nothing is graded:** every M2 claim stays
UNTESTED in the scoreboard until M2 closes, because ADR-0008 earns grades at
a close with evidence rather than asserting them mid-milestone. The evidence,
the misses, and a provisional reading that is explicitly not a set of grades
are in [`m2-paved-road.md`](m2-paved-road.md). Claims unchanged.

### M2 grading note (2026-09-17)

Graded at M2 close; full evidence and reasoning in
[`m2-paved-road.md`](m2-paved-road.md) under "Claims graded".

**HELD:** C-05 (second tenant from one file of eleven non-comment lines;
merge to Ready 4m08s, to a running workload about five minutes), C-03 for the
kinds M2 composes (seven kinds created and reconciled at namespaced scope on
v3.0.0; one recorded gap — an IAM `sql.User` cannot be created, upstream
issue #1000, workaround one out-of-band command then adoption; GCS bucket and
Cloud DNS records not hands-on tested, carried to M3).

**ADJUSTED, each with its superseding ADR (ADR-0016):** C-01 (four applies
on the persistent layers after M1 against a target of zero, plus two changes
that rode the ordinary rebuild — identity and reachability for a new
capability are a declared Terraform crossing; nothing a tenant, service,
database or ownership change needs was), C-06 (the one-line move re-creates
three cloud grants in about two minutes, but only once the team is part of
those objects' names; first run moved nothing in the cloud while reporting
Ready; a sibling System loses its team's grant for about five minutes until
ADR-0016 §2 is built), C-07 (passwordless database in 17m45s with one manual
command forced by the provider bug; denial messages clear at the CLI and the
API server for every bad claim and through Argo CD for one; the reality gate admitted what it exists to deny until it was
fixed; claim deletion, survival and 66-second adoption held cleanly).

**Not attempted:** C-08 (stretch). Its test needs organization-level
project-creation power for the provider identity, a governance question that
was not decided. Deferred to M3, the same way C-01 and C-03 were deferred
from M1: the claim and its milestone column are unchanged, only the grade
column carries the deferral. Also not done: ADR-0015 §6's second rebuild
mode (after a deliberate delete), carried forward in the build log.

**C-02**, graded HELD at M1, is not regraded. ADR-0015 changed what it
measures; cycle 5 is its first clean cycle under the new meaning — zero
manual interventions, 35m31s up, three durable resources adopted, none
re-created.

**Added, dated 2026-09-17, from ADR-0016:** C-24 and C-25 below. Existing
claims unchanged per the append-only rule; only the grade column moved.

**The pattern worth naming this time:** M1's note was that a claim's test
constrains the schedule. M2's is that *a readiness signal is not evidence*.
Three of M2's defects — a team move that moved nothing in the cloud, a grant
deleted by a sibling, a gate that matched nothing — were invisible at every
level the platform reports on, and were found only because the tests read the
cloud side rather than the object. Pre-registration should ask of every
remaining claim: what would this test look at if the platform were lying?

## M2b — Engine swap

Added 2026-09-19 from [ADR-0017](../adr/0017-config-connector-engine-platform-rendered-forms.md),
before any build command for it ran. ADR-0017 retires Crossplane from the
reference path in favour of Config Connector with platform-rendered Helm
charts. **No M2 grade changes:** C-05, C-06 and C-07 were graded on Crossplane
and stay as graded. The claims below are the same questions asked of the new
engine, with M2's measured numbers as their baselines, plus two questions the
swap itself raises. C-31 was added on 2026-09-21 from
[ADR-0018](../adr/0018-engine-permissions-are-a-list-in-platform-roles.md),
which folds a second decision into this milestone, also before any build
command ran. C-28 (b) and (b2), C-29 and C-30 were reworded the same day to
account for it — before this section's first commit and before any build
command, the only point at which the append-only rule allows it. On
2026-09-22, from a second outside review and under the same rule, C-28 (b)
and (b2) and C-29's Data were extended again, with (b)'s "per-System budget"
renamed "per-System claim limit" to match ADR-0017 §6, and C-31's Data was
extended; C-32 was added the same day from ADR-0009's cost estimate and ADR-0015's cost
consequence, also before any build command ran. The claim texts of
C-01..C-25 are unchanged per that rule; their grade cells may still gain
scope notes through a linked build-log entry, as C-03's and C-04's already
have. C-25's subject is removed by ADR-0017 and its grade column is dealt
with at M2b's close, by a linked build-log entry like any other. C-03's
carried-forward hands-on checks — a GCS bucket and Cloud DNS records — will
never run on the provider C-03 names, which ADR-0017 retires, so at M2b's
close a linked build-log entry of the same kind records them as never
tested by hand, rather than carrying them to M3.

On 2026-09-23, C-02's grade cell gained a scope note the same way, for its
spend data point, and C-03's was made exact about which kinds it covers. Both
rest on post-close addenda dated 2026-09-21:
[C-02's third data point](m1-spine.md#post-close-addendum-2026-09-21-c-02s-third-data-point)
in the M1 entry, and
[what C-03's HELD covers](m2-paved-road.md#post-close-addendum-2026-09-21-what-c-03s-held-covers)
in the M2 entry. No claim text and no grade changed.

On 2026-09-24, C-33 was added from
[ADR-0019](../adr/0019-a-file-change-deletes-no-tenant-by-itself.md), which
turns three things ADR-0017 left to convention or left open into guards, also
before any build command ran. It tests those guards, and a Kyverno outage, in
the local rehearsal. The texts of C-26..C-32 are unchanged; ADR-0019 §2 also
fixes the order in which C-28 (c) is run — on the System to which C-28 (a)
adds a claim, while both claims exist.

- **C-26 — The tenant files survive the engine.** (ADR-0017 §3; C-05 asked
  again, and C-08's idea tested harder than its own test would have)
  The files in `systems/tenants/` drive the new engine without edit and
  materialize the same tenant surface; a new tenant is still one file.
  **Test:** cut over with `systems/tenants/` untouched; then onboard a third
  tenant with one file. **Data:** bytes changed in existing tenant files
  (target: zero); objects present in M2's tenant surface and absent now, and
  the reverse; wall-clock from merge to a running workload (M2 baseline: 4m08s
  to Ready, about five minutes to a workload).
- **C-27 — Ownership still moves with a YAML edit.** (ADR-0017 §10; C-06
  asked again)
  Changing `spec.owner.team` re-creates the three team-bearing cloud grants
  through Argo CD apply-and-prune, and the cloud agrees.
  **Test:** move `svc-hello` between teams while polling the project and
  repository IAM policies from the cloud side every ten seconds; then
  deliberately break the naming rule on one member and move again.
  **Data:** files touched (target: 1); resources re-created (predicted: 3);
  seconds from merge until the cloud policy shows the new team and not the old
  (M2 baseline: 1m55s); whether anything reported healthy while the cloud
  disagreed (M2 baseline: yes, on the first run); whether the broken rule
  fails loudly or silently (predicted: loudly, as a failed sync).
- **C-28 — Guardrails replace review without a custom API.** (ADR-0017 §4–7 and §9;
  C-07 asked again)
  A `claims.yaml` in the service repo provisions on the golden path; the form
  is enforced by who renders it rather than by an API type; deletion
  protection holds.
  **Test:** (a) PR → usable database, timed, with no out-of-band command;
  (b) each of these is refused — a wrong region, an oversized request, the
  cross-field rule, an extra key (`ipv4Enabled: true`), a string crafted to
  inject YAML, a claim over the per-System claim limit, an unknown key under
  `environment`, a claim name containing a hyphen, size L with
  `tier: critical` in a standard-tier System, also when the `claims.yaml`
  sets `system.tier: critical` (with a tenant file that states
  `tier: standard` and with one that leaves `tier` out) — and so is a raw
  `SQLInstance` placed in the tenant's `k8s/`, an Argo CD Application placed there and aimed at the platform's project,
  with and without `metadata.namespace: argocd`, a Config Connector object in
  a gated group (`sql`, `iam`, `artifactregistry`) created, edited or deleted
  with `kubectl` by a team member, and an edit by anyone that removes or
  changes the abandon annotation on a durable object; (b2) two things
  predicted *not* to be refused, each recorded: a `claims.yaml` that sets a
  real platform value (`environment.projectID`, and separately
  `system.tier: critical`) — `valuesObject` outranks it
  and the schema sees only the merged values (ADR-0017 §4), so the test is
  that the rendered output is byte-identical with and without it; and a
  Config Connector object in an ungated group the identity holds no write
  permission in (a `StorageBucket`) — admitted by the API server and never
  actuated. The group is named because "ungated" does not mean "unpermitted"
  while the broad roles are bound: `roles/cloudsql.admin` also carries a few
  writes outside Cloud SQL, and `roles/compute.viewer` lets the engine read;
  (c) remove the entry — the instance survives; restore it — the
  instance is adopted.
  **Data:** minutes PR→usable and manual commands (M2 baseline: 17m45s, one);
  every refusal verbatim from every surface it appears on — the PR check, the
  claims Application's status, Kyverno — and which surfaces a developer can
  actually see; anything that admitted; seconds to adopt (M2 baseline: 66).
- **C-29 — The cutover is a rebuild.** (ADR-0017 §12; ADR-0015's adoption
  used as the migration mechanism)
  Switching engines costs one `down` and one `up`, re-creates nothing durable,
  and is reversible the same way.
  **Test:** `down` on Crossplane, `up` from the Config Connector branch; then
  the reverse, once. Both legs run with the engine identity still holding the
  built-in roles Crossplane's holds. ADR-0018's custom roles are bound too; a
  desk check before the first leg must show each is a subset of the built-in
  role it replaces, so that they add nothing (ADR-0018 §4). If it does not,
  the leg does not start. C-29's data closes with the reverse leg; the return
  build onto Config Connector belongs to C-31. **Data:** adopted versus
  re-created for each durable
  resource (target: all adopted, none re-created); each non-durable resource
  the new engine could not acquire and what was done about it; seconds any
  grant was absent from the cloud policy; manual interventions (target: zero);
  `up` wall-clock (M2 baseline: 35m31s); and whether `postgres` accepts the
  `<claim>-admin` Secret's value after the forward leg, from one `psql` login
  (it settles ADR-0017 §8's [I] on `rootPassword` riding in an update).
- **C-30 — The new stack is smaller.** (ADR-0017 Context — the reason for the
  swap, made falsifiable)
  For the same tenant surface, the new engine asks less of the cluster and of
  the reader.
  **Test:** count both engines at the same tenant surface, Crossplane from the
  last M2 cycle and Config Connector after cutover. **Data:** engine pods and
  their memory; CRDs installed; non-comment lines of recipe, schema and policy;
  the kinds and concepts a reader must learn to follow one tenant from its file
  to its cloud resources, listed by name; layer-0 and `platform-roles`
  resources that exist for the engine's identity. **Prediction, written now
  and expected to come out mixed:** fewer pods, fewer recipe lines — and more
  CRDs, by a wide margin. Concepts: still fewer on the new engine, by a
  smaller margin than the swap alone would give. The count is reported twice
  — with and without the three things ADR-0018 adds (an eighth repo, the
  custom role, a second home for Terraform) — and "less of the reader" is
  falsified if the total with them is not lower than Crossplane's.
- **C-31 — The engine runs on narrow roles, and what they leave out is the
  third lock.** (ADR-0018; ADR-0017 §1, §7 and §9; ADR-0015 §1 and §5)
  After C-29 has run both legs on the permissions Crossplane holds, one
  `platform-roles` apply made with the cluster down removes the four broad
  built-in grants. The engine identity then holds four custom roles — 21
  permissions, each traced to a Config Connector v1.156.0 call, against about
  270 — plus the read-only `roles/compute.viewer`, unchanged and stated. On
  those it adopts everything as C-29's forward leg did, onboards and offboards
  a tenant, provisions a database, and cannot delete a durable resource, edit
  a role, or grant a project role outside the two-role list.
  **Test:** (control, first session, broad roles) a raw scratch
  `ArtifactRegistryRepository` with no abandon annotation — in a scratch
  namespace that is not a tenant's (no system label, so ADR-0017 §7's gate
  does not match it; it carries the project annotation), created by the
  operator with `kubectl` — is created and deleted through Config Connector,
  and the cloud delete *succeeds* — so the
  lock test can see a real delete. (desk, before narrowing) each role file is
  a subset of the built-in role it replaces; the first session's audit log
  lists the write permissions the engine really used. (desk, after narrowing,
  no cluster) the project's policy shows exactly five grants for the identity
  — the four custom roles and `roles/compute.viewer` — and none of the four
  broad built-in roles; a script acting as the engine passes its
  must-succeed list and is refused on its must-be-refused list — the four
  durable deletes on scratch resources, an edit to its own role, a
  `roles/viewer` grant on the project. (a, return build, must succeed) `up` on
  Config Connector with every durable resource adopted and none re-created,
  the parked instance woken, every Application Synced and Healthy, no object
  with a permission error; a scratch tenant onboarded from one file with one
  database claim, then offboarded. (b, return build, must be refused by
  Google) in the same kind of scratch namespace, the Kubernetes objects for a
  raw scratch registry and a raw scratch database, neither carrying the
  abandon annotation, are deleted — so the role is the only lock left
  standing — and the cloud side is read.
  **Data:** permission counts before and after, from `gcloud iam roles
  describe`; write permissions seen in the first session's audit log that the
  files lacked; every refusal verbatim, paired with a cloud-side read (still
  present, creation time unchanged); the adoption table and `up` wall-clock
  beside C-29's forward leg; every permission error for the engine identity
  after the narrowing, verbatim, split into found at the desk and found in
  the cluster, with the permission that fixed it and whether the error named
  it; `platform-roles` applies needed after the narrowing (target: zero);
  what Config Connector does with a delete Google refuses — stuck finalizer
  or not, the condition's text, whether the namespace hangs, what released
  it; minutes from a fix's apply until the engine's next call succeeds; grants
  found on the identity that are not in the repo (target: none); whether the
  first real project grant through Config Connector passes the condition,
  which the desk script cannot show; the tenants the first session added or
  moved, listed beside the adoption table so the comparison is read with
  them in view; the scratch cloud resources left behind and the `gcloud`
  deletes a person made to remove them (ADR-0015 §5); and the scratch
  tenant's name going onto the retired list when its file is removed and
  coming off after those deletes (ADR-0017 §9).
  **Prediction, written now:** every refusal holds; one or two permissions are
  missing on the first try and the desk finds at least one of them; the
  refused deletes leave their objects stuck until the annotation is added.
  **Falsified if** the engine identity deletes a durable resource from the
  cloud; or a resource C-29's forward leg adopted is re-created; or the golden
  path needs a permission the design forbids (a durable delete, anything
  under `iam.roles.`, an unconditioned `projects.setIamPolicy`); or the
  `roles/viewer` grant appears in the project's policy; or the engine's edit
  to its own role succeeds; or a failure in the return build can neither be
  tied to a denied call by the engine identity in the audit log nor be
  reproduced with the four built-in grants restored.
- **C-32 — Rebuild cost is measured, not estimated.** (ADR-0009's cost
  estimate, at the end of its Decision — "Session burn rises from near-zero
  to roughly $0.50/hour while the cluster exists" — and ADR-0015 §4 with its "Stopping is not free"
  consequence, which promises the cost "is recorded per session when the
  deferred cost analysis is picked up"; added 2026-09-22)
  The billing export makes the project's published cost estimate checkable:
  an M2b session's spend can be read from the export rather than inferred,
  and a parked day has a number too.
  **Test:** read the billing export for each M2b session — every cutover leg,
  the return build, and the narrowing session — and for the parked days
  between them, keyed to the reference project. Compare against ADR-0009's estimate. No
  figure is published that the export does not carry; nothing is estimated
  from memory.
  **Data:** list-price and net (post-credit) spend per M2b session, with the
  session's `up` and `down` wall-clock beside it so the hourly rate is
  derivable; the implied cost per cluster-hour against ADR-0009's roughly
  $0.50/hour; the parked-instance line items (storage and the private IP,
  which ADR-0015 says bill while parked) as a cost per parked day; the
  September baseline the export already holds — 2026-09-01 → 09-21 as read
  on 2026-09-21 (that day still partial), $17.03 list price,
  net zero under the trial credit, which covers M2, its close and the M2b
  design work; and the same spend split by the `system` label the charts put
  on each instance and registry (ADR-0017 §8), so a System's share is read
  from the export rather than allocated by hand [I: that those labels reach
  the billing export rests on secondary sources].
  **Prediction, written now:** the measured hourly rate for a session with the
  cluster up lands within the same order of magnitude as ADR-0009's estimate
  but not on it,
  because the estimate predates the Cloud SQL instance and the Tailscale
  subnet router; a parked day costs little but not nothing; and the largest
  single surprise is a session that was left running, not a rate that is
  wrong.
  **Falsified if** the export cannot be tied to a session — no session's spend
  can be isolated from it — or if a published cost figure still has to come
  from an estimate after M2b closes.
- **C-33 — A removed, renamed or broken file deletes no tenant by itself,
  and a Kyverno outage recovers with one manual sync per failed
  Application.** (ADR-0019; ADR-0017 §3, §4, §7 and its Consequences; added
  2026-09-24)
  None of these deletes a tenant's Application, namespace or claim objects
  without a person's step: a merge that removes or renames the
  ApplicationSet, renames `tenants/`, removes a tenant file, removes a
  System's last claim, misnames its `claims.yaml`, carries a malformed tenant
  file, or makes `charts/system` stop rendering the tenant Namespace; nor a
  `kubectl delete` of the ApplicationSet with the default background cascade.
  And once a Kyverno outage ends, the namespace teardown it blocked completes
  by itself, and each Argo CD sync it blocked completes with at most one
  manual sync per Application. (The engine's own finalizer updates are not
  tested: no engine runs in the rehearsal.)
  **Test:** in the local rehearsal, with the engine's CRDs installed and no
  engine running, and three tenants: two with one claim each, one with no
  `claims.yaml`. (a) Remove the ApplicationSet's file, then restore it; rename
  the ApplicationSet; delete it with `kubectl` (background cascade); merge a
  `charts/system` change that drops the tenant Namespace. Restore the
  original state after each. (b) Rename `tenants/`, then restore it; then
  offboard one tenant as ADR-0019 §1 describes — the PR, then a person's
  `argocd app delete` of its `<system>-system` Application. (c) Remove one
  System's only claim; separately, rename its `claims.yaml` to `claims.yml`;
  then run the manual sync with prune on the first; and read the status of
  the tenant with no `claims.yaml`. (d) Merge a tenant file with a required
  field missing. (e) Scale Kyverno's admission controller to zero; merge one
  change to a gated object, and delete a tenant namespace that holds one.
  Wait at least 45 minutes: the repo's retry block is ten retries doubling
  from 15 seconds and capped at five minutes, and because Argo CD v3.4.6
  counts a retry before computing its wait, the first comes after 30 seconds
  and the block takes about 38 minutes [C, argo-cd v3.4.6
  `controller/appcontroller.go`, `processRequestedAppOperation`;
  `pkg/apis/application/v1alpha1/types.go`, `NextRetryAt`]. Confirm each
  failed operation has used every retry; then restore Kyverno. No other
  commit reaches the rehearsal's sources during the wait, so a new revision
  cannot start a sync that hides the count. (e) counts only if at least one
  write was refused, recorded verbatim; if Argo CD's self-heal restores
  Kyverno before the wait ends, that is recorded and (e) is repeated with the
  `kyverno` Application's automated sync off.
  **Data:** Applications, namespaces and gated objects deleted in (a)–(d)
  before any person's step (target: none; then exactly the one tenant
  offboarded in (b)); each affected Application's sync status and condition
  text, verbatim; in (e), every refused write verbatim, whether and when the
  namespace finished terminating after Kyverno returned, and every step
  recovery needed.
  **Prediction, written now:** (a)–(d) delete nothing before a person acts;
  (c) shows Argo CD's "auto-sync will wipe out all resources" condition and
  the manual prune completes the removal; the tenant with no `claims.yaml`
  reports Synced and Healthy. In (e) the namespace finishes terminating by
  itself once Kyverno returns. An Application whose sync failed through the
  whole retry block stays failed on that revision until someone syncs it
  again, because Argo CD does not start a new automated sync for a revision
  whose last sync operation failed [C, argo-cd v3.4.6
  `controller/appcontroller.go`, `autoSync`]: one manual sync per such
  Application.
  **Falsified if** any case in (a)–(d) deletes an Application, a namespace or
  a claim object without a person's step; or recovery in (e) needs more than
  one manual sync per failed Application.

**On 2026-10-28, before S3 counts the Config Connector side, C-30's two open
readings are fixed.** C-30's text and prediction are unchanged.

*Lines.* "Non-comment lines of recipe, schema and policy" is read narrowly.
Recipe means the files that turn a tenant file or a claim into objects: the two
Compositions on Crossplane, and the templates of `charts/system` and
`charts/claims` on Config Connector. Schema means the two XRDs on Crossplane and
the two charts' `values.schema.json` on Config Connector. Policy means
`kyverno/policies/`. Lines are counted non-comment, with template comments
stripped, which is the 2026-10-23 desk count's first method. This reading was
chosen after that desk count had been seen on both sides. It is the reading
less favourable to the prediction (+12.3% for recipe, schema and policy on the
narrow set, against −7.8% on the widest), and it is the one whose rows sum
exactly the three categories the text names. The wider sets are reported
beside it and are not graded. The count is re-run at the close on the commit
S3 runs.

*Concepts.* The method is fixed here, before either list is written. The
reader follows `svc-hello` with one database claim, from its tenant file and
its claim to its cloud resources. Two kinds of item are listed by name, each
once, with the file where the reader first meets it:
- every API kind met on that path, whether it creates, is created or refuses;
- every mechanism that is not a kind but must be known to predict what
  happens.

Items the two engines share, such as Argo CD's and Kyverno's, are counted on
both sides. The total is reported twice: without and with ADR-0018's three
additions (an eighth repo, the custom role, a second home for Terraform), each
counted as one item. The Crossplane list is made from `3bfc72d`, the last
Crossplane `main`; the Config Connector list from the commit S3 runs. Both
lists are published in M2b's entry so a reader can dispute any item, and the
claim's "Falsified if" is read on the second total.

## M3 — Approval boundary

- **C-09 — The flip forces review.** (ADR-0002)
  Moving a route file `internal/` → `external/` structurally requires security
  approval via CODEOWNERS; there is no path around it.
  **Test:** simulated PRs — the flip, and attempted bypasses (new file straight
  into `external/`, field flip without the move). **Data:** which paths were
  actually blocked vs. slipped.
- **C-10 — Folder ↔ field consistency is CI-enforceable.** (ADR-0002)
  **Test:** CI check catches every folder/`exposure:` mismatch in a
  deliberately adversarial PR set. **Data:** the mismatch cases tried; escapes.
- **C-11 — Reality gates back intent gates.** (ADR-0001/0002; metadata spine)
  Kyverno denies edge objects from app namespaces and denies untagged
  resources.
  **Test:** create violating resources directly (not via golden path).
  **Data:** deny behavior and message quality; anything that admitted.
- **C-12 — The spine join is structural.** (`platform-pattern.md`)
  An alert arrives already carrying service, team, repo, and the PRs merged
  since last healthy — no archaeology.
  **Test:** synthetic incident on `svc-hello`. **Data:** labels present on the
  firing alert; manual steps to reach "which PR did this" (target: zero).
- **C-13 — The edge model works on GKE as designed.** (ADR-0002/0004;
  `research/api-gateway-2026.md` — GKE Gateway conformance [unverified])
  The `exposure:` field selects WAF policy, DNS zone, and gateway class;
  NS-delegated env zones and private zones resolve; certificates issue.
  **Test:** external and internal route for `svc-hello`, both env zones.
  **Data:** Gateway API conformance findings; Cloud Armor attachment path;
  cert issuance time; deviations from the designed hostname schema.
- **C-14 — FQDN egress policy availability.** (`research/egress-control-2026.md`,
  [unverified] for GKE tier/availability)
  **Test:** primary-source check, then hands-on FQDN egress rule on the
  reference cluster tier. **Data:** availability, tier constraints, behavior
  under DNS churn.

- **C-24 — Per-System grants are independent.** (ADR-0016 §2; added
  2026-09-17 from M2's shared-grant finding)
  An IAM Condition scoped to a System's own instances makes that System's
  Cloud SQL grants separate cloud objects, so another System's lifecycle
  cannot remove them, and narrows `roles/cloudsql.instanceUser` from
  project-wide to the instances the System owns.
  **Test:** two Systems owned by one team; move one to another team while
  polling the project policy every ten seconds for fifteen minutes; the
  application still connects through the Auth Proxy under the condition.
  **Data:** seconds the sibling's grant was absent (target: zero; M2 baseline:
  about 300); the condition expression as shipped; any API call the condition
  broke.

- **C-34 — Tenant namespaces are isolated by default.** (ADR-0021 §15;
  platform-config#21; `platform-pattern.md`, "The tenant model"; added
  2026-10-27)
  Every tenant namespace carries Pod Security `baseline` (enforced, with
  `restricted` as warn and audit, the version pinned) and a default-deny
  NetworkPolicy with a named allow list, so a pod in one tenant's namespace
  cannot connect to another tenant's, and a pod that asks for privilege is
  refused, while the golden path keeps working. This is the cooperative model
  between tenants: it stops accidents and noisy neighbours, and hostile-tenant
  isolation is not claimed.
  **Test:** (a) from a pod in `svc-ledger`'s namespace, connect to `svc-hello`'s
  Service and to its pod IP, before the default is applied and after;
  (b) `svc-hello`'s `/healthz` and a fresh `GRANT` Job stay green after it;
  (c) a privileged pod in a tenant namespace is refused; (d) a tenant-written
  allow-all NetworkPolicy is applied, and what it opened is recorded.
  **Data:** each connection and its result, before and after; the allow list as
  shipped, and every allow the golden path needed that platform-config#21's list
  missed; the refusal message for (c); the set of pods (d) reached.
  **Prediction, written now:** before the default, (a) succeeds, because no
  policy selects the pods; after it, both
  connections fail; (b) and (c) hold; (d) opens only the tenant's own pods
  [I, from platform-config#21; none of it tried]. An allow-table note: the
  external Gateway's proxies and health checks arrive from `130.211.0.0/22` and
  `35.191.0.0/16`, and `gke-l7-rilb`'s from the proxy-only subnet's range plus
  the same health-check ranges.
  **Falsified if** a cross-tenant connection succeeds under the default, or the
  golden path breaks.

### M3 readiness note (2026-10-27)

The question the M1 grading note asked for was applied to M3 before its first
build command, together with the decisions in
[ADR-0021](../adr/0021-the-approval-boundary-on-the-config-connector-engine.md)
and the vocabulary and ceilings of
[ADR-0020](../adr/0020-a-failed-test-is-graded-without-abandoning-the-design.md).
Result: M3 is split into **M3a** (the approval boundary: C-09, C-10, C-11) and
**M3b** (the edge it governs: C-12, C-13, C-14, C-24 and the new C-34). The
milestone column stays M3 for every claim, and both halves close in one
build-log entry. platform-config#33, which closed a way around the gate C-11
tests, merged on 2026-10-27, before any C-11 case runs. Each entry below gives the half, the
test and its cases, the pass line, "Falsified if", the pre-committed ceiling and
what one person on both teams cannot show. A ceiling is the highest grade the
close may give; it never forbids a lower one or NOT REACHED. **Claims
unchanged**, per the append-only rule: no claim's text above is touched, and the
paragraphs below say how each is run.

**Entry gate, archive and order.** Nothing that touches the cluster runs before
M2b's build-log entry exists (ADR-0021 §1); the GitHub-only part of M3a may.
Before the first M3 build command of either kind, this register with this note,
ADR-0020 and ADR-0021 are tagged on `main` and archived by a GitHub release and
a Software Heritage save request; the tag and the archive identifiers are the
first lines of M3's build-log entry, since they cannot be written here. The
cluster work runs in this order, and a later step does not start until the
earlier one has its data: the layer-0 and layer-1 applies, the layer-2 rebuild
(timed; NAT logs re-read for C-23), C-34, C-11's cluster cases, C-13, C-24,
C-12, C-14. Under ADR-0020 a claim whose test does not run inside the milestone
is graded NOT REACHED with its precondition named, never carried in silence; the
claims most exposed to that are C-14 and C-12, which run last, and C-24, which
needs a second billed instance and the M2b cleanup of `second` and
`svc-onboard`. No claim is cut from M3 here, because a cut would be a reschedule
and NOT REACHED is the honest word for one.

**C-09, M3a.** *Test:* pull requests to `edge-config` under the ruleset of
ADR-0021 §10 (required reviewer `security`, one approval, `external/**`). Cases:
the flip, `git mv internal/svc-hello/<route>.yaml external/svc-hello/<route>.yaml`,
without a `security` approval; the same flip merged by the organisation owner
editing the ruleset, read from the ruleset history endpoint and listed as
slipped (ADR-0021 §10); a new file straight into `external/`; a
rename into `external/`; a pull request that edits the workflow that produces the
required check; and the organisation owner editing the ruleset, read from the
ruleset history endpoint. The registered "field flip without the move" has no
case, because there is no field; it becomes C-10's `exposure` case. The ruleset
JSON is recorded as `gh api` returns it, before and after each change, in
`platform-factory/.github`. *Pass line:* no change under `external/` reaches
`main` by any path through the rule without an approval from `security`, and
every case is recorded as blocked or slipped. *Falsified if* any such change
reaches `main` without one, or any C-11 case is admitted by either door, which
is a path around the review (concept#26); either is graded WRONG. The owner-edits-the-ruleset case is
recorded and not scored against the rule, because the owner role is a bypass in
itself. *Ceiling:* **ADJUSTED**. *One person cannot show:* separated duties or
independent review. The run shows that GitHub enforces a path-scoped rule, and a
permitted flip may not be producible at all, since an author cannot approve
their own pull request.

**C-10, M3a.** *Test:* the check and the route chart's closed schema are run
against an adversarial set of route files, each also kept as a must-fail fixture
(ADR-0019 §4): a file with an `exposure` key, in each folder; a `host` outside
`^[a-z][a-z0-9-]{0,30}[a-z0-9]$`; the same `host` in both folders; a `system`
folder that names no tenant file; an unknown key; a non-integer `port`; two
external routes in one System naming the same `service` (ADR-0021 §7). The
check is written against this list before the run. A case that both the schema
and the check accept is an escape, recorded as a failed case; if the check is
then changed, the entry reports the escape, the change and the re-run, as C-07's
entry did. The check becomes a required status
check once it has failed once and passed once. *Pass line:* every case is
refused before merge, by the schema or the check. *Falsified if* any case is
accepted by both. *Ceiling:* **ADJUSTED**, because the registered subject (a
field that can disagree with its folder) no longer exists (ADR-0021 §2). *One
person cannot show:* that a second reviewer would read a failing check. Making it
required shows that GitHub honours it.

**C-11, M3a.** *Test:* each case is created three ways: by a team member's
`kubectl --as=<user> --as-group=<team>@<domain>`, through the service repo's
`k8s/`, and by the operator's own `kubectl`, which no rule exempts, so that a
refusal RBAC makes before admission (likely for the Gateway API kinds, whose
definitions do not aggregate into `admin`) does not stand in for the policy's.
The cluster cases run on the rebuilt cluster with the Gateway API channel on, so
every kind `deny-edge-objects` names exists; a policy naming an absent kind is
not enforced [I]. Cases: a Service of type `LoadBalancer`; one of type `NodePort`; an
`Ingress`; a `Gateway`; an `HTTPRoute`; a `ReferenceGrant`; a `GCPBackendPolicy`;
and, for the second half, a gated cloud kind and a route kind, each with no
System label and with a label naming another namespace. The DNS-annotation case
is not run, because there is no external-dns. Cases predicted admitted: the
platform's own routes and cloud objects. *Pass line:* every violating object is
refused at the door it uses, with the message recorded; for the label cases on
the `kubectl` doors the recorded message names `require-system-label`, since
`deny-edge-objects` or deny-raw would refuse those objects anyway and the label
rule is otherwise untested; and every platform object is admitted. *Falsified if* any violating object is admitted, by either door
(which also makes C-09's "no path around it" false), or a platform-rendered
object is refused. *Ceiling:* **ADJUSTED**, because "denies untagged resources"
is re-scoped to cloud and route kinds (ADR-0021 §9). *One person cannot show:*
what a real team member's RBAC allows beyond `admin`. `--as` shows how the policy
treats an impersonated identity.

**C-12, M3b.** *Test:* a synthetic incident on `svc-hello` (a restart loop) fires
the Cloud Monitoring alerting policy on container restarts per namespace. The
join is alert → namespace → System label on the namespace → tenant file → team
and repo. *Pass line:* each hop is reached by a lookup on a key already in hand,
and the number of hops that needed a hand search or a guess is zero. *Falsified
if* the firing alert does not carry the namespace, or any hop needs a search, a
guess or a label that is not there. "The PRs merged since the last healthy state"
is not tested: it needs a deploy event that does not exist, and moves to the
prod-tag-bump class at M4. *Ceiling:* **PARTIAL**. *One person cannot show:* that
anyone other than the person who built the join can follow it without
archaeology.

**C-13, M3b.** *Test:* an external and an internal route for `svc-hello`. "Both
env zones" means the one environment's public and private zones. Cases: the
external name resolves from outside to the external IP; the wildcard certificate
issues (time recorded) and HTTPS reaches `svc-hello`; a request with a
SQL-injection pattern returns 403 through Cloud Armor; an unknown host gets the
Gateway's 404; the internal name resolves inside the VPC and answers on HTTP 80;
the `git mv` flip moves a route from the internal Gateway to the external one.
*Pass line:* all of these hold. *Falsified if* the certificate does not reach
ACTIVE within the session once its DNS-authorization record resolves from public
resolvers (a certificate still provisioning at the session's end is a case not
run: PARTIAL under ADR-0020 §1), a name does not resolve, the Cloud Armor policy does not attach through the
rendered `GCPBackendPolicy`, or the flip does not change the route's Gateway.
*Data* as registered, with Gateway API conformance findings. *Ceiling:*
**ADJUSTED**: C-13's text says the `exposure:` field selects the policy, zone
and class; under ADR-0021 §2 the folder does. The external route and the flip
are `external/` merges made after C-09's rule is live, merged as ADR-0021 §10
says, and each is also listed in C-09's data. *One person cannot show:* more than one
environment, one route and one service.

**C-14, M3b.** *Test:* the primary-source check is recorded (ADR-0021 §13), then,
on the reference cluster with Dataplane V2 and FQDN policy on, a pod under a
default-deny egress that allows `www.google.com:443` only. It runs in a namespace
that does not carry C-34's allow list, so the refusal can be attributed to the
FQDN rule. DNS churn is read from TTL behaviour and recorded as data. *Pass
line:* the allowed host is reached, `example.com` is refused, and the feature is
available on the cluster's tier. *Falsified if* either reachability check comes
out the other way, or the feature cannot be enabled on the reference cluster.
*Ceiling:* none pre-committed. *One person cannot show:* anything beyond one
allowed and one refused name on one cluster.

**C-24, M3b.** *Test:* as registered, with `svc-onboard` moved to `payments` so
that `svc-hello` and `svc-onboard` share a team, one claim each. Then one moves
to another team while the project policy is polled every ten seconds for fifteen
minutes, and the application still connects through the Auth Proxy under the
condition. The grants are the conditioned pairs of ADR-0021 §14. Before the
connect test the project policy is read from the cloud side and shows no
unconditioned `cloudsql.client` or `cloudsql.instanceUser` grant for the app's
or the team's identity; any found is removed first and the removal recorded
(ADR-0017 §10's cleanup), because with one present the connect test could not
fail. A second billed instance is accepted for the fixture. ADR-0021 §14's
exact-name condition, replacing ADR-0016 §2's prefix before the run, is not by
itself a design change for grading, because C-24's text names no prefix. *Pass
line:* the sibling's grant is absent for 0 seconds (M2 baseline: about 300), the
application connects with only the conditioned grants in the policy, and the
condition as shipped names only the System's own instance.
*Falsified if* the grant is absent for any time, the application cannot connect,
or the condition breaks an API call. *Ceiling:* none pre-committed. *One person
cannot show:* an overmatch between names that extend each other, because that
fixture needs a second instance and is not planned.

**C-34, M3b.** The new claim above. *Ceiling:* none pre-committed. *One person
cannot show:* hostile-tenant isolation, which is not claimed. A team holds `admin`
and can delete or widen its own namespace's policy with `kubectl`, and database
access between tenants is limited by IAM alone.

**C-08.** C-08 stays UNTESTED and is **graded NOT REACHED at M3's close**
(ADR-0020, ADR-0021 §16), naming two preconditions not met: the governance
decision the M2 grading note records as undecided, and a composition layer to
swap, which ADR-0017 removed.

**The Terraform crossings declared for C-01's count** (ADR-0021 §12), before any
M3 command. Layer 0: the APIs `certificatemanager`, `dns` and, if needed,
`networkservices`. Layer 1: the two zones, two wildcard records, the global and
the regional address, the proxy-only subnet, the DNS authorization, certificate,
certificate map and its entry, the Cloud Armor policy and the alerting policy.
Layer 2: `datapath_provider = ADVANCED_DATAPATH`, the Gateway API channel
standard and FQDN network policy. One apply on each persistent layer (0 and 1);
layer 2 rides the rebuild. The Cloudflare delegation and the ruleset changes are
not applies. As with M2's, each is declared before the milestone that needs it.

**Correction before the build, 2026-10-27, after the `m3-preregistered` tag
and before any C-11 case.** ADR-0021 §9 lets the garbage collector through
`deny-edge-objects` on DELETE only. Argo CD prunes in the foreground, and the
garbage collector then removes the `foregroundDeletion` finalizer with an
UPDATE. Refused that UPDATE, every route Argo CD prunes would stay terminating.
That is the failure M2b Session A found in `deny-raw-cloud-resources` on
2026-10-19 (platform-config `5da7d4a`), and under C-11's *Falsified if* it is a
platform object refused. The code review of the M3 pull requests found it
before anything ran. The create-and-update rule therefore exempts
`kube-system/generic-garbage-collector` too (platform-config#40, `add0fd4`),
and `scripts/check_charts.py` now fails if the rule leaves the garbage
collector out or adds the namespace controller. ADR-0021 is not edited. Its §9
is read with this correction, and M3's build log lists it under ADR errata.
C-11's test, pass line, *Falsified if* and ceiling are unchanged. One case is
added to C-11's data: a route file removed from `edge-config`, with the
HTTPRoute gone within one sync and no refusal in Kyverno's log. It tests this
exemption the same way M2b's first team move tested deny-raw's.

## M4 — Factory slice

- **C-15 — A rung is machinery.** (ADR-0007)
  The dependency-bump class's rung is enforced by branch protection +
  CODEOWNERS on its paths alone — no bespoke gatekeeper bot in the enforcement
  path.
  **Test:** the L1→L2 move is a reviewable ruleset diff; after it, agent
  merges succeed on class paths and fail off-path. **Data:** the exact ruleset
  diff; enforcement gaps that needed extra machinery.
- **C-16 — The packet keeps L1 real.** (ADR-0007; `research/approval-packet-2026.md`)
  With an approval packet, human review of agent PRs is fast *and* catches
  real defects.
  **Test:** run ≥20 agent-authored dependency bumps through L1.
  **Data:** median human review minutes; defects caught (near-misses); packet
  size; reviewer-reported usefulness per packet section.
- **C-17 — The scorecard is derivable.** (ADR-0007)
  Per-class scorecard (changes, escapes, near-misses) computes from PR
  metadata + verification results with no manual bookkeeping.
  **Test:** scorecard generated by CI from repo state alone. **Data:** fields
  that required human entry (each one is a design miss).
- **C-18 — Demotion can be automatic.** (ADR-0007)
  Escapes (failed verification, revert for cause, spine-attributed incident)
  are detected mechanically and demote the class without a meeting.
  **Test:** inject each escape type; observe detection and the rung change.
  **Data:** which escape types auto-detect vs. require human attribution — the
  honest boundary of "automatic."
- **C-19 — Autonomy is earnable.** (ADR-0007 — the headline claim)
  The dependency-bump class reaches L2 on an accrued escape-free streak, with
  the displaced approver signing the promotion PR.
  **Test:** run the class from L1 to a real promotion. **Data:** N chosen and
  why; streak history including resets; wall-clock time-to-promotion; the
  promotion PR itself.
- **C-20 — Gates are three-outcome.** (ADR-0007)
  The verifier returns pass / inconclusive→human / fail, and inconclusive
  routes one change to a human without demoting the class.
  **Test:** force each outcome. **Data:** how often "inconclusive" occurs
  naturally; whether the chosen verification tool supports it or it had to be
  built.
- **C-21 — The agent is a governed principal.** (ADR-0006/0007;
  `knowledge-as-code.md`)
  Agents author via PRs only, under their own identity, holding no cluster
  credentials; every action attributes to the principal that took it.
  **Test:** audit-trail walkthrough of one agent change end-to-end; attempt
  (in a controlled way) the actions the design says are impossible.
  **Data:** the identity mechanism chosen (ADR pending); trail gaps.
- **C-22 — Knowledge works as CI.** (`knowledge-as-code.md`; ADR-0006)
  A dependency change flags the dependent doc as a failing check; the question
  bank grades an agent and produces gap tickets; skills are release-pinned and
  revertible.
  **Test:** change a dependency of a prose doc; run the bank; revert a skill
  release. **Data:** staleness false-positive/negative cases; bank scores over
  time.

- **C-25 — The provider bump removes the manual step.** (ADR-0016 §4; added
  2026-09-17 from M2's provider finding)
  On a provider release containing the fix for upstream issue #1000, a fresh
  `Database` claim reaches usable with no out-of-band command — and the bump
  itself arrives as a dependency-bump change of exactly the class M4 builds.
  **Test:** bump `provider-gcp-sql` through the M4 change class; create a
  database claim in a System that has never had one. **Data:** provider
  version; manual commands (target: zero; M2 baseline: one); minutes from
  claim to usable (M2 baseline: 17m45s).

## Out of scope here

The historical anchors in `factory-framing.md` (SAE levels, Phoenix, the 2023
permit suspension, AF447) are facts to verify before the blog publishes, not
build-testable claims; they stay tracked in that doc's final section.
