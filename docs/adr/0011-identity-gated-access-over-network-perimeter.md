# ADR-0011: Operator and user access is identity-gated, not perimeter-gated

Status: Accepted · August 2026 · supersedes the access half of ADR-0009

## Context

ADR-0009 chose a perimeter answer for reaching private things: a site-to-site
HA VPN to the home lab standing in for a corporate interconnect, with the
cluster's public control-plane endpoint restricted by
`master_authorized_networks_config` in the meantime, "moving to the private
endpoint once the VPN is live."

That ADR predicted its own cost and got the direction right but the magnitude
wrong. It recorded: *"Operator friction increases slightly (an
authorized-networks entry must be maintained)."* In practice, over five days:

- **2026-08-13** — the operator's residential lease turned over. The API
  server became unreachable. GKE *drops* unauthorized source IPs rather than
  refusing them, so it presented as a network hang, not an authorization
  failure, and cost real debugging time before the cause was obvious.
- **2026-08-18** — the lease cycled back to a previously held address,
  breaking the first scripted `cycle.sh` run mid-flight. The rebuild created
  a cluster authorized for an address the operator no longer had, so
  `3-argocd` could not reach it. This is churn within a small ISP pool, not a
  one-way reassignment, so it recurs indefinitely.

The assumed structural fix was to finish the VPN. That assumption does not
survive checking: **Cloud VPN requires a static external IP on the peer
gateway.** A residential dynamic lease cannot supply one. Building the VPN
would therefore not have solved the problem — it would have inherited it and
enlarged the blast radius, because a changed address breaks the
`external_vpn_gateway` resource and the whole private path rather than one
allowlist entry. HA VPN additionally requires BGP on the peer device, which
the UniFi gateway may not support, and an idle tunnel meters at roughly
$35–40/month against a trial budget already ~42% consumed.

Separately, the *requirement* was restated during M1 and turns out not to be
what the VPN answers. The reason to want private connectivity is so **users**
can reach internal applications, SSH to hosts, and connect to databases. A
site-to-site VPN serves **locations** — it would grant access to exactly one
building, and every remote developer would still need a second mechanism.

## Decision

**Separate the two concerns ADR-0009 fused, and gate access by identity
rather than by network position.**

1. **Control-plane access uses the GKE DNS-based endpoint.** A public,
   immutable FQDN resolved through Google's API front door and authorized by
   IAM. Google documents this as the preferred control-plane access path. The
   IP endpoints are disabled outright (`ip_endpoints_config.enabled = false`),
   `master_authorized_networks_config` is deleted, and the
   `authorized_networks` variable is removed from `2-cluster` entirely.
   `3-argocd` reaches the cluster by this name and, because `*.gke.goog`
   presents a publicly trusted certificate, no longer passes the cluster CA.

2. **User access to private workloads uses Tailscale, on a jump box in
   `1-network`.** A small VM with no external IP carries the Tailscale subnet
   router advertising `local.private_ranges` into the tailnet, so reaching a
   private address is `psql -h 10.x.x.x` rather than a tunnel inside a tunnel.
   Access policy is **not yet** expressed as code. The tailnet runs
   Tailscale's default allow-all ACL, which means any device on it reaches
   every advertised range. That is tolerable at two nodes, both belonging to
   the operator, and is not tolerable the moment a third party joins. Managing
   ACLs with the `tailscale/tailscale` Terraform provider, so access policy is
   reviewed in pull requests like the rest of the platform, is the intended
   follow-up and is deliberately recorded here as **not done** rather than
   described as if it were.

   The first draft of this ADR put the subnet router in the cluster, as the
   Tailscale Kubernetes operator's `Connector` resource managed by Argo CD.
   That was wrong on this repo's own terms: it puts *reachability* in the
   disposable layer, so every `cycle.sh` teardown would destroy user access to
   the VPC. "Identity and reachability persist; compute is disposable" is the
   rule the four-layer split exists to enforce, and a subnet router is
   reachability. The in-cluster operator remains available later for a
   different job — exposing Services by name on the tailnet — which is
   additive to routing by IP, not a replacement for it.

3. **The jump box also carries a break-glass path that shares nothing with
   the everyday one.** IAP TCP forwarding reaches it over
   `gcloud compute ssh --tunnel-through-iap`, with ingress allowed only from
   `35.235.240.0/20` and authorization by `roles/iap.tunnelResourceAccessor`.
   The two paths fail independently by construction: Tailscale needs
   *outbound* internet to reach its coordination server, while IAP needs
   *inbound* from Google and no egress at all. So IAP still works when NAT is
   down, when the tailnet is broken, or when a Tailscale key has expired —
   which are exactly the moments you need to get in. OS Login ties the login
   itself to IAM, so both the network path and the shell are IAM decisions.

4. **Cloud NAT moves from `2-cluster` to `1-network`.** Its original placement
   was reasoned as "NAT serves nodes specifically, so it should be created and
   destroyed on the same schedule they are." The jump box gives it a second
   consumer on a persistent schedule, and a persistent resource cannot depend
   on a disposable one. Running it continuously costs roughly $5/month: the
   gateway is `$0.0014/hr × attached VMs` (flattening to `$0.044/hr` only
   above 32) plus `$0.005/hr` for the external IP.

5. **The HA VPN stays in `1-network`, re-scoped.** It remains unbuilt
   (`enable_vpn = false`) and is no longer the user-access story. It is the
   hybrid/on-prem interconnect story — a genuine enterprise requirement that
   identity-based access does not cover — and now carries an explicit
   precondition that the peer side needs a static IP and BGP.

The routes do not change; only the transport does. `1-network` defines the
three ranges once as `local.private_ranges` (subnet primary plus both
secondary ranges) and both the VPN and the Tailscale subnet router consume
the same list.

## Consequences

- **The dynamic-IP failure mode is deleted, not mitigated.** There is no
  allowlist to keep current and no operator IP anywhere in `2-cluster`.
  Verified conclusively: with `ipEndpointsConfig.enabled: false` live,
  `kubectl get nodes` and a `3-argocd` plan refreshing both Helm releases
  both still succeed, so the allowlist was never carrying that traffic.
- **Defense in depth is reduced, and the mitigation is named.** The allowlist
  was a network-layer backstop behind IAM; leaked credentials now face no
  second barrier. VPC Service Controls is the documented replacement (adding
  `container.googleapis.com` and `kubernetesmetadata.googleapis.com` to a
  perimeter) and is **not yet configured** — this is an open gap, not a
  solved one.
- **A third-party control plane enters the access path.** Tailscale's
  coordination server is SaaS and cannot be moved inside the VPC: every node
  must reach it to exchange keys, so a private one is a chicken-and-egg. It
  carries no traffic — it is a public-key drop box, and private keys never
  leave their node — but it *can* add nodes to the tailnet. Tailnet Lock is
  the mitigation, and it is **named but deliberately deferred**.
  The reasoning, revised once the tailnet actually existed: Tailnet Lock
  requires at least two signing nodes, and this tailnet has exactly two — a
  laptop and a VM that Terraform can replace. Every new node then needs
  signing by an existing one, so a rebuilt jump box becomes a manual step, and
  the effective redundancy is closer to one than two. Initialization emits ten
  disablement secrets shown once; per Tailscale's docs, losing them all means
  "the tailnet cannot be recovered" unless one was entrusted to Tailscale
  support. Taking real lockout risk to defend a reference build with no
  production data against Tailscale-the-company being compromised is the wrong
  trade at this size.
  Revisit when the tailnet has three or more durable signing nodes, or when
  anything on it is worth more than the cost of being locked out of it. On
  enabling: send a disablement secret to Tailscale support, store the rest in
  a password manager, and add a third signing node first.
- **Two open gaps, both named rather than solved.** VPC Service Controls (the
  replacement for the allowlist's defence in depth) is not configured, and
  Tailscale ACLs are not yet code — the tailnet is allow-all by default. Both
  are acceptable at the current size and neither should survive a third party
  getting access to either plane.
- **Two access-control systems now exist** (GCP IAM and Tailscale ACLs) where
  ADR-0009 implied one. Accepted as the price of the L3 ergonomics: with a
  subnet router, reaching a private database is `psql -h 10.x.x.x` rather
  than the two-hop tunnel an IAP-only design would require.
- **The DNS endpoint is a prerequisite, not an alternative.** Moving the
  subnet router onto the jump box removes the sharpest version of the
  bootstrap problem — the tailnet no longer depends on a workload inside the
  cluster it is meant to reach — but it does not remove the need. The control
  plane has no IP at all now, so the tailnet cannot carry `kubectl` even in
  principle: a subnet router routes addresses, and there is no address. Every
  `cycle.sh` rebuild therefore reaches an empty cluster over the DNS endpoint,
  and `3-argocd` installs Argo CD through it. The two planes stay separate on
  purpose — Tailscale is the data plane, the DNS endpoint is the control
  plane, and nothing on the jump box is on the `kubectl` path.
- **ADR-0009's "slightly" is superseded.** Its posture reasoning stands; its
  estimate of allowlist friction, and its plan to resolve that friction with
  the VPN, do not.
- **The jump box is a pet, and that is a real cost.** It is a VM to patch,
  where the in-cluster `Connector` would have been a managed workload. It runs
  Debian rather than Container-Optimized OS on purpose — the host exists to be
  used interactively (tailscale, psql, the Cloud SQL Auth Proxy) and COS has
  no package manager — so the tradeoff is accepted and bounded by the machine
  having no public IP and no inbound path except IAP.
- **The Tailscale auth key is deliberately not in Terraform.** The startup
  script installs the client and enables kernel forwarding but does not run
  `tailscale up`; joining is a one-time operator step over IAP. Putting an
  auth key in instance metadata would write a credential into Terraform state
  and into the metadata server. The cost is that a rebuilt jump box needs one
  manual command, and advertised routes need approving in the Tailscale admin
  console.
- **C-02's baseline shifts.** Removing NAT's create/destroy from `2-cluster`
  takes it out of the measured rebuild path, so cycle timings after 2026-08-20
  are not directly comparable to the cycle-1 numbers. The improvement is an
  artifact of relocation, not of rebuild-cheapness.
- The VPN's ~$35–40/month idle cost is never incurred.
- No claim in the register covers the VPN, so nothing requires regrading.
  The operability evidence belongs to C-02.
