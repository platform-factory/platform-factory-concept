# ADR-0012: A System is the unit of ownership; its team is a field on it

Status: Accepted · September 2026 · gates the C-05 XRD and defines what C-06 measures

## Context

The pattern doc (`platform-pattern.md`, "The System resource") says a `systems`
entry is one YAML per tenant that materializes a namespace, an Argo
`AppProject`, RBAC bindings, quotas, and an Artifact Registry prefix, and that
"identity binds at exactly one point (IdP group → cloud IAM → namespace RBAC),
so reorganizations are a YAML edit, not a migration." The claims register
turned that into C-06: *move `svc-hello` between teams; count files touched
(target: one) and anything that had to be re-created.*

The M2 readiness walk (`m2-paved-road.md`, surprise 2) found that the design
never says what a *team* is in this model, and the test means something
different under each reading:

- **If a team is a System** — one namespace per team, services inside it —
  then moving `svc-hello` to another team is moving it to another namespace:
  a redeploy, a new Artifact Registry path, a new `AppProject`. C-06 would
  measure deployment mechanics and say nothing about identity binding.
- **If a team is a field on a System** — one namespace per service, owned
  by a team — then the move is a one-line edit and the Composition rebinds
  the group everywhere it appears. C-06 measures what the claim says.

The same decision shapes the C-05 schema, so it has to land before the XRD is
written, not before C-06 is run. Two mechanics found in the walk also bear
on it: GKE requires `container.clusters.get` (in
`roles/container.clusterViewer`) just to *authenticate* to a cluster, before
any RBAC applies [C, GKE RBAC doc, 2026-09-02]; and GKE resolves Google Group
membership only for groups nested under a Workspace group named exactly
`gke-security-groups` that the cluster was created with [C, Google's setup
doc, 2026-09-02].

## Decision

**A System is one deployable unit with its own namespace. A team is a field
on it, never a System itself.**

1. **The System's name is its identity.** It is the namespace name, and every
   composed resource derives its name from it: the `AppProject` is
   `<system>`, the Artifact Registry repository is `<system>`, the labels
   say `system=<system>`. A System corresponds to a service repo (the `repo`
   tag the metadata spine already requires); `svc-hello` is a System named
   `svc-hello`. The schema does not forbid one repo declaring several
   Systems, but the reference build has one each.

2. **`spec.owner.team` names the owning team.** Exactly one team per System;
   a team may own any number of Systems. A team *is* an IdP group — on this
   org, a Google Group — and nothing else: no `Team` kind, no team
   namespace, no team-level cloud resources.

3. **The team → group binding is a naming convention, resolved at
   composition time.** Team `payments` is group `payments@<org-domain>`; the
   domain comes from the per-environment `EnvironmentConfig` the research
   recipe already calls for. There is no second registry file mapping teams
   to groups, because a second file would be a second binding point. The
   group must exist in the Workspace and be nested under
   `gke-security-groups`; creating groups is a Workspace-admin task outside
   the paved road for M2 and is recorded as a manual prerequisite, not
   hidden.

4. **What the team field binds — and nothing else does.** Changing
   `spec.owner.team` changes exactly four things, all in the Composition:
   the `RoleBinding` subjects in the namespace (the team's group gets the
   namespace-admin role), the `AppProject`'s roles (the group may sync into
   the project), the `team` label on everything the System materializes, and
   the group's IAM member on the System's Artifact Registry repository. No
   other resource references the team.

5. **The GKE authentication hop is granted once, in layer 0, to the umbrella
   group.** `roles/container.clusterViewer` goes to
   `gke-security-groups@<org-domain>`, so every member of every team group
   can authenticate and — until a `RoleBinding` says otherwise — see nothing.
   A team move therefore touches no cloud IAM, and the Composition does not
   need project-IAM-admin power for tenancy. IAM counts members of nested
   groups [I — verify on the first login; if it does not, the grant moves
   to a per-team `ProjectIAMMember` in the Composition and C-06's
   "re-created" column gets an entry].

6. **The schema speaks intent, not mechanism.** It carries what the metadata
   spine needs and a size class, and nothing that names a Kubernetes or GCP
   resource:

   ```yaml
   apiVersion: platform.example.com/v1alpha1   # group chosen at build
   kind: System
   metadata:
     name: svc-hello                            # = namespace, AR repo, AppProject
   spec:
     owner:
       team: payments                           # = group payments@<org-domain>
       repo: platform-factory/svc-hello
     tier: standard                             # service-tier
     securityTier: internal
     size: S                                    # quota class, not CPU numbers
   ```

   `size` is a class the Composition maps to a `ResourceQuota`; the schema
   never says `cpu:`. A Composition that maps a System to a whole GCP
   project instead of a namespace (C-08) reads the same file — a project has
   no `ResourceQuota`, but it has a size class. This is the entire content
   of C-08's evidence, decided here so the C-05 build cannot leak
   implementation detail by accident.

## Consequences

- **C-06 tests what the register says it tests.** Predicted data, written
  before the run: files touched, one (`systems/svc-hello.yaml`); re-created,
  one — the Artifact Registry IAM member, because changing `member` on an
  IAM-member resource forces replacement under Terraform's IAM semantics
  [C], while `RoleBinding.subjects` and `AppProject.spec.roles` update in
  place [C, Kubernetes; `roleRef` is the only immutable field]. Anything
  else in the "re-created" column is a finding.
- **The real C-06 check needs a human, not a flag.** `kubectl auth can-i
  --as-group` skips the IdP. The honest test is one person whose group
  membership changes between the two checks, or two people. Recorded as a
  manual step; the paved road does not manage group membership.
- **C-05's "usable" is defined:** the `System` reports Ready, *and* a
  `Deployment` applied to the namespace is admitted under the System's
  `AppProject`. Merge time comes from the GitHub API, Ready time from the
  XR's condition timestamp. The second tenant is simply a second System —
  same team or a different one; the test does not care.
- **Teams have no isolation boundary of their own.** A team with five
  services has five namespaces, and cross-service traffic within a team is
  not privileged by this decision. That is deliberate: isolation follows the
  deployable unit, and anything team-wide (network policy, shared quotas)
  is a later, explicit addition rather than an accident of namespace naming.
- **Renaming a System is a migration.** The name is the namespace, and
  namespaces are immutable identity in Kubernetes. That is exactly why the
  team must not be part of the name — the field that changes must not be
  the field that identifies.
- **Team names carry constraints.** A team name must be a valid group
  local-part *and* a valid Kubernetes label value; the XRD enforces the
  intersection with a pattern so a bad name fails at admission, not at
  group resolution.
- **Group lifecycle is outside the paved road for now.** The provider family
  has a `cloudidentity.Group` kind, but that provider is not installed and
  the Cloud Identity API is off; bringing group creation inside the platform
  is possible later and would make the manual prerequisite in (3) go away.
- **Unverified at decision time:** whether any team groups exist in the
  Workspace (the check needs the Cloud Identity API on a usable quota
  project); nested-group resolution for the umbrella IAM grant (5).
