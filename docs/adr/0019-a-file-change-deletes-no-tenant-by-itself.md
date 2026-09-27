# ADR-0019: A removed, renamed or broken file deletes no tenant by itself, and a rule that relies on memory gets a check

Status: Accepted · September 2026 · decides three things ADR-0017 left to convention or left open — the ApplicationSet its Consequences say is "never renamed or deleted", what a claims Application does when its render is empty (§4), and the retired-name step that "nothing refuses" (§9) — and supersedes, in ADR-0017's ApplicationSet consequence, only that removing a tenant file deletes its namespace by itself: offboarding becomes a merge plus one person's step; narrows ADR-0017 §8's open choice of which chart owns the seed Job's service account; widens ADR-0018 §6's reality gate from three reads to four, and reads ADR-0018 §1's "everything it may do" as the intended set; fixes the order in which C-28 (c) is run, without changing its text; pre-registers C-33 · from an outside reassessment of 2026-09-23, decided on 2026-09-24, before any M2b build command ran

## Context

ADR-0017 and ADR-0018 merged on 2026-09-23, and nothing of M2b has been
built. Later that day an outside reviewer re-read the organisation at
`503e895`, this repo's `main` once the tenant-model docs had merged. Its
verdict was to proceed, and most of what it raised is already tracked. Four
points were new, and they share one shape: the design writes a rule down and
then relies on someone remembering it.

**The ApplicationSet owns every tenant.** ADR-0017's Consequences: "The
ApplicationSet itself is never renamed or deleted: it owns the Applications it
generates, so either would delete every tenant's Application and, through the
finalizer, its namespace." That is a convention, and one merge breaks it. The
ApplicationSet is a file in `platform-config`, read by an Application that
prunes automatically — the root Application does [C, `platform-bootstrap`
`layers/3-argocd/charts/root-app/templates/root-application.yaml` and
`variables.tf`], and so does every child in `apps/` today [C, `platform-config`
`apps/*.yaml`]. A PR that removes or renames the file is enough.

There is a second route that ADR-0017 did not name. By default an
ApplicationSet deletes any Application whose file its generator no longer
finds [C, argo-cd v3.4.6
`docs/operator-manual/applicationset/Controlling-Resource-Modification.md`],
and nothing stops it when the generator suddenly finds none [C, argo-cd
v3.4.6 `applicationset/controllers/applicationset_controller.go`,
`deleteInCluster`]. So a PR to `systems` that renames `tenants/` offboards
every tenant at once. Under Crossplane that route was closed by accident. The
`systems` Application read `tenants/` as its path, so a renamed or emptied
directory could not render — "app path does not exist" [C, argo-cd v3.4.6
`util/app/path/path.go`] — and Argo CD does not auto-sync an Application it
cannot render [C, argo-cd v3.4.6 `controller/appcontroller.go`, `autoSync`].
Behind that, automated sync also refuses to prune an Application down to
nothing: "Skipping sync attempt to %s: auto-sync will wipe out all resources"
[C, same function]. An ApplicationSet has neither guard.

Either route leaves the durable cloud resources in place, under ADR-0017 §9's
abandon annotation. It removes everything else: every tenant's workloads, its
grants and its namespace — including each `<claim>-admin` Secret, the only
place the platform keeps that database's superuser password [I, from
ADR-0013 §3 and ADR-0017 §8].

A third route is already on record: a merge to `platform-config`'s `main`
re-renders every tenant at once (ADR-0017, Consequences). A chart change that
stopped rendering the tenant Namespace would prune every tenant's namespace,
and because the render is not empty, no guard would fire.

**An empty claims render is undecided.** ADR-0017 §4 has a service with no
claims render nothing, and C-28 (c) removes a claim and expects its instance
to survive. Neither says what happens when the claim removed is the *last*
one. By the guard above, Argo CD then prunes nothing and reports an error —
unless the Application sets `allowEmpty`, which ADR-0017 does not mention. The
same guard decides what a misnamed `claims.yml` does, because
`ignoreMissingValueFiles` turns a missing file into an empty render too
(ADR-0017 §4). And if the claims chart emitted any object that does not
belong to a claim, no render would ever be empty and the guard would never
fire. A C-28 (c) run on `svc-hello`, which has one claim, could therefore
pass for the wrong reason: the instance survives because nothing was pruned
at all.

**The retired-name step is on the honour system.** ADR-0017 §9: "the pull
request that removes a tenant file also adds its name to a retired list in
`systems` … nothing refuses a removal that forgets that step."

**The permission reality gate reads one level.** ADR-0018 §6's three reads —
the plan, the project's policy and the identity's own policy — see grants made
on the project. Google works out a principal's effective access from the
allow policy on a resource together with the policies it inherits from each
ancestor [C, IAM "Using resource hierarchy for access control" doc]. So a
grant on the organisation, or on one repository or another service account,
is invisible to those reads. The engine can make the second kind itself:
ADR-0018's Consequences record that it could grant itself admin on one
repository and then delete it. The third read, the identity's own policy,
answers who may act *as* the engine, not what the engine may do. ADR-0018 §1
calls `platform-roles` "everything it may do". With only those three reads, it
is everything the engine was *meant* to hold.

Two smaller facts came from the same pass. The old claim check in `svc-hello`
passed 23 times without validating a claim (the M2 build log's fourth
post-close addendum), so a green check is evidence only once it has been seen
to fail. And ADR-0017's Consequences predict that a Kyverno outage now blocks
Argo CD's syncs, the engine's finalizer updates and a namespace's teardown
[I], with no test that would show it.

## Decision

**No removed, renamed or broken file — the ApplicationSet's, the tenant
folder, a tenant file, a `claims.yaml` — deletes a tenant without a person's
step; an empty render deletes nothing; a rule that relied on memory gets a
check; and the engine's reality gate reads every level of the hierarchy. A
change to the charts themselves can still reach every tenant in one sync
(ADR-0017, Consequences); of what that could delete, only the tenant
Namespace is protected here.**

1. **The ApplicationSet never deletes an Application, and neither a merge nor
   a background `kubectl delete` removes its Applications.** Four settings,
   one line each:
   - `spec.syncPolicy.applicationsSync: create-update` on the ApplicationSet.
     Its controller then creates and updates Applications but never deletes
     one [C, the ApplicationSet doc above]. Layer 3's install sets no
     controller-wide policy: its Helm values carry no
     `applicationsetcontroller.policy` [C, `platform-bootstrap`
     `layers/3-argocd/argocd.tf`; argo-helm 10.2.2 `values.yaml`]. With none
     set, the per-ApplicationSet field is honoured [C, argo-cd v3.4.6
     `cmd/argocd-applicationset-controller/commands/applicationset_controller.go`
     and `applicationset/utils/policy.go`; the doc page says
     `--enable-policy-override` defaults to off, but in the code its default
     is on whenever no policy is set, and the code is what runs].
   - `argocd.argoproj.io/sync-options: Prune=false,Delete=false` on the
     ApplicationSet. The Application that reads it then does not prune it
     when its file goes away, and does not delete it if that Application is
     itself deleted; it only reports OutOfSync [C, argo-cd v3.4.6
     `docs/user-guide/sync-options.md`], which `cycle.sh`'s gate reads as a
     failed build.
   - `resources-finalizer.argocd.argoproj.io` on the ApplicationSet itself.
     When its policy forbids deleting Applications, the controller removes
     their owner references before it lets the ApplicationSet go, so a
     `kubectl delete` with the default background cascade leaves every
     Application in place [C, `Controlling-Resource-Modification.md`,
     "How to prevent Application controller from deleting Applications when
     deleting ApplicationSet"; `applicationset_controller.go`]. A foreground
     delete still cascades — "there's no guarantee to preserve applications"
     [C, same doc] — and that takes a cluster admin choosing it.
   - `argocd.argoproj.io/sync-options: Prune=false` on the tenant Namespace
     `charts/system` renders. A chart change that stops rendering it leaves it
     standing and OutOfSync instead of deleting it. Deleting the tenant's
     Application still deletes the Namespace, because a cascade delete reads
     only the `Delete=` option [C, argo-cd v3.4.6
     `controller/appcontroller.go`, `shouldBeDeleted`].

   `platform-config`'s CI fails if any of the four is missing (decision 4).

   Offboarding one tenant therefore takes two steps. The PR removes the tenant
   file and adds its name to the retired list (ADR-0017 §9; `retired.yaml`
   here). Once it merges, the tenant's `<system>-system` Application can no
   longer render, because its value file is gone [C, argo-cd v3.4.6
   `reposerver/repository/repository.go`]; its sync status becomes Unknown and
   Argo CD does not auto-sync it [C, argo-cd v3.4.6 `controller/state.go`;
   `autoSync`], so nothing is pruned. A person then deletes that Application
   with `argocd app delete`, which cascades by default, and its finalizer
   removes the tenant's namespace and non-durable cloud objects as ADR-0017
   described. This supersedes the part of ADR-0017's consequence in which a
   tenant file's removal deleted the namespace by itself. The like-for-like
   comparison that consequence kept "on purpose" is given up; no M2b claim
   grades offboarding by file removal alone (C-26 grades onboarding; C-31
   offboards a scratch tenant and records its steps).

   Two alternatives were set aside. `Prune=confirm` on the ApplicationSet
   would make a removal wait for someone to confirm it [C, `sync-options.md`],
   but while it waits, the Application that reads the ApplicationSet takes no
   automated sync, so every later change to its sources waits too [C,
   `autoSync` skips while an operation is in progress].
   `preserveResourcesOnDeletion` stops the finalizer being added to generated
   Applications [C, argo-cd v3.4.6
   `docs/operator-manual/applicationset/Application-Deletion.md`]. A mass
   deletion would then leave every tenant running, but with its namespace, and
   the two Applications the System chart emits, owned by nothing the
   platform manages [I]. An Application removed with `kubectl` would leave its
   namespace behind [I]; `argocd app delete` adds the finalizer first and
   would not [C, argo-cd v3.4.6 `server/application/application.go`,
   `Delete`]. `create-update` keeps every Application in place and managed,
   which is the simpler state to recover from.

2. **No Application is pruned to empty automatically.** Every Application
   keeps Argo CD's default: `allowEmpty` is not set. The claims chart emits
   nothing that does not belong to a claim. So the seed Job's service
   account, which ADR-0017 §8 leaves to one chart or the other, is either one
   per claim or in `charts/system`. A `claims.yaml` with no `databases:`
   entries, a missing `claims.yaml` and a misnamed `claims.yml` are therefore
   all an empty render, and Argo CD refuses the automated sync that would
   delete everything [C, `autoSync`]. Removing a System's last claim on
   purpose is a person's manual sync, with prune, of `<system>-claims`; the
   guard sits in the automated path only [C, same function]. The durable
   resources survive that prune as they survive any other (ADR-0017 §9). An
   Application that is empty from the start has nothing to prune; that it
   reports Synced and Healthy is already on ADR-0017's unverified list and is
   settled by C-33. C-28 (c) is run on the System to which C-28 (a) adds a
   claim, while both claims exist, so it tests a prune rather than the guard.
   C-28 (a)'s instance exists until M2b's cleanup in any case, so the order
   costs nothing; it is recorded with the data.

3. **The retired-name step gets a check.** The `systems` repo's check
   (ADR-0017 §3) compares a PR's `tenants/` with its base. Every tenant file
   the PR removes or renames must have its name added to `retired.yaml` in the
   same PR, or the check fails. A name *leaving* `retired.yaml` is not
   checked: that is ADR-0015 §5's runbook, after the last durable resource is
   gone, and it stays a person's step. The check is advisory: no repo's rules
   for `main` require a passing check (ADR-0018 §7), and making them required
   has no ADR or claim yet. It makes a forgotten step visible, not
   impossible.

4. **Every PR check is shown to fail.** Each check the platform relies on —
   `platform-config`'s render CI, the `systems` check, the service repo's
   `helm template` check and `platform-roles`' role-file check — runs, in the
   same job, at least one input it must refuse, and the job fails if that
   input passes. `platform-config` and `platform-roles` are already designed
   to carry refusal fixtures (ADR-0017 §6, ADR-0018 §6); the other two gain
   one each. `platform-config`'s CI also asserts decision 1's four settings.

5. **The reality gate gains a fourth read, and it covers the whole
   hierarchy.** Before each session, beside ADR-0018 §6's three reads:
   Policy Analyzer at the organisation's scope, for the engine's identity
   (`gcloud asset analyze-iam-policy --organization=<id>
   --identity=serviceAccount:<engine> --expand-roles
   --analyze-service-account-impersonation`).
   - **Coverage.** It targets every allow policy "at or below" the
     organisation [C, `gcloud asset analyze-iam-policy` reference], and
     Artifact Registry repositories and service accounts are among the
     resources whose policies it analyses [C, Cloud Asset Inventory
     "Supported asset types"]. A query by identity covers roles assigned
     "either directly to them or to the groups they belong to, directly or
     indirectly" [C, Cloud Asset API `IdentitySelector`; I until run].
     `--expand-roles` lists each role's permissions as well as the role, and
     `--analyze-service-account-impersonation` adds what the engine reaches by
     acting as another account, returned in a field of its own [C, `gcloud`
     reference; Cloud Asset API `analyzeIamPolicy` response].
   - **What passes.** The expected answer is exactly the grants
     `platform-roles` holds, all on the project. Anything else stops the
     session, as a stray project grant already does under ADR-0018 §6. So
     does a result that says it is incomplete: `fullyExplored: false`, or any
     analysis state other than OK [C, same response reference].
   - **Scope.** The reference project sits directly under the organisation,
     with no folder [C, layer 0's `terraform.tfvars`, which is not in git: it
     sets `org_id` and no `folder_id`; re-read with `gcloud projects
     get-ancestors` at the desk].
   - **What it needs.** The Cloud Asset API on the project, added to layer 0's
     API list in the layer-0 apply ADR-0017 §1 already declares, so no new
     crossing. And, for the operator, Cloud Asset Viewer at the organisation,
     Role Viewer because the engine's roles are custom, Service Usage Consumer
     where the call is billed, and the Workspace `groups.read` privilege to
     see access that comes through a group [C, Policy Analyzer "Analyze allow
     policies" doc]. These are granted once by hand and recorded in the
     `platform-roles` README.
   - **Limits.** Policy Analyzer's data is best-effort fresh: almost all
     policy changes appear within minutes, and Cloud Asset Inventory allows up
     to about 36 hours for IAM policies [C, both pages]. So a clean read can
     miss a grant made just before it. Without a paid Security Command Center
     tier, the organisation gets 20 analysis queries a day, and the
     impersonation analysis is documented as expensive [C, Policy Analyzer
     doc], so the read runs once per session. And a read is not a test, so
     C-31's must-be-refused calls stay.

   `platform-roles` holds the intended grants; the four reads are how
   "everything it may do" is checked.

6. **A Kyverno outage is measured before anything changes for it.** Two
   options exist: `failurePolicy: Ignore` for some verbs, or exempting the
   namespace controller at the webhook itself (the API server's
   `matchConditions`) rather than inside Kyverno, where ADR-0017 §7 already
   lets it through. Each one weakens the gate or adds machinery, and so far
   the outage's real effect is only a prediction. C-33 measures it first.

## Consequences

- **Offboarding costs one more step, and the step sits where the others
  already are.** Removing a tenant was already a runbook: ADR-0015 §5 for its
  durable resources, ADR-0017 §9 for its retired name. Deleting one
  Application with `argocd app delete` joins it. What that buys: no removed,
  renamed or broken file, in `platform-config` or in `systems`, deletes a
  tenant by itself.
- **Chart changes still reach every tenant.** ADR-0017's Consequences stand: a
  merge to `platform-config`'s `main` re-renders every tenant at once, and a
  chart change that breaks the grants or the claims objects breaks them
  everywhere. Only the Namespace, which takes the workloads and the
  `<claim>-admin` Secret with it, is protected here. Staging a chart change
  between tenants stays undesigned.
- **A merged removal leaves the platform visibly unfinished.** Between the
  merge and the person's step, the offboarded tenant's Application shows a
  render error. After a removed ApplicationSet, the Application that read it
  shows OutOfSync. `cycle.sh` fails a build in either state. That is the
  intent: loud and stopped, rather than quiet and gone.
- **Removing a team's last database is not self-service.** A team that
  empties its `claims.yaml` sees its claims Application refuse to sync, and a
  person completes the removal. With cooperative teams, and a removal that
  keeps the instance anyway, that costs little. The same refusal protects a
  team whose file is misnamed.
- **The local rehearsal carries C-33.** ADR-0017 plans a local rehearsal
  before any cluster, and C-33 runs there. It costs desk time, about half a
  day [I], and no cloud spend. It installs the engine's CRDs and not the
  engine, so the engine's own finalizer updates during an outage stay
  ADR-0017's [I].
- **The fourth read adds one API, one command, and the operator's read access
  at the organisation.** It turns C-31's "grants found on the identity that
  are not in the repo" into a count taken where such a grant would actually
  sit.
- **C-30 counts things, not time.** Its text is fixed. M2b's build log records,
  beside it and ungraded, the minutes from noticing each failure to knowing
  its cause, so "simpler" can also be read as time spent.
- **Still unverified, each with the step that settles it.** In the local
  rehearsal (C-33):
  - what a renamed ApplicationSet does to the Applications the old one owns;
  - that the ApplicationSet's finalizer keeps its Applications through a
    background delete;
  - that the Namespace's `Prune=false` holds through a chart change and does
    not block offboarding;
  - the status of an Application that is empty from the start;
  - what a Kyverno outage blocks, and how it recovers.

  At the desk, before ADR-0018's apply A: that the operator's roles and
  Workspace privilege let Policy Analyzer run and see access that comes
  through a group, and the project's ancestors, read with `gcloud`.
