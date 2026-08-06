# ADR-0010: The image plane rides the Google-API path; internet egress reduces to git

Status: Accepted · August 2026

## Context

ADR-0009 made the environment corp-real: private nodes, Cloud NAT,
authorized-networks endpoint. But NAT was still carrying general internet
egress, because the platform's images live on public registries — Argo CD on
quay.io, dex on ghcr.io, Crossplane packages on xpkg.upbound.io. A corporate
environment does not let workloads pull from arbitrary public registries;
images come through the organization's artifact plane (a registry
proxy/cache), and egress is a named exception, not a default.

The initial idea was preloading images to a GCS bucket. That fails on
mechanics — image pulls speak the OCI registry protocol, which GCS does not —
but the instinct (the cluster should only need Google APIs) is the right
posture. GCP's native answer is Artifact Registry **remote repositories**:
pull-through caches for upstream public registries, served from `*.pkg.dev`,
which Private Google Access covers without any internet egress.

What no registry trick fixes: Argo CD's source of truth is
`github.com/platform-factory/platform-config`. GitHub is not a Google API.
Mirroring git into GCP (Secure Source Manager; Cloud Source Repositories is
closed to new customers) was considered and rejected for this build — an
instance-priced service and mirror plumbing to eliminate one well-understood
pinhole.

## Decision

- **All cluster image pulls go through Artifact Registry remote
  repositories** (quay.io, ghcr.io, Docker Hub, registry.k8s.io,
  xpkg.upbound.io upstreams), created in Terraform layer 0 alongside the
  project — the registry is platform substrate, needed before anything that
  would pull through it.
- **Argo CD's chart values pin its images to the Artifact Registry paths**;
  the same applies to every addon and Crossplane package installed from
  `platform-config` at M2+.
- **Internet egress reduces to a single named pinhole:** Argo CD's git pulls
  from GitHub, over Cloud NAT. M3's egress-control work (C-14) formalizes
  the pinhole as an FQDN allowlist rather than open NAT.
- Registered as **C-23**: with the cluster up, image traffic is entirely on
  the Google-API path and the only internet egress observed is git.

## Consequences

- The M1 cluster needs the internet for exactly one thing, and everyone can
  name it. "What can this cluster reach?" has a one-line answer — which is
  the property a corp security review actually asks for.
- First pulls populate the remote-repo cache; rebuilt clusters pull warm
  from Google's edge — teardown/rebuild (C-02) gets slightly cheaper and
  less dependent on upstream registry availability.
- A new external registry becomes a deliberate layer-0 change (add a remote
  repo) instead of an invisible new egress dependency.
- Upstream coverage of AR remote repositories is verified against primary
  docs during authoring; any registry it cannot proxy is reported and
  handled explicitly rather than silently falling back to NAT.
