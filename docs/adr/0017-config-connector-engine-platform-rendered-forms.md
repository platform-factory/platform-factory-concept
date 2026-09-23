# ADR-0017: Config Connector is the cloud engine; the form is a values file only the platform renders

Status: Accepted · September 2026 · supersedes ADR-0005's second decision (Crossplane as the composition layer) and its third consequence; replaces the *mechanism* — not the decision — of ADR-0003, ADR-0012 §3 and §6, ADR-0013 §1–3 and §6, ADR-0014 §1–4, ADR-0015 §1–2 and §4 and ADR-0016 §1, §3 and §6; extends ADR-0015 §1's durable set to the database user; adds the System's tier to ADR-0014 §2's size-L rule; adds a name-reuse rule to ADR-0015; adds reserved System names to ADR-0012 §1; widens the exception to ADR-0010's first decision that ADR-0013 §4 opened; removes the subject of C-25; pre-registers C-26..C-30; its engine identity's roles are narrowed by ADR-0018 · checked against primary sources and corrected on 2026-09-19, edited on 2026-09-21 to meet ADR-0018, and edited again on 2026-09-22 for a second outside review, all before its first commit (see the last three consequences)

## Context

M2 built the paved road on Crossplane and graded it: C-05 HELD, C-06 and
C-07 ADJUSTED. It works. What it costs is the list of things a maintainer —
and every reader of a public reference implementation — has to learn before
they can follow one tenant from its file to its cloud resources: the XRD, the
Composition, a three-function pipeline, the EnvironmentConfig, a provider
family that runs one pod per Google service, `managementPolicies`, two
opposite rules for the external-name annotation, and the RBAC manager's
private aggregation labels. The two recipes are 1,051 and 794 lines, and in
the System recipe a large share of that is commentary on engine mechanics
rather than on what a tenant gets. Two of ADR-0016's four findings were
defects in the provider layer itself: it silently refuses a replacement while
reporting Ready, and it cannot create a passwordless database user at all.

ADR-0005 chose Crossplane over Config Connector for one reason: the provider
family is the swappable part, "which makes the portability story concrete."
Two milestones in, that reason has bought nothing the build has used — C-08,
the only claim that would exercise it, was not attempted — and it is paid for
on every page.

So the question was reopened on 2026-09-19: **what is the least machinery that
still keeps the one property the pattern depends on?** That property is worth
naming exactly, because the first two candidates lose it without looking like
they do:

> **The developer-facing form is an allowlist.** A developer can say five
> things about a database and nothing else. ADR-0014 §1 put it as "the denial
> for 'oversized' is that the field cannot say it."

The mental model that sorts the candidates: **a form is enforced by whoever
turns it into resources.** A Terraform module handed to a team is a
convenience — they can fork it or write resources beside it, so the guardrail
has to be policy over everything they might write. An internal API is a
boundary — they submit five fields and something they do not control does the
work. Four candidates, judged on which of those they are:

1. **Config Connector + kustomize; the developer writes raw cloud objects,**
   perhaps starting from a platform base. Fewest concepts. But the tenant's
   Argo CD project must then allow raw `SQLInstance`, and Kyverno has to
   enumerate every field of every kind a developer must not set. The form
   becomes a blocklist, and C-07 becomes a claim about policy coverage.
2. **Config Connector + a Helm chart rendered by the tenant's own
   Application.** The same thing with better ergonomics: the rendered output
   is applied under the tenant's project, so the chart is a module, not an API.
3. **Config Connector + kro.** Keeps the allowlist and keeps denial at the API
   server. kro is at v0.9.4 with its API still `v1alpha1`, and the
   cluster-scoped instances `System` needs shipped in v0.9.0, March 2026
   [C, kro GitHub releases, 2026-09-19]. A new controller and a new concept,
   both pre-1.0, in a repo people copy from.
4. **Config Connector + a Helm chart rendered by a platform-owned
   Application; the developer supplies only a values file.** The allowlist is
   kept by *who runs the recipe*. No new controller: Argo CD and Helm are
   already here.

Mechanics checked against primary sources on 2026-09-19, before deciding:

- An Argo CD multi-source Application may take Helm value files from a second
  Git repository; a source with `ref` set and `path` unset is used "solely as
  a source of value files" and generates no manifests [C, argo-cd
  `docs/user-guide/multiple_sources.md`].
- Argo CD ships a wildcard health check for `*.cnrm.cloud.google.com`
  alongside the ones for `*.crossplane.io` and `*.upbound.io` [C, argo-cd
  `resource_customizations/` listing]. M1's "nil health is instant success"
  trap (and layer 3's hand-written checks) should not recur for these kinds.
- Config Connector's `SQLUser` documents `type` as `BUILT_IN`,
  `CLOUD_IAM_USER` or `CLOUD_IAM_SERVICE_ACCOUNT`, makes `password` required
  for Postgres "unless type is set to either CLOUD_IAM_USER or
  CLOUD_IAM_SERVICE_ACCOUNT", and has a `resourceID` "used for creation and
  acquisition" [C, CRD `sqlusers.sql.cnrm.cloud.google.com`, master]. The
  passwordless user the pinned Crossplane provider panics on is the documented
  case here. `CLOUD_IAM_GROUP` is **not** in that list.
- The same CRD is labelled `tf2crd: "true"`: this kind is generated from the
  Terraform provider inside Config Connector. "No Terraform underneath" is not
  what this ADR claims; "one vendor maintains the wrapper and the cloud" is.
- Config Connector's cluster mode runs one controller under one Google service
  account named in `ConfigConnector.spec.googleServiceAccount` [C,
  `docs/features/config-connector-mode.md`] — the same posture as today's
  single provider identity.
- A reference to another resource is either a Kubernetes `name`/`namespace` or
  an `external` cloud identifier, and the project can be given by the
  `cnrm.cloud.google.com/project-id` annotation, on the resource or on its
  Namespace: "You can set a default project ID for newly-created resources by
  annotating your Kubernetes namespace" [C, Config Connector docs,
  project-scoped resources]. Two edges from the same page: the annotation has
  to be on the Namespace before any resource is created in it, and a Namespace
  without one makes Config Connector use the namespace's *name* as the project.
  (The Namespace half was confirmed after drafting; the first draft carried it
  as [I].)

Four more were checked the same day, after the decision was drafted and before
it was committed. Each one changed the text below:

- Helm merges value files in order and the last one wins; Argo CD adds its own
  order on top, `parameters > valuesObject > values > valueFiles > helm
  repository values.yaml` [C, argo-cd v3.4.6 `docs/user-guide/helm.md`].
  `values.schema.json` is checked against the merged result [C, helm v3.19.4
  `pkg/chartutil/values.go`], so a schema cannot tell which file supplied a
  key.
- Config Connector installs a ClusterRole, `cnrm-admin`, labelled
  `rbac.authorization.k8s.io/aggregate-to-admin` and `aggregate-to-edit`, with
  `get`, `list`, `watch`, `create`, `update`, `patch` and `delete` on every
  resource in each of its 164 resource API groups [C, k8s-config-connector
  v1.156.0 `config/installbundle/components/clusterroles/cnrm_admin.yaml`, and
  the manifest rendered inside the 1.156.0 operator image]. Anyone bound to
  Kubernetes' built-in `admin` in a namespace can create, edit and delete cloud
  objects there from the moment the engine is installed.
- `SQLInstance` is not one of the Terraform-generated kinds. A direct
  controller arrived in 1.125 as an opt-in and became the only reconciler for
  every `SQLInstance` in 1.129.2 [C, k8s-config-connector release notes]; with
  it, "KCC no longer treats unspecified field as unmanaged" [C, maintainer,
  k8s-config-connector issue #6360]. On acquiring an existing instance it
  fills most fields the manifest leaves out with its own default — public IPv4
  *on*, edition `ENTERPRISE`, availability `ZONAL` — copies a few from the
  live instance instead, and, whenever anything differs, sends the whole
  object as an update [C, v1.156.0
  `pkg/controller/direct/sql/sqlinstance_defaults.go`,
  `sqlinstance_controller.go`].
- For the other kinds the charts use, Config Connector's own default copies
  the cloud's values back into `spec` (`state-into-spec: merge`), and Google's
  page names Argo CD as a tool that fights with it. The annotation that turns
  this off cannot be added or changed once the object exists [C, docs
  `concepts/ignore-unspecified-fields`].

## Decision

**Config Connector creates cloud resources. Helm charts owned and rendered by
the platform are the recipes. The developer-facing form is a values file, and
it is enforced by the fact that only the platform's Argo CD project may turn
it into cloud objects. Crossplane is retired from the reference path.**

1. **The engine is Config Connector in cluster mode, under one Google
   identity.** Through both legs of C-29 it holds the same roles
   `crossplane-provider-gcp` holds today, including
   `roles/resourcemanager.projectIamAdmin` under the same two-role IAM
   Condition (ADR-0013 §6), so the cutover changes the engine and not its
   permissions. From the start it also holds the four custom roles of
   ADR-0018 — each checked at the desk, before it is bound, to be a subset of
   the built-in role it replaces, and so adding nothing (ADR-0018 §4;
   unverified until that read is done) — and the built-in ones are removed afterwards, with the
   cluster down, as a measured change (C-31). Namespaced mode — one identity per
   tenant namespace — is the tighter design and is deferred for the reason
   layer 0 already gives for not splitting the provider identity five ways.
   This is a Terraform crossing in ADR-0016 §7's sense, declared here before
   the milestone that needs it: one layer-0 apply adds the identity and its
   Workload Identity binding, and the old identity leaves in a later apply
   once the cutover has held and its rollback has been exercised. The new
   identity's role grants are not in layer 0 at all: they are two further
   declared crossings, applied by hand from `platform-roles` (ADR-0018 §1 and
   §4). The binding's member is
   `serviceAccount:<project>.svc.id.goog[cnrm-system/cnrm-controller-manager]`,
   and it is the only one the install needs [C, Config Connector
   manual-install doc]. The install itself is one vendored file — the operator
   manifest from the version-pinned bundle, 1.156.0, never `latest` — and one
   `ConfigConnector` object with `mode: cluster` and `stateIntoSpec: Absent`.
   Until C-29's reverse leg — the rollback decision 12 describes — has run
   once, every change to `platform-bootstrap` is additive: the old identity, Crossplane core's
   registry-reader grant and layer 3's health checks for `System` and
   `Database` all stay, because that repo is not selected by branch and a
   Crossplane rebuild has to keep working. `platform-roles` never names the
   old identity.

2. **The recipes are two charts in `platform-config`: `charts/system` and
   `charts/claims`.** The body of each Composition is already a Go template,
   so the tenant surface moves across nearly verbatim with Config Connector
   kinds in place of provider kinds. What does not come along is everything
   that existed to satisfy the engine: the pipeline steps, the readiness
   annotations, the namespace ordering gate, the composition-resource-name
   annotations, and the status document.

3. **The tenant file does not change — not one byte.**
   `systems/tenants/<name>.yaml` keeps today's content, envelope included. It
   stops being applied as an object and starts being read as values
   (`.Values.metadata.name`, `.Values.spec.owner.team`). An ApplicationSet in
   `platform-config` generates one Application per file, in the platform's
   project, rendering `charts/system` with that file and then the environment
   file (decision 8) — in that order, because the last value file wins and the
   platform's facts must. Changing the form and the engine in the same step
   would make the re-test uninterpretable, and an unchanged file is the
   strongest evidence C-08's idea has been offered so far. `tenants/` stays
   flat — no subdirectories — because the generator's default globbing matches
   `*.yaml` at any depth, so a file in a subfolder would become a tenant [C,
   argo-cd v3.4.6
   `docs/operator-manual/applicationset/Generators-Git-File-Globbing.md`]. The
   generated Application is named `<system>-system`, not the bare file name:
   the System chart still emits the tenant's own Application as `<system>` in
   the same namespace (decision 4), and an Application that rendered an
   Application of its own name would overwrite itself. So each tenant has
   three: `<system>-system`, `<system>` and `<system>-claims`. A check in the
   `systems` repo — that repo's first; like `platform-config` it has only a
   CODEOWNERS file today — holds the file name equal to `metadata.name`. The
   API server no longer guarantees that System names are unique, and the
   ApplicationSet's templates run with `missingkey=error` (a missing field is
   an error, not an empty string), so one malformed file stops generation for
   every tenant [C, argo-cd v3.4.6
   `applicationset/controllers/applicationset_controller.go`]. "The platform's
   project" is, for M2b, Argo CD's built-in
   `default` project, as it is for every platform Application today. A
   dedicated project would be one more object to learn and would not be the
   wall anyway. The wall is the *tenant's* project — its own namespace only, no
   cluster-scoped kinds, Config Connector's groups blacklisted — together with
   Argo CD honouring Applications only in the `argocd` namespace, which no
   tenant project can write to.

4. **The System chart emits a second Application per tenant,
   `<system>-claims`, and this is where the form is enforced.** It belongs to
   the platform's project and renders `charts/claims` from `platform-config`.
   It takes exactly one value file — `claims.yaml` from the tenant's repo,
   through a `ref` source with no `path`, with `ignoreMissingValueFiles` so a
   service with no claims renders nothing. The platform's facts (the
   environment values, the System's name, its team and its tier) arrive a
   different way: the System chart already holds them when it renders this
   Application, so it writes them into the Application's `helm.valuesObject`,
   which outranks every value file. Helm merges value files key by key [C,
   helm v3.19.4 `pkg/cli/values/options.go`, `mergeMaps`], so `valuesObject`
   protects only the keys it writes: it writes every key of `environment` as
   the environment file gives it, and every key of `system` with the
   template's defaults applied (`tier` is `standard` when the tenant file in
   `systems` leaves it out), and leaves none for a `claims.yaml` to fill. The
   claims chart builds every name from `.Release.Namespace`. Argo CD sets that
   from the Application's destination (or its `helm.namespace`, which the
   platform also writes), and no value file can override it [C, argo-cd
   v3.4.6 `reposerver/repository/repository.go`, `helmTemplate`, and
   `util/helm/cmd.go`]. The chart fails the render if `system.name` differs
   from `.Release.Namespace`. The first draft of this decision listed the
   environment file first and the tenant's file second. Since the last file
   wins and the
   schema sees only the merged tree, that would have let a `claims.yaml`
   replace the project, the network, or the registry that supplies the `GRANT`
   Job's image — the one container that runs holding the database's superuser
   password. The schema closes the
   `environment` and `system` objects too, and a fixture proves that a
   tenant-supplied `environment.projectID` changes nothing. The tenant's own
   Application and AppProject are as they are today, except that the project's
   blacklist names `*.cnrm.cloud.google.com` where it named `*.upbound.io` and
   `*.crossplane.io`, and gains `argoproj.io`; the blacklist matches group
   globs [C, argo-cd v3.4.6 `pkg/apis/application/v1alpha1/types.go`,
   `isResourceInList`]. A tenant has no use for an Application, an AppProject
   or an ApplicationSet of its own. Without that entry one placed in its `k8s/`
   with no namespace would be created in the tenant's namespace, where Argo CD
   never reads it — harmless, but admitted; with it, the sync refuses it by
   name. (Argo Rollouts shares the group. It is not installed, and adopting it
   would mean narrowing this entry to kinds.) The
   developer controls the values and nothing else: not the chart, not its
   version, not the project it renders under. Two limits of
   `ignoreMissingValueFiles`, both from the same code [C, argo-cd v3.4.6
   `reposerver/repository/repository.go`]: it forgives a missing *file*, not a
   missing repository or branch, so a tenant's repo has to exist with a `main`
   branch before its tenant file merges; and the skip is logged at debug level
   only, so a file named `claims.yml` is ignored in silence — the service
   repo's check asserts the name.

5. **The `Database` form keeps its five fields; its envelope changes, and a
   claim's name may no longer contain a hyphen.** The form leaves `k8s/` —
   that directory is applied as manifests, and this is no longer one — for a
   `claims.yaml` beside it:

   ```yaml
   databases:
     main:                 # was metadata.name; still yields <system>-main
       engine: POSTGRES_16
       region: us-central1
       size: S
       tier: standard
       backups: true
   ```

   A map keyed by claim name, so names are unique by construction and a second
   kind of claim (`buckets:`) is a second key, not a second mechanism.

   A claim name may not contain a hyphen: the schema binds the keys of
   `databases:` to `^[a-z][a-z0-9]{1,33}$` with `propertyNames`, the same 2 to
   34 characters as before, and a refusal fixture keeps the keyword honest.
   The instance is `<system>-<claim>` and System names may contain hyphens, so
   under M2's rule System `orders` with claim `api-main` and System
   `orders-api` with claim `main` both name `orders-api-main`, and adoption by
   name would hand one tenant the other's instance. With no hyphen in the
   claim, whatever follows the last hyphen is the claim; `main` complies, so no
   existing claim changes.

   `backups: true` keeps what M2's recipe does, now described as it is: daily
   backups on every size, taken only while the instance runs, so a parked one
   takes none [C, Cloud SQL for PostgreSQL backups doc; for `svc-hello-main`,
   `gcloud sql backups list` on 2026-09-22 beside `cycle.sh`'s
   `cycle-results.tsv`: its only backup is from the day before it was first
   parked]; and point-in-time recovery only on size L, because Cloud SQL
   requires it for a regional instance [C, Cloud SQL for PostgreSQL "Enable
   and disable high availability" doc, REST steps]. The live
   `svc-hello-main` has point-in-time recovery off [C, `gcloud sql instances
   describe`, 2026-09-22], and the chart renders what is live (decision 9), so
   whether recovery follows `backups` on every size is a measured change after
   the cutover, not part of it.

6. **The schema still denies first — at render time and in CI, no longer at
   the API server.** `charts/claims/values.schema.json` closes every level
   with `additionalProperties: false`; engine, region, size and tier are
   enums; the per-System claim limit is `maxProperties` on `databases:`
   (retiring the Kyverno policy `database-claim-budget`); cross-field rules
   are `if`/`then` (replacing CEL). The claim limit counts live claims, not
   instances: a removed claim's instance is kept (decision 9), so the limit
   bounds neither retained instances nor spend. The Kyverno policy it retires
   fails open and counts the claims already in the namespace each time one is
   created; the limit reads one file at one revision and fails closed. Two
   pull requests that each pass on their own can both merge; the next render
   then refuses the combined file and holds every change to that System's
   databases until one claim is removed. Three chart rules make
   the schema a security boundary rather than a courtesy: every
   tenant-supplied value is enum-bound or quoted where it is used; the chart
   never calls `tpl` on a tenant value; names are pattern-bound (decision 5
   gives the claim-name pattern).

   The size-L rule gains an entitlement: besides the claim's own
   `tier: critical`, it requires the System's `tier` to be `critical`. The
   claim's `tier` is the developer's statement; the System's is the
   entitlement, approved in `systems` and written into `valuesObject`
   (decision 4). M2's rule read only the first, although the System schema's
   own description called the System's tier the gate.

   `charts/system` gets a closed
   `values.schema.json` too. It carries the System XRD's rules over unchanged
   (the name, team and repo patterns; the tier, security-tier and size enums)
   and refuses four more kinds of System name. One ending in `-system` or
   `-claims`: every tenant's three Applications share the `argocd` namespace,
   and System `orders` owns `orders-claims`, which is also the tenant
   Application of a System named `orders-claims`. One naming a namespace the
   platform or GKE already uses (`argocd`, `default`, `kyverno`, or one
   starting `kube-`, `gke-`, `gmp-` or `cnrm-`): the chart would bind the team
   as `admin` there, overwrite an Argo CD project, and on offboarding delete
   the namespace. One naming a platform Application in `argocd` (`systems`,
   for one), which the tenant's Application of the same name would overwrite.
   And one naming a Google service account or an Artifact Registry repository
   the platform's Terraform creates (for example `jumpbox`,
   `crossplane-provider-gcp` or `quay-io`), which the chart would acquire by
   name: it would bind the account to the tenant's pods, and try to make the
   repository the tenant's registry, with the team's grant and the tenant's
   `system` label [I]. The tenant
   file is approved by the platform, so this schema guards against mistakes;
   the claims schema is the tenant boundary. Both are enforced at Argo CD's
   render, which no merge can skip. The suffix, reserved-name and
   retired-name (decision 9) refusals are fixtures in `platform-config`'s CI,
   run under `helm template`; none is tried against a live cluster.

   The gate
   order of ADR-0014 §4 stands with new occupants: `helm template`
   against the platform's chart in the service repo's PR check; then Argo CD's
   render of the claims Application; then Kyverno for what neither can judge.
   Two details keep the gates honest. Helm does not apply a schema's
   `default`, where the Database XRD relied on the API server defaulting a
   field before its CEL rule ran; so defaults live in the template, and the
   size-L rule also requires both the claim's and the System's `tier` to be
   present. And `platform-config`, which has no CI today, gets one, because
   its claims schema is now a security boundary.
   That CI renders the real tenant files with the real environment file, runs
   every refusal fixture under `helm template`, and fails if any Application
   sets `helm.skipSchemaValidation` — the gate's only off switch, which the
   platform writes and the tenant cannot. `helm lint` is not that gate: it
   reports a failed `required` or `fail` only as information [C, helm v3.19.4
   `pkg/engine/engine.go`]. It does check the schema, against the chart's
   `values.yaml` and the files it is given [C, same release,
   `pkg/lint/rules/values.go`], so CI runs it only with the real value files.
   Neither chart's `values.yaml` carries an environment key, because that
   file is a value layer beneath every other,
   and a key left in it would quietly fill one the environment file forgot.
   (The Helm 3.19.x pin that keeps the
   service repo's check in step with Argo CD v3.4.6 is recorded in the chart
   README.)

7. **The reality gate keeps its job, loses one of its senses, and gains two
   verbs.** Kyverno denies **create, update and delete** of Config Connector
   kinds in tenant namespaces by any identity other than Argo CD's application
   controller and Config Connector's own two writers — `cnrm-controller-manager`,
   and `cnrm-deletiondefender`, which removes its finalizer with an update on
   every delete [C, v1.156.0 `pkg/controller/deletiondefender/controller.go`].
   It lists concrete API groups, because a wildcard group matches nothing
   (ADR-0016 §3): `sql`, `iam` and `artifactregistry`. Four of the engine
   identity's five roles — the custom ones of ADR-0018, once its narrowing
   has run — act through those three groups, and the fifth, `compute.viewer`,
   is read-only. ADR-0016 §3's coupling rule moves with the engine: every
   Config Connector group is installed at once, so it is giving the engine
   identity a new service's permissions — not installing a package — that
   means adding a group to the gate. The two now live in different repos, so
   "the same PR" becomes an order: the gate merges in `platform-config`
   first, then the permission is applied from `platform-roles`. Today's rule
   covers
   create only, and that is no longer enough. The team's `crossplane-edit`
   binding goes away and the team holds the built-in `admin` role alone — and
   the first draft's hope, that `admin` would then carry no verbs on these
   kinds, is refuted. `cnrm-admin` aggregates into `admin` and `edit`
   (Context), so the update/delete seam the System recipe documents does not
   close by itself. It widens, to every resource kind the engine installs, and
   the operator owns that role, so labels stripped by hand are expected to
   come back at its next reconcile [I — the operator re-applies its own
   manifests; not tested]. The alternative — binding teams to a
   platform-written role in place of `admin` — was set aside: RBAC cannot
   subtract, so it means keeping a hand-made copy of `admin` in step with
   Kubernetes. The delete rule has to let the namespace controller and the
   garbage collector through, or an offboarded tenant's namespace never
   finishes terminating. Which identities those present to admission is
   [unverified]: a local rehearsal settles it for upstream Kubernetes before
   any cluster session, and the first cluster session confirms GKE presents
   the same ones. What changes in kind: today the gate tells the platform from
   the tenant by *identity* (Crossplane's service account). Now both arrive as
   Argo CD, so on the git path the AppProject is the only wall between them
   and Kyverno's reality gate covers only `kubectl`. Two gates remain; they no
   longer overlap. (The one Kyverno rule that does bind Argo CD is decision
   9's, which guards a single annotation, not the form.) Config Connector's
   include-list (`spec.experiments.resourceSettings`) would make every unused
   kind inert rather than open, but it first shipped in 1.149.1 (GitHub
   release 5 May 2026), is documented only in the repository, and compares
   `mode` to the exact string `include` — any other spelling *disables* the
   listed kinds [C, v1.156.0
   `pkg/controller/registration/registration_controller.go`]. It is left off
   for the cutover and tried afterwards as a measured change. Containment
   rests on Kyverno and on the engine identity holding roles for three
   services only. Those roles are project-wide, and even after ADR-0018's
   narrowing three of the four carry a `setIamPolicy`. The project one sits
   under the two-role condition. The other two — on any repository, on any
   service account — are granted without one: Artifact Registry does not
   recognise the condition's attribute at all, and on service accounts a
   condition could limit which role is granted but never which account
   (ADR-0018, Consequences). The honest list of what IAM does not stop is kept beside the grants in
   `platform-roles`, as layer 0's containment note keeps it for the old
   identity.

8. **What Helm under Argo CD cannot do, and what stands in for each.** It
   renders from git with no view of the cluster, so:
   - *Per-environment facts* (project, region, org domain, registry, network)
     come from `platform-config/environments/<env>.yaml`, which replaces the
     EnvironmentConfig. Every key in that file, `platformRevision` included,
     is required and non-empty: both charts read each with Helm's `required`;
     no template gives one a default, and neither chart's `values.yaml` does;
     and `charts/system` copies each into the claims Application's
     `valuesObject` (decision 4). M2's System recipe filled a missing key with
     the reference project's value, so a second environment that forgot one
     would silently have rendered against the reference build's values. And a
     key left out of `valuesObject` is one a `claims.yaml` could supply. A render with a key removed is a refusal
     fixture in `platform-config`'s CI. The project is stamped on every cloud object by the
     chart helper (below) and also rides on the tenant Namespace's annotation
     as a fallback, written in the same manifest that creates the Namespace:
     without it, an object that lost its own annotation would target a project
     named after the namespace (Context). Taking the project from the
     platform's file means no *tenant-authored* file names a project — not
     that no cloud object does: a project-level IAM member names it in
     `resourceRef.external`, and a database user's `resourceID` is
     `<system>@<project>.iam`.
   - *Ordering* is Config Connector's: a resource whose reference is not ready
     waits on a watch and proceeds when it is, at worst one reconcile period
     (five to fifteen minutes) later, with no give-up timeout [C, v1.156.0
     `pkg/controller/tf/controller.go`, `pkg/controller/jitter/jitter.go`].
     `external` references are not waited on — the string is passed through
     as given [C, v1.156.0 `pkg/krmtotf/references.go`]. While a resource
     waits, Argo CD's built-in health check reports it `Degraded` or
     `Suspended`, and in a sync with more than one wave a Degraded resource
     fails the operation [C, argo-cd v3.4.6
     `resource_customizations/_.cnrm.cloud.google.com/_/health.lua`; same
     tree, `gitops-engine/pkg/sync/sync_context.go`]. So each chart is one
     wave with no hooks, and every platform-rendered Application carries the
     repo's retry block.
   - *The `GRANT` Job's address.* The Job is not a Config Connector object, so
     it retries until it can connect — but today it and the connection
     ConfigMap read the instance's private IP from observed state, which a
     render from git does not have. The Job looks the address up when it runs,
     from the Cloud SQL Admin API, with the metadata token its script already
     fetches and the `roles/cloudsql.client` its service account already holds
     [I]. Same image, no new RBAC. (The ConfigMap's `privateIp` key and the
     Job's spec-hash name are chart details, recorded in the chart README.)
   - *The generate-once root password* of ADR-0013 §3 cannot be a template
     function, because every render would mint a new one. A seed Job in the
     claims chart creates `<claim>-admin` if it is absent; Argo CD does not
     track a Secret it did not apply, so it is neither pruned nor reported as
     drift. ADR-0016 §4's stale-after-rebuild caveat turns out to be a
     property of this engine too, not a leftover of the old one: on an
     *acquired* instance a `rootPassword` that differs from the real one never
     triggers an update, because the controller's diff ignores the field [C,
     v1.156.0 `pkg/controller/direct/sql/sqlinstance_equality.go`]. The value
     still rides in the body of any update another field triggers, and the
     Cloud SQL Admin API documents it as "Initial root password. Use only on
     creation" — so that it never changes the live password is [I until
     C-29]. Which
     image runs
     the seed Job — it needs `kubectl` and a shell, through a registry remote
     that already exists — and which chart owns its service account are
     [unverified].
   - *Status written back to the developer* is dropped. The `platform-system`
     ConfigMap already carries the same facts and all of them are derivable at
     render time.
   - *Three annotations on every cloud object, from its first apply:*
     `state-into-spec: absent`, `management-conflict-prevention-policy: none`
     and `project-id`, stamped by one shared helper. If both the first
     annotation and decision 1's cluster-wide `stateIntoSpec: Absent` were
     missing, Config Connector's own default (`merge`) would write the cloud's
     values into `spec`, which Argo CD reads as drift and `cycle.sh` reads as
     a failed rebuild; and it can never be corrected on a live object
     (Context). Belt and braces is cheap here. Conflict prevention
     stays off because its forty-minute lease would stall every rebuild [C,
     docs `concepts/managing-conflicts`].
   - *Cloud labels are spelled without a prefix.* Config Connector derives a
     resource's cloud labels from `metadata.labels` and drops any key
     containing `/` [C, v1.156.0 `pkg/label/label.go`]; `SQLInstance` has no
     `userLabels` field. `cycle.sh park` and its adoption check find resources
     by a `system` label, so the instance and the registry both carry a plain
     `system` key beside the prefixed spine. The registry keeps the plain
     `team`, `tier` and `security-tier` keys it has today and the instance
     keeps `database`; `managed-by: crossplane` is the one label that goes,
     and it is named in the adoption diff of decision 9. The adoption check's
     own guard against a broken label contract counts `System` and `Database`
     objects, kinds the new engine does not have, so before the cutover it is
     extended — additively, per decision 1 — to count tenant Applications as
     well. Otherwise dropped labels would read as a clean "nothing to adopt".

9. **Durable resources stay durable by different spellings (ADR-0015), and
   the third lock becomes Google's.**
   `cnrm.cloud.google.com/deletion-policy: abandon` replaces
   `managementPolicies` without Delete; `resourceID` replaces the external-name
   annotation; Cloud SQL's own `deletionProtectionEnabled` stays. That is two
   locks where ADR-0015 §1 had three, and Config Connector has no
   controller-side equivalent of the third: its deletion-defender finalizer
   guards only an uninstall, and pausing actuation is cluster-wide [I — none
   found in v1.156.0 `pkg/k8s/constants.go`, the Terraform, direct and
   IAM-member reconcilers, or the deletion docs; absence of evidence, not a
   documented statement]. The third lock is supplied from outside the engine
   instead: from ADR-0018's narrowing on, the engine's roles contain no delete
   permission for the instance, the database, the database user or the
   registry, so Google refuses the call whatever state the cluster is in. It
   binds only the engine (a human can still delete, which is ADR-0015 §5's
   point), it stops accidents rather than an engine under hostile control, and
   it does not exist until the narrowing — so everything below still has to
   hold by itself. The abandon annotation is also an ordinary, editable
   annotation, and under Argo CD a removed `databases:` key, a renamed
   `claims.yaml`, or a chart refactor that drops the helper each become a
   prune, then a Kubernetes delete, then a cloud delete. So the annotation is
   on the instance, the database, the database user and the registry from
   their first sync. The database user is the fourth durable kind on purpose,
   and that is a change to ADR-0015 §1, which listed users among what deletes
   with its claim: Cloud SQL cannot delete a Postgres role that has been
   granted privileges, so M2's recipe already abandoned the user and asked for
   the difference to be written down. It is written here. (Neither `SQLUser`
   nor `ArtifactRegistryRepository` has a `deletionPolicy` field, so until
   the narrowing the annotation is the only lock on each. `SQLDatabase` has
   one; the chart sets it to `ABANDON`, which is that kind's second lock.) A
   render test asserts the annotation, and one Kyverno rule — for
   every identity, Argo CD included — denies any update that removes or
   changes it on those four kinds, while allowing it to be added where it is
   absent. That rule stays after the narrowing, as the guard of the first
   lock: a delete that Google refuses is expected to leave the object stuck
   on its finalizer, and a tenant's teardown with it [I until C-31], so the
   engine should never reach that wall. Adding the annotation is expected to
   release a stuck object [I until C-31]. In a tenant's namespace that edit
   cannot come from `kubectl` — decision 7's gate refuses it for everyone but
   Argo CD and the engine's two writers — so the release path there, through
   git or a recorded break-glass exclusion in the gate, is [unverified] and
   is settled with C-31. Adoption also has a precondition the old engine did not. Because the
   instance's controller overwrites what the manifest leaves out (Context),
   the live instance is described with `gcloud` and compared field by field
   with the rendered `SQLInstance` before the first build, and every
   difference is written into the chart. The
   `cnrm.cloud.google.com/unmanaged` annotation is a fallback for some fields
   only: it takes a list drawn from a fixed set of paths, and
   `availabilityType`, `tier` and `databaseFlags` are not among them [C,
   v1.156.0 `pkg/controller/direct/sql/sqlinstance_controller.go`,
   `supportedUnmanageableFields`]. Adoption by name has one more edge. A
   System's name is not reused while any durable resource carrying it exists:
   a new System of that name would acquire the old one's instances, registry
   and database user, its re-created service account would log in as that
   user [I], and the old orphans would drop off ADR-0015's runbook list. So
   the pull request that removes a tenant file also adds its name to a
   retired list in `systems`, kept outside `tenants/`; nothing refuses a
   removal that forgets that step. The list reaches `charts/system` as one
   more value file, between the tenant file and the environment file, and a
   System whose name is on it is refused at render. ADR-0015 §5's runbook
   takes the name off after the last durable resource is deleted. Re-adding a
   removed claim inside the same
   System is still adoption, on purpose (C-28 (c)).

10. **ADR-0016's rules carry over; the engine-specific ones retire.** Any
    composed resource whose identity includes a value a developer may edit
    still has that value in its name — an IAM member's fields are immutable
    here too, and an edit is refused by the API server, through the CRD's own
    `self == oldSelf` rules and again by the webhook, where the old provider
    reported Ready and did nothing [C, v1.156.0 `iampolicymembers` CRD;
    `pkg/webhook/immutable_fields_validator.go`] — and on a team move Argo CD
    applies the three new members and prunes the three old. The per-System IAM
    Condition (C-24) is unchanged in intent and is *not* introduced during
    M2b. A binding's identity is its member, role and condition together [C,
    v1.156.0 `pkg/controller/iam/iamclient/tfiamclient.go`], so a conditioned
    member is a new binding: nothing would be adopted, and the unconditioned
    bindings would stay in the project's policy with no owner. M3 ships it
    with a recorded cleanup. ADR-0016 §2's name prefix matches too much — for
    System `orders` it also matches `orders-api-main`, an instance of System
    `orders-api` — so the ADR that ships C-24 replaces it with exact instance
    names. An exact name picks out one System's instance only because
    decision 5 keeps hyphens out of claim names. Config Connector also
    refuses a condition on an
    Artifact Registry member at admission [C, v1.156.0
    `pkg/webhook/iam_validator.go`], so C-24 covers the project-level Cloud
    SQL grants only. "Readiness is not evidence that the cloud agrees" is unchanged
    and the tests keep reading the cloud side. "Emit the Namespace alone until
    it is observed" retires: Argo CD applies Namespaces first.

11. **A recipe exists only where someone fills in a form.** Infrastructure the
    platform team writes and reviews itself has no form to enforce, so it is
    plain Config Connector manifests with a kustomize base and one overlay per
    environment — no chart. Nothing in the reference build needs this yet:
    there is one environment, and its network stays a Terraform crossing,
    because the engine runs on a cluster that sits on that network and is torn
    down between sessions. The rule is written down so the first such resource
    is not wrapped in a chart out of habit. One exception is named now so
    that nobody tidies it away later: the engine's own role definitions and
    grants are never Config Connector manifests (ADR-0018). The engine does
    not exist when they are needed, and an identity that can edit custom
    roles can add any permission to the role it holds.

12. **The cutover is a rebuild, and it happens before M3.** `cycle.sh down`
    never deletes what Crossplane created, so after a `down` every cloud
    resource is simply there. `up` from a `platform-config` branch carrying
    this ADR's engine lets Config Connector acquire each one by name. Until
    the merge, rolling back is a rebuild from `main`; after it, from the
    Crossplane branch described below — and once decision 1's later apply has
    removed the old identity, a Crossplane rebuild needs that identity
    restored first. There is no period in which both engines run. Three things
    make "from a branch" and "rolling back is a rebuild" true rather than
    hoped:
    - *The branch has to reach below the root.* `root_app_target_revision`
      only chooses the revision of `platform-config` the root Application
      reads `apps/` from. Six of the seven child Applications then read
      `platform-config` at a literal `main`, and the ApplicationSet's and the
      claims Application's chart sources would too. On the branch every
      `platform-config` source names the branch — the literals in `apps/`
      directly, chart-emitted ones through one `platformRevision` key in the
      environment file — and both flip back at merge. Sources that read
      another repo keep `main` on both engines: the seventh child reads
      `systems`, and the tenant's Application reads the tenant's own repo,
      which is why the next point exists. A check fails if a `platform-config`
      source on the branch says `main`, and `cycle.sh` records the revision it
      built from, so a plain `up` cannot become a rollback nobody noticed.
    - *Rollback needs more than `platform-config`.* `svc-hello`'s `main` has
      to serve both engines, so for the length of M2b it carries
      `k8s/database.yaml` and `claims.yaml` side by side, and the tenant
      Application in `charts/system` excludes the former. `platform-bootstrap`
      is not selected by branch at all, which is why decision 1 keeps its
      changes additive.
    - *After the merge, `main` is the new engine.* A tag of the old `main`
      would not roll back, because its children say `main`. Before merging,
      the Crossplane engine is kept on a long-lived branch whose child
      Applications name that branch. That branch is what "the documented
      alternative" physically is.

    The first build also meets a stopped database: the last cycle ended in
    `down` and `park`, and under Crossplane it was drift correction that woke
    the instance. The chart states `activationPolicy: ALWAYS`. Config
    Connector in cluster mode has no in-place downgrade — "Fully downgrading
    Config Connector is not supported", and the documented route is uninstall,
    reinstall and re-apply [C, manual-install doc] — which suits a rollback
    that rebuilds. It is its own milestone, **M2b**, ahead of M3,
    because M3 adds a third form (routes) and building that on an engine being
    retired would be doing the work twice.

## Consequences

- **Portability is restated honestly rather than defended.** The reference
  engine is now GCP-only. What is portable is the arrangement — a form in git,
  a recipe only the platform may run, a cloud controller behind it — and the
  forms themselves; the equivalent controllers elsewhere are ACK on AWS and
  ASO on Azure, with charts per cloud. ADR-0005's reasoning is withdrawn, not
  refuted: nobody tested it. Crossplane becomes the documented alternative,
  and it has what Config Connector never had in that role — two milestones of
  graded evidence in this repo.
- **No M2 grade is touched.** C-05, C-06 and C-07 were graded on Crossplane
  and stay as graded. The new engine earns its own grades on new claims,
  C-26..C-30, registered today with M2's numbers as their baselines.
- **C-25 loses its subject.** There will be no provider bump to run. It stays
  UNTESTED with a dated note at M2b's close: the register has no grade for a
  claim whose subject was removed, and inventing one quietly would be a worse
  precedent than an honest UNTESTED. Its test was to run that bump through
  M4's change class; which kind of dependency bump M4 starts on is decided
  at M4's test-readiness walk.
- **C-03 stays HELD for what it tested**; its carried-forward hands-on checks
  (GCS bucket, Cloud DNS records) will not be run on that provider. **C-08
  stays as worded**; C-26's unchanged-file result speaks to its intent, and
  its namespace-to-project variant becomes an alternate chart for the same
  file. **C-24's claim is unchanged**; it runs on the new engine's IAM member
  kind, and decision 10 says why its condition will name exact instances.
- **What M2 learned is kept; what it learned about Crossplane is archived.**
  Kept: the naming rule, the per-System condition, adoption by name, reading
  the cloud side. Archived with the build log: the ordering gate, readiness
  annotations, the two external-name rules, the RBAC manager's aggregation,
  the watch circuit breaker, revision-suffixed provider service accounts.
- **Costs accepted, stated plainly.** The API server no longer rejects a bad
  form, so a developer who skips CI meets the error on a platform-owned
  Application they may not be able to see until Argo CD has SSO. JSON Schema's
  messages are worse than a CEL rule's hand-written one. And
  `charts/claims/values.schema.json` plus the three chart rules are now
  security-critical:
  a loose schema or an unquoted value is a hole in the tenant boundary, so
  those files are reviewed as policy, not as templating. And Kyverno is now
  load-bearing on the `kubectl` path for edit and delete as well as create,
  because the engine hands every team those verbs (decision 7). That also
  widens what a Kyverno outage stops. ADR-0014 §3 could say that Kyverno being
  down "blocks managed-resource creation and nothing else"; the exclusions are
  evaluated inside Kyverno, so with `failurePolicy: Fail` and three verbs an
  outage now blocks every write to the gated kinds — Argo CD's syncs, the
  engine's own finalizer updates, a namespace's teardown — until it returns
  [I].
- **"Simpler" is a claim, so it gets a test.** Config Connector installs 265
  resource CRDs and eight of its own at 1.156.0, and runs six pods that
  request about 2.3 GiB of memory however few Google services are used [C,
  manifests inside the 1.156.0 operator image] — which may well cost more
  than it saves on a small cluster. C-30 measures pods, CRDs, recipe lines and
  concepts on both engines, and its prediction is written down mixed.
- **ADR-0010's one exception widens; its intent does not change.** Config
  Connector's six images come from `gcr.io/gke-release` with no pull secret;
  five set `imagePullPolicy: Always` and the `prometheus-to-sd` sidecar sets
  none [C, 1.156.0 operator bundle and the manifests inside the operator
  image]. That host is on the Google-API path [C, Private Google Access
  docs]. "All cluster image pulls go through Artifact Registry remote
  repositories" has not been literally true since ADR-0013 §4 let the Cloud
  SQL proxy pull from `gcr.io`, and these six images widen that same
  exception, while "image pulls ride the Google-API path" (C-23) should still
  hold. No new remote is planned; the first cluster session reads the NAT
  logs as C-23's re-check.
- **The ApplicationSet changes what "healthy" and "deleted" mean.** Its
  health reads its own conditions, not its Applications' [C, argo-cd v3.4.6
  `resource_customizations/argoproj.io/ApplicationSet/health.lua`]. A Healthy
  `systems` no longer means tenants reconciled, and one that generated nothing
  would pass `cycle.sh`'s gate, so the script fails a build that has fewer
  tenant Applications than tenant files. Generated Applications carry the
  resources finalizer, so deleting a tenant file deletes its namespace and
  the non-durable cloud objects in it. That is the same outcome as pruning a
  `System` today and is kept on purpose, so that the re-test compares like
  with like. The ApplicationSet itself is never renamed or deleted: it owns
  the Applications it generates, so either would delete every tenant's
  Application and, through the finalizer, its namespace. The root Application
  still carries no finalizer (ADR-0015, Context), so `cycle.sh down` reaches
  none of this.
- **A merge to `platform-config`'s `main` re-renders every tenant at once.**
  Both charts are read at one revision for the whole environment, so a chart
  change reaches every System and every database in one sync; approving a
  change does not make applying it everywhere at once safe. The levers exist
  and are unused: pinned to a tag rather than `main`, the literal revisions in
  `apps/` (the System chart) and decision 12's `platformRevision` (the claims
  chart) would let one environment run ahead of another and give the service
  repo's check a fixed revision to render against. Neither stages a change
  between tenants in one environment; a per-tenant canary is not designed
  here.
- **The human database path may have a gap.** ADR-0013 §5 wants a
  `CLOUD_IAM_GROUP` user and the `SQLUser` CRD does not list it. That path was
  not built in M2 either; it is now a named risk instead of an unknown.
- **Docs that describe Crossplane change at the cutover, not today.** The
  README topology, `platform-pattern.md`, the cloud-target line in
  `CLAUDE.md` and the diagrams stay accurate for as long as Crossplane is what
  runs. They are updated in the session that closes M2b.
- **Checked on 2026-09-19, after drafting and before the first commit.**
  Every item this ADR first listed as unverified was taken to a primary
  source, and the answers are written into decisions 1, 3, 4, 6 to 10 and 12
  above (only 2, 5 and 11 stood as first drafted then). Decisions 1, 4, 7
  and 9 were edited again on 2026-09-21 to meet ADR-0018, and decision 11
  for the first time, also before the first commit. *Confirmed:* the project annotation is
  honoured on a Namespace; the controller's service account is
  `cnrm-system/cnrm-controller-manager`; the ApplicationSet controller is on
  in layer 3's install (chart 10.2.2 has no switch for it and its `replicas`
  is 1); a `ref` source with no `path` renders nothing, and must still be
  permitted by the project's `sourceRepos`; Config Connector's documentation
  and controller code say it acquires a service account, a database, a
  database user and a registry by name — none is on its cannot-acquire list,
  though no acquisition has been run yet, and the only database user here is
  the IAM-type one listed as unverified below; re-applying an IAM member over
  an existing binding changes nothing; `IAMPolicyMember`'s fields are
  immutable and an edit is refused; `SQLInstance` takes its root password
  from a Secret reference. *Refuted:* that the built-in `admin` role carries
  no verbs on these kinds; that a field left out of an acquired `SQLInstance`
  is left alone. *Looked for and not found:* a controller-side deletion lock.
  *Confirmed absent:* `CLOUD_IAM_GROUP` on `SQLUser`, in the 1.156.0 release
  as well as on master.
- **Edited again on 2026-09-22, before the first commit, for a second outside
  review.** Decisions 4, 5, 6, 8, 9 and 10 and these consequences changed;
  decision 2 alone still stands as first drafted. The edits: the rule that
  `valuesObject` writes every key, and names built from `.Release.Namespace`
  (4); the claim-name rule and the backups wording (5); the claim limit's
  reach and the race between two merges, a closed schema for `charts/system`
  with its reserved names, the System's tier in the size-L rule, and the CI,
  `helm lint` and `values.yaml` rules (6); required environment keys, and
  the `rootPassword` [I]'s pointer to C-29 (8);
  the retired-name rule (9); the reason C-24's condition will name exact
  instances (10); and, among these consequences, which kind of dependency
  bump M4 starts on, the C-24 bullet's pointer to decision 10, the claims
  chart's schema named in the security-critical bullet, the re-render of
  every tenant by one merge, the 2026-09-19 bullet's record of decision 11,
  and the unverified `global`-key item widened to both charts' closed
  schemas. The live `svc-hello-main` was
  re-read with `gcloud` on 2026-09-22 for decision 5.
- **Still unverified, each with the step that settles it.** In a local
  rehearsal, before any cluster: that the git-files generator hands a tenant
  file to a Helm source as a `$ref` value file (each part is confirmed, the
  combination has not been run); that an Application rendering zero resources
  reports Synced and Healthy, since `cycle.sh` gates on every Application;
  which identities the namespace controller and garbage collector present to
  admission (upstream here; re-read on GKE in the first cluster session);
  that Helm adds no `global` key that trips either chart's closed root
  schema; that
  `POSTGRES_16` is accepted (the field is a plain string); that the tenant
  boundary holds as *inferred* — taking tenant repos as value sources in the
  platform's project widens nothing only because no tenant can author an
  Application there. The test is a tenant `k8s/` holding a raw `SQLInstance`
  and an Application aimed at `default`, with and without
  `metadata.namespace: argocd`. All three are refused by the blacklist — the
  `SQLInstance` by the Config Connector entry and both Applications by
  `argoproj.io` — because Argo CD checks the kind before the destination [C,
  argo-cd v3.4.6 `controller/sync.go`, `validateSyncPermissions`]. For the
  Application naming `argocd` the destination rule is a second wall behind
  the first, and this test no longer reaches it. At the desk: the seed Job's image and owner, and that
  `wget` in the `GRANT` Job's image can make the HTTPS call (decision 8). In
  the first cluster session: that the `GRANT` Job's address lookup succeeds
  with the token and the `roles/cloudsql.client` it already has (decision 8's
  [I]); that the images pull on the Google-API path here (C-23); that Config Connector
  adopts a *stopped* instance and wakes it; that an IAM-type `SQLUser` is
  acquired and stays up to date, a path Config Connector ships no sample or
  test for. Afterwards, as a measured change of its own and not in C-31's
  return build: the include-list.
