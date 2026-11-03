# ADR-0023: What the M3 build changed in the approval-boundary decisions

Status: Accepted 2026-11-03, at M3's close · supersedes ADR-0021 §15's DNS
allow, as shipped in platform-config#21, with one that names every resolver
path the cluster uses; reads C-10's headline with a stated exception until the
workflow that produces the required check is protected, and decides the guard;
accepts, with the fact stated, that a fail-closed admission webhook can strand a
Gateway · from the M3 build (2026-10-27 to 2026-10-29 at the desk; sessions on
2026-10-31 and 2026-11-03)

## Context

ADR-0020 and ADR-0021 were written before any M3 build command ran. The build's
evidence is in [`docs/build-log/m3-approval-boundary.md`](../build-log/m3-approval-boundary.md).
ADR-0020 says a claim that holds only after a design change is graded ADJUSTED
and needs a superseding ADR, linked. ADR-0016 did this for M2 and ADR-0022 for
M2b; this ADR does it for M3. It changes only what the build showed to be wrong
or incomplete, and names the evidence for each change. The findings, one line
each:

1. **The tenant allow list broke the golden path.** platform-config#21's list,
   which ADR-0021 §15 adopts, allowed DNS to `kube-dns` only. On this cluster a
   pod's lookups are answered by NodeLocal DNSCache, so with the default in
   place every lookup failed. Running pods kept serving and hid it; new pods and
   the GRANT Job failed, from 16:22 to 16:38 on 2026-10-31 (build log, C-34;
   fixed by platform-config#44).
2. **A pull request can switch off the required check that judges it.** On
   `pull_request` GitHub runs the workflow from the pull request's own head.
   edge-config#19 edited `.github/workflows/check.yml` beside a route the check
   refuses, and GitHub read it CLEAN. ADR-0021 §10's required reviewer covers
   `external/**` only, so nothing stopped it. It was not merged (build log,
   C-10).
3. **One timed-out webhook call left a Gateway unprogrammed.** GKE's Gateway
   controller failed to add its finalizer to `edge/external` because
   `validate.kyverno.svc-fail` timed out once, three minutes after Kyverno
   started, and it never tried again. The `edge` Application stayed Progressing
   and root's later wave never ran. Deleting the Gateway, which Argo CD restored
   in two seconds, recovered it (build log, Surprise 5; cycle 10).

## Decision

### 1. The DNS allow names every resolver path — supersedes ADR-0021 §15's allow list on DNS

`charts/system`'s default-deny NetworkPolicy allows egress on UDP and TCP 53 to
both `k8s-app=kube-dns` and `k8s-app=node-local-dns` in `kube-system`. Each is a
resolver path a GKE pod may use: `kube-dns` when NodeLocal DNSCache is off,
`node-local-dns` when it is on. `scripts/check_charts.py` asserts both rules,
with a mutant that removes the second (platform-config#44, `abc208e`). The rest
of the allow list, Pod Security `baseline` with `restricted` as warn and audit,
and the version pin stand as ADR-0021 §15 says.

### 2. C-10's headline, read with an exception until the workflow is protected

"CI holds the route files" is read with this exception: a pull request that
also edits the workflow producing `route files` can make the check pass. The
rules that only the check enforces are then not enforced on that pull request:
`[host-unique]`, `[service-unique]` and `[system]`, the cross-file ones. The
single-file rules still fail when Argo CD renders the chart, because the
schema refuses them.

Two guards were candidates:
- adding `.github/**` to the file patterns of the `security` team's required
  review in edge-config's ruleset;
- making the check an organisation required workflow, run from a repo the
  pull request cannot edit.

This ADR decides the first. It keeps the rule in the one ruleset C-09 already
tests, needs no second repo, and is recorded like every other ruleset change:
by hand, before and after, in `platform-factory/.github`. It is applied by the
owner, and until that record exists the exception stands. With one person on
both teams it shows that GitHub enforces the pattern, not that a second person
reads the workflow change, which is C-09's limit too.

### 3. A fail-closed webhook can strand a Gateway: accepted, and stated

Kyverno's resource webhook is fail-closed (ADR-0021 §9) and is registered for
the Gateway API kinds, because `deny-edge-objects` names them. A call that
times out refuses the write. When the refused write is GKE's Gateway controller
adding its finalizer, the controller does not requeue: the Gateway stays
"Waiting for controller" until its spec changes or it is recreated. An
annotation does not wake it [C, one attempt, cycle 10].

This ADR accepts that, with the fact stated, as ADR-0022 §3 did for what an
outage blocks. Recovery is deleting the Gateway so that Argo CD recreates it;
nothing in the cloud exists for an unprogrammed Gateway, so nothing is lost.
`cycle.sh up` does not do it for the operator: if it did, the step would still
be an intervention, and it would hide it. Two remedies are left open, because a
change to the webhook's scope is C-11's subject and needs a test of its own:
- scoping Kyverno's webhooks to tenant namespaces, so that the `edge`
  namespace's Gateways never meet them;
- an `up` check that names a Gateway still "Waiting for controller" after a
  fixed time, and counts it.

## Consequences

- C-34's ADJUSTED links §1. C-10's scope links §2. C-11's grade is unaffected,
  because the stranding is a webhook timeout, not a policy decision; the entry
  says so beside the grade and links §3.
- Not decided here: the SQL-injection form that Cloud Armor did not refuse (the
  build log records it, and the backend service had request logging off, so its
  source was not read); the public wildcard answering `internal.ref` names,
  which ADR-0021's Consequences predicted and which fails at the TLS handshake
  rather than with a 404; the FQDN source's implicit deny, which changes no
  decision of ADR-0021 §13; and `cycle.sh down`'s Gateway drain
  (platform-bootstrap#32), which is a script change, not a design decision.
- No grade of an earlier milestone changes, and no M3 ceiling moves.
