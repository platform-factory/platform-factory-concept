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

- **C-23 partial live evidence via NAT logs (Aug 13).** With NAT translation
  logging on (`filter = "ALL"`, see surprise 9), a 20-minute window of
  `resource.type="nat_gateway"` logs contains exactly one destination:
  `140.82.114.3:443` — reverse DNS `lb-140-82-114-3-iad.github.com`, WHOIS
  `OrgName: GitHub, Inc.` Zero connections to any public registry. This is
  the predicted shape (image plane on the Google-API path, one git pinhole),
  and it's the first evidence that doesn't rely on reading image *references*.
  **Deliberately not called conclusive.** Cloud NAT only sees traffic that
  needs NAT, and no pod started during the window, so no image pull would
  have crossed it either way — the absence of registry traffic is currently
  consistent with both "pulls ride Private Google Access" and "nothing
  pulled." Grading C-23 needs a forced pull: start a pod whose image isn't
  cached on its node, confirm it arrives from `*.pkg.dev`, and confirm NAT
  logs stay GitHub-only across the same interval. Recorded here rather than
  held back because the ledger of *what the evidence does not yet cover* is
  the part surprise 8 says gets skipped.
- **Operator IP churn is a recurring manual intervention (Aug 13).** The
  residential IPv4 in `authorized_networks` turned over
  ([redacted-residential-ip-1] → [redacted-residential-ip-2]) and the API server became unreachable.
  Failure mode is a `dial tcp ... i/o timeout`, not an authz error — GKE
  drops unauthorized source IPs rather than refusing them, so it presents as
  a network hang with no hint at the cause. Cost: one tfvars edit plus a
  `2-cluster` apply before any cluster work could happen. Same family as the
  per-session reauth, and both belong in C-02's manual-intervention count.
- **The two credentials expire independently (Aug 13).** `gcloud auth login`
  refreshed the CLI credential while Terraform continued failing on
  `invalid_rapt`, because the provider authenticates through Application
  Default Credentials — a separate token needing its own
  `gcloud auth application-default login`. Sharpens the earlier entry: it
  isn't "two logins per session," it's two independent credentials whose
  expiries don't coincide and where fixing one gives no signal about the
  other.

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
9. **A pre-registered test can require evidence the build never ships.**
   C-23's test reads "inspect NAT logs / VPC flow logs: no egress to public
   registries" — but the built infrastructure emits neither. The subnet
   (`1-network/network.tf`) has no `log_config` block, so VPC flow logs are
   off; Cloud NAT (`2-cluster/nat.tf`) is set to `filter = "ERRORS_ONLY"`,
   which records *failed* connections only. A node successfully pulling from
   a public registry — precisely the thing the claim denies happens — would
   generate no log line at all. The claim isn't wrong and the config isn't
   wrong; they were written independently, and nothing forced them to meet.
   General form worth carrying into the pattern docs: **a claim's test is
   itself a requirement on the build.** Pre-registration (ADR-0008) catches
   vague success criteria, but it doesn't catch un-runnable ones — the
   register should have asked, at write time, "what must exist for this test
   to produce data?" Every remaining claim with an observability-shaped test
   (C-11's deny behavior, C-12's alert labels, C-18's escape detection) is
   worth re-reading with that question now, before its milestone.
10. **A conservative bootstrap default quietly made a claim's target
    unreachable.** `3-argocd`'s `root_app_automated_sync` defaults to
    `false` — sensible while bootstrapping, since it stops a
    half-authored `platform-config` from being applied to a live cluster
    the moment Argo CD starts. But C-02's target is *zero manual
    interventions* on rebuild, and with auto-sync off, every rebuild ends
    with a human clicking Sync. The claim cannot score zero against the
    configuration as shipped, and no amount of scripting fixes that —
    `scripts/cycle.sh` deliberately fails rather than papering over it.
    Same family as surprise 9 (the test is a requirement on the build),
    but the mechanism is worth separating: surprise 9 is a *missing*
    capability, this is a *present and deliberate* setting that happens to
    contradict a claim written elsewhere. Both were invisible until
    something tried to actually run the test. Decision needed before the
    C-02 runs: flip the default to `true` (and treat the bootstrap-safety
    concern as handled by `platform-config` being reviewed, not by Argo CD
    being idle), or amend C-02 to count the sync as an accepted manual
    step. The first is the honest version of the claim; the second is the
    honest version of the config.
11. **The cluster discarded every log it produced, for six days, silently —
    and the reason it went unnoticed is that the *other* half of
    observability was switched on for free.** `0-foundation` enables an
    explicit list of 8 APIs; `logging.googleapis.com` was not among them.
    Meanwhile the cluster runs with
    `loggingService = logging.googleapis.com/kubernetes` and
    SYSTEM_COMPONENTS + WORKLOADS enabled, and the node SA holds
    `logging.logWriter` through `container.defaultNodeServiceAccount` — so
    the config and the permissions were both right, and the writes went
    nowhere anyway. Nothing errored. Found only because C-23's NAT-log test
    tried to read logs that were never being stored.
    The trap is the asymmetry: creating a GKE cluster auto-enables a long
    tail of APIs as dependencies of `container.googleapis.com` — monitoring,
    autoscaling, gkebackup, telemetry, containerfilesystem, and more — and
    `monitoring.googleapis.com` is on that list. Metrics and managed
    Prometheus therefore worked from day one. Logging is *not* on that list.
    Half of observability enabled itself and masked the half that didn't, so
    every casual check ("is monitoring working?") returned yes.
    Two durable lessons. **An explicit allowlist is only as good as the
    review that built it** — the 8-API list was authored by reasoning about
    which layer needs which API, and logging isn't consumed by any Terraform
    resource here, so it never came up; it's consumed by the *running
    platform*, which no layer's code mentions. **And a dependency you get
    for free is a dependency you haven't declared**: monitoring worked by
    accident of GKE's behavior, not by anything in this repo. Both APIs are
    now pinned explicitly in `main.tf`. Worth carrying into ADR-0009's
    corp-real posture, which this quietly violated — a corporate cluster
    with no logs is not corp-real, and the posture checklist should name
    observability alongside private nodes and authorized networks.

## Research verified

- **[C] provider-upjet-gcp kind coverage — C-03's doc half (verified
  2026-08-11** against `crossplane-contrib/provider-upjet-gcp` at tag
  v3.0.0: the shipped CRDs under `package/crds/`, the hand-curated
  `examples/` tree, and `docs/family/Configuration.md`). Every kind the
  Compositions need exists, at **both** scopes — cluster
  (`*.gcp.upbound.io`) and namespaced (`*.gcp.m.upbound.io`): Cloud SQL
  `DatabaseInstance` / `Database` / `User`; storage `Bucket`; cloudplatform
  `ServiceAccount` + `ProjectIAMMember` / `ServiceAccountIAMMember` /
  `BucketIAMMember`; artifact `RegistryRepository`; dns `RecordSet` and
  `ManagedZone`. This closes the research doc's standing watch item
  ("non-AWS providers lagged AWS on namespaced-MR support as of v2.3 —
  verify first"): at v3.0.0 the GCP family ships a `namespaced/` example
  directory for every service group, so the lag is gone. Two caveats worth
  carrying into M2: the Cloud SQL instance kind is **`DatabaseInstance`**,
  not `Instance` as the AWS-shaped matrix implies; and `RecordSet` ships as
  a CRD but appears only under `examples-generated/` (auto-generated from
  the Terraform docs) rather than the uptest-run `examples/` tree — present,
  but the one needed kind with no tested upstream example.
  C-03's remaining half (hands-on create of each kind) genuinely can't run
  until Compositions exist, which is M2 work — the register assigned this
  claim to M1, but only its doc half is an M1-shaped test.
- **[C] The GCP provider family is at v3.0.0 (released 2026-08-04) — a
  breaking major, five days before this check** (verified against the repo's
  release list and the v3.0.0 release notes). It removes all deprecated
  `v1beta1` APIs on the back of an upstream Terraform provider jump
  (v6.47.0 → v7.39.0) and ships a **mandatory Storage Version Migrator**
  that must run as an init container via `DeploymentRuntimeConfig` before
  upgrading from 1.x/2.x — skipping it leaves CRDs inconsistent and
  controllers failing to start. Greenfield here, so the migrator itself is
  moot; the finding is the timing. The research doc's pins (2026-07-28) were
  stale at exactly the moment they were first needed, which is the same
  lesson as the helm-provider-v3 catch, now with a second data point: pin
  the family explicitly in `platform-config` and treat its ~2-month cadence
  as a tracked dependency, not a detail.
- **[C] The Crossplane community package track is ghcr.io content — so the
  one lower-confidence remote repo isn't needed at all (verified 2026-08-11**
  by direct registry probe). `xpkg.crossplane.io` is a pass-through front
  for ghcr.io: it answers with ghcr.io's own bearer challenge
  (`realm="https://ghcr.io/token", service="ghcr.io"`), **accepts a
  ghcr.io-issued token**, and returns the **identical manifest digest**
  (`sha256:a2170b5c…46cfdf5`) for
  `crossplane-contrib/provider-gcp-storage:v3.0.0`. Consequence for
  ADR-0010 and C-23: the community track the research doc recommended is
  already covered by the existing `ghcr-io` remote, so provider packages can
  be referenced as
  `<region>-docker.pkg.dev/<project>/ghcr-io/crossplane-contrib/provider-gcp-*`
  and the `xpkg-upbound-io` repo in `registry.tf` — the only one carrying an
  [I] tag and an explicit "recheck at M2 before Crossplane depends on this"
  note — can be **deleted rather than re-verified**. The dependency flagged
  as riskiest turned out to be an unnecessary one.
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
