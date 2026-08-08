# M1: Spine — in progress (started 2026-08-01)

> **Status: IN PROGRESS.** This entry accumulates evidence in real time and is
> finalized — claims graded — only when the milestone closes. Nothing below is
> a grade yet.

## Built

- **Org scaffold (Aug 1):** seven public repos live in the
  [`platform-factory`](https://github.com/platform-factory) GitHub org
  (`platform-bootstrap`, `platform-config`, `systems`, `edge-config`,
  `template-service`, `svc-hello`, `platform-knowledge`). Each has a README
  stub, Apache-2.0 LICENSE, CODEOWNERS, and an active `main-protection`
  ruleset (PRs required, force pushes and branch deletion blocked). Org teams
  `platform` and `security` exist; `edge-config`'s CODEOWNERS routes
  `/external/` to `@platform-factory/security` — the approval boundary in
  file form, present from the first commit.
- **Terraform layer 0 (Aug 1–6, in review as PR #1):** `platform-bootstrap`
  authored as four independently-applied layers, split by lifecycle —
  `0-foundation` (project under the org, billing link, 8 APIs, state
  bucket, 6 Artifact Registry remote repos, dedicated `gke-nodes` service
  account), `1-network` (custom VPC, subnet + secondary ranges, gated HA
  VPN pair), `2-cluster` (regional private-node GKE, Cloud NAT,
  authorized-networks endpoint), `3-argocd` (Argo CD 10.2.2 via Helm +
  root Application as its own second `root-app` release — corrected
  mid-build from the `extraObjects` design, see surprise 8 — images
  rerouted through the AR caches). The boundary rule: **identity and reachability persist; compute
  is disposable** — teardown between sessions destroys only layers 2–3
  (this is the shape of the C-02 test, now with the corporate-shaped bill
  per ADR-0009). Seven review-hardened commits; nothing applied yet —
  apply waits on human review of the PR.

## Data

- Org scaffold wall-clock: under two hours for repos + rulesets + teams,
  fully via `gh` CLI; zero cloud cost.
- Build runs on a GCP Free Trial billing account ($300 / 90 days) under a
  fresh identity — trial window comfortably covers the M1–M4 timeline.
- **First apply (Aug 6, after human review of PR #1):** `0-foundation`
  applied clean on the first attempt — all 19 resources, including the six
  Artifact Registry remote repos. Notably the `xpkg.upbound.io` remote
  (the one flagged lower-confidence in `registry.tf`) was accepted by the
  AR API at create time; whether it actually proxies pulls is still the
  M2 check. Local→GCS state migration worked as designed. `1-network`
  applied clean immediately after (3 resources; VPN gated off).
- **First cluster apply hit a GCE stockout (Aug 6→7).** GKE's automatic
  zone selection for the regional cluster included `us-central1-f`, which
  had no `e2-standard-4` capacity; after 35 minutes the cluster landed in
  ERROR with 3 of 4 zones' nodes running. Fix required zero code change:
  the `node_locations` variable (already in the design as an override)
  pinned zones a/b/c via gitignored tfvars, and `terraform apply` with
  the tainted cluster replaced it cleanly. Two general notes: capacity
  stockouts are a fact of life even in a top-3 region on a mainstream
  machine type, and the "parameterize the escape hatch you hope not to
  need" habit is what kept this a tfvars edit instead of a PR.
- **The two browser logins recur per session, not once.** The Workspace
  org's default reauthentication policy expired both the CLI credential
  and ADC overnight (`invalid_rapt`). C-01's "irreducibly manual" list is
  therefore a per-session cost, not a one-time bootstrap cost — worth
  stating precisely when grading C-01.
- **Spine complete (Aug 7).** Cluster replacement (destroy ERROR cluster
  + regional recreate + node pool, zones pinned a/b/c): ~29 min
  end-to-end. `3-argocd`: argo-cd release live in 84s, root-app release
  in 3s; all pods Running; `root` Application Synced/Healthy against
  platform-config. **First live C-23 evidence:** every image on the
  cluster arrived through the Artifact Registry remotes — quay
  (argocd v3.4.6), ghcr (dex v2.45.1), and ECR Public (redis
  8.2.3-alpine) paths all resolve to
  `us-central1-docker.pkg.dev/platform-factory-ref/...` — zero direct
  internet image pulls on the private-node cluster, three distinct
  upstreams proxied.

## Surprises (running list — raw material for grading and the next ADR)

1. **The design said "a GitHub org, because the topology is the point" — and
   the build confirmed the org is load-bearing, not cosmetic.** The initial
   instinct was to scaffold under the existing user account; but user
   accounts cannot have teams, and team-based CODEOWNERS
   (`@platform-factory/security`) is what makes approval routing real rather
   than one username pasted everywhere. The claim C-15 test (rung enforceable
   by branch protection + CODEOWNERS alone) would have been untestable in a
   user-account topology.
2. **Solo-builder deadlock in the ruleset design.** The pattern assumes
   required reviews via CODEOWNERS, but GitHub won't let a PR author approve
   their own PR — with one human, `required_approving_review_count: 1` blocks
   every merge. Resolution: rulesets active from day one but with approval
   count 0; the count tightens when agent-authored PRs begin (M4), which is
   itself a cleaner story — the human reviewer requirement turns on exactly
   when a non-human author appears. This needs to be explicit in the M4 test
   design for C-15/C-16.
3. **The cost-optimal layer layout was not the corporate-real layout.** The
   first cut put the VPC in the torn-down-between-sessions layer, because a
   VPC is free and the cluster needs one. Ronak's correction: "in a real
   corporate environment, the cloud wouldn't live by itself — it would have
   other network connections." The mechanical forcing function agrees: an HA
   VPN gateway gets new Google-side public IPs when recreated, so a VPC that
   dies with the cluster means manually reconfiguring the peer (corp
   firewall, or the home-lab UniFi standing in for it) on every rebuild.
   Resolution: four layers — foundation and network persist; cluster and
   Argo CD are disposable. The boundary rule that fell out is worth keeping
   for the pattern docs: **identity and reachability persist; compute is
   disposable.** Cost yields to realism where they conflict (an idle VPN
   tunnel's ~$35–40/mo is the price of the theory being provable).
4. **The one-off network correction generalized into a standing rule
   (ADR-0009).** After the VPC promotion, the remaining cost shortcuts got
   re-examined and reversed as a set — zonal → regional control plane,
   public → private nodes (+ Cloud NAT + Private Google Access), spot →
   on-demand, unrestricted → authorized-networks API endpoint. Ronak's
   framing (Aug 6): "mock a real env as much as possible over costs —
   configure it like a normal corp environment would look and behave."
   Caught pre-apply, so it's a diff on an open PR; a week later it would
   have been a live migration. Recorded as ADR-0009: cost yields to realism;
   C-02's cheapness claim now gets tested with the corporate-shaped bill.
5. **"Only GCP APIs" was almost right — and the wrong storage idea led to
   the right architecture (ADR-0010, C-23).** Ronak proposed preloading
   Crossplane to a GCS bucket so the cluster needs nothing but Google APIs.
   GCS can't speak the OCI registry protocol, but the posture was correct:
   Artifact Registry remote repositories put every image pull on the
   Google-API path (Private Google Access), and the cluster's internet
   egress collapses to one named pinhole — Argo CD's git pulls from GitHub —
   which M3's FQDN work (C-14) then formalizes. Added C-23 to the register
   (dated, append-only) to grade this at M1.
6. **"Terraform's last job" cuts earlier than the first execution attempt.**
   The first instinct was `gcloud projects create` + billing link by hand,
   then Terraform from the VPC down. Corrected mid-session (Ronak): if
   Terraform ends at layer 0, layer 0 should *begin* at the project. The
   irreducibly manual set is small and worth naming as C-01 evidence:
   interactive browser auth (`gcloud auth login`,
   `gcloud auth application-default login`), free-trial signup, and one
   `terraform init -migrate-state` after the state bucket exists. Everything
   else is code.

7. **A from-nothing runbook step can become a hazard the moment the
   bootstrap succeeds.** `0-foundation` shipped with its GCS backend
   commented out — required for genesis, since the backend's bucket is
   created by the very layer it serves. But post-genesis that same state
   is a trap: a fresh clone running `terraform init` lands silently on
   empty local state, and its next plan proposes re-creating all of live
   foundation. Caught during the first real bootstrap, minutes after the
   apply; fixed by committing the backend enabled and inverting the
   runbook (commenting it out becomes the documented genesis-only
   exception). General form, worth carrying into the pattern docs:
   **bootstrap instructions have a half-life — the correct default flips
   from "assume nothing exists" to "assume everything exists" after
   exactly one successful run.**
8. **The researched workaround had its own apply-only failure mode — the
   evidence hierarchy is validate < plan < apply, and only apply is
   real.** The `extraObjects` design for the root Application was itself
   the fix for a verified plan-time wall (`kubernetes_manifest` can't
   bootstrap CRD-typed objects), and it validated and planned clean — then
   failed on the first real apply, because its premise (Helm installs a
   chart's `crds/` directory before templates) doesn't hold for the
   argo-cd chart, which *templates* its CRDs; Helm validates every
   rendered object against the cluster before applying any of them.
   Fixed as a second Helm release of a tiny in-repo `root-app` chart,
   validated at its own install time (platform-bootstrap PR #3, verified
   live). The durable rule: **a CRD-typed object never belongs in the
   same lifecycle step as the thing that defines its type** — the same
   boundary, one step earlier, that pushes Crossplane XRs into GitOps
   instead of Terraform (C-01).

## Research verified

- (pending) provider-upjet-gcp kind coverage (C-03) — to be verified when
  Compositions are authored.
- **[C] Provider/chart versions (verified 2026-08-01** against the Terraform
  registry API and Artifact Hub API, not memory): `hashicorp/google` latest
  stable 7.42.0; `hashicorp/helm` 3.2.0; `hashicorp/kubernetes` 3.2.1;
  `argo-cd` chart 10.2.2 (app v3.4.6). Notable: **helm provider v3 is a
  breaking change** — the v2 `provider "helm" { kubernetes { ... } }` block
  syntax is gone, replaced by a `kubernetes = { ... }` nested object
  (confirmed against the provider docs at the v3.2.0 tag). The kubernetes
  provider's config block did NOT make the equivalent change across its own
  2.x → 3.x. Exactly the class of drift the "verify against primary sources"
  rule exists for — training-data memory would have produced broken code.
- **[C] `terraform validate` does not evaluate cross-variable `validation`
  blocks — `terraform plan` does.** Confirmed by isolated reproduction during
  layer-0 authoring (a guard requiring `peer_gateway_ip` when `enable_vpn`
  is true fires at plan time but not under bare `validate`). Consequence for
  the method: "validate passed" is weaker evidence than it reads as; CI
  gates built later (C-11, C-22 territory) should not treat validate as
  proof that input guards hold.
- **[C] Artifact Registry remote-repo upstream coverage (verified
  2026-08-06** against the AR product docs and provider v7.42.0 docs):
  Docker Hub is the only preset; ghcr.io, registry-1.docker.io,
  public.ecr.aws, and registry.k8s.io are documented custom Docker
  upstreams; quay.io works (live example in terraform-provider-google
  issue #20278). Provider note: `docker_repository.custom_repository` is
  deprecated in v7.42 in favor of `common_repository` — another
  training-data trap avoided by checking. **[I] xpkg.upbound.io**: speaks
  the Docker Registry V2 protocol (direct probe) but has no documented or
  observed use as an AR remote — flagged in-code, re-verify at M2 before
  Crossplane depends on it.
- **[C] argo-cd chart 10.2.2 pulls redis from AWS's ECR Public Gallery**
  (`ecr-public.aws.com`, an alias of `public.ecr.aws`), not Docker Hub as
  assumed. Found by reconciling the chart's actual values at the pinned tag
  against the remote-repo list — the kind of silent egress dependency the
  ADR-0010 posture exists to surface. A sixth remote repo covers it.
- **[C] GKE node-SA guidance consolidated (verified 2026-08-06** against
  Google's current node-service-accounts and cluster-hardening docs): the
  long-standing four-granular-role pattern (logging.logWriter,
  monitoring.metricWriter, monitoring.viewer,
  stackdriver.resourceMetadata.writer) is superseded by one predefined
  role, `roles/container.defaultNodeServiceAccount`. The old pattern is
  still what blogs — and this build's own first spec — repeat; the fourth
  legacy role no longer appears in current docs at all. Also verified: orgs
  created after 2024-05-03 disable automatic IAM grants for default
  service accounts, which is what makes a dedicated node SA a functional
  requirement here, not just hygiene.
- **[C] `kubernetes_manifest` cannot bootstrap a CRD-typed object** onto a
  cluster that doesn't have the CRD yet (schema validation runs at plan
  time). The root Argo CD Application therefore rides in via the chart's
  `extraObjects` value, landing in the same Helm release that installs the
  CRDs. Relevant beyond M1: any Terraform-side creation of Crossplane XRs
  would hit the same wall — one more reason claims/XRs belong in GitOps, not
  Terraform (supports C-01's boundary).
  **Apply-time correction (2026-08-07): the second half of this entry was
  wrong.** The `kubernetes_manifest` limitation is real, but the
  `extraObjects` workaround fails too, one step later: the argo-cd chart
  templates its CRDs rather than shipping them in Helm's special `crds/`
  directory, and Helm validates all rendered objects against the cluster
  API before applying any — so the same release can't carry both the CRD
  and an instance of it. Working fix: the root Application as its own
  second Helm release (in-repo `root-app` chart, `depends_on` the argo-cd
  release), verified live. Kept uncorrected above deliberately — this
  entry carried a [C] tag and still broke, which is exactly the
  validate-vs-apply evidence gap (surprise 8) this log exists to record.

## Claims graded

None yet — grading happens at milestone close per ADR-0008.
