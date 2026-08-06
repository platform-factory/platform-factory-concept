# ADR-0009: The reference build mirrors corporate posture, not minimal cost

Status: Accepted · August 2026

## Context

The reference implementation runs on personal infrastructure with trial
credits, which creates constant pressure to configure for cheapness: zonal
control plane (free management tier), spot nodes, public node IPs (no NAT to
pay for), a cloud that connects to nothing. The first cut of the layer-0
Terraform took several of those shortcuts.

Each shortcut individually looked harmless, but together they change what the
build *proves*. A pattern demonstrated on an environment no corporation would
run leaves the reader doing the translation work themselves — "would this
survive private nodes? an interconnect? restricted API access?" — which is
exactly the credibility gap a reference implementation exists to close. The
build already hit this once: the VPC originally lived in the torn-down layer,
which works only for a cloud that talks to nothing (superseded during M1 —
see the build log; a real corporate cloud always has other network
connections, so the network was promoted to a persistent layer).

## Decision

**Configure the environment the way a normal corporate environment would look
and behave; keep it small rather than cheap.** Scale is not the goal — one
cluster, a handful of nodes — but every *posture* choice follows the corporate
default:

- **Network:** VPC persists independently of compute; site-to-site VPN
  (HA pair, BGP) to a peer network — the home lab stands in for the corporate
  interconnect; Private Google Access on; firewall changes logged.
- **Cluster:** regional control plane; private nodes with Cloud NAT for
  egress; the public API endpoint restricted to authorized networks, moving
  to the private endpoint once the VPN is live; on-demand, shielded nodes;
  workload identity (already in place).
- **Structure:** projects parented under the GCP organization when one is
  available, rather than floating org-less.
- **Deliberate exceptions, named:** egress lockdown (FQDN policies) arrives
  with the M3 egress-control work, not before; teardown-between-sessions
  stays (a corp wouldn't tear down nightly, but rebuild-cheapness is a
  registered claim being tested — C-02); scale stays minimal.

Cost yields to realism wherever the two conflict. Session burn rises from
near-zero to roughly $0.50/hour while the cluster exists; the teardown rhythm
keeps the absolute number trivial against the trial-credit budget.

## Consequences

- Claims graded by the build (C-01..C-22) are tested against a
  corporate-shaped environment, which is what makes their grades portable to
  readers' environments.
- C-02's cost data changes: "cheap enough to rebuild between sessions" is now
  tested with a regional control plane and NAT in the bill — a more honest
  test of the claim.
- The zonal-free-tier rationale recorded in early layer-0 comments and
  private notes is superseded by this ADR.
- Operator friction increases slightly (an authorized-networks entry must be
  maintained; NAT and the management fee bill during sessions) — accepted as
  the price of the demonstration being believable.
