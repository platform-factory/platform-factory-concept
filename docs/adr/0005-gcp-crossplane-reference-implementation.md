# ADR-0005: GCP + Crossplane for the public reference implementation

Status: Accepted · July 2026

## Context

The pattern (ADR-0001..0004) is cloud-agnostic; a reference implementation has
to pick a home. The author's deepest production experience with this pattern is
on GCP/GKE. The repo is public and Apache-2.0 licensed, and the portability of
the pattern is itself a claim worth demonstrating rather than asserting.

## Decision

- **GCP is the reference cloud:** GKE, Cloud DNS, Artifact Registry, Cloud SQL,
  Secret Manager, Cloud Armor; GCP project per environment as the tenant
  isolation unit; Workload Identity Federation for pod → cloud identity.
- **Crossplane (provider-upjet-gcp family) is the composition layer**, chosen
  over the GCP-only Config Connector precisely because the provider family is
  the swappable part — the same XRDs/Compositions structure targets any cloud's
  provider family, which makes the portability story concrete.
- **Public from the start, Apache-2.0** (the license used by the core stack —
  Argo CD, Crossplane, Kyverno — and it carries a patent grant).

## Consequences

- Cross-cloud survey research (e.g., the egress-control and gateway digests,
  which examine AWS primitives in depth) stays in the repo, labeled — comparing
  primitives across clouds is part of the reference value.
- GCP-specific claims inherited from AWS-verified research are marked
  unverified until re-checked against primary sources at build.
- Config Connector remains a documented alternative for GCP-only shops, not the
  reference path.
