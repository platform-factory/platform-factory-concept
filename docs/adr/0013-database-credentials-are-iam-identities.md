# ADR-0013: The application's database credential is its own IAM identity, not a password

Status: Accepted · September 2026 · defines "usable" in C-07(a); refines ADR-0003 for GCP

## Context

ADR-0003 says database credentials "flow via managed secret rotation into the
app namespace; no credentials in git." The Crossplane research digest chose,
for AWS, `manageMasterUserPassword` → Secrets Manager → an ESO
`ExternalSecret` in the app namespace, and then recorded that "Cloud SQL has
no direct `manageMasterUserPassword` equivalent; secret flow needs its own
design." The M2 readiness walk listed three candidates and could not define
C-07(a)'s test — *PR → usable database, timed* — until one was chosen,
because "usable" means the credential path works end to end.

The three candidates, with what each costs:

- **(i) A Crossplane-generated password in a namespace Secret.** Simplest.
  A static credential exists in the app namespace; rotation is manual;
  ADR-0003's "managed rotation" is not delivered, so C-07 would grade
  ADJUSTED at best.
- **(ii) External Secrets Operator + Secret Manager.** Matches the stack list
  in `CLAUDE.md`, but ESO is not installed, the API is off, and it does not
  actually solve the problem: Secret Manager *stores* a password, it does not
  generate or rotate a database one. Somebody still has to make and turn
  the credential, so (ii) is (i) with an extra hop.
- **(iii) Cloud SQL IAM database authentication.** No password exists for
  the application at all. It presents a short-lived token minted from its
  own Google identity; Cloud SQL checks IAM. Verified against Google's docs
  on 2026-09-14: principals may be user accounts, service accounts, or
  groups; authentication is by temporary token; SSL is required; login
  needs `roles/cloudsql.instanceUser` and, when connecting through the Auth
  Proxy or a language connector, `roles/cloudsql.client` [C]. The instance
  needs the flag `cloudsql.iam_authentication=on` [C]. The Postgres user
  for a service account is its email with the `.gserviceaccount.com` suffix
  removed, because of the database username length limit [C]. The
  provider's `User` kind supports `type: CLOUD_IAM_SERVICE_ACCOUNT`,
  `CLOUD_IAM_USER` and `CLOUD_IAM_GROUP` at v3.0.0 [C, CRD checked
  2026-09-14].

One mechanic constrains (iii). Cloud SQL's IAM login doc lists no
workload-identity-federation principal form; the GKE connection doc uses a
Google service account bound to the pod's Kubernetes service account with
`roles/iam.workloadIdentityUser` and the `iam.gke.io/gcp-service-account`
annotation [C]. The direct `principal://` form that layer 0 already uses for
Crossplane's package pulls is therefore assumed *not* to work for database
login [I], and each System needs a Google service account.

A second mechanic constrains every option. An IAM database user is created
with no privileges; someone must run `GRANT` as a privileged user, and on a
fresh Cloud SQL Postgres instance the only privileged path is the built-in
`postgres` user with a password [C, Cloud SQL "manage IAM users" doc]. So a
password cannot be avoided entirely — the question is who holds it and what
it is for.

## Decision

**The application authenticates to its database as itself. No password is
ever handed to it.**

1. **Identity is part of the tenant surface (ADR-0012), not the database.**
   The `System` Composition creates, per System: a Google service account
   `<system>@<project>.iam.gserviceaccount.com`; a Kubernetes service
   account `<system>` in the namespace, annotated to it; the
   `roles/iam.workloadIdentityUser` binding between them; and project grants
   of `roles/cloudsql.client` and `roles/cloudsql.instanceUser` to the
   Google service account. A System with no database carries the two grants
   unused — harmless, because without a user row on an instance the
   identity can log in nowhere.

2. **The `Database` Composition creates the instance, the database, and the
   IAM user — and no application password.** The instance is private-IP
   only (`ipv4Enabled: false`, on the Private Services Access range
   1-network provides), with `cloudsql.iam_authentication=on`. The `User`
   is `type: CLOUD_IAM_SERVICE_ACCOUNT`, named
   `<system>@<project>.iam`, derived from the claim's namespace — which by
   ADR-0012 *is* the System name — so the Composition needs no lookup.

3. **The privileged bootstrap credential belongs to the team, is generated
   by the platform, and is never mounted by the application.** The
   Composition generates a root password once, keeps it stable across
   reconciles by re-reading the observed Secret [I — the documented
   go-templating pattern, verified at build], and stores it as
   `<claim>-admin` in the claim's namespace. Its only consumer is the
   Composition's own `GRANT` step: a Kubernetes `Job` composed into the same
   namespace that runs `psql` as `postgres` over the private IP and grants
   the IAM user its privileges on the database. Native Kubernetes objects
   are composable in Crossplane v2 without a wrapper [C, research digest].
   The team can read this Secret — it is *their* database, per ADR-0003 —
   but nothing on the golden path uses it, so it is a break-glass
   credential, not a runtime one.

4. **The application connects through the Cloud SQL Auth Proxy as a sidecar,
   with `--private-ip --auto-iam-authn`.** The proxy handles TLS and token
   minting; the app talks plaintext Postgres to `127.0.0.1`. The image
   (`gcr.io/cloud-sql-connectors/cloud-sql-proxy`) is Google-hosted and on
   the Google-API path, so it pulls without an ADR-0010 remote and without
   registry egress. The sidecar is a snippet the paved road gives the team
   for their `Deployment`; it is *not* injected. A language connector in the
   app is an equally valid substitute and changes nothing here.

5. **Humans use the same door.** The `Database` Composition also creates a
   `type: CLOUD_IAM_GROUP` user for the owning team's group, and the
   `System` Composition grants that group `roles/cloudsql.instanceUser` and
   `roles/cloudsql.client`. A developer runs `psql` from the tailnet (ADR-0011)
   with a login token from their own account; audit logs show the person,
   not a shared account [C, IAM group authentication doc]. There is no
   shared database password for anyone to paste into a chat.

6. **The provider identity gets grant power, bounded by an IAM Condition.**
   Steps 1 and 5 need the Crossplane provider identity to hold
   `roles/iam.serviceAccountAdmin` and `roles/resourcemanager.projectIamAdmin`.
   The second is a strong grant; it is conditioned so it may only add or
   remove the two Cloud SQL roles:

   ```
   api.getAttribute('iam.googleapis.com/modifiedGrantsByRole', [])
     .hasOnly(['roles/cloudsql.client', 'roles/cloudsql.instanceUser'])
   ```

   The attribute lists the roles a request modifies and returns an empty
   list for anything else, so unchanged bindings do not trip it [C, IAM
   conditions attribute reference, 2026-09-14]. Both roles land in layer 0
   with the floor apply the readiness walk already counted.

**"Usable" in C-07(a) means:** `svc-hello`'s pod, running as its Kubernetes
service account with the proxy sidecar and *no Secret mounted*, creates its
table and serves a request from it. Two timestamps are recorded from the
merge: the `Database` XR reporting Ready, and the app's readiness probe —
which checks the database — going green.

## Consequences

- **ADR-0003 is refined for GCP, not superseded.** "Managed rotation" is
  delivered by construction: the runtime credential is a token that expires
  in an hour, minted fresh by the proxy [C]. "No credentials in git" holds.
  The one long-lived secret is the bootstrap password, held by the team,
  used by the platform's `GRANT` step, and mounted by nothing.
- **The rotation story for the bootstrap credential is stated, not built.**
  Deleting the `<claim>-admin` Secret makes the Composition generate a new
  one and update the instance's root password [I — verify that the provider
  re-applies `rootPasswordSecretRef` on change]. That is a manual operator
  action in M2, recorded as such.
- **ESO + Secret Manager is deferred, not dropped.** It remains the path for
  third-party secrets a service genuinely holds — an API key for an outside
  vendor — and nothing in M2 needs one. It is installed when the first claim
  needs it, not before.
- **The `GRANT` step is an extra moving part, and it is the honest one.** A
  Job that runs once per grant-spec change (named by a hash of what it
  grants) is more machinery than a password in a Secret. It is also the
  only way an IAM-authenticated application gets table privileges on a
  fresh instance, so the alternative was pretending the step did not exist.
  The postgres client image pulls through the existing Docker Hub remote.
- **The developer-facing cost is one sidecar block.** The paved road's
  contract with the team is a documented snippet, and a `Deployment` that
  omits it simply cannot reach the database. Kyverno *mutation* to inject
  the sidecar automatically is the obvious refinement and is deliberately
  not in M2: mutation interacts with Argo CD drift detection and needs
  `ServerSideDiff` and `IncludeMutationWebhook` (research digest), which is
  its own change.
- **Provider roles accrete, as surprise 17 predicted.** Cloud SQL admin,
  service-account admin, and conditioned project-IAM admin all join the
  provider identity in one layer-0 apply. The condition is the difference
  between "Crossplane can grant two Cloud SQL roles" and "Crossplane can
  grant Owner," and it is the thing to check first if a grant fails.
- **Two providers' worth of kinds, all installed.** `DatabaseInstance`,
  `Database`, `User` (provider-gcp-sql); `ServiceAccount`,
  `ServiceAccountIAMMember`, `ProjectIAMMember` (provider-gcp-cloudplatform)
  — every one verified at both scopes in M1 or the M2 walk [C]. No new
  provider package is needed for this ADR.
- **Unverified at decision time:** that IAM database login rejects direct
  workload-identity-federation principals (the reason for the per-System
  Google service account — if it accepts them, step 1 shrinks to a single
  IAM binding); that a namespaced `DatabaseInstance` reads
  `rootPasswordSecretRef` from its own namespace only; the observed-Secret
  password-stability pattern in function-go-templating; root-password
  re-application on Secret change.
