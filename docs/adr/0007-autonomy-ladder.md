# ADR-0007: The autonomy ladder — agent autonomy is a promotion process

Status: Accepted · July 2026

## Context

In the factory model, agents author, review, and test changes; humans gate. At
factory throughput the human review queue either stalls the line or degrades to
rubber-stamping — a review rule past a certain volume becomes a suggestion.

Research (`research/approval-packet-2026.md`) established the whitespace: no
published scheme exists anywhere in which agent-authored changes earn merge
autonomy from accrued track record. Shipping agent products use static admin
toggles; mature promotion systems earn trust per-artifact (Kargo Freight) or
per-change-class (Renovate automerge rules), never from an actor's history.
Two of their patterns are worth adopting: trust attaches to *classes of
change*, and mature gates are three-outcome (pass / inconclusive→human /
fail), never binary.

## Decision

**Autonomy is a promotion process, not a setting.** Trust attaches to a
**change class** — a kind of diff (dependency bump, drift fix, internal route,
external route, database claim change, prod tag bump) — never to an agent.

Every class sits on a rung:

- **L0** — human writes it
- **L1** — agent writes, human approves (every class starts here)
- **L2** — agent merges, human notified after
- **L3** — agent merges silently

A rung is enforced machinery, not policy prose: branch protection +
CODEOWNERS on the class's paths. Changing a rung is itself a PR to the
ruleset — versioned, auditable, revertible.

**Promotion** (earned, human-granted):

- A class becomes *promotable* when it clears a track-record bar: N
  consecutive escape-free changes (N set per class in the ruleset, tunable).
- Near-misses block promotion: an L1 human rejection that caught a real
  defect resets the streak — a class cannot graduate while humans are still
  catching real bugs in it.
- The **displaced approver** approves the promotion PR: whoever's review is
  being removed signs its removal — security for `edge-config` classes,
  platform for `systems`/`platform-config` classes, the owning team for its
  own service-repo classes. This is ADR-0001 applied to the ladder itself.

**Demotion** (automatic, no meeting):

- An **escape** is any post-merge wrongness: failed verification against the
  intent's done-criteria, a revert for cause, or an incident attributed to
  the change via the metadata spine. Harm is not required; being wrong is
  enough.
- On escape, the class demotes one rung immediately. A mandatory root-cause
  triage follows: if the cause is class-local (weak criteria), done; if it is
  shared machinery (a skill, a verifier, a Composition), **every class
  depending on that machinery freezes to L1** until the machinery is fixed.
- Individual changes stay three-outcome even at L3: a gate returning
  *inconclusive* routes that one change to a human without demoting the
  class.

The ratchet is deliberately asymmetric: losing autonomy is automatic; gaining
it is a human decision.

## Consequences

- A per-class **scorecard** must exist (changes, escapes, near-misses) —
  derivable from PR metadata plus verification results. The scorecard is the
  factory's trust UI; demotion being visible is what makes promotion
  credible.
- The approval packet (`research/approval-packet-2026.md`) is what keeps L1
  review real rather than ritual — packet quality bounds ladder credibility.
- "No checkable done-criteria, no autonomy": a change class cannot enter L1
  automation until its criteria and verification are defined.
- On demotion, in-flight changes of that class drain back to the human queue.
- ADR-0006's release pinning is the rollback lever when triage blames a
  skill: revert the knowledge release pointer, unfreeze on the fixed tag.
