# ADR-0021: The approval boundary on the Config Connector engine: the folder is the only input, and every door to the edge has a gate

Status: Accepted · October 2026 · supersedes ADR-0002's second mechanism (an `exposure:` field that drives materialization) and the folder-to-field check that went with it, so C-10's subject changes; supersedes, for the reference implementation, ADR-0004's "external-dns → Cloudflare for public records" and keeps its decision of one domain with a zone delegated into the environment's project; re-scopes the "untagged resources" half of ADR-0014 §3's deferred metadata-spine pre-check and of C-11, which a validate-only Kyverno cannot meet as worded; replaces the name prefix of ADR-0016 §2 with exact instance names, as ADR-0017 §10 said this ADR would; extends ADR-0017 §4's tenant AppProject blacklist and §7's reality gate to the edge kinds; applies ADR-0019 to routes with one stated exception (§3); fixes ADR-0020's grade ceilings for C-09, C-10, C-11, C-12 and C-13; splits M3 into M3a and M3b; pre-registers C-34 · from platform-factory-concept#21, #22, #26, #27 and #37, platform-config#21, #24 and #33, and the 2026-10-23 review, decided on 2026-10-27, before any M3 build command ran

## Context

M3 was written for an engine that no longer exists. ADR-0002 made every route
a file in `edge-config`, split by folder *and* by a field, and gave the reason
for the field in one clause: "admission controllers and Compositions can't see
file paths". Under Crossplane that was true. Under ADR-0017 the platform
renders the tenant's form itself, and the renderer is an Argo CD Application
that knows which folder it read. ADR-0017 §4 already uses this: the platform
writes `helm.valuesObject`, which outranks every value file, so a key the
platform writes cannot be overridden by a tenant's file. The same move makes
the field unnecessary. That is what the engine swap changed for M3, and why
this ADR cannot restate ADR-0002.

Six open issues promise M3's gating ADR and readiness note
(platform-factory-concept#21, #22, #26 and #27, platform-config#21 and #24),
and a third outside review (#37) added what it found in the clones. What exists
at that snapshot (`platform-config` `6949999`, `platform-bootstrap` `7ad4843`,
`edge-config` `9864f39`, `systems` `a79a910`, `svc-hello` `76a6294`):

- **`edge-config` is a scaffold.** It holds a CODEOWNERS with two lines,
  `* @platform-factory/platform` and `/external/ @platform-factory/security`,
  and no `external/` folder [C, `edge-config` `.github/CODEOWNERS`]. Every
  ruleset requires a pull request and zero approvals, and none requires a
  status check [C, read-only `gh api`, 2026-09-23, in #21]. One person is the
  organisation's only owner and the only member of both `platform` and
  `security`.
- **Nothing refuses an edge object in a tenant namespace.** The tenant
  AppProject's blacklist names no Service, Ingress or Gateway kind [C,
  `platform-config` `charts/system/templates/argocd.yaml:32-55`]. The team is
  bound to the built-in `admin` role, which allows them [C,
  `charts/system/templates/access.yaml:24-27`]. The one Kyverno gate covers
  Config Connector groups only [C, `kyverno/policies/deny-raw-cloud-resources.yaml`].
  A team can create a `LoadBalancer` Service today with no review; the tenant
  model in `platform-pattern.md` discloses this and says M3 closes it. Nothing
  is exposed in practice: the tenant Services are `ClusterIP` [C, `svc-hello`
  `k8s/service.yaml:16`].
- **The cluster cannot enforce a NetworkPolicy and has no gateway.** Layer 2
  sets no `datapath_provider`, no network policy and no Gateway API channel [C,
  `platform-bootstrap` `layers/2-cluster/gke.tf`, searched for each]. Layer 1
  has one subnet and no proxy-only range [C, `layers/1-network/network.tf:14-18`].
  No zone, address or certificate exists in the Terraform. Tenant namespaces
  carry no Pod Security label and no NetworkPolicy [C,
  `charts/system/templates/namespace.yaml:16-35`].
- **Two registered claims cannot be built as worded.** C-11's "denies untagged
  resources" needs a Kyverno that adds labels. This one is validate-only on
  purpose, because a mutation shows as permanent drift [C, `apps/kyverno.yaml:3-12`],
  and refusing every unlabelled tenant Deployment would refuse every tenant.
  C-12 names no alert source, and "healthy" is undefined.
- **A hole sat beside the gate C-11 tests.** The blacklist has `kyverno.io` but
  not `policies.kyverno.io` [C, `argocd.yaml:53-55`], and the vendored Kyverno
  chart aggregates a role that can create policies of that group into `admin`
  (#37 item 2). Offline, a tenant-written mutating policy changed a
  platform-rendered object that then passed the deny-raw gate [C, offline; not
  run on a cluster]. platform-config#33 closed it, merged on 2026-10-27
  (`e8eedeb`), before any C-11 case runs; without it a HELD on C-11 would sit
  beside a known way around the gate.

## Decision

**The folder a route file sits in is the only input that says how exposed it
is. The platform renders the route, and it reaches a load balancer only through
a Gateway the platform owns. Every other way to make an edge object is refused
at both doors, git (the AppProject) and `kubectl` (Kyverno). `external/` is
approved by a rule GitHub enforces, recorded as the API returns it, and what
each test cannot show with one person on both teams is written down before the
run.**

1. **M3 is split in two, and both halves close in one build-log entry.** M3a,
   the approval boundary: C-09, C-10, C-11. M3b, the edge it governs and the
   rest: C-12, C-13, C-14, C-24 and a new dated claim, C-34. The register's
   milestone column keeps saying M3; the readiness note says which half. The
   split orders the work and does not change grading. Cluster-touching M3 work
   waits for M2b's close: ADR-0017 §1 and §12 keep `platform-bootstrap` additive
   until C-29's reverse leg has run, and M2b had no build-log entry at this
   ADR's date, so no layer-2 change, policy, chart change or claim that touches
   the cluster runs before that entry exists. The GitHub-only part of M3a — the
   route chart rendered offline, `edge-config`'s tree and check, the ruleset
   change, C-09 and C-10 — may start as soon as this ADR, ADR-0020 and the
   readiness note are committed and archived. ADR-0017 §1 and §12
   already put the engine swap (M2b) before M3.

2. **The route form: the folder is the only input.** A route is one file,
   `edge-config/<exposure>/<system>/<route>.yaml`, with `<exposure>` either
   `external` or `internal`. It has four fields: `host`, a service-class label
   matching `^[a-z][a-z0-9-]{0,30}[a-z0-9]$`; `service`, a Service in the
   System's namespace; `port`, an integer; and optionally `path`, a prefix,
   default `/`. There is no `exposure` field. The platform's keys live under one
   object of their own, `platform`, beside `environment`, and the schema closes
   the file's top level to the four fields, so a file carrying `exposure`,
   `system` or `route` at its top level is refused as an unknown key. The
   nesting is what makes the refusal real: Helm merges value files key by key
   and the schema sees only the merged tree (ADR-0017 §4; C-28 (b2)), so a
   top-level `exposure` that the platform also wrote would be overridden in
   silence, not refused.

   A new `platform-config` chart, `charts/route`, renders the file. Two
   platform-owned ApplicationSets, `edge-external` and `edge-internal`, each
   read their own folder with a git-files generator and write into every
   generated Application's `helm.valuesObject`, under `platform`:
   `platform.exposure` (a literal per ApplicationSet), `platform.system` and
   `platform.route` (from the file's path), beside the environment's facts under
   `environment`. That is ADR-0017 §4's pattern, for the same reason:
   `valuesObject` outranks the route file, so the file cannot set what the
   folder says. There is one Application per route, in the System's namespace.

   **This supersedes ADR-0002's second mechanism.** ADR-0002 had two views of
   one fact, a folder for CODEOWNERS and a field for the machinery, and a CI
   check to keep them from disagreeing. The renderer now sees the folder, so
   the field that could disagree with it does not exist. The first mechanism
   stands: the folder routes the review, and the internal-to-external flip is
   one `git mv` in one pull request. C-10's registered subject, a field that
   can disagree with its folder, is gone, so its ceiling is fixed now:
   **ADJUSTED**, not HELD on a smaller claim. C-13's text says the `exposure:`
   field selects the WAF policy, the zone and the gateway class; under this
   section the folder does, so C-13's ceiling is **ADJUSTED** for the same
   reason.

3. **What a route renders, and where it may write.** In the System's namespace:
   an `HTTPRoute` attached to the Gateway for its exposure (namespace `edge`,
   §4) with the hostname `<host>.<zone>`, the zone being `ref.thecloudgeek.io`
   for external and `internal.ref.thecloudgeek.io` for internal; and, for an
   external route only, a `GCPBackendPolicy` that targets the Service and
   carries the Cloud Armor policy (§7). Every object carries the System label.

   The Applications run under one platform-owned AppProject, `edge`: sources
   `edge-config` and `platform-config`, destinations any namespace, and
   `namespaceResourceWhitelist` exactly `HTTPRoute` and `GCPBackendPolicy`. A
   whitelist suits here, unlike the tenant blacklist, because these
   Applications render two kinds and nothing else. "Any namespace" is bounded
   twice more, by the Gateways' `allowedRoutes` (§4) and by `require-system-label`
   (§9) [I; C-11's cases try both].

   **Pruning a removed route is normal.** ADR-0019 protects namespaces and data
   from a removed file; a route file is neither, and removing it removes its
   `HTTPRoute`. That is the one stated exception to ADR-0019's rule, limited to
   the kinds the `edge` project may write. There is one Application per route,
   so a removed file is not an empty render: the ApplicationSet deletes that
   route's Application under Argo CD's default `applicationsSync` policy, and
   the Application's resources finalizer removes its `HTTPRoute` and
   `GCPBackendPolicy` [C, argo-cd v3.4.6
   `docs/operator-manual/applicationset/Controlling-Resource-Modification.md`
   and `Application-Deletion.md`]. The two edge ApplicationSets therefore do not
   carry ADR-0019 §1's four settings, which `scripts/check_charts.py` asserts on
   the tenants' ApplicationSet only; that is the exception, and its cost is
   stated: renaming or emptying `external/` or `internal/` removes every route
   in it in one sync, and a route file the generator cannot parse stops that
   ApplicationSet generating until it is fixed, deleting nothing [I; the first
   M3 session reads what the controller does].

4. **Gateways and addresses.** A platform Application, `edge`, reads
   `platform-config`'s `edge/` directory and creates the namespace `edge` and
   two Gateways. `external`: class `gke-l7-global-external-managed`, a reserved
   global static IP, HTTPS on 443 with a Certificate Manager map attached by
   annotation [I]. `internal`: class `gke-l7-rilb`, a reserved regional internal
   IP, HTTP on 80 only. **Internal TLS is out of M3, and recorded as such.**
   Both set `allowedRoutes` to namespaces carrying the System label. `edge` joins `charts/system`'s reserved System names (platform-config#22), so no System can take the namespace. The
   regional internal load balancer needs a proxy-only subnet, a layer-1 resource.

5. **DNS.** A public Cloud DNS zone, `ref.thecloudgeek.io`, in
   `platform-factory-ref`, delegated from Cloudflare by one manual step
   (recorded, not in Terraform). A private zone, `internal.ref.thecloudgeek.io`,
   bound to the VPC. One wildcard A record in each: `*.ref…` to the external IP,
   `*.internal.ref…` to the internal IP. **No external-dns and no per-route DNS
   write: a route changes no DNS.**

   This supersedes ADR-0004's "external-dns → Cloudflare for public records"
   for the reference implementation and keeps its decision: one registered
   domain, an environment zone delegated into the environment's project,
   private zones in Cloud DNS, and the `<service-class>.<env>.<domain>` schema.
   ADR-0004's check that a route's hostname agrees with its folder and
   `exposure:` has nothing left to compare, because the chart builds the
   hostname from the folder [I].

6. **Certificates.** Certificate Manager holds a DNS authorization for
   `ref.thecloudgeek.io`, a Google-managed certificate for
   `*.ref.thecloudgeek.io`, and one certificate map, `edge-external`, with one
   entry, attached to the external Gateway (§4).

7. **The WAF.** One Cloud Armor policy, `edge-external`: the preconfigured
   WAF rule for SQL injection returns 403, and the default is allow. It is
   attached per external route through the rendered `GCPBackendPolicy`.
   Internal routes get none. GKE attaches one `GCPBackendPolicy` to a Service at
   a time [C, GKE "Configure Gateway resources", read 2026-10-27], so the chart
   names it after the Service, and C-10's check refuses a second external route
   that names the same `service` in the same System. ADR-0004's stricter policy for an `auth.`
   hostname is not built: every external route has this one.

8. **No new engine permission.** The zones, records, addresses, subnet,
   certificate, map and policy are Terraform crossings, declared in §12 and
   counted for C-01. GKE's own Gateway controller builds the load balancers.
   `platform-roles` is unchanged.

9. **The reality gates (C-11).** Two validate-only Kyverno policies, with
   `failurePolicy: Fail` and `allowExistingViolations: false` (the setting
   whose default let an edit through the first deny-raw rule,
   `deny-raw-cloud-resources.yaml:112-119`):
   - **`deny-edge-objects`**, in tenant namespaces. CREATE and UPDATE of a
     Service of type `LoadBalancer` or `NodePort` are refused **for everyone**:
     the tenant's own Application is Argo CD too, and an AppProject cannot
     filter by Service type, so exempting Argo CD would leave the git door
     open. CREATE, UPDATE and DELETE of an `Ingress`, of the Gateway API kinds
     (`gateway.networking.k8s.io/*/*`) and of the GKE gateway policy kinds
     listed by name (`GCPBackendPolicy`, `HealthCheckPolicy`,
     `GCPGatewayPolicy`, `GCPSessionAffinityPolicy`,
     `GCPTrafficDistributionPolicy`, `LBPolicy`) are refused for everyone except
     the writer of the platform's edge Applications, Argo CD's application
     controller, and, on DELETE, the namespace controller and the garbage
     collector, which deny-raw's delete rule already lets through for the same
     reason. The GKE group is not matched whole: GKE's own controllers write
     `ServiceNetworkEndpointGroup` objects of that group into a Service's
     namespace when a route reaches it [I; the first external route records the
     kind and the identity]. Exempting Argo CD is safe here only because, for
     these kinds, the git door has its own wall, below; the same identity
     carries a tenant's `k8s/`, which is why the Service rule exempts nobody.
   - **`require-system-label`**, in tenant namespaces: the gated cloud kinds
   (ADR-0017 §7) and the route kinds (the kinds enumerated above, on CREATE and
   UPDATE) must carry
     `platform.thecloudgeek.io/system` equal to the namespace's name.

   **The git door.** The tenant AppProject's blacklist gains
   `networking.k8s.io/Ingress`, `gateway.networking.k8s.io/*` and
   `networking.gke.io/*`. A team's own manifests cannot create these through
   Argo CD; the platform's `edge` project, with its whitelist, can.

   **"Denies untagged resources" is re-scoped.** A validate-only Kyverno cannot
   add a label, and a rule refusing every unlabelled Deployment would refuse
   every tenant. The rendered Namespace already carries the System label
   (`_helpers.tpl:27-32`), so the namespace is the join key for what is inside
   it. The registered half holds where it can by construction, for cloud kinds
   and route kinds, which the platform's charts render with the label. This
   supersedes the wording of ADR-0014 §3's deferred "metadata spine pre-check"
   and of C-11. C-11's ceiling is **ADJUSTED**, because its second half is
   re-scoped.

   **This reverses the 2026-09-22 decision** (#26; `platform-pattern.md`, "The
   tenant model") that the Service rule ships with the gateway. With one
   maintainer and `ClusterIP` Services, "no approved way to expose yet" costs
   nothing, so the gap is closed now. The cost is on the Kyverno side
   (Consequences).

10. **C-09: the mechanism, the identity, the record.** `edge-config`'s ruleset
    gains GitHub's *required reviewers* rule (generally available 2026-02-17 [I;
    re-read before the change]): team `security`, one approval, file pattern
    `external/**`. Required approvals stay at zero elsewhere and CODEOWNERS is
    unchanged, so `internal/` keeps ADR-0002's light gate. C-09's text names
    CODEOWNERS, so it is graded against a mechanism it does not name. C-10's
    check becomes a required status check once it has failed once and passed
    once (ADR-0019 §4).

    The ruleset JSON is recorded by hand, as `gh api` returns it, before and
    after each change, under `rulesets/` in `platform-factory/.github`, a repo
    other than the one whose rules it describes. With one human the record and
    the rules have the same author; the ruleset history endpoint is the read
    that does not depend on the record. It is not Terraform: ADR-0016
    §7 keeps tenants, services, databases and policy out of it, and C-01
    counts every apply [I].

    **The test identity is the one person on both teams; no GitHub App in
    M3.** The ceiling is **ADJUSTED**: one person shows that GitHub enforces
    a rule, not that two people kept their duties apart (#21). The claim is
    **WRONG** if any change under `external/` reaches `main` without an
    approval from `security`. The bypass set is run and each case recorded as
    blocked or slipped: a new file straight into `external/`, a rename into it,
    a pull request that edits the workflow that produces the required check,
    and the organisation owner editing the ruleset. A permitted `external/`
    merge needs an approval the author cannot give, and C-13 needs two such
    merges after the rule is live (its external route and its flip). Every such
    merge is made by the organisation owner editing the ruleset, read from the
    history endpoint and listed in C-09's data as slipped; this is one more
    reason the ceiling is ADJUSTED and not higher. C-09's "field flip without
    the move" cannot be made, because there is no field (§2); it becomes C-10's
    case, a file with an `exposure` key.

11. **C-12 is narrowed, with a ceiling of PARTIAL.** The alert source is a Cloud
    Monitoring alerting policy, defined in Terraform at layer 1, on GKE
    container restarts per namespace. The join is alert → namespace → the
    System label on the namespace → the tenant file, which names the team and
    the repo. The test is a synthetic incident on `svc-hello`. **"The PRs merged
    since the last healthy state" is not tested in M3:** no deploy event exists
    to say when "healthy" was, so it moves to the prod-tag-bump class at M4.

12. **Terraform crossings, declared before the milestone (C-01).**
    - Layer 0: the APIs `certificatemanager`, `dns`, and `networkservices` if
      needed.
    - Layer 1: the two zones, the two wildcard records, the DNS authorization's
      CNAME record, the global and the regional address, the proxy-only subnet,
      the DNS authorization, certificate, certificate map and its entry, the
      Cloud Armor policy, and the alerting policy.
    - Layer 2: `datapath_provider = ADVANCED_DATAPATH`, the Gateway API channel
      set to standard, and FQDN network policy on.

    One apply on each persistent layer (0 and 1); layer 2 rides the rebuild.
    The Cloudflare delegation is a manual step, and the ruleset changes are
    by hand; neither is an apply.

13. **C-14.** The primary source [C, Google Cloud GKE "FQDN network policies"
    how-to, read 2026-10-27]: it requires Dataplane V2 and kube-dns or Cloud
    DNS; it states no edition requirement; a Standard cluster enables it at
    creation with `--enable-fqdn-network-policy`; the object is
    `networking.gke.io/v1alpha1` `FQDNNetworkPolicy`. The hands-on test: a pod
    under a default-deny egress that allows `www.google.com:443` only reaches
    it and is refused `example.com`. DNS churn is read from TTL behaviour. The
    register entry has no claim sentence, so the readiness note supplies the
    pass line.

14. **C-24.** The app's and the team's `cloudsql.client` and
    `cloudsql.instanceUser` members move from `charts/system`, where they are
    project-wide, to `charts/claims`: one conditioned pair per database, with
    the condition `resource.name == "projects/<project>/instances/<system>-<claim>"`
    (platform-config#24). That names one instance exactly, which holds only
    because a claim name has no hyphen (ADR-0017 §5). It replaces ADR-0016 §2's
    prefix, which matched `orders-api-main` for System `orders`. The test is the
    registered one, with `svc-hello` and `svc-onboard` both owned by `payments`
    and one claim each, before one of them moves team. The M2 unconditioned
    bindings stay in the project's policy, ownerless, until a recorded cleanup
    removes them (ADR-0017 §10).

15. **C-34, new, dated 2026-10-27: tenant namespaces are isolated by default.**
    `charts/system` adds `pod-security.kubernetes.io/enforce: baseline`, with
    `warn` and `audit` at `restricted` and the version pinned, and a default-deny
    NetworkPolicy with the allow list in platform-config#21. The test: from
    `svc-ledger`, connect to `svc-hello`'s Service and pod IP, before and after;
    `svc-hello`'s `/healthz` and a fresh `GRANT` Job stay green; a privileged
    pod in a tenant namespace is refused; a tenant-written allow-all opens only
    that tenant's own pods. **Falsified if** a cross-tenant connection succeeds
    under the default, or the golden path breaks. It is a claim of its own
    because C-11 covers Kyverno only and the register is append-only. It states
    the model the tenant-model section already holds, cooperative between
    tenants and untrusted toward the platform; hostile-tenant isolation is not
    claimed.

16. **Out of M3.** ESO (deferred out of M3: the GCP-native edge needs no
    Cloudflare token, and ADR-0013's "deferred, not dropped" stands for
    third-party secrets); kgateway and a Cloudflare-proxied edge as swap-ins;
    restore and point-in-time recovery (#28, platform-config#23); break-glass
    (platform-bootstrap#23); a GitHub App (M4); the human-login test; and C-08,
    **graded NOT REACHED at M3's close** under ADR-0020, naming two
    preconditions not met: the governance decision the M2 grading note records,
    and a composition layer to swap, which ADR-0017 removed; it stays open until
    an ADR retires it or restates it for the chart engine.

## Consequences

- **The `git mv` flip is still one pull request, and the folder is the only
  thing it changes.** Nothing in the file can disagree with its place. That
  removes a class of CI case and leaves a smaller claim for C-10: no file sets
  what the folder sets, and names and hosts are well formed. The cost is a
  grade, ADJUSTED, stated before the run.
- **Wildcard DNS resolves every name under each zone.** A route adds no record,
  so a name with no route also resolves, to a Gateway that answers 404. That is
  the price of "a route changes no DNS" and of having no external-dns. A name
  that resolves does not mean a service exists. The public wildcard
  `*.ref.thecloudgeek.io` also answers `x.internal.ref.thecloudgeek.io` from
  outside, because a wildcard matches deeper labels when no closer name exists:
  the answer is the external IP and a 404, which is harmless and not a leak.
- **Internal routes have no TLS.** Traffic to an internal route is plain HTTP
  between the client and the load balancer. No claim tests for it in M3.
- **One human.** C-09 and C-11 are run by the one person on both teams. C-09
  shows that GitHub enforces the rule, not that duties are separated. C-11's
  `kubectl --as` cases show how the policy treats an impersonated identity,
  not every path a real team member might have.
- **The Service rule makes Kyverno load-bearing for every tenant deploy.** The
  webhook now matches Services in tenant namespaces, so with `failurePolicy:
  Fail` an outage blocks tenant deploys, not only cloud writes. `helm template`
  of the vendored chart renders no `replicas` and no PodDisruptionBudget [C,
  #37 item 7; platform-config#36]. This ADR accepts that for the reference
  build and does not make a replica count a prerequisite; C-33 measures what an
  outage blocks.
- **What it supersedes in earlier ADRs.** ADR-0002's field and its CI check;
  ADR-0004's external-dns for public records; ADR-0014 §3's "untagged" wording;
  ADR-0016 §2's prefix. It extends ADR-0017 §4 and §7. `docs/adr/README.md`
  carries each pointer; the earlier ADRs are not edited.
- **`FQDNNetworkPolicy` is platform-only in the reference build.** Blacklisting
  `networking.gke.io/*` for tenants (§9) keeps a team from writing its own
  egress allow by FQDN through git.
- **The `edge` project is a new writer.** The route chart is reviewed as policy,
  as the claims chart is (ADR-0017, Consequences): its schema is closed and its
  output is two kinds. The Gateways' `allowedRoutes` and the label rule are the
  other walls [I].
- **C-24's limits.** IAM allows at most 20 bindings with one role and principal
  under different conditions, so a team covers at most 20 databases per role.
  Config Connector IAM members are immutable, so a changed instance list needs
  a new member name (platform-config#24).
- **Still open, each with the step that settles it.**
  - The ingress allow that lets the Gateways reach tenant pods under the
    default-deny, which platform-config#21 defers to the gateway's choice.
    C-34's data records every allow the golden path needed that the list
    missed, and C-13's external route runs after C-34's default is on.
  - The comparison #26 asks for, of what the Gateway API's roles delegate and
    what routing every internal route through `edge-config` displaces, is not
    made here.
  - C-24's overmatch fixture (two Systems where one name extends the other)
    needs a second Cloud SQL instance and is not planned.
  - ADR-0005's wording of the project as the tenant isolation unit, which #22
    said this ADR would mark as superseded, is not touched here;
    `platform-pattern.md` already carries the corrected rows.
