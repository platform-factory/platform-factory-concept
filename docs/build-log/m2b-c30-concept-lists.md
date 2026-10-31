# C-30 concepts and lines: desk lists made by the 2026-10-28 method

Method followed verbatim from claims-register.md, "On 2026-10-28, before S3 counts the Config Connector side, C-30's two open readings are fixed." The reader follows `svc-hello` with one database claim from tenant file and claim to cloud resources. Two kinds of item, each listed once by name with the file where the reader first meets it. Shared items counted on both sides. Total reported without and with ADR-0018's three additions, each one item.

Commits read fresh read-only clones:
- Crossplane: platform-config 3bfc72d (2026-09-21), platform-bootstrap c64cc5a (matching time; Terraform layers 0-3), systems and svc-hello at the same period (tenants/svc-hello.yaml; svc-hello k8s/database.yaml, 2da78e5 form).
- Config Connector: platform-config e8eedeb (main, 2026-10-27), systems a79a910, svc-hello 76a6294 (claims.yaml), platform-bootstrap 7ad4843, platform-roles 08178fb.

All paths below: `pc/` = platform-config, `sys/` = systems, `sh/` = svc-hello, `pb/` = platform-bootstrap. Crossplane paths are at 3bfc72d; Config Connector paths at main.

## 1. Rules used (applied identically to both engines)

Path order: tenant file (sys/tenants/svc-hello.yaml) -> what reads it (pc/apps/systems.yaml) -> engine recipe (XRD + Composition, or the two charts) -> the objects it creates -> the claim (sh/k8s/database.yaml, or sh/claims.yaml) -> its recipe -> cloud resources -> what refuses.

**Kind met on the path.** One apiVersion-group + kind, listed once, if it is: (a) the kind of an object a file on the path creates, applies or defines (tenant/claim objects, XRD/Composition, Applications, every cluster and cloud object the recipe emits, including core kinds like Namespace and Job); (b) a kind a path file names as a reference that must exist for the object to work (roleRef ClusterRole, secretKeyRef Secret, functionRef Function, the default providerConfigRef); (c) a kind that refuses (Kyverno ClusterPolicy); (d) an engine-install kind only if it decides which kinds exist, which identity acts, or how objects reconcile (Provider, DeploymentRuntimeConfig, ClusterProviderConfig on Crossplane; ConfigConnector on Config Connector). Not kinds: CRDs, vendored upstream chart internals and operator Deployments, image-mirror config (ImageConfig), RBAC subject `Group`, `resourceRef` targets that are only pointed at (`Project`), kinds in files the engine's Application skips, repos, Google-side services.
Same-named kinds in different API groups (core ServiceAccount vs the GCP ServiceAccount; platform `Database` vs Cloud SQL `Database`) are different kinds.

**Mechanism.** Not a kind, but a file on the path cites it as the reason an object is shaped, ordered, named, kept or refused as it is, so a reader who did not know it would mispredict what happens (readiness, adoption, deletion, drift, ordering, admission, identity, rendering). One item per named field, annotation, label, policy setting or behaviour, however many places use it. Sub-details are folded into the parent item (e.g. `spec.deletionPolicy: ABANDON` folds into the durability item; the Kyverno exclusion list folds into the identity gate). Where Crossplane and Config Connector solve the same problem with different named things, each side lists its own and they are NOT marked shared; "shared" means the same named thing.

Repos are not items (neither kind nor mechanism); only ADR-0018's eighth repo is added, as the note says. Terraform in platform-bootstrap is one shared mechanism (its "last job" boundary is met at pc/README.md:7); ADR-0018's "second home for Terraform" is the addition.

## 2. Crossplane list (3bfc72d) — 28 kinds, 36 mechanisms

### Kinds
1. System (platform.thecloudgeek.io) — XR kind of the tenant file — sys/tenants/svc-hello.yaml:30 — what the tenant file is; cluster-scoped
2. Database (platform.thecloudgeek.io) — namespaced XR kind of the claim — sh/k8s/database.yaml:24 — what the claim is
3. Application (argoproj.io) [SHARED] — pc/apps/systems.yaml:24 — reads tenants/ and applies each System
4. CompositeResourceDefinition — pc/crossplane/compositions/system/xrd.yaml:43 — turns the kind into an API, with its schema
5. Composition — pc/crossplane/compositions/system/composition.yaml:126 — the recipe
6. Function (pkg.crossplane.io) — pc/crossplane/platform/functions.yaml:55 — each pipeline step is a package that must be installed
7. EnvironmentConfig — pc/crossplane/platform/environment.yaml:36 — project, domain, network the recipe reads (Required vs Optional resolution differs between the two Compositions: database/composition.yaml:122 vs system/composition.yaml:151)
8. Namespace [SHARED] — system/composition.yaml:289 — the tenant's namespace
9. ResourceQuota [SHARED] — system/composition.yaml:322 — size class
10. RoleBinding [SHARED] — system/composition.yaml:385 (two: -admin, -crossplane-edit, :488) — team access
11. ClusterRole [SHARED] — pc/crossplane/platform/rbac.yaml:34; roleRef `admin` at composition.yaml:397 — grants to Crossplane; the role the team is bound to
12. AppProject (argoproj.io) [SHARED] — composition.yaml:525 — tenant's wall
13. ServiceAccount (core) [SHARED] — composition.yaml:723 — the pod's identity
14. ConfigMap [SHARED] — composition.yaml:998 (platform-system); database/composition.yaml:570 (connection)
15. ServiceAccount (cloudplatform.gcp.m.upbound.io) — composition.yaml:761 — the Google service account
16. ServiceAccountIAMMember — composition.yaml:798 — Workload Identity binding
17. ProjectIAMMember — composition.yaml:867 — project-level Cloud SQL login grants
18. RegistryRepository (artifact.gcp.m.upbound.io) — composition.yaml:912 — the tenant's registry
19. RegistryRepositoryIAMMember — composition.yaml:958 — the team's registry writer grant
20. DatabaseInstance (sql.gcp.m.upbound.io) — database/composition.yaml:370 — the Cloud SQL instance
21. Database (sql.gcp.m.upbound.io) — database/composition.yaml:459 — database `app`
22. User (sql.gcp.m.upbound.io) — database/composition.yaml:499 — the IAM database user
23. Secret [SHARED] — database/composition.yaml:546 — the break-glass admin password
24. Job [SHARED] — database/composition.yaml:608 — the GRANT
25. ClusterProviderConfig (gcp.m.upbound.io) — pc/crossplane/platform/provider-config.yaml:44 — the one credential every managed resource defaults to
26. Provider (pkg.crossplane.io) — pc/crossplane/providers/provider-gcp-sql.yaml:7 (and provider-family-gcp.yaml:11) — supplies the `*.gcp.m.upbound.io` kinds; one pod per Google service
27. DeploymentRuntimeConfig — pc/crossplane/providers/runtime-configs.yaml:71 — pins the provider pod's service account and its Workload Identity annotation
28. ClusterPolicy (kyverno.io) [SHARED] — pc/kyverno/policies/deny-raw-managed-resources.yaml:57 (also database-claim-budget.yaml:29) — the refusers

### Mechanisms
1. XR scope, Cluster vs Namespaced (v2, no Claim) — pc/crossplane/compositions/system/xrd.yaml:67; database/xrd.yaml:41 — decides which namespaces a recipe may write to
2. XRD schema becomes a CRD; root-level CEL; metadata pattern dropped — system/xrd.yaml:37, :132 — what is refused at admission
3. Composition pipeline mode (steps, context passed between functions) — system/composition.yaml:138, :162 — order and data flow
4. Go-template rendering of desired resources: resource-name annotation, `missingkey=error` — system/composition.yaml:191, :295 — object identity across reconciles
5. Observed-state gating (later objects render only after earlier ones are observed) — system/composition.yaml:303, :315; database/composition.yaml:348 — why things appear late
6. XR readiness roll-up: function-auto-ready, per-kind health, `gotemplating.../ready` annotation — system/composition.yaml:73, :87, :334 — when the System reports Ready
7. Crossplane applies composed objects with its own service account; RBAC must be granted via `rbac.crossplane.io/aggregate-to-crossplane` — pc/crossplane/platform/rbac.yaml:15, :39 — an ungrated kind fails Forbidden
8. RBAC-manager aggregation (`crossplane-edit`, `aggregate-to-admin/edit`) opens raw managed resources to the team — system/composition.yaml:488; deny-raw-managed-resources.yaml:8 — why a reality gate is needed
9. Upjet managed-resource shape (forProvider / initProvider / providerConfigRef; Terraform underneath) — system/composition.yaml:771 — where each cloud field goes
10. ProviderConfig default reference (`default`, InjectedIdentity) — provider-config.yaml:28 — which credential acts
11. `crossplane.io/external-name` (adoption by deterministic name; two opposite rules) — database/composition.yaml:380; system/composition.yaml:769 — a rebuild adopts, not recreates
12. `managementPolicies` without Delete (plus provider-level `deletionPolicy: ABANDON` on Database/User) — database/composition.yaml:385, :493 — durability
13. Garbage collection of composed resources that leave desired state — system/composition.yaml:858 — a team move removes the old objects
14. Composition-minted admin password, re-minted on rebuild, Cloud SQL never learns it — database/composition.yaml:38, :273 — what the admin Secret is worth
15. Kyverno `apiCall` context and its read grant (`rbac.kyverno.io/aggregate-to-background-controller`) — database-claim-budget.yaml:78; database-claim-budget-rbac.yaml:28 — how the two-database budget counts
16. Kyverno `crdWatcher` (a CRD created in the same sync wave) — pc/apps/kyverno.yaml:68 — policy on a generated kind
17. Team is a Google group by naming convention (`<team>@<orgDomain>`), group must pre-exist [SHARED] — sys/tenants/svc-hello.yaml:12
18. App-of-apps root Application [SHARED] — pc/README.md:9 (root created by pb/layers/3-argocd/argocd.tf:247)
19. Argo CD sync waves [SHARED] — pc/apps/crossplane.yaml:14; system/composition.yaml:132
20. Automated sync, prune + selfHeal (deleting a file prunes the System) [SHARED] — pc/apps/systems.yaml:47, :53
21. Tenant AppProject resource blacklist (the git-path wall) [SHARED] — system/composition.yaml:553
22. Kyverno gate keyed on requester identity (`subjects` exclusions) [SHARED] — deny-raw-managed-resources.yaml:164
23. Kyverno kind selectors list groups by name; a wildcard group matches nothing [SHARED] — deny-raw-managed-resources.yaml:36, :135
24. Kyverno `failurePolicy` Fail vs Ignore [SHARED] — deny-raw-managed-resources.yaml:92; database-claim-budget.yaml:56
25. Workload Identity (KSA to Google SA) [SHARED] — system/composition.yaml:800; runtime-configs.yaml:10
26. IAM database authentication (`cloudsql.iam_authentication`, passwordless user) [SHARED] — database/composition.yaml:422
27. The GRANT Job (an IAM user starts with no privileges) [SHARED] — database/composition.yaml:93
28. Cloud SQL Auth Proxy with `--auto-iam-authn` [SHARED] — sh/k8s/deployment.yaml:83
29. Private Services Access peering for private-IP Cloud SQL (from pb layer 1) [SHARED] — database/composition.yaml:100
30. Terraform layers 0-3 in platform-bootstrap (the base the engine stands on) [SHARED] — pc/README.md:7
31. Cloud SQL deletion protection on the instance [SHARED] — database/composition.yaml:368, :389
32. IAM member fields are immutable; the team is in the object name, a move re-creates [SHARED] — system/composition.yaml:59
33. Size is a class mapped to machine type and quota in the recipe [SHARED] — system/composition.yaml:250; database/composition.yaml:232
34. The System name is the identity: namespace, project, registry, GSA, instance `<system>-<claim>`, `system` label [SHARED] — system/composition.yaml:30 (header), :289; database/composition.yaml:175
35. GCP labels cannot hold `/`: a second unprefixed label set for the cloud side [SHARED] — database/composition.yaml:443, :447
36. Jobs are named by a hash of their spec (a Job's pod template is immutable) [SHARED] — database/composition.yaml:296, :308, :610

**Crossplane totals: kinds 28, mechanisms 36, total 64.** (Shared with Config Connector: 11 kinds, 20 mechanisms.)

## 3. Config Connector list (main e8eedeb) — 20 kinds, 36 mechanisms

### Kinds
1. ApplicationSet (argoproj.io) — pc/apps/systems.yaml:35 — one Application per tenant file
2. Application [SHARED] — pc/charts/system/templates/argocd.yaml:81 (and apps/systems.yaml:61 template) — the tenant's `<name>`, `<name>-claims` and `<name>-system`
3. AppProject [SHARED] — argocd.yaml:17 — tenant wall
4. Namespace [SHARED] — charts/system/templates/namespace.yaml:17
5. ResourceQuota [SHARED] — namespace.yaml:46
6. RoleBinding [SHARED] — charts/system/templates/access.yaml:18
7. ClusterRole [SHARED] — access.yaml:26 (roleRef `admin`); cnrm-admin in deny-raw-cloud-resources.yaml:4
8. ServiceAccount (core) [SHARED] — access.yaml:39 (and database-seed :54)
9. Role — access.yaml:62 — the seed Job's Secret rights
10. ConfigMap [SHARED] — charts/system/templates/configmap.yaml:11; claims/templates/databases.yaml:176
11. IAMServiceAccount (iam.cnrm.cloud.google.com) — identity.yaml:10
12. IAMPolicyMember — identity.yaml:27 (also :61, :77; registry.yaml:50) — Workload Identity, Cloud SQL logins, registry writer
13. ArtifactRegistryRepository — registry.yaml:18
14. SQLInstance — charts/claims/templates/databases.yaml:70
15. SQLDatabase — databases.yaml:135
16. SQLUser — databases.yaml:152
17. Secret [SHARED] — databases.yaml:98 (secretKeyRef; created by the seed Job, _jobs.tpl:18)
18. Job [SHARED] — databases.yaml:189 (seed and grant)
19. ConfigConnector (core.cnrm.cloud.google.com) — pc/config-connector/configconnector.yaml:20 — cluster mode and the Google service account it acts as
20. ClusterPolicy (kyverno.io) [SHARED] — pc/kyverno/policies/deny-raw-cloud-resources.yaml:35 (also keep-abandon-annotation.yaml:21)

### Mechanisms
1. Tenant file and claims.yaml are Helm values files, never applied (the `kind: System` in the tenant file is not a registered kind) — pc/apps/systems.yaml:86; sh/claims.yaml:29; sys/tenants/svc-hello.yaml:30
2. Argo CD multi-source with `ref` and `$`-valueFiles layering (tenant file, retired.yaml, environments/reference.yaml; `ignoreMissingValueFiles`) — apps/systems.yaml:74-92; argocd.yaml:134, :141
3. Helm chart rendering as the recipe (named templates, dict tables, `hasKey` vs `default`) — charts/system/templates/_helpers.tpl:1; databases.yaml:56
4. `values.schema.json` checked at render; Helm applies no schema defaults — charts/system/values.schema.json:6; _helpers.tpl:15
5. Render-time guards: `required` and `fail` — _helpers.tpl:10; system guard.yaml:15; claims guard.yaml:12
6. Retired-names list (sys/retired.yaml) — apps/systems.yaml:88; system guard.yaml:2
7. Never-prune-to-empty: `Prune=false,Delete=false`, `applicationsSync: create-update`, Namespace `Prune=false`, empty render refused — apps/systems.yaml:15-20; namespace.yaml:32; databases.yaml:12-15
8. `cnrm.cloud.google.com/project-id` (namespace fallback for the project) — _helpers.tpl:42; namespace.yaml:29
9. `cnrm.cloud.google.com/state-into-spec: absent` — _helpers.tpl:40
10. `cnrm.cloud.google.com/management-conflict-prevention-policy: none` — _helpers.tpl:41
11. `cnrm.cloud.google.com/deletion-policy: abandon` (plus SQLDatabase `spec.deletionPolicy: ABANDON`, plus the keep-abandon Kyverno guard on the old object) — registry.yaml:34; databases.yaml:25, :149; keep-abandon-annotation.yaml:57
12. `resourceID` adoption by name, with defaulting and copy-from-live of unset fields (public IPv4 default, zone, disk, edition) — registry.yaml:36; databases.yaml:88, :105, :126
13. Seed Job and cluster Secret for the root password, not tracked by Argo CD, `rootPassword` used only at create — _jobs.tpl:18; databases.yaml:96
14. Kyverno policy closed by chart value (`rbac.roles.aggregate.admin: false`) and the `policies.kyverno.io` blacklist rows — pc/apps/kyverno.yaml:139; argocd.yaml:61
15. Kyverno `allowExistingViolations: false` (UPDATE judged on old and new object) — deny-raw-cloud-resources.yaml:119
16. Config Connector's `cnrm-admin` aggregates into the built-in `admin` (parallel to, not the same as, Crossplane's aggregation) — deny-raw-cloud-resources.yaml:4
17. Team is a Google group by convention [SHARED] — sys/tenants/svc-hello.yaml:12; identity.yaml:4
18. App-of-apps root Application [SHARED] — pc/README.md:9
19. Argo CD sync waves [SHARED] — apps/config-connector.yaml:20; configconnector.yaml:28
20. Automated sync, prune + selfHeal [SHARED] — apps/systems.yaml:98
21. Tenant AppProject resource blacklist [SHARED] — argocd.yaml:32
22. Kyverno gate keyed on requester identity [SHARED] — deny-raw-cloud-resources.yaml:84
23. Kyverno kind selectors list groups by name [SHARED] — deny-raw-cloud-resources.yaml:21
24. Kyverno `failurePolicy` [SHARED] — deny-raw-cloud-resources.yaml:61
25. Workload Identity [SHARED] — identity.yaml:29
26. IAM database authentication [SHARED] — databases.yaml:115, :169
27. The GRANT Job [SHARED] — _jobs.tpl:126
28. Cloud SQL Auth Proxy [SHARED] — sh/k8s/deployment.yaml:83
29. Private Services Access [SHARED] — environments/reference.yaml:43
30. Terraform layers 0-3 in platform-bootstrap [SHARED] — pc/README.md:7
31. Cloud SQL deletion protection [SHARED] — databases.yaml:111
32. IAM member immutability, team in the object name [SHARED] — identity.yaml:49
33. Size is a class mapped in the recipe [SHARED] — namespace.yaml:8; databases.yaml:49
34. System name is the identity [SHARED] — databases.yaml:30, :61; _helpers.tpl:22
35. GCP labels cannot hold `/`: second unprefixed label set [SHARED] — databases.yaml:77, :80
36. Jobs named by a hash of their spec [SHARED] — _jobs.tpl:3; databases.yaml:191

**Config Connector totals: kinds 20, mechanisms 36, total 56.**

With ADR-0018's three additions, each counted once:
- 57 eighth repo, platform-roles (README.md:1; "one of eight")
- 58 the custom role(s): four custom roles + one IAM Condition (platform-roles/roles/*.yaml; terraform/roles.tf)
- 59 a second home for Terraform: platform-roles/terraform/ with its own state prefix (README "The mental model")

**Config Connector total with additions: 59.**

## 4. Totals

| | Kinds | Mechanisms | Total | With ADR-0018 additions |
|---|---|---|---|---|
| Crossplane (3bfc72d) | 28 | 36 | 64 | n/a |
| Config Connector (e8eedeb) | 20 | 36 | 56 | 59 |

"Less of the reader" is falsified if the with-additions total is not lower than Crossplane's. 59 < 64: lower, by 5 (7.8%). The margin without additions is 8 (12.5%). The claim's prediction ("still fewer, by a smaller margin") holds on this desk count.

## 5. Shared items
Kinds (11): Application, AppProject, Namespace, ResourceQuota, RoleBinding, ClusterRole, ServiceAccount (core), ConfigMap, Secret, Job, ClusterPolicy.
Mechanisms (20): team = Google group, app-of-apps, sync waves, automated sync prune/selfHeal, AppProject blacklist, Kyverno identity gate, Kyverno groups by name, Kyverno failurePolicy, Workload Identity, IAM database authentication, GRANT Job, Auth Proxy, Private Services Access, Terraform layers, Cloud SQL deletion protection, IAM member immutability, size classes, name-is-identity, GCP label slash rule, hash-named Jobs.
Engine-specific: Crossplane 17 kinds + 16 mechanisms = 33; Config Connector 9 kinds + 16 mechanisms = 25. The 8-point gap is entirely in engine-specific items.

## 6. Items a reader might dispute (listed, not tuned)
Granularity is the largest lever: each of the three `cnrm.cloud.google.com/*` annotations is one item (3); if bundled as one, CC falls by 2. Crossplane's `forProvider`/`managementPolicies`/`external-name` are likewise separate.
1. ImageConfig excluded (image-mirror rule). Included, XP +1. Same for CC's vendored operator manifest (excluded).
2. DeploymentRuntimeConfig and Provider included as "identity/which kinds exist". If install kinds were excluded: XP -2, CC -1 (ConfigConnector).
3. `Project` as an IAMPolicyMember resourceRef target (CC) and `Group` subject: excluded on both.
4. Kind `Database` (Crossplane form) in sh/k8s/database.yaml is still present on the Config Connector path's repo but skipped by the tenant Application (argocd.yaml:100 `exclude`); excluded as a CC kind and the `exclude` itself not counted as a mechanism (rollback scaffolding). Including it: CC +1 kind (+1 mechanism).
5. Crossplane `crdWatcher` is still set in CC's apps/kyverno.yaml:78; counted only on XP (reason given is the XRD-generated CRD). CC +1 if counted.
6. CC mechanism 14 (Kyverno chart value closing the kubectl door) and 15 (`allowExistingViolations: false`) exist only because of a9e2bdd and the 2026-10-06 rehearsal; XP at 3bfc72d had neither (XP policies set `allowExistingViolations: true`). Counted on CC as the files cite them. If dropped, CC -2.
7. Crossplane's budget policy (apiCall, mechanism 15) is a policy the CC side no longer has; counted where the file is.
8. Terraform layers as a shared mechanism (not a kind): if Terraform is not an item, both fall by 1 and the "second home" addition still counts.
9. Not counted on either side: the image plane (ADR-0010 remotes, Docker Hub remote for the Job image), Argo CD `ServerSideApply`, GKE/Google Workspace groups, PITR/edition settings other than as folded.
10. Mechanism 1 (CC) and XRD-as-CRD (XP 2) are parallel "how the form is enforced" items; one could argue they are one concept each way, or four.

## 7. Narrow line count (non-comment, template comments stripped; matrix.py logic copied to c30-concepts/matrix.py; `python3 -I`)

Scripts: c30-concepts/run.py (output: linecount.txt). Recipe = Compositions vs charts/system + charts/claims templates; schema = two XRDs vs two values.schema.json; policy = kyverno/policies/.

| Category | Crossplane 3bfc72d | CC 6949999 | CC main e8eedeb |
|---|---|---|---|
| recipe | 633 | 654 | 656 |
| schema | 297 | 383 | 383 |
| policy | 144 | 153 | 153 |
| total | 1074 | 1190 | 1192 |

Main vs Crossplane: +118 lines, +11.0%. Note: the 2026-10-23 table's 1206 for 6949999 also included Chart.yaml in chart meta (16); the fixed reading counts templates only, which gives 1190 for 6949999 (re-run here and confirmed).
Changed since 6949999 within the counted sets: only `charts/system/templates/argocd.yaml` (+2 non-comment lines: the `policies.kyverno.io` group rows added by a9e2bdd). `kyverno/policies/` is unchanged in a9e2bdd (no file added or changed there); a9e2bdd also changed apps/kyverno.yaml (outside the count), charts/kyverno, and scripts/check_charts.py.
