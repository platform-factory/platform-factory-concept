# Kubernetes Egress Control — 2026 State of the Art

Research digest, 2026-07-28. Motivating question: in a 2020-era platform build we
wanted DNS-layer protection AND to block outbound calls that didn't initiate with
a DNS query first (devs declare hostnames like `api.xyz.com`; hardcoded IPs
should fail). How is that solved in 2026? All load-bearing claims verified
against primary sources on 2026-07-28 by a research agent; source list at bottom.

Ported into this repo 2026-07-29. The implementation survey below ran against
**AWS/EKS**; the mechanism section and the Cilium material are cloud-neutral. GCP
notes added at port are [unverified].

---

## TL;DR

**The property you want — "a pod can only connect to an IP it first resolved via a
sanctioned DNS query" — is now a solved, productized pattern with exactly three
per-pod implementations:**

1. **EKS Auto Mode `ApplicationNetworkPolicy`** (AWS-native, launched Dec 15 2025) — zero ops, Auto Mode ONLY
2. **Cilium `toFQDNs`** (OSS, deepest/most mature) — requires owning the CNI
3. **Calico Enterprise DNS policy** (commercial-only, license cost)

The 2021-era pain (TTL races, dropped long-lived connections, DNS dying with the
agent) is either fixed (Cilium) or absorbed by the cloud provider (Auto Mode).

**The fork this creates on AWS:** Auto Mode and Cilium are mutually exclusive.
AWS: "EKS Auto Mode does not support alternate CNI plugins or network policy
plugins." So the egress decision and the compute-management decision are ONE
decision:

- **Auto Mode + ApplicationNetworkPolicy** — ops burden ~1/5, but the feature is 7 months old, `v1alpha1`, TTL internals undocumented
- **Standard EKS + Karpenter + Cilium** — battle-tested for ~7 years, every knob public, but you own a CNI (ops ~3-4/5)

Either way, back it with the VPC-perimeter pair (DNS-firewall walled garden +
network-firewall egress SNI allowlist in a centralized inspection VPC behind a
Transit Gateway hub) and a detective layer. Every individual layer has a
documented bypass; the stack together doesn't.

---

## The mechanism (how "no connection without DNS first" actually works)

All three per-pod implementations use the same architecture:

1. Default-deny egress on the pod.
2. Policy declares allowed **hostnames** (`api.xyz.com`, wildcards supported).
3. A **per-node DNS proxy** intercepts the pod's DNS queries. Queries for
   non-allowlisted names are refused — so a blocked destination fails at the
   *resolve* step with a clean DNS error, not a hang.
4. For allowed queries, the proxy forwards to the real resolver, then **records the
   answered IPs (with their TTLs) into the datapath policy** (an eBPF map keyed to
   that pod) *before* handing the answer back to the app.
5. eBPF on the pod's veth allows egress **only to IPs currently in that map**.

A hardcoded IP was never in a DNS answer → it's not in the map → the packet drops.
That is literally the desired semantics, enforced in the kernel per pod.

---

## Option 1: EKS Auto Mode ApplicationNetworkPolicy (the new thing)

- **Launched Dec 15, 2025**, GA all commercial regions, K8s ≥ 1.29. CRDs in API
  group `networking.k8s.aws/v1alpha1`.
- `ApplicationNetworkPolicy` = standard NetworkPolicy fields **plus
  `egress[].to[].domainNames`** (exact names + wildcards like
  `*.s3.us-east-1.amazonaws.com`).
- Also `ClusterNetworkPolicy` — cluster-scoped Admin/Baseline tiers (AWS's
  implementation of the upstream AdminNetworkPolicy work, merged into one CRD).
  Admin-tier rules can also carry `domainNames` on Auto Mode — i.e., a
  platform-owned, cluster-wide FQDN allowlist devs can't override.
- **Confirmed DNS-gated semantics** (AWS docs "How does it work?"): node agent
  filters DNS against the allowlist → allowed queries proxied to the node-local
  CoreDNS → Route 53 Resolver → "resolved IPs with TTL … written in an eBPF map …
  eBPF probes attached to the Pod veth interface filter egress traffic." Same
  model as Cilium.
- Setup: `amazon-vpc-cni` ConfigMap `enable-network-policy-controller: "true"`;
  NodeClass `networkPolicy: DefaultDeny`, `networkPolicyEventLogs: Enabled`.
- **FQDN rules are Auto-Mode-exclusive.** On standard EKS, the VPC CNI (≥ v1.21.1)
  gets `ClusterNetworkPolicy` L3/L4 only — no `domainNames`. Parity for standard
  EKS is an open ask (containers-roadmap #2801, open since Apr 2026).
- **Honest gaps:** `v1alpha1` API 7 months post-launch; no public equivalents of
  Cilium's TTL knobs (min-TTL, max-IPs-per-name, idle-connection grace); no
  documented DNS behavior during node-agent restart; documented footgun — an
  ApplicationNetworkPolicy and a NetworkPolicy with the **same name** in one
  namespace silently corrupt policy state (roadmap #2772); no L7 rules yet.
  → All POC-verifiable in a week; none disqualifying on their face.

## Option 2: Cilium toFQDNs (the 2021 approach, now grown up)

Current stable **1.19.6** (Jul 16, 2026). What changed since the 2021-era pain:

| 2021 pain | 2026 state |
|---|---|
| App connects before policy is programmed (resolve→connect race) | `--tofqdns-proxy-response-max-delay` (default 100 ms): DNS answer is **withheld** until the datapath has the IP |
| Long-lived connections cut when TTL expires | Live connections survive TTL expiry — deferred deletes, up to 10,000 retained IPs; "live connections are not expired until they terminate" |
| IP churn / CDN rotation blowing caps | 1,000 IPs per hostname per endpoint (was 50); oldest-unused expire, in-use retained |
| Single-level wildcards only | `**.example.com` multi-level wildcards since 1.19.0 (Feb 2026) |
| DNS dies if cilium-agent dies | **Standalone DNS Proxy** DaemonSet (alpha, 1.19): keeps resolving already-seen domains during agent downtime. Hardened HA proxy = Isovalent (Cisco) Enterprise feature |

- On EKS: full CNI replacement (ENI mode = pods keep real VPC IPs). CNI *chaining*
  with the VPC CNI impairs the L7/DNS-proxy path — not viable for FQDN policy.
  kube-proxy replacement NOT required for toFQDNs.
- AWS support posture: Cilium on EC2 nodes is "partner supported" (Isovalent), not
  AWS-supported. **Not possible at all on Auto Mode.**
- Still real ops: you own CNI upgrades, DNS-proxy latency/error metrics
  (`cilium_ipcache_errors_total`), and wildcard-cardinality hygiene (a
  `*.amazonaws.com`-style rule inserts every cached IP — identity explosion).
  A wedge bug as recent as Mar 2026 (#44714, 1.18.x nodes dropping all FQDN
  egress until agent restart) shows this layer still needs monitoring.

## The JVM angle (for JVM-heavy shops)

Modern OpenJDK caches successful DNS lookups **30 seconds** by default
(`networkaddress.cache.ttl` unset; the cache-forever behavior only existed under a
SecurityManager, removed in JDK 24). 30s sits inside typical TTL windows → fine.
The failure case is apps/HTTP clients that pin resolved IPs indefinitely and
reconnect later: the new connection targets an IP whose policy entry expired →
dropped. Mitigations: min-TTL knob (Cilium), idle-connection grace, or fix the
client. Worth one line in the platform's service standards: "resolve per
connection; don't cache IPs."

## Perimeter layers (AWS survey — VPC-level, work under either cluster choice)

- **Route 53 Resolver DNS Firewall**, walled-garden mode: "deny all domains except
  the ones you explicitly trust." Blocks *resolution only* — never packets, so it
  cannot stop hardcoded-IP egress by itself. DNS Firewall Advanced (Nov 2024) adds
  DNS-tunneling/DGA detection.
- **AWS Network Firewall** in a centralized egress/inspection VPC behind a
  Transit Gateway hub (the standard AWS multi-VPC reference architecture):
  strict rule order + default-drop, egress allowlist by TLS SNI. Without TLS
  inspection, SNI is client-controlled (spoofable); with egress TLS inspection
  (GA all regions Dec 2023) you decrypt with your own CA and close that hole at
  the cost of latency + cert distribution. Pricing: $0.395/endpoint-hr +
  $0.065/GB (NAT charges waived when chained in-path).
- Key honesty: DNS firewall and network firewall are **not linked** — nothing at
  the perimeter ties "flow allowed" to "was previously resolved." Only the
  pod-level options do that. Perimeter granularity is per-VPC, not per-pod (pod
  IPs churn too fast to key firewall rules on).

## DNS plumbing (required under every option)

- Pin pods to the sanctioned path: default-deny egress + allow 53 **only to
  CoreDNS** (on Auto Mode: allow kube-dns via an Admin-tier ClusterNetworkPolicy).
  CoreDNS forwards to the VPC resolver → DNS firewall and the detective layer
  see every pod query.
- DoH/DoT bypass: DoT is TCP/853 — dead under default-deny. DoH rides 443 — block
  known DoH provider domains at the DNS firewall/SNI layer. Under
  default-deny-egress this is mostly moot (unlisted DoH endpoints are
  unreachable anyway); it matters for the perimeter-only layers.

## What is NOT an enforcement layer

- **Service mesh egress (Istio REGISTRY_ONLY, Linkerd EgressNetwork):** both
  projects' own docs state a malicious app can bypass the sidecar entirely. Istio
  ambient deliberately doesn't implement REGISTRY_ONLY in ztunnel. Mesh egress is
  routing/audit, not a security boundary — only adopt it as egress *plumbing* if
  a mesh is wanted anyway, and pin it with NetworkPolicy underneath.
- **Cilium Egress Gateway:** static egress source IP (so external firewalls can
  key on it) — routing, not FQDN filtering. Composes with toFQDNs.
- **Portability note:** upstream sig-network has FQDN egress on the roadmap
  (NPEP-133, status Implementable, Admin-tier/allow-only) but the released
  ClusterNetworkPolicy v1alpha2 has no `domainNames` yet. Every FQDN API today is
  vendor-specific. AWS shipped tomorrow's semantics behind its own API group.

## Detective layer (AWS survey)

- GuardDuty: DNS-log findings (C2, DNS exfil) — only for queries through the VPC
  resolver (another reason to pin resolution); Runtime Monitoring for EKS (eBPF);
  **Extended Threat Detection for EKS** (Jun 2025) correlates audit + runtime +
  API into attack-sequence findings.
- Cheap compliance-evidence win: join resolver query logs with VPC flow logs —
  "which flows had no preceding sanctioned resolution?" is answerable in a
  warehouse query. Cilium adds Hubble per-pod policy-verdict + DNS flow logs.

---

## Decision matrix (AWS survey)

| Approach | Blocks hardcoded-IP egress? | Layer | Granularity | On Auto Mode? | Ops (1-5) |
|---|---|---|---|---|---|
| Cilium toFQDNs | **Yes** (DNS-gated eBPF) | Pod datapath | Per-pod | **No** | 3-4 |
| EKS Auto ApplicationNetworkPolicy | **Yes** (DNS-gated eBPF) | Pod datapath | Per-pod/ns + cluster Admin tier | **Only there** | 1-2 |
| Standard-EKS VPC CNI + ClusterNetworkPolicy | No (L3/L4 CIDR only) | Pod datapath | Per-pod/ns | n/a | 2 |
| R53 DNS Firewall walled garden | No (resolution only) | VPC resolver | Per-VPC | Yes | 1 |
| Network Firewall SNI allowlist | Partial (SNI spoofable w/o decrypt) | Inspection VPC | Per-VPC | Yes | 2 |
| NFW + egress TLS inspection | Stronger (decrypt) | Inspection VPC | Per-VPC | Yes | 3 |
| Istio/Linkerd egress | Bypassable by design | Sidecar/proxy | Per-pod | runs, but not a boundary | 4 |
| Calico Enterprise DNS policy | **Yes** (same model) | Pod datapath | Per-pod | No | 3 + license |

## Recommended shape (defense in depth, four layers)

1. **Pod level (the DNS-gated allowlist):** the per-pod eBPF option your compute
   mode allows. Dev contract: the FQDN allowlist lives in the service's repo
   next to its k8s config, Argo-synced — same ergonomics as the rest of the
   platform pattern.
2. **Resolver pinning:** 53 only to CoreDNS; CoreDNS → the VPC resolver;
   DNS-firewall walled garden pushed org-wide.
3. **Perimeter:** centralized egress/inspection VPC + SNI allowlist (TLS
   inspection as a later hardening step, not day 1).
4. **Detective:** cloud-native threat detection + resolver-log/flow-log join for
   compliance evidence.

**POC question that decides the AWS fork** (fits a personal-account POC): stand
up an Auto Mode cluster, apply a default-deny NodeClass + ANP with
`domainNames`, then verify: (a) hardcoded-IP egress drops, (b) allowed-FQDN
flows survive TTL expiry mid-connection, (c) DNS behavior during a forced
node-agent restart, (d) wildcard behavior. If (b) or (c) is ugly → standard EKS
+ Cilium.

## GCP mapping (added at port, 2026-07-29 — ALL [unverified])

This repo's reference cloud is GCP (ADR-0005). Verify at build:

- **GKE Dataplane V2 is Cilium-derived** — the eBPF datapath is there by
  default. Google ships an `FQDNNetworkPolicy` CRD for domain-based egress;
  availability/tier to confirm (may require GKE Enterprise rather than
  Standard-tier GKE).
- Alternative: full Cilium OSS on GKE Standard (own-the-CNI trade, as on AWS).
- Perimeter analogs: Cloud DNS response policies / DNS policies for the
  resolver layer; Secure Web Proxy or NGFW for egress inspection.
- The mechanism section, Cilium detail, JVM note, mesh caveats, and the
  four-layer recommended shape transfer unchanged.

## Unverified / watch items

- EKS ANP TTL internals (min-TTL, IP caps, proxy hold behavior) — undocumented; POC.
- ANP on existing (pre-Dec-2025) clusters — "coming weeks" at launch; likely landed; confirm.
- Auto Mode management fee (~10-12% on top of EC2) — from memory, not re-verified.
- Whether Cilium 1.19.x fully resolves the #44714 FQDN-wedge — fix not confirmed.
- The entire GCP mapping section above.

## Primary sources (all fetched 2026-07-28)

- AWS What's New Dec 15 2025 "Amazon EKS introduces enhanced network security policies"; docs.aws.amazon.com/eks/latest/userguide/auto-net-pol.html; blogs/containers Dec 15 + Dec 22 2025; containers-roadmap #2801, #2772; aws-network-policy-agent releases (v1.3.0, v1.4.0)
- docs.aws.amazon.com/eks/latest/userguide/alternate-cni-plugins.html (Auto Mode CNI stance)
- docs.cilium.io v1.19: security/policy/language, security/standalone-dns-proxy, cmdref/cilium-agent, network/egress-gateway; cilium/cilium releases, `pkg/defaults/defaults.go`, issues #44714, #30984
- Route 53 DNS Firewall docs + What's New Nov 2024 (Advanced), Jun 2026 (Palo Alto preview); aws.amazon.com/network-firewall/pricing (2026-07-28); AWS multi-VPC whitepaper (centralized egress)
- istio.io egress-control task + blog 2026 egress-dynamic-dns (Istio 1.30.3); linkerd.io 2-edge egress (Linkerd 2.20)
- network-policy-api.sigs.k8s.io: NPEP-133, v0.2.0 release, implementations page
- docs.tigera.io Calico Enterprise domain-based-policy; tigera.io Calico 3.30 blog
- OpenJDK `sun/net/InetAddressCachePolicy.java` (30s default); GuardDuty What's New Jun 17 2025
