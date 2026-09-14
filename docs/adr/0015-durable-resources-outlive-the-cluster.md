# ADR-0015: What the paved road creates outlives the cluster, and the rebuild adopts it

Status: Accepted · September 2026 · resolves M2 surprise 1; changes what C-02 measures from M2 on

## Context

M1's `cycle.sh` enforces the four-layer boundary in code: it can only ever
destroy and rebuild `2-cluster` and `3-argocd`, because identity and
reachability persist and compute is disposable. Its finish line is every
Argo CD Application reporting Synced and Healthy, and its data — minutes
down, minutes up, manual interventions — is C-02's evidence. "Rebuild from
empty" was literally true: nothing in the cluster had created anything
outside it.

M2 ends that. The paved road's whole purpose is that a merged claim creates
something durable in the cloud — a Cloud SQL instance, an Artifact Registry
repository — and C-07(c) *requires* that deleting the claim does not delete
the database. The readiness walk recorded the collision as surprise 1: a
resource whose claim deletion cannot delete it also survives `cycle.sh
down`, so on `up` the platform meets its own leftovers. Three questions the
M1 script has no notion of:

- **What does `down` do to them?** Today, nothing — and for two reasons
  that are not the same. The root Application carries no cascade-delete
  finalizer, so removing it orphans its children rather than deleting their
  resources [C, `3-argocd/charts/root-app`], and a cluster destroy is a
  cluster destroy, not a Kubernetes delete. Neither reason is a policy; a
  finalizer added later would turn `down` into a delete storm.
- **What does `up` expect to find?** Argo recreates the claim, Crossplane
  recreates the managed resource, and the provider must *adopt* the instance
  that already exists rather than fail on it or create a second one.
  Crossplane's import mechanism is the `crossplane.io/external-name`
  annotation: the provider observes the resource with that identifier, and
  `Create` applies only "if the external resource doesn't exist" [C,
  Crossplane managed-resource docs and import guide, 2026-09-14].
- **What does it cost to leave them?** An idle Cloud SQL instance bills
  between sessions. Google's own answer is the activation policy: `NEVER`
  stops the instance and "suspends instance charges"; storage and IP
  charges continue [C, Cloud SQL start/stop doc, 2026-09-14]. Deleting is
  the other option, and instance names are reusable immediately [C,
  verified in the walk].

## Decision

**Durable resources are never deleted by automation. The rebuild adopts
them by name, and idle cost is managed by stopping them, not removing
them.**

1. **The durable set is anything that holds data or images.** In M2 that is
   `DatabaseInstance`, `Database`, and `RegistryRepository`. Each is composed
   with `managementPolicies: ["Observe", "Create", "Update", "LateInitialize"]`
   — everything but `Delete` — *and* `deletionProtection: true`, the
   provider-side guard, *and* `settings.deletionProtectionEnabled: true` on
   the instance, the Cloud SQL-side guard [C, CRD fields checked
   2026-09-14]. Three independent locks, because C-07(c) is the claim that
   a destructive mistake is bounded by construction. Everything else a
   Composition creates — users, IAM members, service accounts, in-cluster
   objects — is fully managed and deletes with its claim.

2. **External names are deterministic and derived from the claim.** The
   Composition sets `crossplane.io/external-name` on every durable resource
   from the XR's identity — `<system>` for a registry repository,
   `<system>-<claim>` for an instance — never Crossplane's generated suffix.
   This is the entire mechanism by which `up` adopts instead of duplicates,
   and it is also why a System's name is immutable (ADR-0012).

3. **`cycle.sh down` and `up` do not change.** `down` cannot address durable
   resources and must never learn to. `up` gains one verification: after
   every Application is Healthy, it records each durable resource's
   cloud-side creation timestamp, so the results file can distinguish
   *adopted* (timestamp older than this cycle's `up`) from *re-created* —
   because a rebuild that silently made a fresh empty database would pass
   the Healthy check and be a false C-07(c) result.

4. **Idle cost is an explicit operator command, separate from the cycle.**
   `cycle.sh park` sets the activation policy to `NEVER` on every instance
   the platform created — found by the metadata-spine labels the
   Composition puts on every cloud resource, not by a hand-kept list — and
   records the action. It is not part of `down`, on purpose: C-02 measures
   rebuild, and folding cost hygiene into it would change the number and
   hide the choice. On `up`, no `unpark` is needed: the Composition's spec
   says `activationPolicy: ALWAYS`, `Update` is in the policy set, and
   Crossplane reconciles the instance back on [I — expected upjet
   drift-correction behaviour; verified by the first parked rebuild].
   The paved road heals its own instance.

5. **Deleting durable resources is a human act with a runbook.** Removing a
   tenant or a database for real is `gcloud`, performed deliberately, after
   the claim is gone, and recorded. The platform offers no automation for
   it in M2. That is ADR-0003's "bounded by deletion protection and orphan
   policies" made literal.

6. **Both rebuild modes are exercised at least once at M2.** A parked
   rebuild (adoption, the default rhythm) and a rebuild after a deliberate
   delete (fresh creation) — because the second is the path a real
   disaster takes, and the first is the path every evening takes.

## Consequences

- **C-02 changes meaning, and the log must say so.** "Rebuild from empty"
  becomes "rebuild from persisted identity, reachability, and data." Cycle
  timings after the first `Database` claim include instance restart and
  adoption on the critical path and are not comparable to M1's numbers —
  the same caveat ADR-0011 recorded when NAT moved layers. The results file
  gains a column for it rather than the number quietly growing.
- **C-07(c) and C-02 are now one test seen twice.** Every parked rebuild
  is evidence that the database survived claim deletion *and* cluster
  deletion. The adoption check in (3) is what makes it evidence rather than
  an assumption.
- **Forgetting to park costs money, not correctness.** The failure mode of
  an explicit command is that nobody runs it. That is accepted: a wrong
  number on a bill is recoverable and a wrong delete is not, and the
  results file shows whether it ran.
- **Adoption trusts the name.** The provider will adopt *any* resource with
  the external name, including one the platform did not create. The
  mitigation is that names are derived from claims in a project the
  platform owns, and the labels from (4) make "did we create this?" an
  auditable question after the fact rather than a guess.
- **Removing a System does not remove its registry or its databases.** An
  off-boarded tenant leaves durable resources behind by design, and the
  runbook in (5) is how they go. Orphans are visible: every durable resource
  carries a `system` label, and one whose System no longer exists is
  exactly the list the runbook works from.
- **`cycle.sh` grows, but not past its boundary.** `park` and the adoption
  check touch Crossplane-created cloud resources through `gcloud`, never a
  Terraform layer, so the script's "only 2-cluster and 3-argocd" assertion
  stands unchanged and stays enforceable in code.
- **Stopping is not free.** Storage and the private IP bill while parked
  [C]. The smallest instance class the Composition offers is what the
  reference build parks, and the cost is recorded per session when the
  deferred cost analysis is picked up.
- **Unverified at decision time:** that an upjet `DatabaseInstance` adopts by
  instance name alone (the Terraform identifier includes the project, and
  the provider's external-name configuration is expected to handle it);
  drift correction from `NEVER` back to `ALWAYS` on `up`.
