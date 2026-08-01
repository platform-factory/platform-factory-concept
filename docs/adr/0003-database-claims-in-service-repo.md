# ADR-0003: Database claims live in the service repo

Status: Accepted · July 2026

## Context

Databases are the classic exception argument: "stateful, risky, so centralize
review." But ADR-0001 says the question is who approves, and the answer for a
team's own database is the team.

## Decision

Database claims (namespaced Crossplane XRs) live in the service repo, next to
the Deployment they serve. Teams own their databases. Guardrails replace review:

- The Composition pins engine versions, instance classes, backup policy, and
  deletion protection (`managementPolicies` omitting Delete on stateful
  resources — the external database survives claim deletion).
- Kyverno bounds what a claim may request (sizes, regions, flags).
- Credentials flow via managed secret rotation into the app namespace; no
  credentials in git.

## Consequences

- Provisioning a database is a PR in your own repo on the golden path — minutes,
  not tickets.
- The platform team's leverage moves from reviewing requests to authoring
  Compositions and policies — one-time cost, fleet-wide effect.
- A destructive mistake is bounded by deletion protection and orphan policies,
  not by a human catching it in review.
