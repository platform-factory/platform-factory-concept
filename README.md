# Platform Factory

An opinionated developer-platform pattern — GitOps control plane, approval-boundary
repo topology, and a knowledge-as-code layer — with a **GCP reference
implementation** built on GKE, Argo CD, Crossplane, Gateway API, and Kyverno.

> **Status: design phase.** Design docs and research are in; the org scaffold and
> live cluster come next. Follow along — this repo is being built in public.

## The pattern in five sentences

1. **Git is the source of truth for everything; Kubernetes is the control plane.**
   Terraform's only job is to bootstrap what the control plane can't create yet.
2. **The repo boundary is the approval boundary.** Everything a team ships lives in
   the team's repo by default; something moves to a central repo only when its
   *approver* is someone other than the team shipping it.
3. **Developers declare intent as namespaced resources next to their Deployment**
   (Crossplane XRs for databases, buckets, identities); Compositions materialize
   the cloud resources, and admission policy — not human review — enforces the
   guardrails.
4. **The edge is config, not tickets.** All routes and DNS live in one
   `edge-config` repo where folder structure routes the review (security approves
   `external/`) and a schema field drives what gets materialized.
5. **Knowledge is treated exactly like code** — born via PR, owned via CODEOWNERS,
   flagged stale by CI when its dependencies change, and regression-tested by a
   question bank that agents are graded against.

## Planned repo topology

This repo is the design seed. The reference implementation will be a GitHub org of
seven repos, because the topology itself is the point:

```
├─ platform-bootstrap    Terraform layer 0: project, VPC, GKE, workload identity,
│                        Argo CD bootstrap — Terraform's last job
├─ platform-config       Argo root (app-of-apps): Crossplane, Kyverno, ESO,
│                        external-dns, gateway + XRDs/Compositions (golden paths)
├─ systems               one YAML per tenant (System XR)        [platform approves]
├─ edge-config           ALL routes/DNS/WAF                     [security approves external/]
├─ template-service      "create repo from template" golden path
├─ svc-hello             canonical service: app code + k8s/ (Deployment + XRs)
└─ platform-knowledge    skills/, adr/, standards/, questions/  [light review, no deploy risk]
```

Three tiers of change, three approval speeds: service-repo `k8s/` (team approves;
schema + admission policy enforce) → `edge-config` (security via CODEOWNERS) →
`systems` (platform approves).

## Why GCP, and why it still counts as cloud-agnostic

The pattern is the deliverable; the cloud is an implementation detail. The
reference implementation runs on GKE with Crossplane's GCP provider family —
swapping the provider family is the portability story, and the research here
deliberately covers both clouds' primitives. See
[ADR-0005](docs/adr/0005-gcp-crossplane-reference-implementation.md).

## What's in this repo now

- `docs/design/` — the platform pattern, the knowledge-as-code layer, and the
  factory framing (the personas it serves + the autonomy narrative), in full
- `docs/adr/` — decision records (the knowledge layer, dogfooded from day one)
- `research/` — primary-source research digests: API gateway landscape 2026,
  Crossplane v2 review, Kubernetes egress control 2026

## Provenance

These are patterns I've designed and run in production at fintech scale —
multi-year GKE operation, PR-gated system provisioning, one-repo-per-service
GitOps, and a curated knowledge layer. This repo is the from-scratch, generic
reference implementation of those patterns: no client or employer code, config,
or data — pattern only.

## License

Apache-2.0.
