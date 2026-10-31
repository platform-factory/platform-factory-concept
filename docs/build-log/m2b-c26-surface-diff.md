# C-26 surface diff: tenant surface at M2 (Crossplane) vs now (Config Connector)

Claim (claims-register.md, M2b): "The files in systems/tenants/ drive the new engine without edit and materialize the same tenant surface; a new tenant is still one file." Data asked: objects in M2's surface and absent now, and the reverse.

## Sources and SHAs fresh read-only clones
- platform-config main e8eedebdb57052547cbdd77a978f413e7b654b10 (now; charts, apps/systems.yaml, environments/reference.yaml)
- platform-config 3bfc72d749bdaa45597f4e6116c2fbf6ef474b4d (M2 end; crossplane/compositions/{system,database}/composition.yaml, apps/systems.yaml), read via git show
- systems main a79a910afbf0fa48d96fdb9889a51508c6f2a606 (tenants/svc-hello.yaml, svc-ledger.yaml, svc-onboard.yaml, retired.yaml = `retired: []`)
- svc-hello main 76a629486569e891570ed201d079f1a9e8b435a8 (claims.yaml)
- concept repo d771a90c12e8339b58d54449abab65c92efef3cd (m2-paved-road.md, ADR-0017, ADR-0021)
- Bytes: `git diff 8c7b0cb HEAD -- tenants/svc-hello.yaml tenants/svc-ledger.yaml` in systems is empty (8c7b0cb = 2026-09-21, the last M2 edit); 0 bytes changed.
- Render method: c26/render.py imports platform-config/scripts/check_charts.py (render_system, values_object, render_claims) with Helm 3.19.4; value files in Argo CD order (tenant file, retired.yaml, environments/reference.yaml). Outputs in c26/out/. svc-hello's claims.yaml defines `main` and `second`; the claims render uses `main` only, as instructed (the claims Application's valuesObject is from the real system render).
- M2 inventory cross-check: m2-paved-road.md C-05 data ("namespace; ResourceQuota; two RoleBindings (admin, crossplane-edit); SA; AppProject and Application; Google SA and Workload Identity binding; four project IAM members; Artifact Registry repository and writer member"; "crossplane render shows ... 17 after"). 17 composed objects = the System rows below minus the XR.

## What apps/systems.yaml makes per tenant
- M2: ONE Application `systems` (path tenants/ of the systems repo) applying each file as a cluster-scoped `System` XR; the XR composed 17 objects. A Database claim `main` (k8s/database.yaml in svc-hello, applied by the tenant's own Application) made a `Database` XR.
- Now: ONE ApplicationSet `systems` (git files generator `tenants/*.yaml`; create-update; Prune=false,Delete=false; finalizer) generating per tenant ONE Application `<name>-system` (project default, charts/system, 3 value files). That Application renders the rows below, including the tenant's AppProject `<name>`, Application `<name>`, and Application `<name>-claims`.
- Rendered per tenant: svc-hello 19 objects (system chart) + 6 (claims chart, claim main); svc-ledger 19 + 0 (claims chart renders an empty document).

## Table A: svc-hello (team payments, size S, claim main)
"M2" = composed by System/Database Compositions at 3bfc72d. "Now" = helm render.

| Object (kind / name) | M2 | Now | Note |
|---|---|---|---|
| System XR `svc-hello` (cluster-scoped) | y | n | engine machinery |
| Application `systems` (applies System files) | y | n | replaced by ApplicationSet `systems` + Application `svc-hello-system` (n at M2, y now) |
| Namespace `svc-hello` | y | y | same 4 `platform.thecloudgeek.io/*` labels, same `repo` annotation. Now adds `cnrm.cloud.google.com/project-id` and `argocd.argoproj.io/sync-options: Prune=false`; M2 had `composition-resource-name` |
| ResourceQuota `svc-hello` | y | y | identical hard limits (2/4Gi/4/8Gi/20 pods) |
| RoleBinding `svc-hello-admin` (ClusterRole admin, group payments@thecloudgeek.io) | y | y | same |
| RoleBinding `svc-hello-crossplane-edit` (ClusterRole crossplane-edit) | y | n | |
| ServiceAccount `svc-hello` (workload-identity annotation) | y | y | same |
| ConfigMap `platform-system` | y | y | same 11 data keys and values |
| AppProject `svc-hello` | y | y | sourceRepos, destinations, developer role identical. Blacklist: M2 `*.upbound.io`, `*.crossplane.io`, rbac, ResourceQuota, LimitRange, `kyverno.io`; now `*.cnrm.cloud.google.com`, `argoproj.io`, rbac, ResourceQuota, LimitRange, `kyverno.io`, `policies.kyverno.io` |
| Application `svc-hello` (tenant's repo, path k8s) | y | y | now adds `directory.exclude: database.yaml` |
| Application `svc-hello-claims` | n | y | new |
| Application `svc-hello-system` (generated per tenant) | n | y | new |
| ServiceAccount `database-seed` | n | y | new |
| Role `database-seed` (secrets get/create) | n | y | new |
| RoleBinding `database-seed` | n | y | new |
| Google SA `svc-hello@platform-factory-ref` (M2 MR `ServiceAccount svc-hello-gsa`; now `IAMServiceAccount svc-hello`) | y | y | same cloud resource; description differs (M2 "...(team payments). Composed by the System XR..."; now "...Rendered by the platform..." with no team) |
| Workload Identity binding (`ServiceAccountIAMMember` / `IAMPolicyMember svc-hello-workload-identity`) | y | y | same role and member |
| Project grants `svc-hello-sql-client-app`, `-sql-instance-user-app`, `-sql-client-team-payments`, `-sql-instance-user-team-payments` (M2 `ProjectIAMMember`; now `IAMPolicyMember`) | y | y | same names, roles, members |
| Artifact Registry repo `svc-hello` (M2 MR `RegistryRepository svc-hello-registry`; now `ArtifactRegistryRepository svc-hello`) | y | y | same location/format/description; labels system/team/tier/security-tier kept; `managed-by: crossplane` gone from the template; no Delete in M2's managementPolicies, now `deletion-policy: abandon` |
| Registry writer (`RegistryRepositoryIAMMember` / `IAMPolicyMember svc-hello-registry-writer-payments`) | y | y | same role and member |
| Database XR/claim `main` (from svc-hello k8s/database.yaml) | y | n | excluded from the tenant Application now; claims.yaml read instead |
| Cloud SQL instance `svc-hello-main` (M2 `DatabaseInstance`; now `SQLInstance`) | y | y | same Postgres 16, db-f1-micro, ZONAL, private IP, IAM auth flag, deletion protection, backups on. GCP label `managed-by: crossplane` dropped from the template; `database`, `system` kept |
| Database `app` (`SQLDatabase svc-hello-main-app`) | y | y | same |
| IAM DB user `svc-hello@platform-factory-ref.iam` (`SQLUser svc-hello-main-iam-user`) | y | y | same |
| Secret `main-admin` | y (composed) | n (rendered) | now created at runtime by the seed Job, not rendered or tracked by Argo CD |
| ConfigMap `main-connection` | y | y | M2 keys connectionName, privateIp, databaseName, iamUser; now no `privateIp` key |
| Job `main-grant-<hash>` | y | y | hash differs; M2 ran as ServiceAccount `svc-hello`, now `svc-hello` too (grant Job) |
| Job `main-seed-<hash>` (runs as `database-seed`) | n | y | new |

## Table B: svc-ledger (team checkout, size S, no claim)

| Object | M2 | Now | Note |
|---|---|---|---|
| Same 14 rows as svc-hello (Namespace, ResourceQuota, RoleBinding admin, ServiceAccount, ConfigMap, AppProject, Application, Google SA, WI binding, 4 project grants, registry repo, registry writer) with name `svc-ledger`, team `checkout` | y | y | same notes as Table A, including the AppProject blacklist change and `directory.exclude: database.yaml` |
| System XR `svc-ledger`; RoleBinding `svc-ledger-crossplane-edit` | y | n | as Table A |
| Application `svc-ledger-system`; Application `svc-ledger-claims`; ServiceAccount, Role, RoleBinding `database-seed` | n | y | present although there is no claim (claims Application renders nothing) |
| SQL instance, database, user, Secret, ConfigMap, Jobs | n | n | no claim, none either side |
| The four project grants and the registry writer for svc-ledger | y | y | granted without a database in both engines |

## Present in M2, absent now
1. System XR (cluster-scoped) per tenant; the Application `systems` that applied the files; the Database XR/claim `main` (k8s/database.yaml). Class (a): engine machinery. The claim is replaced by `claims.yaml` read by `<name>-claims` (ADR-0017 §4, §5, §12).
2. RoleBinding `<name>-crossplane-edit`. Class (b) and (c): ADR-0017 §7 says "the team's `crossplane-edit` binding goes away and the team holds the built-in `admin` role alone"; §7 also records that `cnrm-admin` aggregates into `admin`, which widens what the team can do on Config Connector kinds.
3. Secret `main-admin` as a rendered object. Class (a): generated at runtime by the seed Job (ADR-0017 §5, generate-once root password).
4. `privateIp` key in ConfigMap `main-connection`. Class (c) with a visible consequence: ADR-0017 §5 (the Job looks the address up at run time); a consumer of the key would lose it.
5. GCP label `managed-by: crossplane` (SQL instance, registry). Class (c): ADR-0017 §8 ("the one label that goes").
6. AppProject blacklist entries `*.upbound.io` and `*.crossplane.io`. Class (a): replaced by `*.cnrm.cloud.google.com` (ADR-0017 §4).
7. Team name in the Google SA description. Class (b), cosmetic; no ADR found.

## Present now, absent in M2
1. Application `<name>-system` per tenant (from the ApplicationSet) and Application `<name>-claims`, even for svc-ledger with no claim. Class (a) and (c): ADR-0017 §3, §4.
2. ServiceAccount, Role (secrets get/create) and RoleBinding `database-seed` in every tenant namespace, claim or not. Class (b)/(c): ADR-0017 §5 seed Job; the same ADR says which chart owns the account is "[unverified]".
3. Job `main-seed-<hash>`. Class (c): ADR-0017 §5.
4. AppProject blacklist entries `argoproj.io` (ADR-0017 §4) and `policies.kyverno.io` (platform-config#33; ADR-0021 Context, not ADR-0017). Class (b) and (c): a tenant can no longer sync those kinds.
5. Application `<name>` field `directory.exclude: database.yaml`. Class (c): ADR-0017 §12 (rollback needs both engines' files).
6. Namespace annotations `cnrm.cloud.google.com/project-id`, `Prune=false`. Class (a)/(c): ADR-0017 §8; ADR-0019 §1.
7. Cloud-side only: Config Connector adds `managed-by-cnrm: true` to cloud labels (visible live). Class (a).

## Cross-check with Session A (the Session A audit reads, `evidence/session-a/`)
- reconstruction.md C-26, third tenant svc-onboard: first cloud writes are AR CreateRepository, CreateServiceAccount, SetIAMPolicy on the SA (workload identity), project SetIamPolicy (cloudsql.client and instanceUser for the GSA; team grants already present), AR SetIamPolicy (writer). That is the 8 cloud-facing objects of the "now" list (repo, GSA, WI, 4 project grants, writer); the project grants appear as one SetIamPolicy that added only the GSA because the team members already existed. No cloud object beyond the render appeared.
- project-policy-now.json: cloudsql.client and instanceUser each hold group:checkout, group:payments and the three GSAs, matching 4 grants per tenant. repo-svc-hello-policy-now.json: writer = group:payments, matches.
- sql-instances-now.json: svc-hello-main is the M2 instance (createTime 2026-09-16), userLabels still `managed-by: crossplane` (the render no longer carries it; the live instance keeps it, cause not determinable from the files). svc-hello-second exists live (second claim of claims.yaml); not in the single-claim render. No instance for svc-ledger or svc-onboard, matching.
- ar-repos-now.json: all three repos carry `managed-by: crossplane` plus `managed-by-cnrm: true`.
- service-accounts-now.json: live descriptions for svc-hello, svc-ledger and svc-onboard still read "Workload identity for System X (team T). Composed by the System XR; do not edit by hand.", whereas the chart renders "Rendered by the platform; do not edit by hand." Git shows the chart text has been unchanged since 90a522d. UNRESOLVED mismatch: not explained by the files; svc-onboard's SA was created by Config Connector at 22:32:45 yet shows the old text.
- In-cluster objects (database-seed trio, claims Application, Secret main-admin) are not in the cloud audit; this cross-check covers cloud objects only.
