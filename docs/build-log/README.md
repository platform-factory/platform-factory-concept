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
touches them.

## Milestones

| # | Name | What it builds | What it proves | Claims |
|---|------|----------------|----------------|--------|
| M1 | Spine | Org repos, `platform-bootstrap` (Terraform layer 0), GKE, Argo CD app-of-apps from `platform-config` | Terraform's-last-job + GitOps control plane; rebuild cheapness | C-01..C-04 |
| M2 | Paved road | System XR, `svc-hello` + database claim, Compositions + Kyverno guardrails | Declare intent in your own repo → infrastructure materializes, policy replaces review | C-05..C-08 |
| M3 | Approval boundary | `edge-config` (folders + field + CI), CODEOWNERS, Kyverno reality gates, metadata spine, DNS/edge | Repo boundaries can carry the approval model; the spine joins incidents structurally | C-09..C-14 |
| M4 | Factory slice | One change class (dependency bump) end-to-end: done-criteria, approval packet, agent-authored PRs, scorecard, L1→L2 promotion | The autonomy ladder works — the novel claim nobody has published a working example of | C-15..C-22 |

Milestones are vertical slices: each leaves the system in a complete, honest
state and grades its claims before the next begins.

## Entry template

```markdown
# M<N>: <name> — <date>

## Built
<what exists now that didn't before; key commits/PRs>

## Data
<the numbers the register asked for: wall-clock, cost, counts, review times>

## Claims graded
<C-NN: HELD/ADJUSTED/WRONG — evidence, one paragraph each; ADR links for
ADJUSTED/WRONG>

## Research verified
<[unverified] items resolved this milestone, with primary-source citations>

## Surprises
<what the design didn't anticipate, good or bad — raw material for the next ADR>
```
