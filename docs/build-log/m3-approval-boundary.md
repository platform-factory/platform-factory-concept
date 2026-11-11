# M3: Approval boundary — closed 2026-11-03 (opened 2026-10-27; built 2026-10-29 to 2026-11-03; tested 2026-10-29, 2026-10-31 and 2026-11-03)

> **Status: closed 2026-11-03.** Every M3 test ran. The grades are the
> decisions of the maintainer's delegate on the data, under the maintainer's
> instruction to finish M3, and use ADR-0020's six words with the ceilings the
> readiness note fixed before the run. An independent reviewer (a separate
> model instance, given the register, ADR-0020, ADR-0021 and the raw data, and
> told to argue from the pre-committed rules) reached the same grade on every
> claim; its corrections to the data are taken into this entry.
>
> **How to read the markers.** `[C, source]` read or observed; `[I]` inferred;
> `[unverified]` not checked. All times are UTC.
>
> **Grades in one table** (reasoning under "Claims graded").
>
> | Claim | Grade, 2026-11-03 | Ceiling | What it rests on, and what is open |
> |---|---|---|---|
> | C-09 | ADJUSTED, superseding ADR = [ADR-0021](../adr/0021-the-approval-boundary-on-the-config-connector-engine.md) §10 | ADJUSTED | every path into `external/` through the rule was blocked; the one merge into `external/` was the owner lifting the rule for ten seconds, recorded and listed as slipped |
> | C-10 | ADJUSTED, scoped; [ADR-0021](../adr/0021-the-approval-boundary-on-the-config-connector-engine.md) §2, [ADR-0023](../adr/0023-what-the-m3-build-changed.md) §2 | ADJUSTED | all eight registered cases refused before merge; a counter-example outside them (#19): a pull request that edits the check's workflow passes it |
> | C-11 | ADJUSTED, superseding ADR = [ADR-0021](../adr/0021-the-approval-boundary-on-the-config-connector-engine.md) §9 | ADJUSTED | 33 attempts on three doors, 0 admitted; platform objects admitted; the route prune passed. A Gateway stranded by a webhook timeout is outside the claim ([ADR-0023](../adr/0023-what-the-m3-build-changed.md) §3) |
> | C-12 | PARTIAL | PARTIAL | the alert carries the namespace and the join took three lookups, none a search; "the PRs merged since the last healthy state" did not run, as registered |
> | C-13 | ADJUSTED, superseding ADR = [ADR-0021](../adr/0021-the-approval-boundary-on-the-config-connector-engine.md) §2 and §5 | ADJUSTED | every case held; one SQL-injection form was not refused by Cloud Armor and its source is not known |
> | C-14 | HELD | none | allowed host reached, `example.com` refused, available on Standard; 24 of 24 under churn |
> | C-24 | HELD | none | the sibling's grants in 75 of 75 polls through the move; connects with only the conditioned grants |
> | C-34 | ADJUSTED, superseding ADR = [ADR-0023](../adr/0023-what-the-m3-build-changed.md) §1 | none | the golden path broke for 15 minutes as shipped, the falsifier the register named; fixed by platform-config#44, then every case held |
> | C-08 | NOT REACHED | — | as the readiness note pre-committed: no governance decision, and no composition layer to swap |

## Archive

The register, its M3 readiness note, ADR-0020 and ADR-0021 were tagged
`m3-preregistered` on `main` at `5ff8568` and published as a GitHub release at
2026-10-29T05:31:04Z
(<https://github.com/platform-factory/platform-factory-concept/releases/tag/m3-preregistered>).
Software Heritage save request 2588830 was accepted at 05:31:05Z; its visit
(2026-10-29T10:55:46Z) holds snapshot
`swh:1:snp:dde88fbb5a2fa2aa71e3c4b055d2fd52c405d6ee`. The release and the save
came after M2b's Apply B (05:29Z) and before the first M3 build command, which
was the fix push on edge-config#6 at about 05:36Z (GitHub-only).

## Readiness and provenance

**What was registered, and when.** C-09..C-14 were registered on 2026-07-31,
C-24 on 2026-09-17 and C-34 on 2026-10-27. The M3 readiness note (2026-10-27)
gave each one its test, pass line, falsifiers and ceiling, before any M3 build
command. No claim text has changed since.

**The order held.** The readiness note says the cluster work runs in a fixed
order, and a later step does not start until the earlier one has its data. It
ran in that order: the layer-0 and layer-1 applies, the layer-2 rebuild (with
C-23's NAT re-read), C-34, C-11's cluster cases, C-13, C-24, C-12, C-14. The one
break in time is the wait for the Cloudflare delegation. On 2026-10-31 C-13's
internal cases ran, and then nothing more until the delegation was published on
2026-11-03. The cluster was taken down and rebuilt in between (cycle 10), and
C-13's internal cases were run again on the new Gateways before its external
half.

| Source | When | What it is | Standing |
|---|---|---|---|
| M3a at the desk | 2026-10-27 to 2026-10-29 | `edge-config`'s tree and check; C-09's and C-10's pull requests; the ruleset changes | Observed; `gh pr view` reads, ruleset JSON in `platform-factory/.github` |
| Session 1 | 2026-10-31, 15:21Z to 17:00Z | Layer-0 and layer-1 applies; rebuild (cycle 9); C-23's NAT re-read; C-34; C-11's cluster and repo doors; C-13's internal cases | Observed; `evidence/m3/`, `evidence/m3-applies/`, `cycle-results.tsv` |
| The wait | 2026-10-31 17:00Z to 2026-11-03 16:59Z | The delegation was not published. Watchers on public resolvers (2 h) and on Cloudflare's own nameserver (10 h) saw nothing. Cycle 10's `down` and `park` on 2026-11-01 | Observed |
| Session 2 | 2026-11-03, 17:11Z to 20:21Z | Rebuild (cycle 10); C-13 again, then its external half and the flip; C-24; C-12; C-14 | Observed |
| IAM poller | from 2026-10-27, through the close | Project and registry policies, median 13 s apart (target 10 s) | Observed; the C-24 data |

All times are UTC. A path under `evidence/` names a raw file kept with the
operator outside the repos. They hold IAM policies and addresses, so they are
not published; every fact cited from one is quoted here.

**What the instrument could not show.** One person holds every role, so
separated duties are not shown anywhere in M3 (each claim's "one person cannot
show" says what that costs it). The auto-mode classifier that runs this build's
commands refused merges on blocked PRs ("Merge Without Review") and the
ruleset lift ("CI Bypass"), so C-09's blocked results are GitHub's reported
merge state, not refused merge calls, and the flip's ruleset step was run by
the operator by hand. Kyverno's admission log at TRC level records the UPDATEs
of a pruned route and not the DELETE itself.

## Built

| Repo | PR | Merged | What |
|---|---|---|---|
| edge-config | #6 | 2026-10-29 05:38:39Z, `1f80629` | The route tree (`internal/`, `external/`, one folder per System) and `scripts/check_routes.py`, with must-fail fixtures |
| .github | #4 | 2026-10-29 05:52:07Z | Ruleset records for C-09's and C-10's changes |
| platform-config | #39 | 2026-10-31 15:02:37Z, `5dcf446` | `charts/route`, the `edge` AppProject and the two route ApplicationSets |
| platform-config | #40 | 2026-10-31 15:03:01Z, `f202c35` | The two Gateways, `deny-edge-objects`, `require-system-label` |
| platform-bootstrap | #30 | 2026-10-31 15:23:10Z, `3bae159` | The Terraform crossings ADR-0021 §12 declared |
| platform-config | #43 | 2026-10-31 16:18:57Z, `9a715cd` | `charts/route` writes out `backendRefs`' defaults (Surprise 2) |
| platform-config | #41 | 2026-10-31 16:21:32Z, `20e9ecf` | Tenant isolation by default (C-34): Pod Security `baseline` enforced, version pinned to v1.36; default-deny and the allow list |
| platform-config | #44 | 2026-10-31 16:37:17Z, `abc208e` | The allow list gains egress to NodeLocal DNSCache (C-34's first run) |
| platform-bootstrap | #32 | 2026-11-01 05:31:58Z, `1277011` | `cycle.sh down` drains what Gateways made before it destroys the cluster (Surprise 4) |
| edge-config | #7 | 2026-11-03 19:10:58Z, `b1c3f99` | The flip: one `git mv` of `hello-internal` into `external/` |
| .github | #5 | 2026-11-03 19:12:04Z | Ruleset records around the flip |
| platform-config | #42 | 2026-11-03 19:34:05Z, `b6ec260` | Cloud SQL grants per database, conditioned on the instance name (C-24) |

Test fixtures, each reverted or recorded: svc-ledger#6–#9 (C-11's repo door),
edge-config#8–#19 (C-09 and C-10 cases, left open), systems#23 and #24 and
svc-ledger#10 (C-24's fixture and its move).

**Terraform crossings (C-01).** Exactly the ones ADR-0021 §12 declared, one
apply each on 2026-10-31. Layer 0: 2 adds (the `certificatemanager` and `dns`
APIs; `networkservices` was not needed), 24 s. Layer 1: 14 adds (the two zones,
the two wildcard records, the DNS authorization's CNAME record, the global and
regional addresses, the proxy-only subnet, the DNS authorization, the
certificate, the certificate map and its entry, the Cloud Armor policy and the
alerting policy), 35 s. Layer 2's three settings (Dataplane V2, the Gateway
API's standard channel, FQDN policy) rode the rebuild and read back as set.
The plans matched the declaration line for line. Manual steps that are not
applies: the Cloudflare delegation (the operator's) and the ruleset changes
(by hand). GKE created its own `gkegw1-*` firewall rules for the load
balancers (130.211.0.0/22 and 35.191.0.0/16, plus the proxy-only subnet
regionally); nothing in Terraform names them.

**Rebuilds (C-02's data, recorded in `cycle-results.tsv`).**

| Cycle | Step | Seconds | Interventions | Note |
|---|---|---|---|---|
| 9 up, 2026-10-31 | 2-cluster, 3-argocd, verify | 580, 113, 2402 (timeout) | 1 | 11/12 ready: the route Application looped OutOfSync (Surprise 2); fixed by #43 |
| 10 down, 2026-11-01 | edge drain by hand, then 3-argocd, 2-cluster | about 780 by hand, then 46, 535 | the drain's steps (Surprise 4) | first teardown with Gateways |
| 10 park | | 55 | 0 | one instance stopped |
| 10 up, 2026-11-03 | 2-cluster, 3-argocd, verify | 615, 101, 2401 (timeout) | 2 | a Gateway stranded by one webhook timeout (Surprise 5); everything ready 65 min after the start |
| 11 down, 2026-11-03, M3's close | edge drain (scripted, #32), 3-argocd, 2-cluster | 242, 46, 586 | 0 | the drain's first live run: two forwarding rules and svc-hello's NEGs in three zones cleared, nothing left |
| 11 park | | 113 | 0 | both instances stopped |

A first attempt at cycle 10's `up` (17:12:32Z) failed in 0 s. The operator's
working copy had lost files to macOS's `/tmp` cleanup (Surprise 7). That says
nothing about the platform, and the row says so.

## Data

### C-09 — The flip forces review

The ruleset `main-protection` (edge-config, id 20204888) gained a required
reviewer on 2026-10-29 at 05:40:28Z: team `security`, one approval, files
`external/**` (history version 45124681 → 55281350). The team has write
access, so its approvals count. Its only member is the one person who writes
every pull request here.

| Case | PR | GitHub's merge state | Result |
|---|---|---|---|
| The flip, `git mv internal/svc-hello/hello-internal.yaml external/svc-hello/` | #7 | BLOCKED, `security` requested | blocked |
| A new file straight into `external/` | #8 | BLOCKED | blocked |
| A rename into `external/` under another name | #9 | BLOCKED | blocked |
| The workflow that produces the required check edited (check step replaced by `echo`), plus a file in `external/` | #18 | BLOCKED, `security` requested | blocked |
| The same workflow edit, plus an internal route the check refuses (outside C-09's pass line) | #19 | **CLEAN, mergeable** | the required check passed because the PR changed it |
| The owner lifts the rule (the flip, 2026-11-03) | #7 | merged after the lift | **slipped**, as the readiness note says to list it |

No merge was attempted on a blocked PR; the classifier refused the calls. The
result is GitHub's reported state. #7, the same pull request, merged without
`--admin` eight seconds after the rule was lifted, so BLOCKED meant blocked.

**The flip (2026-11-03).** The operator ran the ruleset step by hand
(`evidence/m3/c13-flip.txt`; records in `.github` #5): version 55281693 before;
`required_reviewers` emptied at 19:10:50Z (56118778); #7 merged without
`--admin` at 19:10:58Z; restored at 19:11:00Z (56118811). The restored ruleset
equals the one before, compared with `jq -S`. The rule was off for ten seconds.

**#19** (its data is under C-10, whose counter-example it is). On `pull_request` GitHub runs the workflow from the pull request's own
head. A pull request that edits `.github/workflows/check.yml` can make
`route files` pass while it carries a route the check refuses. Nothing under
`external/` changes, so the required reviewer does not apply. The route
chart's schema still refuses every single-file case when Argo CD renders it.
The cross-file cases (a duplicate host, two external routes on one Service, an
unknown System) would reach `main`. Two remedies were noted and are not
adopted in M3: add `.github/**` to `security`'s required-reviewer patterns, or
make the check an organisation "required workflow" from a repo the pull request
cannot edit.

### C-10 — CI can hold the route files

The check failed for real on #10–#17 and passed for real on #6 and #7–#9. It
was then made a required status check, at 2026-10-29 05:45:41Z (context
`route files`, GitHub Actions; version 55281693). Before that, a failing
internal-only PR was UNSTABLE, so it could have merged; after, BLOCKED.

| Case | PR | Schema (Helm 3.19.4, `charts/route`) | Check |
|---|---|---|---|
| `exposure` key in `internal/` | #10 | refused (additional property) | refused |
| `exposure` key in `external/` | #11 | refused | refused |
| host outside the grammar (`Hello_World`) | #12 | refused (pattern) | refused |
| the same host in both folders | #13 | accepted | refused |
| a System folder with no tenant file (`svc-nobody`) | #14 | accepted | refused |
| an unknown key (`timeout`) | #15 | refused | refused |
| a non-integer port (`"80"`) | #16 | refused (type) | refused |
| two external routes in one System on the same Service | #17 | accepted (each file alone) | refused |

No case was accepted by both. Every case is also a must-fail fixture in
`edge-config/tests/check-fixtures/`. The first CI run of #6 failed for a reason
in the workflow (the `systems` checkout sat inside the walk), fixed in
`2b93f87`, and is not counted as the check's "failed once". #19 (above)
carries an internal route the check refuses, a duplicate host
(`hello-internal`) in `internal/svc-ledger/`, past a check the same PR edited.

**The counter-example outside the registered cases (#19).** #19 carries a
duplicate host across two Systems in `internal/`, a file the check on `main`
refuses (`[host-unique]`), and edits `.github/workflows/check.yml` so that the
`route files` step only echoes. GitHub read it CLEAN and mergeable: the required
check was satisfied by the pull request's own workflow, and nothing under
`external/` changed, so no reviewer was required. It was not merged. The schema
accepts the file (it is valid alone), so merged, a duplicate host would have
rendered. The class this opens is the cross-file rules (`[host-unique]`,
`[service-unique]`, `[system]`); single-file cases still fail when Argo CD
renders the chart. The remedies are in C-09 above; ADR-0023 §2 decides one.

### C-11 — Every other door to the edge is refused

Run on 2026-10-31 at 16:44Z on the rebuilt cluster, Gateway API channel on, so
every kind `deny-edge-objects` names exists (`evidence/m3/c11-kubectl.txt`,
`c11-repo-door.txt`).

Eleven cases, three doors, 33 attempts, **0 admitted**. Who refused each:

| Case | Team (`kubectl --as`) | Operator (`kubectl`) | Service repo (`k8s/`, through Argo CD) |
|---|---|---|---|
| Service `LoadBalancer` | Kyverno | Kyverno | Kyverno (`deny-edge-objects`, #8) |
| Service `NodePort` | Kyverno | Kyverno | Kyverno (#8) |
| `Ingress` | Kyverno | Kyverno | AppProject (#6) |
| `Gateway` | RBAC | Kyverno | AppProject |
| `HTTPRoute` | RBAC | Kyverno | AppProject |
| `ReferenceGrant` | RBAC | Kyverno | AppProject |
| `GCPBackendPolicy` | RBAC | Kyverno | AppProject |
| cloud kind, no System label | Kyverno: `require-system-label`, `deny-raw-cloud-resources` | the same two | AppProject |
| cloud kind, another namespace's label | Kyverno: the same two | the same two | AppProject |
| route kind, no System label | RBAC | Kyverno: `require-system-label`, `deny-edge-objects` | AppProject |
| route kind, another namespace's label | RBAC | Kyverno: the same two | AppProject |

The team identity is refused by RBAC first for the Gateway API and GKE kinds,
because they do not aggregate into `admin`; that is why the operator's own
`kubectl`, which no rule exempts, is the door that tests the policy. On it every
case names a Kyverno policy, and every label case names `require-system-label`,
as the pass line requires (`evidence/m3/c11-kubectl.txt`).

On the service-repo door, svc-ledger#6 (`f4e4aee`) carried nine of the cases,
and Argo CD's AppProject refused all nine ("resource X is not permitted in
project svc-ledger"). Argo validates a whole sync before applying any of it, so
the two Services in the same commit were never sent. Alone in #8 (`9efcfb7`),
both were refused by Kyverno with `deny-edge-objects`, read through Argo's sync
result. #7 and #9 reverted them; nothing was left in the namespace. Argo's retry
pinned the old revision twice; `terminate-op` cleared it
(`evidence/m3/c11-repo-door.txt`).
- **Platform objects admitted:** the two Gateways, the routes, the
  GCPBackendPolicy and svc-hello's cloud objects, all Synced.
- **The data case, a route file removed (at the flip, 2026-11-03):** the old
  HTTPRoute was pruned 148 s after the merge with no Kyverno refusal and no
  PolicyViolation event. Kyverno's log shows the old route's UPDATE at
  19:13:21Z by an identity holding
  `system:controller:generic-garbage-collector`, passed (the exemption M2b's
  Surprise 1 added), and the new HTTPRoute and GCPBackendPolicy CREATEs by
  Argo CD's controller at 19:13:45Z, passed. The DELETE is not in the log at
  this level [I].

### C-34 — Tenant namespaces are isolated by default

`evidence/m3/c34-*`, 2026-10-31.

- **Before** (16:20Z, no policy): from svc-ledger, svc-hello's Service name,
  ClusterIP and pod IP all answered 200.
- **#41 merged at 16:21:32Z**, Pod Security pinned to the cluster's version,
  v1.36, read from the rebuilt cluster.
- **The golden path broke, as shipped.** The policies were in by 16:22:50Z. The
  running svc-hello pod kept serving, because its connections were already
  open, and that hid it. New pods and new Jobs failed. A restarted pod sat at 1/2
  for fifteen minutes ("database not ready", readiness 503), and a fresh GRANT
  Job retried "no private IP for svc-hello-main yet". Every name lookup failed:
  pods on this cluster resolve through NodeLocal DNSCache, and the allow list
  named only `kube-dns`. A test policy allowing `k8s-app=node-local-dns`
  restored names. #44 (16:37:17Z) added egress to it on UDP and TCP 53, with a
  check and a mutant. Broken from 16:22 to 16:38.
- **After #44:** svc-hello's `/healthz` 200; a restart succeeded; a fresh GRANT
  Job completed in about 15 s.
- **(a) after:** from svc-ledger, svc-hello's Service name, ClusterIP and pod
  IP all timed out. So did svc-ledger to its own pod (in-namespace traffic is
  denied too), and port 80 to anywhere. Google APIs on 443 answered.
- **(c):** a privileged pod, by the operator and by the team, refused:
  `violates PodSecurity "baseline:v1.36": privileged`.
- **(d), a tenant-written allow-all in svc-ledger:** it opened svc-ledger's own
  pods, its own egress on port 80 to anywhere, and its pods to a namespace with
  no policy. It did not open svc-ledger to svc-hello, svc-hello to svc-ledger,
  or a no-policy namespace to svc-hello. "Opens only that tenant's own pods" is
  true with respect to other tenants; the allow-all also opens the tenant's
  egress and its ingress from unpoliced namespaces. A policy a tenant adds is
  not in git and stays until deleted. **Two prediction misses:** the
  registered prediction "(d) opens only the tenant's own pods" was partly wrong
  (it also opens the tenant's egress and its ingress from unpoliced namespaces),
  and the prediction did not say that the default also denies traffic inside a
  namespace.
- **On 2026-11-03** the external load balancer reached svc-hello through the
  same allow list: GKE's own firewall rules use 130.211.0.0/22 and
  35.191.0.0/16, the ranges the list admits.

### C-13 — The edge model works on GKE

- **Internal** (2026-10-31 16:54:51Z, and again on cycle 10's new Gateways at
  2026-11-03 18:22:14Z): from a pod in the VPC,
  `hello-internal.internal.ref.thecloudgeek.io` resolved to 10.10.0.8, `/healthz`
  answered 200 on port 80, and an unknown host got 404. On the new Gateways the
  route served about three minutes after its Application existed; before that,
  404 "fault filter abort".
- **The delegation.** The operator's four NS records were in place by 16:59Z
  on 2026-11-03 (when they were added was not observed), read on Cloudflare's nameserver and on 8.8.8.8, 1.1.1.1 and
  9.9.9.9. The DNS-authorization CNAME resolved publicly.
- **The certificate.** `edge-ref-wildcard`'s authorization attempt at 17:04:01Z
  read AUTHORIZED, and the certificate was ACTIVE when read at 17:11Z, with the
  cluster down (it is layer 1's). When the records were published was not
  observed; they were in place by 16:59Z. Issuer Google Trust Services WR3,
  valid to 2027-02-01. Served on 443 as `CN=*.ref.thecloudgeek.io`.
- **The flip** (`evidence/m3/c13-external.txt`): merged 19:10:58Z. The old
  HTTPRoute was gone at 19:13:26Z (148 s, the ApplicationSet's git poll); the
  new one, `hello-internal-external`, parent `external`, hostname
  `hello-internal.ref.thecloudgeek.io`, was Accepted at 19:14:19Z (201 s). The
  route's Gateway changed. GCPBackendPolicy `svc-hello` Attached; the external
  backend service carries security policy `edge-external`, the internal one
  none.
- **From outside:** the name resolved on 8.8.8.8 to 34.120.72.225. HTTPS
  answered 404 until 19:20:27Z, then 200: 569 s from merge to serving. `/healthz`,
  `/` and `/notes` (which reads the database) 200. An unknown host
  (`nobody.ref.thecloudgeek.io`) 404. Plain HTTP: no listener, empty reply.
- **Cloud Armor** (request logging is off on the backend service, so no log
  says which component answered): `' OR '1'='1`, `UNION SELECT 1,2,3`, `;DROP TABLE users` and
  a bare `--` each got Armor's 403 page. **`1 UNION SELECT password FROM users`,
  with or without `--`, got a 404 "fault filter abort"** of 18 bytes: neither
  Armor's 403 nor svc-hello. Its source was not determined [I]. A benign query
  got 200.
- **After the flip, inside the VPC:** the internal name answered 404 (the route
  left the internal Gateway).
- **The hostname schema, as ADR-0021 predicted, with one detail different.**
  The public wildcard `*.ref.thecloudgeek.io` also answers names under
  `internal.ref` from outside, so `hello-internal.internal.ref.thecloudgeek.io`
  resolves publicly to the external address. ADR-0021's Consequences say so and
  call it harmless: "the answer is the external IP and a 404". The 404 is the
  detail that differs: such a request fails at the TLS handshake, because the
  certificate map has no entry that covers two labels. Nothing internal is served
  either way.
- **Gateway findings.** (1) One timed-out call to a fail-closed admission webhook
  left a Gateway unprogrammed for good (Surprise 5). (2) Merge to serving was
  569 s: about 148 s of ApplicationSet polling and about six minutes of load
  balancer programming. (3) GKE's controller acted on a spec change, not on an
  annotation [I, from one nudge].

### C-24 — Per-System grants are independent

Deviation: `svc-ledger` stands in for the retired `svc-onboard`.

- **The fixture** (2026-11-03): systems#23 moved svc-ledger to `payments`
  (19:22:19Z) and svc-ledger#10 gave it one database (19:22:24Z). Instance
  `svc-ledger-main`, the second billed instance the register accepts, was
  created at 19:23:18Z and RUNNABLE at 19:32:00Z; its claims Application was
  Healthy at 19:33:38Z.
- **Before #42:** one unconditioned binding per role, holding `group:payments`
  and both Systems' service accounts, owned by eight IAMPolicyMembers in the two
  Systems' `charts/system`. svc-hello's and svc-ledger's team members were the
  same policy member. No ownerless M2 binding.
- **#42 merged at 19:34:05Z.** From the poller: svc-ledger's conditioned pairs
  appeared at 19:36:34Z; the unconditioned `group:payments` and svc-ledger
  account went at 19:36:47Z; svc-hello's account at 19:37:13Z; svc-hello's
  conditioned pairs at 19:39:10Z. **A changeover window:** svc-hello's account
  had no Cloud SQL grant for about 117 s, and the team group on svc-hello's
  instance for about 143 s. The group went when svc-ledger's prune removed the
  shared member: M2's shared-grant defect, on its way out. The window was named
  in #42 before the merge. svc-hello's `/healthz` was not probed during it.
- **Before the move,** read from the cloud: four bindings, all conditioned,
  `resource.name == "projects/platform-factory-ref/instances/svc-hello-main"`
  and `.../svc-ledger-main`, each with the team group and that System's account.
  No unconditioned grant for any application or team identity, so nothing was
  removed.
- **The move:** systems#24 moved svc-ledger back to `checkout` at 19:40:41Z. In
  the 75 polls to 19:55:41Z (median 13 s, max 14 s), **svc-hello's four grants
  were present in all 75: absent in 0 of 75 polls, at 13 to 14 s spacing**
(the register asks for 10 s; M2 baseline: about 300 s). svc-ledger's
  team grant changed from `payments` to `checkout` within one poll, at 19:44:41Z.
  svc-hello's `/healthz` answered 200 each minute.
- **The connect test** (19:57Z): with no unconditioned grant in the policy, a
  restarted svc-hello pod's Auth Proxy accepted its first connection four
  seconds after starting; the application logged "database ready ... (attempt
  1)"; `/healthz` (`SELECT 1` through the proxy) and `/notes` answered 200. The
  proxy logged no permission error. A fresh GRANT Job also succeeded. No API call
  broken by the condition was seen.

### C-12 — The spine joins an alert to a team and a repo

- **Steady state:** no incident opened on 2026-11-03 before the test. On
  2026-10-31 the policy opened one on `kyverno` (16:13:58Z to 16:23:43Z) when
  Kyverno's controllers restarted during cycle 9; that namespace has no System
  label, so its join ends at the namespace.
- **The synthetic incident:** a pod in svc-hello that exits 1, started at
  19:58:59Z, admitted (Pod Security warned at `restricted`, enforces
  `baseline`). The incident opened at 20:05:48Z, 409 s later, at six restarts
  (the threshold is more than three in 600 s). It carries the resource labels
  `project_id`, `location`, `cluster_name` and `namespace_name=svc-hello`.
- **The join, each step on a key already in hand:** the alert gives the
  namespace; the namespace's `platform.thecloudgeek.io/system` label gives
  `svc-hello`; `systems/tenants/svc-hello.yaml` gives team `payments` and repo
  `platform-factory/svc-hello`. Hops that needed a search or a guess: **0**. The
  second step needs read access to the cluster; the label key is also in the
  policy's user labels.
- No notification channel exists, by design (ADR-0021 §11); the incident was
  read through the Monitoring API. "The PRs merged since the last healthy state"
  was not tested, as the readiness note says.

### C-14 — FQDN egress policy

- **The primary source, re-read on 2026-11-03** (GKE "FQDN network policies"
  how-to): it needs Dataplane V2 and kube-dns or Cloud DNS; it names no edition
  or Enterprise requirement; it does not mention NodeLocal DNSCache; Autopilot
  enables it only by updating an existing cluster; a Standard cluster enabling
  it later restarts `anetd`. It adds two things to ADR-0021 §13's reading of
  2026-10-27, and contradicts none: an FQDNNetworkPolicy carries an implicit
  deny for the pods it selects, and it does not affect DNS. It honours the
  record's TTL, keeps established connections, allows at most 50 addresses per
  policy, and a shared address opens every domain on it.
- **The cluster:** Standard, 1.36.4, Dataplane V2, FQDN policy enabled at
  creation, CRD served at v1alpha1.
- **The test,** in a namespace with no System label (so outside C-34's
  policy), one client pod: with no policy, `www.google.com` and `example.com`
  both answered. Under a default-deny egress that allows only DNS, plus an
  FQDNNetworkPolicy allowing `www.google.com` on TCP 443: **`www.google.com:443`
  200; `example.com` refused** (timed out); `www.google.com:80` refused. With the
  FQDN policy alone, the same results: the implicit deny.
- **Under churn:** every 30 s for twelve minutes, 24 of 24 connections to
  `www.google.com` succeeded across seven different addresses (TTL 190 s). The
  lookup and the connection often landed on different addresses, and none
  failed.

### C-23 — re-read on the rebuilt cluster

From the rebuild to the end of session 1 (15:35Z to 16:55Z on 2026-10-31), 210
NAT log entries: 209 to GitHub on 443, one to `example.com` on 80 from C-34
(d)'s own probe under the tenant allow-all. No public registry. C-23's grade
stands on the rebuilt cluster.

## Claims graded

### C-09 — The flip forces review

**Grade: ADJUSTED** (2026-11-03; ceiling ADJUSTED; superseding ADR
[ADR-0021](../adr/0021-the-approval-boundary-on-the-config-connector-engine.md)
§10, which puts the rule in a ruleset's required reviewer, not the CODEOWNERS
the claim names). The pass line is that no change under `external/` reaches
`main` by any path through the rule without an approval from `security`, and
every case is recorded as blocked or slipped. The flip, a new file, a rename and
a workflow edit with an `external/` file were all BLOCKED (#7, #8, #9, #18). The
one change that reached `external/` was the flip after the owner lifted the rule
for ten seconds, which the readiness note says to record from the history
endpoint, list as slipped and not score, because the owner role is a bypass in
itself. The falsifiers are not met: nothing under `external/` reached `main`
through the rule, and no C-11 case was admitted.
**What keeps this from being cleaner:** one person is the author and the only
member of `security`, so the run shows that GitHub enforces a path-scoped rule,
not that duties are separated. A permitted flip could not be produced at all
without the owner's bypass, as the readiness note predicted. #19 is not a C-09
failure, because it changes nothing under `external/`; it is C-10's
counter-example.

### C-10 — CI can hold the route files

**Grade: ADJUSTED, scoped** (2026-11-03; ceiling ADJUSTED; ADR-0021 §2, the
folder as the only input, and
[ADR-0023](../adr/0023-what-the-m3-build-changed.md) §2). Cell words: "all eight
registered cases refused before merge, by the schema or the check; the check
became required after it failed and passed; a counter-example outside the
registered cases: a pull request that edits the workflow producing the check
passes it, so the cross-file rules are not enforced on it (#19, not merged);
ADR-0023 §2 decides the guard."
The registered falsifier, any case accepted by both, is not met: #13, #14 and
#17, which the schema accepts, were refused by the check. #19 is outside the
registered list. It is graded at the ceiling with the counter-example in the
cell, the shape M2b used for C-33 (x), because the registered text does not
reach it and the claim's headline does not hold against it. The headline is
read with ADR-0023 §2's exception until the guard's record exists.

### C-11 — Every other door to the edge is refused

**Grade: ADJUSTED** (2026-11-03; ceiling ADJUSTED, because ADR-0021 §9
re-scopes "denies untagged resources" to cloud and route kinds). Every
violating object was refused at the door it used, 33 of 33, with the message
recorded. On the operator's `kubectl` every label case names
`require-system-label`. The platform's own Gateways, routes, GCPBackendPolicy
and cloud objects were admitted, and the route prune at the flip passed the
garbage collector's exemption. Falsifiers not met.
**Beside the grade:** cycle 10's stranded Gateway was a platform object's
update refused by Kyverno's webhook, by a timeout and not by a policy. The
readiness note's dated 2026-10-31 entry accepts Kyverno's failure modes for
C-11, so it is not scored here; ADR-0023 §3 states it. What a real team
member's RBAC allows beyond `admin` is not shown, as registered.

### C-12 — The spine joins an alert to a team and a repo

**Grade: PARTIAL** (2026-11-03; ceiling PARTIAL). The named cases held: the
synthetic incident fired the layer-1 policy 409 s after the restart loop
started, the incident carries `namespace_name`, and the join reached team and
repo in three lookups on keys in hand, with zero hops searched or guessed. The
case that did not run is the one the readiness note names: "the PRs merged since
the last healthy state", which needs a deploy event that does not exist and
moves to M4. The design is kept. What one person cannot show, that someone other
than the builder can follow the join, is not shown.

### C-13 — The edge model works on GKE

**Grade: ADJUSTED** (2026-11-03; ceiling ADJUSTED; ADR-0021 §2 makes the folder,
not an `exposure:` field, select the policy, zone and class, and §5 replaces
external-dns with wildcard records). Every case held: the external name
resolves outside to the external address; the wildcard certificate was ACTIVE
(authorized 17:04:01Z) and HTTPS reached svc-hello; SQL-injection patterns got
Cloud Armor's 403; an unknown host got 404; the internal name resolved in the
VPC and answered on port 80; the flip moved the route from the internal Gateway
to the external one. Armor attached through the rendered GCPBackendPolicy, read
from the backend service. Falsifiers not met.
**What keeps this from being cleaner:** one form,
`1 UNION SELECT password FROM users`, got a 404 from the load balancer instead
of Armor's 403, and the source was not read (request logging is off). The
public wildcard answers names under `internal.ref`, as ADR-0021 predicted, and
such a request fails at the TLS handshake rather than with the predicted 404.
One environment, one route, one service.

### C-14 — FQDN egress policy

**Grade: HELD** (2026-11-03; no ceiling). The primary-source check is recorded,
with what it adds to ADR-0021 §13's reading. On the reference cluster (Standard,
Dataplane V2, enabled at creation, no edition requirement) a pod under a
default-deny egress with an FQDN allow for `www.google.com:443` reached it and
was refused `example.com`, in a namespace outside C-34's allow list. Under
twelve minutes of churn across seven addresses, 24 of 24 connections succeeded.
Falsifiers not met. Beyond the registered shape, the FQDN policy alone gave the
same results, as the source says.

### C-24 — Per-System grants are independent

**Grade: HELD** (2026-11-03; no ceiling). Through svc-ledger's team move,
svc-hello's four conditioned grants were present in all 75 polls (13 to 14 s
apart; the register asks for 10 s), against M2's 300 s absence. With only the
conditioned grants in the policy, read from the cloud, a new svc-hello pod
connected through the Auth Proxy and served `SELECT 1`. The condition as
shipped names exactly one instance, `resource.name ==
"projects/platform-factory-ref/instances/svc-hello-main"`. No API call broken
by the condition was seen. Falsifiers not met.
**What keeps this from being cleaner:** svc-ledger stood in for the retired
svc-onboard. The changeover merge before the test (#42) left svc-hello without
a Cloud SQL grant for about two minutes, through M2's shared member: migration
cost, named in #42 beforehand, and not the move the claim is about. An overmatch
between names that extend each other was not tested, as registered.

### C-34 — Tenant namespaces are isolated by default

**Grade: ADJUSTED** (2026-11-03; no ceiling; superseding ADR
[ADR-0023](../adr/0023-what-the-m3-build-changed.md) §1). The register names
"the golden path breaks" as a falsifier, and as shipped it did: with the
default in place no new pod could resolve a name, from 16:22 to 16:38 on
2026-10-31, because the allow list missed NodeLocal DNSCache. HELD is ruled
out by that. The design was changed, by platform-config#44 and ADR-0023 §1, and
after it every case held: no cross-tenant connection by Service name, ClusterIP
or pod IP; svc-hello and a fresh GRANT Job green; a privileged pod refused by
`baseline:v1.36`. ADR-0020 names this shape, a gate that fails its first live
test and is fixed in the hour, as ADJUSTED (C-07). Two parts of the prediction
were wrong: a tenant's allow-all also opens its egress and its ingress from
unpoliced namespaces, and the default denies traffic inside a namespace too.
Hostile-tenant isolation is not claimed.

### C-08

**Grade: NOT REACHED** (2026-11-03), as the readiness note pre-committed. Two
preconditions were not met inside the milestone: the governance decision the M2
grading note records as undecided, and a composition layer to swap, which
ADR-0017 removed. The claim stays open, or a later ADR retires it.

### What the grades add up to

Eight M3 claims and C-08 are graded: two HELD, five ADJUSTED (four at the
ceiling the readiness note fixed, one by a change the build made), one PARTIAL
at its ceiling, and one NOT REACHED. None is UNSUPPORTED or WRONG. The ceilings
did the work they were set for: with one person on both sides, C-09, C-10, C-11
and C-13 could not have shown more. The findings that matter for the design are
the two the build made by looking where the platform would lie: a required check
that the pull request under judgement can edit (#19), and an allow list that
looked right on a running pod and broke every new one.

## Surprises

1. **A pull request can switch off the required check that judges it (#19).**
   On `pull_request`, GitHub runs the workflow from the pull request's head, so
   a change to `.github/workflows/check.yml` in the same pull request decides
   what `route files` reports. Path-scoped review covers `external/**` and
   nothing else. Single-file cases still meet the chart's schema at render time;
   cross-file cases do not. Found at the desk on 2026-10-29, before the
   cluster half.
2. **The first route Application never settled (#43).** Argo CD synced
   `edge-internal-svc-hello-hello-internal` every five minutes, each a
   server-side apply, and it stayed OutOfSync while Argo's own `app diff`
   printed nothing. The live object differed from the render only in the API's
   defaults on `backendRefs` (group, kind, weight). Writing them out fixed it in
   seconds. It cost cycle 9's verify step its timeout.
3. **The allow list missed NodeLocal DNSCache (#44).** On this cluster a pod's
   DNS is answered by the `node-local-dns` pod on its node, not `kube-dns`, and
   platform-config#21's list named only `kube-dns`. A running pod hid it,
   because established connections kept working; a restart showed it. Fifteen
   minutes of a broken golden path, 16:22 to 16:38 on 2026-10-31.
4. **`down` had no step for what Gateways make (#32).** M3 is the first
   milestone with Gateways at teardown. A cluster destroy tells nobody to clean
   up, so the forwarding rules would have stayed on layer 1's reserved
   addresses and the NEGs for good [I: not allowed to happen]. Drained by hand
   on 2026-11-01: forwarding rules went within two minutes of deleting the
   Gateways, and the NEGs only after GKE's `cloud.google.com/neg` annotation was
   removed from the Service. #32 scripts it and refuses to destroy while any
   remain or the cloud cannot be read; its test found that `gcloud`'s
   aggregated lists exit 0 on a failed request. Its first live run, M3's
   closing `down` (cycle 11), took 242 s with no intervention and left no
   forwarding rule, proxy, URL map, backend service, NEG or health check;
   both addresses read RESERVED.
5. **One webhook timeout stranded a Gateway (cycle 10).** At 17:30:03Z on
   2026-11-03 GKE's Gateway controller failed to add its finalizer to
   `edge/external`: "failed calling webhook validate.kyverno.svc-fail ...
   context deadline exceeded". Kyverno had been up three minutes and never
   restarted. The controller did not try again: the Gateway sat at "Waiting for
   controller", generation 1 and no finalizer, for 48 minutes, while
   `edge/internal` resynced every few minutes. `edge` stayed Progressing, so
   root's later wave (the `edge` AppProject and the route ApplicationSets) never
   ran. An annotation did nothing; deleting the Gateway, which Argo CD put back
   in two seconds, did. Two interventions. Not fixed in M3. ADR-0021's
   Consequences name the cost of `failurePolicy: Fail` for tenant deploys; this
   is the same cost reaching a platform object through another controller.
6. **Kyverno's controllers restarted together during cycle 9 (2026-10-31,
   16:09Z),** after "failed to validate certificates ... context deadline
   exceeded" against the API server while the addon manager re-patched CRDs [I:
   a control-plane event]. C-12's alert policy opened an incident on `kyverno`
   for it. That is the policy working: a platform namespace, no System label,
   so the join ends there.
7. **The operator's working copy was eaten (2026-11-03).** The session's
   scratch space sat under `/private/tmp`, and macOS's tmp cleanup deletes files
   not read for three days. Five of layer 2's `.tf` files, git objects in every
   clone, the poller's pid file and older drafts went; the M3 evidence, written
   on 2026-10-31, was inside the window and survived. The first `up` failed in
   0 s on it. All repos were re-cloned and checked, and the evidence copied out
   of `/tmp`. It says nothing about the platform.
8. **The delegation took three days.** The four NS records were added on
   2026-11-03; watchers on 2026-10-31 and 11-01 (12 hours together) saw
   nothing, and cycle 10's teardown was for the wait. About 12 hours of cluster
   ran with no test.
9. **Cloud Armor refused three SQL-injection forms and not a fourth** (C-13).
   `1 UNION SELECT password FROM users` got 404 "fault filter abort" from the
   load balancer, not Armor's 403; it never reached svc-hello. Cause not
   determined.
10. **A shared member removed the sibling's grant one last time** (C-24's
    changeover). Before #42, both Systems' team grants were one member of one
    unconditioned binding, so svc-ledger's prune took svc-hello's away. That is
    M2's finding, and the reason for C-24; it showed once more in the
    migration, for about two minutes, before the test.
11. **The classifier refused the flip's ruleset step as "CI Bypass"** although
    the operator had installed a rule for it, and earlier refused merges on
    blocked PRs. The operator ran the step. The refusals made the record
    better, not worse: every blocked result is GitHub's own state.

## ADR errata found by the build

- **ADR-0021 §15 / platform-config#21:** the allow list's DNS rule named only
  `kube-dns`. On GKE with NodeLocal DNSCache it needs `k8s-app=node-local-dns`
  too (#44). ADR-0023 §1 states the change.
- **ADR-0021 §13:** the primary source adds that an FQDNNetworkPolicy carries an
  implicit deny and leaves DNS alone. §13's test (a default-deny plus an allow)
  still holds; the separate default-deny is not needed for the refusal.
- **ADR-0021, Consequences, "Wildcard DNS":** a deeper name under `internal.ref`
  asked from outside fails at the TLS handshake, not with a 404.
- **ADR-0021, Consequences, `failurePolicy: Fail`:** the cost reaches platform
  objects too, through GKE's Gateway controller, which does not requeue after a
  refused finalizer patch (Surprise 5).
- **ADR-0021 §10:** a required check is enforced only as far as the workflow that
  produces it is protected; the PR under judgement can edit it (Surprise 1).

## Deviations from the readiness note

- **One `external/` merge, not two.** ADR-0021 §7 (one GCPBackendPolicy per
  Service) refuses the flipped `hello-internal` beside an external `hello` on
  the same Service, so C-13's external route is the flipped one. The flip stays
  a pure `git mv`.
- **`svc-ledger` stands in for `svc-onboard`** in C-24, because M2b retired
  `svc-onboard`.
- **The IAM poller's interval was 13 s, not 10 s** (median; maximum 14 s during
  the move).
- **C-13's internal cases ran twice,** on cycle 9's Gateways and on cycle 10's,
  because the delegation wait put a rebuild between C-13's halves.
- **C-14 also ran the FQDN policy alone,** beyond the registered shape, because
  the primary source says it carries its own deny.

## What is pending, in one place

- Delete or keep `svc-ledger-main`, C-24's second instance: parked at this
  close; deletion is the operator's.
- **#19's guard, decided by ADR-0023 §2:** the owner adds `.github/**` to the
  `security` team's required-review patterns in edge-config's ruleset and records
  it in `platform-factory/.github`. Until that record exists, C-10's headline
  reads with the exception.
- Not decided: a remedy for the stranded Gateway (ADR-0023 §3 leaves two open),
  and for the SQL-injection form Armor did not refuse.
- Done at this close: `cycle.sh down` (cycle 11), whose scripted drain ran
  live for the first time (242 s, no intervention, nothing left), and `park`
  of both instances (113 s).
- C-32 (M2b) still waits on the billing export.

## Post-close addendum, 2026-11-11: C-10's guard is recorded

ADR-0023 §2 decided that `.github/**` joins the `security` team's
required-review patterns in edge-config's ruleset, and said C-10's headline
reads with its exception until that record exists. It exists now. On 2026-11-11
at 21:51:53Z the ruleset's `file_patterns` became `external/**` and
`.github/**` (history 56118811 → 57378541; no other rule changed), recorded as
steps 09 and 10 in `platform-factory/.github` (#6). edge-config#19, the
counter-example, was reopened to read it: CLEAN on 2026-10-29, now BLOCKED with
`security` requested, although its edited workflow still reports `route files`
as passing. It was closed again without merging.

C-10's grade does not change. What the read shows is what C-09's shows: GitHub
enforces the pattern. With one person as author and as `security`'s only
member, it does not show that a second person reads a workflow change.
