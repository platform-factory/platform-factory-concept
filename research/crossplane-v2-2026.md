# Crossplane v2 — Design-Support Review

Compiled July 28, 2026 for the reference-platform prototype. Method: three
parallel research passes against primary sources only (docs.crossplane.io, GitHub
releases/repos, Upbound blog + policy docs, Argo CD/ESO/Kyverno/external-dns/AWS
docs), fetched live — no training-data claims. Confidence labels: **[C]**
confirmed from fetched source, **[I]** inferred from confirmed facts.

Ported into this repo 2026-07-29. The provider-coverage verification below ran
against the **AWS provider family**; this repo's reference cloud is GCP
(ADR-0005) — the GCP translation notes at the end are [unverified] until
re-checked at build. Everything about Crossplane core, Argo CD, CI, Kyverno, and
licensing is cloud-neutral and stands as verified.

## Verdict

**Crossplane v2 (current: v2.3.4, GA since Aug 14 2025, CNCF-graduated) natively
supports every element of the prototype design — and v2 is materially *better*
for this design than the v1 model sketched from 2020-era production memory.**
Composing arbitrary Kubernetes resources (ServiceAccounts, AppProjects) is now
first-class, namespaced XRs replace the claim machinery with plain Kubernetes
RBAC, and Argo CD understands Crossplane health out of the box. The licensing
scare around Upbound's providers resolved into a clean two-track model; the
community track is unambiguous and free. No element of the design is blocked.
Several elements change shape — listed below.

## Design-support matrix (verified against the AWS provider family)

| Design element | Verdict | Key facts |
|---|---|---|
| Dev-facing "claim" next to Deployment in app namespace | ✅ **Supported — now called a namespaced XR** | v2 removed claims entirely; XRD `scope: Namespaced` (the new default) makes the XR itself the thing devs create in their namespace. [C] |
| Composition creates ServiceAccount (+ any k8s object) | ✅ **Native in v2** | Headline v2 feature; no provider-kubernetes Object wrapper needed. Namespaced XR composes only into its own namespace (exactly right for SA + Secret). [C] |
| `System` XR → Namespaces, quotas, RoleBindings, Argo AppProject | ✅ **Supported — must be `scope: Cluster`** | Cluster XRs compose cluster-scoped resources *and* namespaced resources in any namespace (AppProject in `argocd` ns). Docs: "Use Cluster scope only for platform-level resources like RBAC or cluster configuration." [C] |
| Pod identity wiring without trust-policy templating | ✅ **Better than designed** (AWS) | `PodIdentityAssociation` exists in provider-aws-eks (namespaced variant confirmed in repo examples). EKS Auto Mode has the Pod Identity agent built in; AWS recommends it over IRSA. [C] — GCP equivalent is Workload Identity Federation; see translation notes. |
| Object storage / IAM / DB / registry / budgets / network-refs coverage | ✅ **Covered (AWS family)** | iam: role, policy, rolepolicy, openidconnectprovider confirmed by namespaced examples; eks: podidentityassociation confirmed; budgets: Budget confirmed (cluster + namespaced); rds: Instance incl. `manageMasterUserPassword` confirmed on both scope variants; s3/ecr/ec2 confirmed by family pattern [I — CI validate catches any gap day one]. |
| App teams create XRs only, never raw cloud resources | ✅ **Simpler in v2** | MRs are namespaced (`*.m.upbound.io` API groups) → plain k8s RBAC: grant create on XR kinds, no grants on MR kinds. Kyverno as belt-and-suspenders. Caveat: composed MRs live in the team's namespace — scope team write permissions by kind, never wildcard. [C mechanics / I recipe] |
| DB credentials to app pods | ✅ **Two documented paths** (AWS) | Chosen there: `manageMasterUserPassword` → cloud secrets manager → ESO ExternalSecret in app namespace (keeps cloud-managed rotation). Alternative: v2 removed XR-level connection secrets; the replacement is composing a Secret explicitly (function-patch-and-transform has built-in support). [C] |
| Orphan/protect stateful resources | ✅ **Supported** | `managementPolicies` (beta, on by default): omit `Delete` → external database survives XR deletion. Must be baked into the composition at authoring time — functions don't run at delete. [C] |
| Argo CD syncs everything; health + ordering | ✅ **Native since Argo v3.1** | Built-in wildcard health checks for `*.crossplane.io` / `*.upbound.io`. Sync waves gate on health → XR in wave -1, Deployment in wave 0 solves the SA-ordering question. Required Argo settings: annotation resource tracking, exclude ProviderConfigUsage, QPS bump. [C] |
| PR-gate CI (render + validate offline) | ✅ **Supported, renamed** | CLI (now separate repo, v2.4.1): `crossplane composition render` (drives the real reconciler since v2.3) and `crossplane resource validate` (offline schema + CEL validation against provider packages). Functions run via Docker in CI. [C] |
| Kyverno on XRs/MRs | ✅ **No v2 friction found** | CRs are ordinary match targets; namespaced MRs *reduce* old scope-mismatch friction. Known interplay is Kyverno-mutate vs Argo drift → `ServerSideDiff=true,IncludeMutationWebhook=true`. [C] |
| external-dns → Cloudflare | ✅ **Supported, active** | v0.21.0; token scopes Zone:Read + DNS:Edit; proxied via annotation; Gateway API route sources at v1. Note new `external-dns.kubernetes.io/` annotation prefix. No Crossplane Cloudflare provider exists in contrib (404) — external-dns is the answer, as designed. [C] |

## The provider licensing story (the thing that could have changed the recommendation — it didn't)

Timeline, all from primary sources:

1. **Nov 2024 → Mar 25 2025** — Upbound announces, then enforces, authentication
   requirements for Official Packages on its marketplace. [C]
2. **Feb 25 2025** — "An Update on Upbound's Official Providers" (with the
   Crossplane Steering Committee): two-track model. **Community track:** source in
   `crossplane-contrib`, Apache 2.0, free monthly releases, published to
   `xpkg.crossplane.io/crossplane-contrib/` (backed by ghcr.io) — vendor-neutral
   registry mandated by new contrib governance. **Commercial track:** "Official
   Providers" = downstream builds of the same source with LTS/backports, SBOM,
   signing, support; subscription required. [C]
3. **Aug 12–19 2025** — Crossplane 2.0 GA; Upbound launches UXP 2.0 and ships
   Official Provider v2.0 builds with a **boot check refusing to start on OSS
   Crossplane** ("only compatible with Upbound Crossplane (UXP)"). Community
   backlash (documented user reports within weeks). Bassam's Aug 19 post confirms
   the policy: Official = UXP-exclusive; community providers "fully supported on
   upstream Crossplane" with "full Crossplane 2.0 features." [C]
4. **Oct 14 2025 → today** — Package Policy v1.0.0 (current, © 2026):
   softened/clarified. Downstream **main** releases: signed + SBOM, **free to
   everyone**, and per the published compatibility matrix "runnable on all
   Crossplane runtimes" (OSS + UXP). What's actually paid: **backport releases**
   (Standard+ subscription + pull secrets), **FIPS artifacts** (Business
   Critical), support SLAs, and note the **12-month availability window** on main
   releases (latest release stays pullable indefinitely). Upstream community
   releases: unlimited availability, unsigned, no SBOM, unit + basic integration
   testing only. [C]
   - Unresolved nit: whether current downstream builds still carry a
     (now-passing) boot check on OSS Crossplane — the written policy says they
     run everywhere; the Aug-2025 builds demonstrably didn't. Doesn't affect our
     path. [I]

**Recommendation:** pull from the `xpkg.crossplane.io/crossplane-contrib/`
community track. (AWS family verified at v2.6.0, released 2026-06-11; steady
cadence since v2.0.0 shipped alongside Crossplane 2.0; Upjet families track the
upstream Terraform provider ~every 2 months. Apache 2.0, no account, no pull
limits, explicitly supported on OSS Crossplane with full v2 features.)

**For regulated production use (PCI lens):** upstream builds are unsigned and
carry no SBOM — a procurement/InfoSec talking point. Mitigations, in order of
preference: (a) pin by digest + generate your own SBOM (syft) + scan
(Grype/Trivy) in the platform pipeline — consistent with a Sigstore/SBOM
supply-chain posture anyway; (b) an Upbound subscription buys signed+SBOM
downstream builds, backports, and a 7-day critical-CVE SLA — a legitimate
procurement line-item; (c) free UXP Community Edition is "100% API compatible"
but couples the runtime to a vendor distribution — hold as option, not default.

## v2 deltas to fold into the prototype (what changes from the sketches)

1. **Vocabulary + API:** stop saying "claims." XRDs on
   `apiextensions.crossplane.io/v2` with `scope: Namespaced` for
   `ObjectStore`/`Database`; `scope: Cluster` for `System`. Dev experience is
   identical to the design (YAML in `k8s/` next to the Deployment).
2. **Machinery moved:** XR plumbing lives under `spec.crossplane.*`; XRD schemas
   must not use `spec.crossplane`, `status.crossplane`, `status.conditions`.
3. **Compositions are function pipelines only** (native patch-and-transform
   removed). Toolkit: function-patch-and-transform v0.10.0, function-go-templating
   v0.11.2, function-auto-ready v0.6.0, function-environment-configs. All refs
   fully qualified — v2 has no default registry.
4. **Per-env values** (network IDs, cluster name): EnvironmentConfigs (beta) +
   function-environment-configs; mock in CI via `--context-values`.
5. **Connection details:** compose the Secret explicitly; for managed databases
   take the managed-master-password → cloud secrets manager → ESO path
   (composition templates the ExternalSecret using the secret ref from MR
   status).
6. **Crossplane's own RBAC:** it can compose "some" k8s kinds out of the box;
   ship an aggregated ClusterRole
   (`rbac.crossplane.io/aggregate-to-crossplane: "true"`) covering
   ServiceAccount, Namespace, ResourceQuota, RoleBinding, AppProject.
   RoleBinding creation is subject to k8s RBAC escalation rules — spike item in
   phase 0.
7. **Version pins:** Crossplane v2.3.4 · Argo CD v3.4.5 (≥3.1 required for
   built-in health; avoid pairing Argo with Crossplane v2.1.x —
   list-reordering sync-loop bug, fixed v2.2) · ESO v2.8.0 · external-dns
   v0.21.0 · CLI v2.4.1. Optional, all alive but likely unneeded given native
   composition: provider-kubernetes v1.2.1, provider-helm v1.3.0,
   provider-argocd v0.14.3.
8. **Ingress reality check (AWS framing, retained):** ingress-nginx retired
   (Mar 2026, SIG-recommended migration to Gateway API). EKS Auto Mode's
   built-in load balancing reconciles Ingress/Service only; Gateway API via
   self-managed AWS LBC v3.4.2 (Gateway support GA Jan 2026) as the forward
   path. GCP shape: GKE's built-in Gateway API controller — see translation
   notes [unverified].
9. **Ops notes:** provider CRD-flood problem addressed in v2 via
   ManagedResourceActivationPolicies (alpha — default activates everything;
   allowlist instead); XRD schema changes need a Crossplane pod restart;
   upgrades one minor at a time; `crossplane.io/paused` blocks deletion
   (finalizer).

## GCP translation notes (added at port, 2026-07-29 — ALL [unverified])

The reference cloud is GCP (ADR-0005). The matrix above transfers as follows;
every row needs the same primary-source verification pass before build:

- Provider family: `provider-upjet-gcp` in crossplane-contrib (same Upjet
  pattern as AWS; namespaced-MR support and per-kind coverage to verify).
- Pod identity: Workload Identity Federation (GKE-native) replaces EKS Pod
  Identity — different mechanism (KSA→GSA annotation binding), likely composed
  from IAM policy bindings rather than a single association kind.
- Kind mapping to verify: GCS bucket ↔ s3; Cloud SQL instance ↔ rds
  (managed-password path differs — Cloud SQL has no direct
  `manageMasterUserPassword` equivalent; secret flow needs its own design);
  Artifact Registry ↔ ecr; project-level budgets ↔ budgets.
- Everything cloud-neutral above (core v2 semantics, Argo integration, CI
  render/validate, Kyverno, licensing) applies unchanged.

## Watch items

- **No v2 multi-tenancy guide exists** — the RBAC lockdown recipe here is
  assembled from confirmed primitives, not a published walkthrough. The
  prototype effectively becomes one; worth an ADR.
- **Default composable-kinds list** for Crossplane's service account is
  undocumented — assume the aggregated ClusterRole is required until tested.
- **v2.0 line is already EOL** (last patch Apr 2026); stay on 2.2+ always.
  v1.20 is the long-support legacy line — irrelevant for greenfield.
- Non-AWS providers lagged AWS on namespaced-MR support as of v2.3 docs —
  directly relevant to the GCP family; verify first.

## Sources (primary)

docs.crossplane.io (latest = v2.3: whats-new, XRD/composition/managed-resources
pages, connection-details + Argo CD + upgrade-to-v2 guides, release-cycle) ·
github.com/crossplane/crossplane releases · github.com/crossplane/cli ·
blog.crossplane.io (CNCF graduation) · upbound.io/blog: "An Update on Upbound's
Official Providers" (2025-02-25), "UXP 2.0 and Crossplane: Clarifying the
Relationship" (2025-08-19) · docs.upbound.io/manuals/packages/policies (v1.0.0,
effective 2025-10-14) · github.com/crossplane-contrib/provider-upjet-aws
(releases to v2.6.0; examples tree incl. namespaced variants; README/Apache-2.0)
· crossplane-contrib releases APIs (provider-kubernetes/helm/argocd) ·
argo-cd.readthedocs.io (health/sync-waves/sync-options/diffing/projects/
declarative-setup) + argoproj/argo-cd resource_customizations ·
external-secrets.io · kyverno.io (match-exclude, platform notes) ·
kubernetes-sigs/external-dns (cloudflare tutorial) ·
kubernetes-sigs/aws-load-balancer-controller (v3.0.0 Gateway GA) ·
docs.aws.amazon.com (EKS Auto Mode + security whitepaper) · kubernetes.io blog
(ingress-nginx retirement) · community reports of the Aug 2025 boot check
(r/devops, 2025-08-29).
