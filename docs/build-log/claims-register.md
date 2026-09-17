# Claims register

Pre-registered 2026-07-31, before the first build command (ADR-0008). Each
claim is a falsifiable prediction made by the design docs, with the test that
grades it and the data to collect. Append-only: claims are added with a date,
never reworded or deleted. Grades change only via a linked build-log entry.

Grades: **UNTESTED** → **HELD** | **ADJUSTED** (superseding ADR linked) |
**WRONG** (ADR linked).

## Scoreboard

| ID | Claim (short) | Milestone | Grade |
|----|---------------|-----------|-------|
| C-01 | Terraform ends at layer 0 | M1 | **ADJUSTED** (2026-09-17, ADR-0016 §7) — deferred from M1 |
| C-02 | Tear-down/rebuild is cheap enough to run between sessions | M1 | **HELD** (2026-08-28) |
| C-03 | provider-upjet-gcp covers the kinds the Compositions need | M1 | **HELD** for the kinds M2 composes, one recorded gap (2026-09-17) — deferred from M1 |
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
