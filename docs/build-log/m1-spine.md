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
- **C-23 conclusive: forced pull, zero registry egress (Aug 13).** The gap
  named in the entry above is now closed. A throwaway pod pulled
  `busybox:1.36` through the `docker-hub` Artifact Registry remote —
  chosen precisely because `registry.tf`'s own comment records that
  nothing this repo installs pulls from it, so neither the node nor the
  remote had ever touched that path — with `imagePullPolicy: Always` to
  force a registry round-trip rather than a cache hit.
  The pull was real, not elided: kubelet reported **2.909s and 2,217,006
  bytes**, and the container status resolved to a `pkg.dev` digest
  (`us-central1-docker.pkg.dev/platform-factory-ref/docker-hub/library/busybox@sha256:73aaf090…`).
  Across the entire window, Cloud NAT logged **exactly one connection**:
  `140.82.113.3:443` — `lb-140-82-113-3-iad.github.com`, `OrgName: GitHub,
  Inc.` Zero connections to Docker Hub or any other registry.
  This is the strong form of the claim, and it demonstrates the mechanism
  rather than just the outcome: because this was the `docker-hub` remote's
  first ever use, Artifact Registry itself had to fetch the layers from
  Docker Hub upstream — and that fetch generated no node-side egress,
  because it happens on Google's side of the boundary. A private-node
  cluster with no internet route to any registry pulled a public image
  successfully. Four upstreams now proxied in practice (quay, ghcr,
  ECR Public, Docker Hub). C-23 is ready to grade at milestone close.
- **Operator IP churn is a recurring manual intervention (Aug 13).** The
  residential IPv4 in `authorized_networks` turned over
  (one residential address to another; the specific values are redacted — this repo is public and they geolocate) and the API server became unreachable.
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

- **C-02 first scripted cycle (Aug 18).** `scripts/cycle.sh cycle 1`, the
  first end-to-end run of the harness. Down: `3-argocd` 65s, `2-cluster`
  486s, **total 9m13**. Up, after the failure below was fixed: `2-cluster`
  753s, `3-argocd` 87s, verify 12s — **~14m12 if it had run clean**. The
  `2-cluster` rebuild at 12m33 is the first clean baseline; the ~29 min
  recorded on Aug 7 included destroying a cluster stuck in ERROR.
  **Manual interventions: 1** (the IP fix in surprise 12; a second stumble,
  a shell left in the wrong directory, was operator error rather than a
  property of the system). C-02's target is zero, so the first cycle does
  not meet it.
  The verify step passed in **12 seconds** — the root Application reached
  Synced/Healthy with nobody clicking Sync, which is surprise 10's flip
  working as intended and the first evidence that the rebuild can finish
  unattended at all.

- **Control-plane access moved off the IP allowlist (Aug 20).** Three
  applies, deliberately sequenced so a failure could not lock the operator
  out. (1) `dns_endpoint_config.allow_external_traffic = false -> true` —
  in-place, nodes untouched; the DNS endpoint already existed, only external
  traffic was off. (2) `3-argocd`'s kubernetes/helm providers repointed from
  `cluster_endpoint` to the new `cluster_dns_endpoint`, dropping
  `cluster_ca_certificate` because `*.gke.goog` presents a publicly trusted
  certificate rather than the cluster's own CA. (3)
  `ip_endpoints_config.enabled = true -> false` plus deletion of
  `master_authorized_networks_config` and the `authorized_networks` variable.
  Evidence at each stage: the FQDN resolves through public DNS (`dig @8.8.8.8`
  -> `216.239.32.27`, a Google address — it is a public name, not an internal
  one); `kubectl get nodes` returned three Ready nodes over it; `3-argocd`
  planned `No changes` while refreshing **both** `helm_release` resources,
  which is the check that matters after surprise 12 — a plan proposing to
  *create* argo-cd would have meant the provider silently could not see the
  cluster. All three still held after the IP endpoint was disabled, which is
  what makes it conclusive rather than suggestive: the allowlist was never
  carrying that traffic. `2-cluster/terraform.tfvars` is now down to
  `node_locations` and contains no address at all. **The recurring failure
  from surprises 12 and the Aug 13 entry is deleted, not mitigated.**

- **Credential expiry, third direction (Aug 20).** Extends the two earlier
  entries. Terraform failed on ADC (`invalid_rapt`); after
  `gcloud auth application-default login` fixed that, `gcloud` itself failed
  separately with `Reauthentication failed`; and after `gcloud auth login`
  fixed *that*, the CLI came back pointed at a different active account and
  project than the cluster lives in, producing a 403 on
  `container.clusters.get` that reads like a permissions problem and is
  actually a context problem. Three distinct failures, three different error
  surfaces, none of which indicates anything about the other two. A `cycle.sh`
  preflight that checks only the credential that broke last time will keep
  being surprised.

- **Jump box, and Cloud NAT moved to the persist layer (Aug 20).**
  `1-network` gained six resources: a dedicated service account, an
  IAP-only SSH firewall rule, an e2-micro jump box with no external IP, and
  Cloud NAT relocated out of `2-cluster`. Ordering was forced — two NAT
  gateways cannot cover the same subnet ranges, so `2-cluster` had to destroy
  its gateway (`0 to add, 0 to change, 2 to destroy`) before `1-network` could
  create one. Argo CD's git traffic was the only casualty of the gap and it
  reconnected on its own; the root Application was `Synced/Healthy` on the
  next check.
  Verified over IAP with no public IP anywhere on the box:
  `gcloud compute ssh --tunnel-through-iap` returned a shell, `tailscale
  1.102.3` was installed, `net.ipv4.ip_forward = 1` was live and persisted to
  `/etc/sysctl.d/`, and `tailscale status` reported `Logged out` — ready for
  the one-time join. That apt reached the Debian repos at all is the
  independent proof the relocated NAT works.
  One latent bug found and fixed rather than left: nothing declared that the
  VM needs NAT, so Terraform created them concurrently and the startup
  script's one network-dependent step won the race by seconds. Now
  `depends_on` the gateway, with a retry loop around the install so a cold NAT
  degrades into a delay instead of a jump box that silently has no Tailscale.

- **Tailnet up; subnet routing measured, not assumed (Aug 22).** The jump box
  joined the tailnet and advertised all three ranges; routes approved in the
  admin console. Two nodes on the tailnet — the operator's laptop
  (`100.126.144.78`) and the jump box (`100.91.151.61`). Reachability tested
  from the laptop, over the tailnet, to private VPC addresses with no VPN, no
  public IP, and no allowlist anywhere in the path:

  | Advertised range | Target | Result |
  | --- | --- | --- |
  | `10.10.0.0/20` (nodes) | `10.10.0.24` jump box | reachable, ~52 ms |
  | `10.10.0.0/20` (nodes) | `10.10.0.21` GKE node | reachable, ~52 ms |
  | `10.20.0.0/16` (pods) | `10.20.1.6` Argo CD pod | reachable, ~55 ms |
  | `10.30.0.0/20` (Services) | `10.30.4.27` argocd-server | **unreachable** |

  Two of three ranges do what the design intended. The third never could —
  see surprise 16.

  Tailnet Lock was evaluated and deferred rather than enabled (ADR-0011): it
  needs at least two signing nodes, this tailnet has exactly two, one of them
  is a VM Terraform can replace, and initialization emits ten
  once-only disablement secrets whose total loss is unrecoverable. Real
  lockout risk against a threat model this build does not have.

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
12. **A "successful" teardown that tore nothing down — the Helm provider
    treats an unreachable cluster as an absent release.** The first
    `cycle.sh` run destroyed `3-argocd` in 65s and reported
    `Destroy complete! Resources: 0 destroyed`, exit 0. What actually
    happened: the operator's residential IP had cycled back to a
    previously-held address, so the workstation was outside
    `authorized_networks` and the kubernetes/helm providers could not reach
    the API server. Rather than failing, the provider refreshed both
    `helm_release` resources, concluded they no longer existed, and dropped
    them from state — leaving a plan containing nothing but output changes,
    annotated with Terraform's own cheerful "without changing any real
    infrastructure."
    It caused no damage here only by luck of ordering: `2-cluster`'s destroy
    removed the whole cluster eight minutes later. **A `3-argocd`-only
    teardown would have left Argo CD running while deleting Terraform's
    record that it exists** — state and world silently diverging, in the
    direction where Terraform believes less exists than actually does.
    Two lessons. The narrow one: unreachable is not absent, and any layer
    whose providers authenticate *through* a resource in another layer can
    fail this way — `terraform destroy` succeeding is not evidence anything
    was destroyed. The broad one belongs with surprise 8's evidence
    hierarchy: **exit 0 is not a result.** The same failure mode also hid a
    diagnosis in this session — an earlier `terraform plan` on this layer
    "succeeded" for exactly the same reason, and was misread as proof the
    cluster was reachable.
    Third occurrence of the dynamic-IP problem in ten days, second to block
    work outright. The structural fix is not tfvars discipline but
    `1-network`'s VPN (`enable_vpn`, still false): a corp environment
    authorizes a stable office/VPN egress CIDR, and pinning a dynamic
    residential /32 is an artifact of building from home without the tunnel
    ADR-0009's posture already calls for.
    **Superseded on Aug 20 (ADR-0011).** The fix named here — finish the VPN —
    was wrong, and wrong in a way the build log had already recorded three
    times without anyone checking the premise. See surprise 13.

13. **The fix this log named three times could never have worked.** Aug 13,
    Aug 18, and the closing note on surprise 12 all concluded the structural
    answer to residential IP churn was `1-network`'s VPN. Checking the primary
    source before building it: **Cloud VPN requires a static external IP on
    the peer gateway.** A dynamic residential lease cannot supply one, so the
    VPN does not solve the dynamic-IP problem — it inherits it, and enlarges
    the blast radius, because a changed address breaks the
    `external_vpn_gateway` resource and the entire private path instead of one
    allowlist line. HA VPN also requires BGP on the peer device, and an idle
    tunnel meters at ~$35–40/month.
    What made the error durable is that the conclusion was *plausible* and
    kept getting reinforced by recurrence: each new outage felt like more
    evidence for the remedy, when it was only more evidence for the problem.
    Three log entries asserted the fix; none tested it. This is surprise 8's
    evidence hierarchy pointed at the log's own reasoning rather than at a
    command's exit code — **a remedy repeated is not a remedy verified.**
    The deeper miss was a requirements error underneath the technical one. The
    stated need is for *users* to reach internal apps, SSH, and databases; a
    site-to-site VPN serves *locations*, and would have granted access to
    exactly one building while every remote developer still needed something
    else. The design had been carrying an unexamined 2010s assumption —
    that reachability is a network property — since ADR-0009.

14. **Disabling the IP endpoint leaves a live-looking address behind.** After
    `ip_endpoints_config.enabled = false`, the provider's `endpoint` attribute
    still reports the old control-plane IP (`34.55.89.82`), and `terraform
    output` still hands it to any consumer that asks. Nothing answers there:
    `curl` to it times out (exit 28) rather than being refused — the same
    silent-drop behaviour the authorized-network allowlist produced, and the
    same reason the Aug 13 outage read as a network hang instead of a config
    fact. A stale value that *looks* current is worse than an absent one,
    because it sends the next reader debugging connectivity for a problem that
    is really a configuration decision. The `cluster_endpoint` output was
    removed rather than documented in place. Same family as surprise 12:
    the dangerous failures here are the ones that return a confident,
    well-formed, wrong answer.

15. **Closing one door re-armed another, and the config still said "open."**
    Disabling the IP endpoints did not just change the field it was set on —
    GKE also flipped `private_cluster_config.enable_private_endpoint` from
    false to true, because with no public IP endpoint left that is what the
    older field now means. The Terraform config still carried the original
    `false`, so the very next plan read `enable_private_endpoint = true ->
    false`: an innocuous-looking one-line diff whose apply would have
    **reopened the public IP endpoint** and silently undone the whole posture
    change made an hour earlier.
    It surfaced only because an unrelated change — moving Cloud NAT to
    `1-network` — forced a plan on that layer and the diff was read rather
    than skimmed. Nothing about the original apply hinted the field existed;
    the security control and the field that drifted are in different blocks of
    the resource.
    The general shape is worth naming: **a hardening change can leave the
    config asserting the pre-hardened value for a neighbouring field, so the
    next routine apply quietly reverts it.** The window between the two is
    invisible, because the drift lives in the provider's view of the world
    rather than in anything the operator wrote. Related to surprises 12 and
    14 — all three are cases where the tooling gave a confident, well-formed
    answer that pointed the wrong way — but this one is worse than those,
    because the wrong answer would have been applied on purpose by someone
    reading a clean plan.

16. **A range can be reserved, advertised, accepted — and still be
    unreachable by construction.** `local.private_ranges` advertises the
    subnet primary range plus both GKE secondary ranges, and its comment
    promised "pod and Service IPs are reachable from the peer network too,
    not just node IPs." Measured over the tailnet: nodes reachable, pods
    reachable, **Service ClusterIPs not**. Nothing is misconfigured. ClusterIPs
    are virtual — kube-proxy rules on each node translate them, and they are
    never real addresses on the VPC network — so routing cannot reach them
    from outside no matter what is advertised.
    What makes this worth recording is how convincing the wrong version
    looked from the GCP side. The Services range is a real
    `secondary_ip_range` on a real subnet, it appears in the VPC IP plan
    alongside the pods range that *does* work, and both are configured
    identically in `ip_allocation_policy`. Nothing at the Terraform or GCP
    layer distinguishes the routable range from the one that only reserves
    addresses; the difference lives entirely in Kubernetes' implementation of
    Services, one abstraction above where the route is written.
    The comment had been wrong since it was written for the VPN, and survived
    because the VPN was never built — the same mechanism as surprise 13, where
    an assertion persisted precisely because nothing exercised it. Two of
    these in one milestone is a pattern rather than a coincidence: **this
    codebase's comments have been carrying untested capability claims, and
    they read exactly like tested ones.** The claims register is graded
    against evidence; comments are not, and that gap is where both errors
    lived.
    Reaching a Service by name from the tailnet is what the in-cluster
    Tailscale operator's Ingress is for — which is the concrete job for the
    operator that ADR-0011 kept as "additive, not a replacement" after moving
    the subnet router out to the jump box.

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

- **Cloud VPN requires a static peer IP — the premise check that killed the
  VPN plan (verified 2026-08-19** against Google's Cloud VPN overview). The
  documentation is unambiguous: the peer VPN gateway must have a static
  external, internet-routable IP address, and that address is required to
  configure Cloud VPN at all. This log had asserted three times that the VPN
  was the structural fix for residential IP churn; the assertion had never
  been checked against the requirement. HA VPN additionally requires BGP on
  the peer device. See surprise 13 — the interesting finding is not the
  requirement, it is that repetition had been substituting for verification.

- **GKE DNS-based control-plane endpoint (verified 2026-08-20** against
  Google's "About network isolation in GKE"). Google states the best practice
  directly: use only the DNS-based endpoint for control-plane access. It is a
  unique immutable FQDN per cluster, resolves to an endpoint reachable from
  any network that can reach Google Cloud APIs — on-premises and other clouds
  included — and explicitly "eliminates the need for a bastion host or proxy
  nodes." Authorization is IAM plus optional VPC Service Controls, not source
  address. Terraform surface confirmed against the provider source
  (`resource_container_cluster.go`, hashicorp/google v7.42):
  `control_plane_endpoints_config` carries `dns_endpoint_config`
  (`allow_external_traffic`, `enable_k8s_tokens_via_dns`,
  `enable_k8s_certs_via_dns`) and `ip_endpoints_config.enabled`. gcloud
  equivalent is `--enable-dns-access`.
  Confirmed empirically rather than only on paper: the endpoint resolves via
  public DNS (`dig @8.8.8.8` -> `216.239.32.27`). **It is a public name, not
  an internal address** — a point worth stating plainly, because the natural
  assumption is the opposite and the whole design depends on it being public.

- **IAP TCP forwarding — evaluated, not adopted (verified 2026-08-20** against
  Google's "Use IAP for TCP forwarding"). Forwards SSH, RDP and arbitrary TCP
  to VMs with no external IP; ingress is opened only to `35.235.240.0/20`, and
  authorization is `roles/iap.tunnelResourceAccessor`. Google's IAP-for-GKE
  page states the positioning outright — it "enables you to control
  resource-level access for employees instead of using a VPN" — and supports
  the Gateway API on GKE 1.24+. Rejected for the user-access plane only
  because IAP TCP forwarding targets Compute Engine instances rather than
  managed services, so a private Cloud SQL instance needs a jump VM running
  the Auth Proxy plus a tunnel to it: two hops where a subnet router gives
  `psql -h 10.x.x.x`. Recorded because it remains the GCP-native answer if
  ADR-0011's third-party dependency later proves unwelcome.

- **Tailscale control-plane architecture (verified 2026-08-20** against
  Tailscale's "How Tailscale works", subnet router, and Tailnet Lock docs).
  The coordination server is "essentially, a shared drop box for public keys";
  the control plane is hub-and-spoke but "carries virtually no traffic," and
  private keys never leave their node. **It therefore cannot be moved inside
  the VPC** — every node must reach it to exchange keys, so a private one is
  a chicken-and-egg, and self-hosting means Headscale, which Tailscale's own
  docs note "removes the availability guarantees and low maintenance
  overhead" of the SaaS. What goes in the VPC is a subnet router; the docs
  name this exact use case, "securely connect to cloud-managed services like
  Amazon RDS or Google Cloud SQL without exposing them to the public
  internet." The residual risk is that Tailscale's control plane can add
  nodes to a tailnet; Tailnet Lock is the documented mitigation, and with it
  "even if Tailscale were malicious or Tailscale infrastructure hacked,
  attackers can't send or receive traffic in your tailnet."

## Claims graded

None yet — grading happens at milestone close per ADR-0008.
