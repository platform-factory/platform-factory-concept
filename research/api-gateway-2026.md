# API Gateway Landscape — 2026 State of the Art

Research digest, 2026-07-28. Motivating question: Gloo worked well in a 2020-era
fintech platform build — what's the right pick in 2026? All load-bearing claims
verified against primary sources on 2026-07-28 by a research agent; sources at
bottom. Ported into this repo 2026-07-29; GCP notes added at port are marked
[unverified].

---

## TL;DR

**Gloo didn't just survive — it became a CNCF project, and the enterprise features
you paid Solo for in 2020 are now open source.** Solo.io donated Gloo Gateway to
CNCF (Nov 2024); it was renamed **kgateway**, accepted as a CNCF Sandbox project
Mar 2025 (incubation application pending since Oct 2025). Current: v2.4.1
(Jul 27, 2026), quarterly releases. External auth, JWT validation with claim-based
RBAC, local + global rate limiting, and request/response transformations — the old
Gloo Enterprise differentiators — are all in OSS kgateway now.

**The shape of the decision changed:** in 2020 you picked a gateway *product* with
proprietary CRDs. In 2026 you adopt the **Kubernetes Gateway API** (v1.6.1, now the
assumed default — the community ingress-nginx controller was retired March 2026 and
Kubernetes itself points everyone at Gateway API) and pick an *implementation* of
it. Routes are portable across implementations; only the policy CRDs are
vendor-specific. Lock-in risk is structurally lower than in 2020.

**Recommendation: kgateway**, with Envoy Gateway as the fallback if CNCF-Sandbox
maturity is a blocker. And the AI kicker: kgateway's sibling project
**agentgateway** (same control-plane family) is the leading agent-native gateway —
it landed in the Linux Foundation's Agentic AI Foundation, the same foundation that
hosts the MCP spec itself. That's a first-class landing zone for an agent-native
platform layer on day 1, not a bolt-on later.

**Important nuance:** classic Gloo Edge — the actual VirtualService/Upstream CRDs
run in that 2020-era build — is **EOL Dec 31, 2026**. The central
gateway-config-in-git pattern ports conceptually (declarative gateway config in
git, Argo-synced), not literally (kgateway is pure Gateway API: HTTPRoute +
TrafficPolicy/ListenerPolicy CRDs).

---

## What changed since the 2020 analysis

1. **Gateway API standardization.** v1.6.1 (Jul 2026). GA: HTTPRoute, GRPCRoute,
   TCPRoute, UDPRoute, TLSRoute, BackendTLSPolicy, ListenerSet. Formal conformance
   program with Conformant/Partially/Stale tiers. Native auth semantics (GEP-1494)
   are coming into the core API. Mesh support (GAMMA) went GA in 2024.
2. **ingress-nginx is dead.** Retired March 2026 after the IngressNightmare CVEs
   (incl. a 9.8 unauthenticated RCE, Mar 2025). Zero patches ever again. The
   ecosystem consolidated on Gateway API implementations.
3. **Envoy stayed the dominant data plane.** The finalists are all Envoy-based or
   Envoy-adjacent; Envoy's steady DoS-class CVE stream means the gateway choice is
   partly "who ships Envoy patches fastest" (kgateway and Envoy Gateway both do).
4. **The Kong cautionary tale.** From Kong Gateway 3.10 (Mar 2025), no prebuilt
   OSS images; unlicensed images run as expired-enterprise. OSS Kong is
   effectively frozen at 3.9.x. Same vendor-squeeze pattern as the Upbound/
   Crossplane-provider scare — and the same hedge applies: prefer
   foundation-owned projects (kgateway is CNCF; Envoy Gateway is under graduated
   Envoy).
5. **AI/agent traffic became a gateway category.** LLM routing, token-based rate
   limiting, MCP federation, A2A — now table stakes in the leading projects.

## kgateway (the Gloo successor) in detail

- **Lineage:** Gloo (2018) → Gloo Gateway donated at KubeCon NA Nov 2024 → renamed
  kgateway → CNCF Sandbox 2025-03-04 → incubation application open (cncf/toc#1913,
  since 2025-10-06, still pending as of today).
- **Releases:** v2.1.0 Oct 2025, v2.2.0 Feb 2026, v2.3.0 May 2026, v2.4.1 Jul 27
  2026. Quarterly cadence.
- **OSS feature set** (formerly-enterprise features marked ★):
  ★ External auth: API key, Basic, OAuth2/OIDC, BYO gRPC+HTTP ext-auth service.
  ★ JWT validation, claim extraction, claim-based RBAC via CEL.
  ★ Rate limiting: local + global (Envoy rate limit service).
  ★ Transformations: new Rust "rustformation" engine — headers + body templating,
  the classic Gloo pattern. Strongest transformation story of the finalists.
  Plus: ExtProc, CORS/CSRF, mTLS listeners, TLS passthrough, backend TLS, route
  delegation (the Gloo delegation model), policy CRDs (TrafficPolicy,
  ListenerPolicy, BackendConfigPolicy), documented Argo CD install, ingress-nginx
  migration tooling, Istio ambient waypoint support.
- **Commercial layer:** "Solo Enterprise for kgateway" (2.2.x line): Coraza-based
  WAF (WAFPolicy CRD, OWASP CRS), FIPS images, enterprise policy CRDs, SLAs.
  Support relationship with Solo is resumable, not mandatory.
- **Fintech signal:** Trust Bank (regulated digital bank) is a public reference.
- **Risks:** CNCF Sandbox, not yet incubating; maintainership still Solo-heavy;
  Solo's commercial energy has visibly shifted to agentic infrastructure;
  enterprise build (2.2.x) trails OSS (2.4.x).

## The finalists

| | kgateway | Envoy Gateway | agentgateway | Istio ingress |
|---|---|---|---|---|
| Version (Jul 2026) | v2.4.1 | v1.8.3 | v1.4.0 | 1.30.3 |
| Foundation | CNCF Sandbox (incubation pending) | CNCF (under graduated Envoy) | LF / Agentic AI Foundation | CNCF Graduated |
| Data plane | Envoy (can also drive agentgateway) | Envoy | Purpose-built Rust | Envoy + ztunnel |
| Ext-auth / JWT / OIDC | Full, OSS | Full, OSS (SecurityPolicy) | Full + MCP authz, token exchange | JWT + ext-auth; no API-key surface |
| Rate limiting | Local + global OSS | Local + global OSS | + LLM token-based | Local; global is DIY |
| Transformations | **Strongest** (body templating) | Header/URL only; body via ExtProc/Wasm | kgateway-style templating | EnvoyFilter/Wasm (fiddly) |
| AI/agent story | Sibling agentgateway, same family | Envoy AI Gateway 1.0 add-on | **Is** the AI gateway (LLM/MCP/A2A native) | Experimental agentgateway waypoint |
| Gateway API conformance | Conformant (v1.6) | Partially (likely report lag) | Conformant (v1.6) | Conformant (gateway + mesh) |
| Enterprise support | Solo | Tetrate | Solo | Solo/Tetrate/Red Hat/Google |
| Key risk | Sandbox; Solo-dominated | No WAF, weak transforms, fast minor EOLs | Youngest (1.0 Mar 2026); Rust proxy less battle-tested | Operational weight for edge-only use |

## The AI/agent angle

- **agentgateway**: created by Solo Mar 2025 (Rust), donated to Linux Foundation
  Aug 2025, joined the **Agentic AI Foundation** June 2026 — the LF foundation
  co-founded around Anthropic's MCP, Block's goose, and OpenAI's AGENTS.md.
  v1.4.0 (Jul 27, 2026). 300+ contributors from 60+ orgs (Microsoft, AWS, Alibaba,
  Adobe, Cisco, Salesforce, Red Hat, CoreWeave).
- Features: one data plane for HTTP/gRPC/TCP + LLM + MCP + A2A. LLM: provider
  failover, virtual models, token budgets/spend limits, cost tracking, token-based
  rate limiting, semantic caching, guardrails (incl. Bedrock Guardrails). MCP:
  tool federation (virtual MCP), guardrails, authz, current MCP spec (2026-07-28),
  OAuth token exchange / Cross-App Access. Also a conformant Gateway API
  implementation in its own right.
- **kgateway's control plane provisions agentgateway data planes since v2.1.0** —
  one control-plane family covers "normal" API traffic (Envoy) and agent/LLM
  traffic (Rust agentgateway). Istio 1.30 added experimental agentgateway
  waypoint support.
- Alternatives: Envoy AI Gateway 1.0 (GA Jun 2026; Tetrate/Bloomberg/Netflix;
  16 providers, MCPRoute) if standardized on Envoy Gateway. Kong AI Gateway is
  commercial-only in practice.

## Cloud-native front doors (survey: why they're the front door, not the gateway)

AWS survey (retained from the original compile, all verified 2026-07-28):

- **EKS Auto Mode built-in controller**: Ingress→ALB and Service→NLB only — **no
  Gateway API support**. ALB gives L7 routing, TLS/ACM, OIDC/Cognito auth actions,
  AWS WAF attachment. It does NOT do JWT bearer validation, API keys,
  transformations, or per-client rate limiting. Conclusion: ALB/NLB is the L4/L7
  front door; the real gateway runs in-cluster behind it.
- **AWS Load Balancer Controller v3.0** (self-managed, Jan 2026): Gateway API GA,
  partially conformant.
- **AWS API Gateway via VPC Link**: $1.00/M requests (HTTP APIs) — per-request
  economics + extra hop lose to a self-run Envoy gateway at scale; keep for
  Lambda-heavy edges only.
- **VPC Lattice**: east-west service networking with IAM/SigV4 auth, not an
  internet edge.

GCP mapping [unverified — not yet re-checked to this doc's standard]: GKE ships
its own Gateway API controller (`gke-l7-*` gateway classes) backed by Cloud Load
Balancing, with Cloud Armor attachment — it plays the ALB role here. Expected
conclusion is the same shape: cloud LB as front door, kgateway in-cluster behind
it. Verify controller conformance + policy surface at build.

## Recommended shape for the reference platform

- **Edge:** Cloudflare (WAF/DDoS/bot at the edge) → cloud L4/L7 front door →
  **kgateway** in-cluster.
- **Pattern port:** the central gateway-config-in-git model survives intact —
  Gateway API routes + kgateway policy CRDs in the `edge-config` repo,
  Argo-synced. Same GitOps contract as the 2020-era build, new CRD vocabulary
  (HTTPRoute + TrafficPolicy instead of VirtualService + RouteOption).
- **Auth at the gateway:** JWT validation + CEL claim RBAC per route; ext-auth
  service when tenant logic demands it; global rate limits per consumer — all
  OSS now.
- **AI layer, when the agent tier lands:** add agentgateway under the same
  kgateway control plane for LLM/MCP/agent traffic — token budgets, guardrails,
  MCP tool federation. This is the strongest "platform-first buys the AI future"
  argument in the whole stack.
- **Fallback:** Envoy Gateway if Sandbox status is unacceptable to
  security/procurement — trade away transformations + the tight agentgateway
  integration, gain multi-vendor governance.
- **Falsifiers to watch:** kgateway incubation application (cncf/toc#1913) stalls
  or fails; Solo maintainer concentration doesn't dilute; enterprise-only drift
  in new features. Any of those → re-run this decision against Envoy Gateway
  before hardening.

## Unverified / watch items

- "Most widely deployed Envoy-based gateway" is Solo's claim; no independent data.
- Envoy Gateway "partially conformant" is likely a report-recency artifact, not a
  capability gap — interpretation, not confirmed.
- Whether Solo's Portal/UI extras have been ported from the legacy Gloo Gateway
  1.21 product into Solo Enterprise for kgateway 2.2.x — unconfirmed.
- EKS Auto Mode Gateway API absence — best evidence is an AWS re:Post article
  (~Mar 2026) + no What's-New; a very recent change could have been missed.
- The GCP mapping note above (added at port) — entirely unverified.

## Primary sources (all fetched 2026-07-28)

- solo.io blog (donation announcement); cncf.io/blog 2025-02-05 + project page
  (Sandbox 2025-03-04) + 2025-07-23 spotlight; github.com/cncf/toc#1913
- github.com/kgateway-dev/kgateway releases (v2.4.1); kgateway.dev/docs;
  docs.solo.io/kgateway/2.2.x; github.com/solo-io/gloo README (OSS EOL 2026-12-31)
- gateway-api.sigs.k8s.io (v1.6.1, conformance list); kubernetes.io/blog
  2025-11-11 (ingress-nginx retirement), 2026-03-20 (Ingress2Gateway 1.0);
  wiz.io IngressNightmare (CVE-2025-1974)
- github.com/envoyproxy/gateway releases (v1.8.3); aigateway.envoyproxy.io 1.0
  announcement (2026-06-23)
- agentgateway.dev; aaif.io blogs (2026-06); linuxfoundation.org AAIF press;
  github.com/agentgateway/agentgateway v1.4.0
- github.com/Kong/kong discussions#14628 (OSS image change); developer.konghq.com
- docs.aws.amazon.com/eks auto-configure-alb; repost.aws Gateway-API-on-Auto-Mode
  article; AWS LBC v3.0 GA blog (2026-01); aws.amazon.com/api-gateway/pricing;
  AWS networking blog 2026-03-31 (ALB + Lattice reference architecture)
