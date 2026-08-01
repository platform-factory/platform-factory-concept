# ADR-0002: All routes live in edge-config, split by folder AND field

Status: Accepted · July 2026

## Context

By ADR-0001, internal routes would colocate with services (team approves) and
only external routes would centralize. But the dangerous moment is the
*transition* — an internal route becoming external — and a split home makes that
transition a cross-repo move that's easy to do wrong and hard to gate.

## Decision

ALL routes — internal and external — live in the `edge-config` repo, a
deliberate deviation from colocate-by-default. Two mechanisms, one per audience:

- **Folders route the review:** CODEOWNERS requires security approval on
  `external/**`, a light gate on `internal/**`. The internal→external flip is a
  `git mv` that structurally forces security review.
- **An `exposure:` field on the route claim drives materialization** — WAF
  policy, DNS zone, gateway class — because admission controllers and
  Compositions can't see file paths.
- **CI enforces folder ↔ field consistency** so the two views can't drift.

Kyverno additionally denies edge objects created from app namespaces (the
reality gate behind the intent gate).

## Consequences

- Teams give up colocation for routes; in exchange, exposure changes are
  impossible to make silently.
- One repo shows security the entire edge surface — review and audit in one
  place.

## Alternatives considered

External-only in edge-config (rejected: the flip becomes a cross-repo move with
no structural gate); annotations instead of folders (rejected: CODEOWNERS can't
see YAML contents, only paths).
