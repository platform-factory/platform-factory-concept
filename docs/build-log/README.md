# Build log

The build phase runs as a pre-registered experiment (ADR-0008): the design
docs made falsifiable claims; the build tests them; this directory is the
evidence trail. Read `claims-register.md` for what was predicted and how each
prediction fared. One entry per milestone records what was built, the data
collected, and the grades — including the misses.

## Method in one paragraph

Before any build command ran, every falsifiable claim in the design docs was
extracted into the claims register with a defined test and the data to
collect. Claims are graded only by build-log entries with evidence: **HELD**
(worked as designed), **ADJUSTED** (worked after a design change, superseding
ADR linked), or **WRONG** (abandoned, ADR published). The register is
append-only; grades cannot be edited without a linked entry. Unverified
research claims get verified against primary sources at the milestone that
touches them. Where a claim's pass criterion needed operationalising, it was
done before that milestone's first build command and recorded, with its
date, in the register — for M2 in the 2026-09-02 readiness walk, and in
ADR-0012 and ADR-0013 on 2026-09-14, which the walk called for. The claim's
wording was never changed.

## Milestones

| # | Name | What it builds | What it proves | Claims | Entry |
|---|------|----------------|----------------|--------|-------|
| M1 | Spine | Org repos, `platform-bootstrap` (Terraform layer 0), GKE, Argo CD app-of-apps from `platform-config` | Terraform's-last-job + GitOps control plane; rebuild cheapness | C-01..C-04, C-23 | [m1-spine.md](m1-spine.md) |
| M2 | Paved road | System XR, `svc-hello` + database claim, Compositions + Kyverno guardrails | Declare intent in your own repo → infrastructure materializes, policy replaces review | C-05..C-08 | [m2-paved-road.md](m2-paved-road.md) |
| M2b | Engine swap (added 2026-09-19, ADR-0017; roles folded in 2026-09-21, ADR-0018; guards added 2026-09-24, ADR-0019) | Config Connector in place of Crossplane; `system` and `claims` Helm charts rendered by platform-owned Applications; the cutover as a rebuild; `platform-roles`, the engine's permissions as four custom roles; guards so that a removed, renamed or broken file deletes no tenant by itself | The paved road's results hold on a smaller engine, the form stays an allowlist without a custom API, what the engine's roles leave out is a deletion lock Google enforces, a rebuild's cost is read from the billing export, not estimated, and a removed, renamed or broken file deletes no tenant by itself | C-26..C-33 | — |
| M3 | Approval boundary | `edge-config` (folders + field + CI), CODEOWNERS, Kyverno reality gates, metadata spine, ESO, external-dns, the Gateway | Repo boundaries can carry the approval model; the spine joins incidents structurally | C-09..C-14, C-24 | — |
| M4 | Factory slice | One change class (dependency bump) end-to-end: done-criteria, approval packet, agent-authored PRs, scorecard, L1→L2 promotion | The autonomy ladder works: a change class earns merge rights from its own track record and loses them automatically when a merged change turns out wrong | C-15..C-22, C-25 | — |

Milestones are vertical slices: each leaves the system in a complete, honest
state and grades its claims before the next begins.

**Claims do not always stay in the milestone they were registered for.** The
Claims column above is the register's assignment, which is deliberately not
rewritten when a claim moves; the Grade column of the scoreboard in
`claims-register.md` records the move instead. C-01 and C-03 were deferred
from M1 and graded at M2's close (2026-09-17). C-08 was registered as an M2
stretch, was not attempted, and is deferred to M3 pending a governance
decision (2026-09-17). C-26 tests C-08's idea on the new engine at M2b; C-08
itself stays deferred. C-25's subject is removed by ADR-0017, and its grade
column is dealt with at M2b's close. C-23 (added 2026-08-06) and C-24 and
C-25 (added 2026-09-17) were registered after the 2026-07-31
pre-registration, as were M2b's C-26..C-33; each is dated in the register.
The scoreboard in `claims-register.md` is the authority in every case.

## Entry template

The four-part layout for ADJUSTED grades was added on 2026-09-23 and applies
from M2b's grading on; the ADJUSTED grades given at M2's close (C-01, C-06,
C-07) are not rewritten to fit it.

```markdown
# M<N>: <name> — <date>

## Built
<what exists now that didn't before; key commits/PRs>

## Data
<the numbers the register asked for: wall-clock, cost, counts, review times>

## Claims graded
<C-NN: HELD/ADJUSTED/WRONG — evidence, one paragraph each; ADR links for
ADJUSTED/WRONG. An ADJUSTED grade keeps four things apart: the prediction as
registered and how it failed; the revised mechanism (superseding ADR); what
the revised mechanism was observed to do; what is still unresolved.>

## Research verified
<[unverified] items resolved this milestone, with primary-source citations>

## Surprises
<what the design didn't anticipate, good or bad — raw material for the next ADR>
```
