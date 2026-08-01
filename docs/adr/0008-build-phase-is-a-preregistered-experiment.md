# ADR-0008: The build phase runs as a pre-registered experiment

Status: Accepted · July 2026

## Context

The design phase is complete (ADR-0001..0007). Every claim in the design docs
is currently an assertion: the platform layer is a GCP translation of patterns
proven elsewhere, and the factory layer (the autonomy ladder) has never run
anywhere — the research found no published working example. The build's job is
not just to produce a working reference implementation; it is to find out how
accurate the design was, and to be trustworthy when it reports the answer.

Build-in-public efforts routinely fail that trustworthiness test in three ways:
narratives written after the fact (survivorship), design drift that never gets
recorded (the doc silently comes to match what got built), and negative results
quietly omitted. The repo's research standard — primary sources, confidence
labels, unverified items listed explicitly — applies to our own claims too.

## Decision

**Pre-register the claims, then grade them with evidence.**

- **Claims register before build.** Every falsifiable claim the design docs
  make is extracted into `docs/build-log/claims-register.md` *before* the
  first build command runs, each with: the claim, its source doc, the
  milestone that tests it, the test, and the data to collect.
- **Register entries are append-only.** New claims may be added (dated) as
  design evolves; existing claims are never reworded or deleted — the same
  discipline as "ADRs are superseded, never edited."
- **Grades are earned, not asserted.** A claim moves from UNTESTED only via a
  build-log entry with evidence: **HELD** (worked as designed), **ADJUSTED**
  (worked after a design change — superseding ADR required, linked), or
  **WRONG** (abandoned — ADR required, published, not buried).
- **Every milestone ends with a build-log entry** in `docs/build-log/`:
  what was built, the data collected, claims graded, ADRs spawned, and
  surprises — including cost and wall-clock numbers, since cheapness-to-rebuild
  is itself a registered claim.
- **Unverified research claims get verified at the milestone that touches
  them**, against primary sources plus hands-on results, and the research
  digests are updated from [unverified] to [C]/[I] with citations.

## Consequences

- `docs/build-log/` becomes a first-class part of the repo: the method README,
  the claims register, and one entry per milestone.
- The blog series draws from the build log rather than being written fresh —
  the public narrative and the evidence trail are the same artifact.
- WRONG gradings are a deliverable, not an embarrassment: a reference
  implementation that reports what its own design got wrong is the credibility
  claim of the whole repo, applied to itself.
- The register is the factory's acceptance contract applied one level up:
  done-criteria for the build itself, written before the work — the same
  "no checkable done-criteria, no autonomy" rule ADR-0007 imposes on change
  classes.
