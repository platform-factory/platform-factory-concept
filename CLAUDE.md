# Platform Factory — project instructions

Public repo (Apache-2.0): an opinionated developer-platform pattern (GitOps
control plane, approval-boundary repo topology, knowledge-as-code layer) with a
GCP reference implementation. This working repo is the design seed; the running
implementation will be a GitHub org of seven repos (see README topology).

## Layout

- `docs/design/` — the core design docs (platform pattern, knowledge layer,
  factory framing/articulation)
- `docs/adr/` — decision records, numbered; ADRs are **superseded, never edited**
- `research/` — primary-source research digests with confidence labels
- `CLAUDE.local.md` — private working context, gitignored; read it at session
  start, never commit it or copy its contents into public files

## Conventions

- **Dogfood knowledge-as-code:** any session that changes the design ends with an
  ADR or design-doc update in the same session. Decisions without an ADR didn't
  happen.
- **Provenance discipline:** no client names, employer names, or internal project
  codenames anywhere in committed content — ever. Provenance is always phrased as
  "patterns run in production at fintech scale." If unsure whether a detail is
  generic, it goes in CLAUDE.local.md, not here.
- **Research standard:** load-bearing claims verified against primary sources,
  labeled [C] confirmed / [I] inferred; unverified items listed explicitly. Don't
  soften this — it's the repo's credibility.
- **Everything public-facing gets written to be read:** plain explanations,
  why → mental model → mechanism, before code or YAML.

## Cloud target

GCP: GKE, Crossplane (provider-upjet-gcp family), Argo CD, Gateway API
(kgateway), Kyverno, ESO + Secret Manager, external-dns, Cloud DNS + Cloudflare
at the public edge. AWS-specific survey material in `research/` is retained and
labeled — the portability story is part of the pattern.
