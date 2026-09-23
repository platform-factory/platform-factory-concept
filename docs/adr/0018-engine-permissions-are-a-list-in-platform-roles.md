# ADR-0018: What the engine may do in the cloud is a list in its own repo, and what the list leaves out is the third lock

Status: Accepted · September 2026 · supersedes, for the Config Connector identity and from apply B, the two built-in role names of ADR-0013 §6, and keeps its two-role IAM Condition as decided — the applied text is copied byte for byte from layer 0; narrows the engine identity of ADR-0017 §1 and supplies the third lock ADR-0017 §9 looked for; adds `platform-roles` to the repos of ADR-0001, approved by security; adds a Terraform root outside `platform-bootstrap` to ADR-0016 §7's declared crossings, with two declared applies from it; pre-registers C-31 · checked against primary sources on 2026-09-20 and 2026-09-21, and edited on 2026-09-22 for a second outside review, before its first commit

## Context

ADR-0017 gives the new engine one Google identity holding what Crossplane's
holds today: `roles/cloudsql.admin`, `roles/artifactregistry.admin`,
`roles/iam.serviceAccountAdmin`, `roles/compute.viewer`, and
`roles/resourcemanager.projectIamAdmin` under a two-role IAM Condition [C,
`platform-bootstrap/layers/0-foundation/iam.tf`]. Those are Google's role
names. The four that can write hold about 270 permissions between them (the
read-only `roles/compute.viewer` holds several hundred more), and they
include deleting any database and any registry in the project — the one
thing ADR-0015 says automation must never do. Layer 0's own note says plainly that the condition
"is not a general containment boundary".

On 2026-09-20 the decision was taken to replace the broad roles with narrow
ones the platform writes itself, to keep them in a new repo, and to do it
inside M2b rather than after it. Two facts shape everything that follows.

**A custom role is only a named list of permissions that you own.** What is
on the list the identity may do; what is not, Google refuses.
`cloudsql.instances.delete` and `artifactregistry.repositories.delete` are
permissions of their own [C, IAM custom-roles permission support list], so a
role can simply leave them out. ADR-0017 §9 looked for a third deletion lock
inside Config Connector and found none [I — absence of evidence]; all the
cluster could add was a Kyverno rule guarding the abandon annotation, which
holds only while admission is working. This lock is enforced by Google,
whether or not anything in the cluster is working.

**The engine cannot be the one to apply it.** It does not exist when its
roles are needed, and the cluster it runs on is torn down between sessions.
There is also a harder reason: "If a principal can edit custom roles in a
project or organization, they can add any permission to any custom role in
that project or organization" [C, IAM roles overview, Custom roles], and the
resource-name, -type and -service conditions cannot narrow that to "every
role but my own", because IAM is not among the services that recognise them
[C, IAM conditions attribute reference]. Google does document tags on custom
roles and tag-based conditions; whether one could fence off a single role
was not explored [unverified], and it would not change the decision. So the
engine never holds `iam.roles.*` —
which rules out Config Connector's own `IAMCustomRole` kind for this job,
however natural it looks beside ADR-0017 §11.

Checked against primary sources on 2026-09-20 and 2026-09-21, before
deciding:

- Every permission the engine needs is `SUPPORTED` in a custom role — none is
  `TESTING` or `NOT_SUPPORTED` [C, IAM custom-roles permission support list].
- The "only these two roles" condition uses the attribute
  `iam.googleapis.com/modifiedGrantsByRole`. "If a limited IAM admin tries to
  grant a role on a resource, and the resource's service does not recognize
  the modifiedGrantsByRole attribute, then the request fails"; Resource
  Manager and IAM recognise it, Artifact Registry and Cloud SQL do not [C, IAM
  conditions attribute reference]. For any call that is not a `setIamPolicy`
  the attribute is undefined, the expression's default `[]` is used, and
  `hasOnly` of an empty list is true — so a role bound under this condition
  can still *read* the project's policy [C for the text; that the read
  succeeds here is I until run].
- The engine's work was traced through Config Connector v1.156.0 for the six
  kinds the charts use: 45 steps, 34 of them Google API calls and 11 routing
  steps that make none; 36 confirmed against source and Google's
  method-to-permission tables, 9 inferred. Every call the charts' paths reach
  needs only the 21 permissions listed below. Four calls exist outside those
  paths — a user delete (MySQL only), a user password update, a repository
  delete, a service-account enable or disable — and need permissions the
  roles leave out on purpose [C and I as marked step by step in the table
  the `platform-roles` README carries from its first commit; that repo is
  empty as this ADR is written, so until then the marks rest on this ADR's
  working trace].
- Deleting a custom role is a soft delete: it can be undeleted for 7 days,
  and its ID cannot be reused for up to 44 days [C, IAM creating-custom-roles
  doc; Terraform's provider docs still say 37]. An ID cannot be changed, and
  may hold only letters, digits, underscores and periods — so no hyphen [C,
  IAM roles overview, Custom roles].
- `SQLInstance`'s controller treats *any* error from `instances.get` — a 403
  included — as "not found" and goes on to create [C, v1.156.0
  `pkg/controller/direct/sql/sqlinstance_controller.go`, `Find`]. A missing
  permission at the cutover would therefore look like the very thing C-29
  exists to catch: a durable resource the new engine failed to adopt.

## Decision

**The engine's permissions are four custom roles, written as files in
`platform-roles`, approved by security, and applied by a person with
Terraform. The engine never holds the power to edit a role. What the files
leave out is the third deletion lock.**

1. **The mental model: layer 0 says "this identity exists"; `platform-roles`
   says "this is everything it may do".** The repo holds one YAML file per
   role, in Google's own Role format, and a small Terraform root that turns
   each file into a project-level custom role and binds it, one plain block
   per grant. (File naming, the always-written `stage` and the setting that
   should make a removed file fail the apply are build details kept in the
   `platform-roles` README; the one of them that is unverified is listed at
   the end of this ADR.) It is best thought of as one more
   Terraform layer that lives in its own repo only because a different group
   approves it (ADR-0001): its own state prefix, reading layer 0 through
   `terraform_remote_state` the way layer 1 does, applied by hand after
   `0-foundation`, never touched by `cycle.sh`. Roles and grants belong to the
   project, so a rebuild applies nothing. Layer 0 keeps only the new service
   account, its one Workload Identity binding and an output. *Every* grant the
   engine holds, custom or built-in, is in this repo, so one place shows
   security all of it.

2. **Four roles, 21 permissions, each traced to the engine call that needs
   it.**

   | Role | Replaces | Holds | Leaves out, on purpose |
   |---|---|---|---|
   | `platformEngineSql` | `cloudsql.admin`, 173 | instances create, get, update; databases create, get, update; users create, list | delete of an instance, a database or a user; `users.update` (a password reset: the one user the charts manage has no password, and `postgres`'s is set only at create — ADR-0017 §8's [I], settled by C-29; resetting it is a person's `gcloud` command); export, import, clone, restore |
   | `platformEngineRegistry` | `artifactregistry.admin`, about 70 | repositories create, get, update, getIamPolicy, setIamPolicy | repository delete; everything about the images inside |
   | `platformEngineServiceAccounts` | `iam.serviceAccountAdmin`, 19 | serviceAccounts create, get, update, delete, getIamPolicy, setIamPolicy | disable, enable, undelete, list, API-key bindings, tags, project get and list. Keys and the act-as-another-account verbs were never in the built-in role; the check keeps them from being added. Delete stays: a tenant's service account is meant to go when the tenant goes |
   | `platformEngineProjectIam` | `projectIamAdmin`, 9 | projects getIamPolicy, setIamPolicy | everything else, including the policy-binding permissions the built-in role carries |

   `roles/compute.viewer` stays as it is for M2b. It is read-only, and which
   compute permission Cloud SQL checks on the caller for a private-network
   instance, if any, is the one thing Google's pages leave unstated. The honest
   headline is *four of five*; narrowing the fifth is the first measured
   change after M2b closes.

3. **The project-IAM role stands alone in the one conditioned grant.** The
   condition's title, description and expression are copied byte for byte
   from layer 0, so ADR-0013 §6's decision stands and only the role carrying
   it changes. It is never merged with another role: a repository grant made
   under that condition would simply fail, and a service-account grant would
   drag `roles/iam.workloadIdentityUser` into the allow-list. The allow-list
   stays the two built-in roles granted *to tenants* and never gains a custom
   one, which Google warns against for any role the limited admin could
   modify. The identity must hold no unconditioned grant of any role that
   contains `resourcemanager.projects.setIamPolicy`; one stray grant silently
   voids the condition.

4. **Broad first, narrow after, and never both changes in one build.**
   *Apply A*, at the desk before any cluster exists, creates the four roles
   and grants the new identity both them and the built-in roles Crossplane
   holds. Before it, a read of the four built-in roles has to show each file
   is a subset of the role it replaces; if one is not, the session does not
   start. With that shown, the first cluster session — C-29's forward leg,
   then C-26, C-27 and C-28 on the new engine — and the reverse leg after it
   run on exactly Crossplane's permissions, and every failure in them is
   about the engine. With the cluster down after the
   reverse leg, session 1's audit log is read for the write permissions the
   engine really used, and *apply B* removes the four broad grants: one PR,
   one apply, revertible in one apply. A short desk rehearsal, `gcloud` acting
   as the engine, tries a must-succeed list and a must-be-refused list for
   cents. Acting as the engine needs a temporary
   `roles/iam.serviceAccountTokenCreator` grant for the operator on the
   engine's service account; the script adds it, removes it and asserts it is
   gone, and decision 6's third read is what catches one left behind. The
   return build onto Config Connector, needed anyway, is C-31's test: the
   same transition as C-29's forward leg with one thing changed on purpose —
   the grants.
   The reason is the `Find` behaviour above. It is the same choice ADR-0017 §7
   makes for the include-list.

5. **The third lock, and its limits written beside it.** From apply B the
   instance has three locks (the abandon annotation, Cloud SQL's deletion
   protection, no delete permission); so does the database (the annotation,
   its own `spec.deletionPolicy: ABANDON`, no delete permission); the database
   user and the registry have two where ADR-0017 §9 left them one. It binds only the engine
   identity: a human with Owner can still delete, which is ADR-0015 §5's
   point. It stops accidents — a mistake in git that cascades into a prune, a
   Kubernetes delete, a cloud delete — and not an engine under hostile
   control (Consequences). It does not exist between apply A and apply B. And
   when it fires it is expected to fail *stuck*, not clean: the object hangs
   on its finalizer with a permission error, and a tenant's namespace with it
   [I until C-31]. That is why ADR-0017 §9's Kyverno rule stays, as the guard
   of the first lock.

6. **Two gates on the repo, neither needing a cloud credential in CI.** A
   plain check, run at the desk and on every PR, refuses a role file that
   adds a durable delete, anything under `iam.roles.`, a service-account key
   or an act-as verb, or that puts `projects.setIamPolicy` anywhere but the
   project-IAM file — with refusal fixtures, the same idea as ADR-0017 §6.
   That is the intent gate. The reality gate is three read-only commands run
   before a session: the plan says "no changes"; the project's policy shows
   exactly the grants in the repo for this identity; and the identity's own
   policy shows only its Workload Identity binding. Terraform's additive
   grants cannot see one made by hand, so the second read is the one that
   matters.

7. **Security is the declared approver, and that is declared, not enforced.**
   One CODEOWNERS line covers the whole repo, the grants as well as the
   definitions. As read from GitHub on 2026-09-22, one person is the only
   member of both teams and the organisation's only owner; no repo's rules
   for `main` require code-owner review, an approval or a passing check; and
   this repo, still empty, has no ruleset yet, so unlike the others, which
   at least require a pull request, its `main` would take a direct
   push. Nobody is blocked by the line yet [C as of that date; re-read at
   M2b's close] — the same state as the
   `/external/` line in `edge-config`'s CODEOWNERS, whose folder is not built
   until M3. Security approves the merge; the platform operator runs the
   apply, from a checkout nothing forces to equal `main`. C-09 (M3, on
   `edge-config`) and C-15 (M4, on one change class's paths) are where the
   mechanism — CODEOWNERS plus branch protection — gets tested. Neither test
   touches this repo, so turning enforcement on here is a follow-up with no
   claim of its own yet.

8. **Left out on purpose.** Config Connector's `IAMCustomRole` for these roles
   (Context). A Terraform module pulled into layer 0 by git reference — the
   codebase's first module, and it would leave the grants under the platform
   team's approval. A GitHub Action that applies — the organisation's first
   CI identity with cloud credentials, holding role-admin power. An IAM deny
   policy, which would make the lock hold against a hostile engine but needs
   an organisation-level role and about four more ideas; revisit in M3, when
   more privileged service accounts arrive and it protects more than four
   permissions. Custom roles for what *tenants* are granted: a binding's role
   is part of its identity, so swapping them would orphan bindings exactly as
   ADR-0017 §10 describes.

## Consequences

- **The upkeep is ours now, and nothing raises an alarm.** Google adding
  permissions to its built-in roles does not matter to a custom role. What
  matters is a Config Connector upgrade that calls a new API method. The
  first person to notice is the operator, in a paid session, as a failed
  resource — or, for a missing `cloudsql.instances.get`, as a misleading
  "already exists". The defences are habits: the README's table of
  permission → engine call is the contract, re-tracing it is a checklist line
  on every Config Connector version bump, and the rehearsal is re-run after
  one. C-31 records how many fixes M2b needed, so the next reader knows the
  going rate.
- **What IAM still does not stop, kept beside the grants.**
  `iam.serviceAccounts.setIamPolicy` is project-wide, and nothing used here
  narrows it. A condition could limit which roles it grants — IAM recognises
  `modifiedGrantsByRole` — but never *which* service account, and the one
  role it has to grant, `roles/iam.workloadIdentityUser`, is itself the way
  one identity acts as another [C, IAM conditions attribute reference; IAM
  roles page]. So the engine could let itself act as any service account in
  the project — including, until the apply that removes it, the old
  Crossplane identity with all its delete permissions.
  `artifactregistry.repositories.setIamPolicy` is as wide: Artifact Registry
  does not recognise that attribute, so the roles it grants cannot be
  limited, and it could grant itself admin on a repository and then delete
  it, or make a registry public. Tag-based conditions exist for both services
  and were not explored [unverified]. `cloudsql.instances.update` can switch
  on a public IP; the fix is an organisation-policy constraint,
  `constraints/sql.restrictPublicIp` [C, Cloud SQL organization-policy doc],
  a named follow-up. `cloudsql.users.create` can make a built-in (password)
  database user, and Cloud SQL by default puts such a user in
  `cloudsqlsuperuser` with the attributes `postgres` has [C, Cloud SQL for
  PostgreSQL users doc]. Leaving out `users.update` stops the engine
  resetting `postgres`'s password through the users API (whether an instance
  update carrying `rootPassword` can reset it is ADR-0017 §8's [I]); it does
  not stop the engine making a second login in `cloudsqlsuperuser`. No
  constraint for that was
  found, so it stays on this list with no fix named yet.
- **Terraform gets further from its last job, and says so.** `platform-roles`
  is one more Terraform layer for ADR-0016 §7's purposes: a directory with
  its own state that outlives every cluster. That section asks for one
  bundled apply per layer. This one declares two, apply A and apply B,
  because the point is to change one thing at a time, and each permission fix
  is one more. Predicted fixes: one or two, against a target of zero (C-31).
- **What folding this into M2b cost, for anyone copying the order.** Done
  after M2b, the engine swap would have closed without a return build to
  grade, and C-30 would have measured the swap alone. Done inside it, the
  return build carries a scratch tenant, a scratch claim and the lock test —
  an estimated twenty to thirty extra paid minutes [I] — leaves scratch
  durable resources for a person to delete with `gcloud`, puts two decisions
  behind one milestone's grades, and weakens C-30's concept count. What it
  bought: M2b never closes on roles that can delete every database in the
  project, and the third lock is graded in the milestone that lost it.
- **C-30's "fewer concepts" gets weaker**, and is reworded before it is
  registered: an eighth repo, the custom role, and a second home for Terraform
  are new to a reader.
- **The coupling with Kyverno's gate becomes an order across two repos**
  (ADR-0017 §7): the gate merges in `platform-config` first, then the
  permission is applied from here, so a new kind is refused before the engine
  is able to create it.
- **"Seven repos" becomes eight at M2b's close, not before.** The design
  README, `platform-pattern.md`, `CLAUDE.md`, the diagram and the repo READMEs
  change in the session that closes M2b, as ADR-0017 already says for the
  Crossplane text. `platform-roles`' own README says "one of eight" from its
  first commit, and the repo joins the close procedure with its own tag.
- **Role lifecycle traps worth a sentence each.** A deleted role turns every
  engine call into a 403 while its grants still show in the policy. A rename
  is a delete plus a create and costs the old ID for up to 44 days. A role's
  title and description are limited in *bytes* and fail only at apply, so the
  check counts bytes. The condition's text is frozen once applied.
- **Still unverified, each with the step that settles it.** At the desk,
  read-only: that every role file is a subset of the built-in role it
  replaces, and whether `roles/cloudsql.admin` carries a Cloud SQL
  `setIamPolicy` at all — two Google pages disagree, which decides whether
  layer 0's "Three of the four unconditioned roles carry their own
  setIamPolicy power" was ever right (`gcloud iam roles describe`); that all
  21 are grantable in a *project-level* role here (`gcloud iam
  list-testable-permissions`); that Terraform's `deletion_policy = "PREVENT"`
  makes the apply fail when a role file is removed, instead of soft-deleting
  the role (the argument is in provider 7.42.0; the behaviour was read, not
  run); that the API treats a role file with no `stage` as `ALPHA`, as its
  enum implies (Terraform's default of `GA` is confirmed from source). In the
  local rehearsal ADR-0017 already plans: that its §7 gate lets the operator
  create and delete a raw Config Connector object in a namespace that is not
  a tenant's, and that its §9 rule admits adding the abandon annotation where
  it is absent — C-31's lock test and its release step need both. In the
  desk rehearsal: that the
  condition behaves the same on a custom role — `roles/cloudsql.client`
  granted, `roles/viewer` refused; that Artifact Registry needs no permission
  to poll a create. In the return build: what Config Connector does with a
  delete Google refuses, and what releases it — in a tenant's namespace the
  release cannot be a `kubectl` edit, because ADR-0017 §7's gate refuses one,
  so the path there (through git, or a recorded break-glass exclusion) is
  open; that the project `setIamPolicy`
  Config Connector sends — with an update mask `gcloud` does not use — passes
  the condition, which the rehearsal cannot show. After M2b: whether any
  compute permission is checked on the caller.
