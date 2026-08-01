# ADR-0001: The repo boundary is the approval boundary

Status: Accepted · July 2026

## Context

A platform's repo topology usually accretes by habit — a monorepo because it's
easy, or a repo per tool because it mirrors the org chart. Both couple unrelated
approval flows: a team's Deployment change and a security-gated external route
end up in the same review queue, so either everything is slow or gates get
rubber-stamped.

## Decision

Colocate by default: everything a team ships lives in the team's repo. A thing
moves to a central repo **only when its approver is someone other than the team
shipping it.** This one rule generates the topology:

- Service code, k8s manifests, database claims → service repo (team approves;
  schema + admission policy enforce guardrails).
- Routes/DNS/WAF → `edge-config` (security approves `external/`).
- Tenant definitions → `systems` (platform team approves).
- Knowledge → `platform-knowledge` (many contributors, light review, zero
  deploy blast radius).

Intent gates (CODEOWNERS) are always paired with reality gates (admission
policy): a review rule without an enforcement rule is a suggestion.

## Consequences

- Three tiers of change with three approval speeds; most changes never leave
  the team's own queue.
- The topology is explainable in one sentence, which is what makes it
  enforceable.
- Deviations must be deliberate and recorded (see ADR-0002, which deviates on
  purpose).
