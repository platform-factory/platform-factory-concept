# ADR-0016: What the M2 build changed in the paved-road decisions

Status: Accepted · September 2026 · supersedes ADR-0012 §4 (count and mechanism), the unconditioned Cloud SQL grants of ADR-0013 §1 and §5, and ADR-0014 §3's group wildcards; restates the Terraform boundary behind C-01; records caveats on the rest of ADR-0013; confirms ADR-0015's decisions with one obligation unmet

## Context

ADR-0012 through ADR-0015 were written on 2026-09-14, before any build
command ran, so that M2 could be held against them. The build ran on
2026-09-16 and 2026-09-17 and the evidence is in
`docs/build-log/m2-paved-road.md`. Three of the four did not survive contact
unchanged — ADR-0015 did — across four findings. ADR-0008 says a claim that works only after a design change
is graded ADJUSTED and needs a superseding ADR, linked. This is that ADR. It
changes only what the build showed to be wrong or incomplete, and says what
evidence forced each change.

The four findings, in one line each:

1. **A team move cannot update a cloud grant in place.** The provider refuses
   to replace an IAM member whose `member` changed, and says nothing at the
   level anyone looks at (build log, C-06 first run; surprise 7).
2. **A team's project-level grant is one cloud object shared by every System
   that team owns.** Removing one System's copy removes the grant for all of
   them, for minutes (surprise 14).
3. **Kyverno cannot match provider groups by wildcard.** The selector
   ADR-0014 §3 described registers a webhook that matches nothing, and the
   reality gate admitted the exact thing it exists to deny (surprise 4).
4. **The credential path of ADR-0013 depends on a provider release that does
   not exist yet.** The pinned provider cannot create a passwordless database
   user at all (surprise 6).

## Decision

### 1. Team-bearing IAM members are named after the team — supersedes ADR-0012 §4's mechanism and count

ADR-0012 §4 said changing `spec.owner.team` changes "exactly four things",
and its Consequences predicted one re-created resource. Both numbers were
wrong. ADR-0013 §5,
written the same day, added two more team grants (the group's two Cloud SQL
roles), so the field binds **six** things, plus a derived copy in the
tenant's `platform-system` ConfigMap:

1. both RoleBinding subjects in the namespace,
2. the AppProject's role groups,
3. the `team` label on everything composed (and its GCP-alphabet twin),
4. the group's writer member on the System's Artifact Registry repository,
5. the group's `roles/cloudsql.client` project member,
6. the group's `roles/cloudsql.instanceUser` project member.

Items 1–3 update in place. Items 4–6 are IAM members, and an IAM member
cannot change its `member`: Terraform would replace it, and the layer that
wraps Terraform into this Crossplane provider (upjet) does not perform
replacements — it refuses the update, permanently, while the old grant stays
live. **So the three IAM members carry the team in their object name and
their composition resource name.** A team move then composes three new
members and Crossplane garbage-collects the three old ones, which removes the
old bindings. Measured on the clean re-run (2026-09-17): one file, three
re-created, 1m55s from merge, no stuck objects.

The general rule this leaves behind: **any composed resource whose identity
includes a value a developer may edit must have that value in its name.**
Otherwise an edit becomes an in-place update the provider will refuse.

### 2. Cloud SQL grants at project scope get a per-System IAM Condition — supersedes the unconditioned grants of ADR-0013 §1 and §5

An unconditioned project-level IAM binding is identified by its role and its
member, and nothing else. Two Systems owned by one team therefore compose two
managed resources for one cloud binding; deleting either removes the grant for both, and the
survivor's object keeps reporting Ready until the provider's next poll
restores it (about five minutes, measured; up to roughly ten).

**Each System's Cloud SQL grants — the team group's and the System's own
service account's — carry an IAM Condition that scopes them to that System's
instances:**

```
resource.service == 'sqladmin.googleapis.com' &&
resource.name.startsWith('projects/<project>/instances/<system>-')
```

In an IAM policy a binding is keyed by its role *and* its condition, so each
System's grant should become its own cloud object that no other System's
lifecycle can touch [I — this is the thing C-24 tests]. It is also
the least-privilege fix ADR-0013 §1 waved at and did not make:
`roles/cloudsql.instanceUser` is otherwise project-wide, held back only by
the absence of a user row. Google documents this exact use — a condition on
`roles/cloudsql.client` "to grant permission to just the named instance"
[C, Cloud SQL IAM Conditions doc, 2026-09-17]. The instance naming rule that
makes the prefix meaningful already exists: ADR-0015 §2 names every instance
`<system>-<claim>`.

**This is decided here and not yet built.** It was found on the last day of
M2 and needs its own verification — that the Auth Proxy's connect and login
calls are evaluated against the instance resource name, and that the
provider's IAM-member kind round-trips a condition without fighting it. It is
the first item of M3's platform work, and the hazard stands, measured and
bounded, until then. The alternative considered was to admit a team *is* a
thing and compose its grants once per team; rejected because it re-opens
ADR-0012's central decision to fix a problem a condition fixes more
narrowly, and leaves the project-wide `instanceUser` grant in place.

The registry writer needs no change: it is a grant on the System's own
repository, which no other System shares.

### 3. The reality gate enumerates provider groups — supersedes ADR-0014 §3's group wildcards

ADR-0014 §3 described the gate as denying raw managed resources in
`*.gcp.m.upbound.io` and `*.gcp.upbound.io`. A Kyverno kind selector with a
wildcard *group* is written verbatim into the webhook's `apiGroups`, and the
Kubernetes API server does not glob that field, so the rule matched nothing:
a raw `DatabaseInstance` applied by hand was admitted and created a real
Cloud SQL instance. A concrete group with wildcard version and kind
(`sql.gcp.m.upbound.io/*/*`) *is* expanded correctly [C, probed live
2026-09-16].

**The policy lists one line per installed provider package, in both
families. Installing a provider means adding its group to the gate in the
same PR.** That coupling is the cost; it is now explicit instead of silently
absent. ADR-0014's other decisions stand: the schema denies first, the
developer-facing message is the same clear one at every surface it was
recorded on (all three for the wrong-region claim; the CLI and the API server
for the rest — §5's full matrix was not run), and Kyverno stays validate-only.

Two smaller corrections from the same run: `validationFailureAction: Enforce`
is stated at the policy's spec level as well as on each rule (Kyverno
defaults the deprecated spec field to Audit and displays it), and the
rule-level fields Kyverno defaults are written out, because Argo CD's diff
cannot see API-server defaults inside a list and reads their absence as
permanent drift — which `cycle.sh` would count as a failed rebuild.

### 4. The rest of ADR-0013 stands, with three caveats the build made concrete

- **It depends on a provider fix.** provider-upjet-gcp v3.0.0 panics creating
  any passwordless `sql.User` (upstream issue #1000, open, root-caused by a
  maintainer). Cloud SQL rejects a password on an IAM user, so the
  Composition cannot work around it. Until a fixed provider ships, **each new
  database costs one manual command** — create the IAM user out of band and
  let the provider adopt it. Re-adding a claim to an existing instance does
  not hit the bug, because the user already exists and is adopted (measured,
  C-07(c)). Bumping the provider is an M4 dependency-bump change; this is the
  first real instance of that class.
- **The human path (§5) is not built.** The team-group database user was
  deferred within M2; until it exists a developer reaches their database with
  the break-glass admin Secret, which is exactly the shared credential §5 set
  out to remove.
- **The break-glass password is stale after a rebuild.** The Secret is
  regenerated with the cluster and Cloud SQL never learns the new value,
  because the provider pushes the root password only at instance create. The
  application path is unaffected (it never uses a password); the break-glass
  path needs one `gcloud sql users set-password`. The real fix — a seed that
  outlives the cluster — is cross-repo and unbuilt.

### 5. ADR-0015's decisions are confirmed; one of its obligations was not met

Deleting the claim left the instance, its database and its IAM user in
place, with the application still serving. Restoring the claim adopted the
instance by external name in 66 seconds, against about fourteen minutes for a
fresh one, with the data intact. `cycle.sh park` found the instance by its
label and stopped it; `down` never touched it; on the rebuild Crossplane
restarted the parked instance by itself and the adoption check recorded
three adopted, none re-created. **What was not done:** §6 asked for both
rebuild modes at M2, and only the parked one ran. The rebuild after a
deliberate delete — fresh creation, the path a real disaster takes — is
carried forward, and `cycle.sh`'s `EXPECT_FRESH` branch stays unexercised
until it runs. One correction to the ADR's tooling, not its decision: the adoption check had to force UTC, because
`gcloud` prints Artifact Registry creation times in local time with no zone.

### 6. Two Composition rules the ADRs never stated

- **Emit the Namespace alone until it is observed.** Crossplane applies
  composed resources in map order and stops at the first error, so a new
  System failed on a different namespaced object every reconcile until the
  Namespace happened to go first. Anything that must exist before its
  siblings is composed in its own pass.
- **Readiness is not evidence that the cloud agrees.** An XR reports Ready
  from its composed resources' Ready conditions, and a refused update or a
  grant deleted by a sibling changes neither. Every C-06 defect was invisible
  at the level a developer or Argo CD looks. Tests for this platform read the
  cloud side, not the object.

### 7. Terraform does not end at layer 0; it recurs at a declared boundary

The design's phrase is "Terraform's last job," and C-01's test counts
`terraform apply` runs after M1 against a target of zero. M2 needed four: one
cleanup (2026-09-02), a bundled layer-0 apply for the provider's cloud
identity, roles and two APIs, a layer-1 apply for Private Services Access, and
an outputs-only re-apply caused by planning a layer before the one beneath it
was applied. Two further Terraform changes — layer 2's Google Groups setting
and layer 3's health checks for the new kinds — rode the ordinary rebuild
and added no apply run, but are crossings all the same. Three APIs were also
enabled by hand. M1's surprise 17 predicted
the shape — one structural crossing per new platform component — and the
readiness walk pre-declared three of these crossings before the build
(layers 0, 1 and 2), which is the only reason they arrived bundled rather
than discovered one at a time.

**The boundary is restated rather than defended:** everything a pull request
can carry arrives by pull request; **identity and reachability for a new
platform capability are a Terraform crossing, declared before the milestone
that needs it and bundled into one apply per layer.** What Terraform must
never own is unchanged — tenants, services, databases, policy. Nothing in M2
required Terraform for any of those, including onboarding a tenant, moving
one between teams, and creating, deleting and re-adopting a database. APIs the
platform depends on belong in layer 0's list; the three enabled by hand are
codified or disabled at the next layer-0 apply.

## Consequences

- **C-01 grades ADJUSTED, with decision 7 as its link.** The zero-applies
  target was wrong in a predictable, bounded way; the boundary that matters
  held.
- **C-06 grades ADJUSTED, and this ADR is its required link.** The one-line
  move is real, but only under a naming rule the original decision did not
  have, and with a shared-grant defect decided here and fixed in M3.
- **C-07 grades ADJUSTED, with decisions 3 and 4 as its link.** The gate that
  failed its first live test was corrected the same hour and has held since;
  the accidental instance it admitted was deleted; the credential path works
  and costs one manual command per new database until the provider is fixed.
- **The other grades need no supersession:** C-05 HELD, C-03 HELD for the
  kinds M2 composes with one recorded gap, C-08 not attempted.
- **The claims register gains claims rather than losing face.** The
  per-System condition (decision 2) and the provider bump (decision 4) are
  pre-registered for the milestones that will test them, so they can be
  graded like everything else.
- **Adding a provider is now a two-file change** (the package and the gate).
  A CI check that the gate lists every installed provider group is cheap and
  belongs with M3's approval-boundary work.
- **The pattern's cost is stated honestly.** "Ownership moves with a YAML
  edit" is true, takes about two minutes, re-creates three cloud grants, and
  — until decision 2 is built — can remove a sibling System's database access
  for a few minutes. That is what the reference implementation measured, and
  it is what the pattern doc should say.
- **Unverified at decision time:** that Cloud SQL evaluates the Auth Proxy's
  connect and IAM-login calls against the instance resource name a condition
  can match; that the provider's `ProjectIAMMember` handles a `condition`
  block without perpetual diff; whether the external consumer account's
  refusal at the cluster endpoint (the IAM analyzer says it has access through
  the nested group, the runtime says 403) is a Google Groups external-member
  setting or something else. The last one blocks a real-login test of C-06
  and is carried forward, not explained away.
