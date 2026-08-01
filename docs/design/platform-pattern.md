# The Platform Pattern

Design doc, July 2026. Generalized from patterns designed and run in production at
fintech scale (multi-year GKE operation, PR-gated provisioning, one-repo-per-service
GitOps), rebuilt here from scratch as a generic reference implementation on GCP.
How to *explain* the pattern — the personas it serves, the autonomy analogy, the
confident handover — is its own design doc: `factory-framing.md`.

## Thesis

**Git is the source of truth for everything; Kubernetes is the control plane.**
Terraform (or any imperative IaC) has exactly one job: bootstrap the things the
control plane cannot yet create for itself — the project, the network, the
cluster, workload identity, and the Argo CD install that takes over from there.
After layer 0, every change to the platform is a pull request.

Why this shape: the platform's value is not the infrastructure, it's the
*contract* it gives product teams — a paved road where declaring intent in your
own repo is all it takes to get infrastructure, routes, and guardrails. Contracts
need enforcement points, and PRs + admission control are enforcement points that
scale; tickets and human review are ones that don't.

## Organizing principle: the repo boundary is the approval boundary

Colocate by default. Everything a team ships lives in the team's repo. Something
moves to a central repo **only when its approver is someone other than the team
shipping it.**

This single rule generates the whole topology:

- A Deployment change is approved by the team → lives in the service repo.
- An external route is approved by security → lives in `edge-config`.
- A new tenant/system is approved by the platform team → lives in `systems`.
- Knowledge is approved lightly by many contributors with zero deploy blast
  radius → gets its own repo, `platform-knowledge`.

## The seven repos

```
├─ platform-bootstrap    Terraform layer 0: GCP project(s), VPC, GKE, workload
│                        identity, Argo CD bootstrap — Terraform's last job
├─ platform-config       Argo root (app-of-apps): Crossplane, Kyverno, ESO,
│                        external-dns, gateway + XRDs/Compositions (golden paths)
├─ systems               one YAML per tenant (System XR)     [platform approves]
├─ edge-config           ALL routes/DNS/WAF                  [security approves external/]
├─ template-service      "create repo from template" golden path
├─ svc-hello             canonical service: app code + k8s/ (Deployment + XRs)
└─ platform-knowledge    skills/ adr/ standards/ questions/  [light review]
```

Three tiers of change, three approval speeds:

1. **Service repo `k8s/`** — team approves; schema validation + Kyverno admission
   enforce the guardrails, so review stays inside the team.
2. **`edge-config`** — security approves external exposure via CODEOWNERS.
3. **`systems`** — platform team approves new tenants.

## The System resource

A `systems` entry is itself a Crossplane XR (cluster-scoped): one YAML per tenant
that materializes the namespace, Argo AppProject, RBAC bindings, resource quotas,
and an Artifact Registry prefix. Two properties matter:

- **Ownership moves without re-plumbing.** Identity binds at exactly one point
  (IdP group → cloud IAM → namespace RBAC), so reorganizations are a YAML edit,
  not a migration.
- **The schema survives the org's unresolved questions.** Whether a "system"
  ultimately maps to a namespace or to a whole cloud project is absorbed by the
  Composition, not the developer-facing schema.

## The edge model

All routes — external *and* internal — live in `edge-config`. Internal routes in a
central repo is a deliberate deviation from colocate-by-default, chosen so that
the promotion path is structural:

- **Folders route the review:** CODEOWNERS requires security on `external/**`,
  light gate on `internal/**`. Flipping a route internal→external is a `git mv`
  that *structurally forces* security review.
- **A field drives materialization:** an `exposure:` field on the route claim
  selects WAF policy (Cloud Armor), DNS zone, and gateway class — because
  admission controllers and Compositions can't see file paths.
- **CI enforces folder ↔ field consistency**, so the two views can't drift.

Intent gates (CODEOWNERS) are always paired with reality gates (Kyverno denies
edge objects created from app namespaces). A review rule without an admission
rule is a suggestion.

DNS: one registered domain, env zones NS-delegated into each environment's GCP
project. Hostname schema `<service-class>.<env>.<domain>`, prod drops the env
segment — `api.example.com`, `api.staging.example.com`,
`api.internal.dev.example.com`. Internal zones are Cloud DNS private zones per
env. `auth.` gets its own service-class hostname (IdP cookie isolation, stricter
WAF). See ADR-0004.

## Databases and other team-owned state

Database claims (namespaced XRs) live in the **service repo**, next to the
Deployment. Teams own their databases; guardrails replace review: the Composition
pins engine versions, sizes, backup policy, and deletion protection
(`managementPolicies` omitting Delete on stateful resources), and Kyverno bounds
what the claim can ask for. See ADR-0003.

## The metadata spine

Incident response — human or agent — starts with joining dots: alert →
endpoint → service → team → repo → recent changes. That join must be
structural, never archaeological. The mechanism is declare-once,
propagate-mechanically:

- **Declared once:** the System XR carries the tags — `team`, `service-tier`,
  `security-tier`, `repo` — in the one file the platform team already
  approves. Route claims in `edge-config` link hostname → service.
- **Propagated by Compositions:** everything a Composition materializes
  (namespace, cloud resources, alert policies) inherits those tags as labels.
  Nobody tags resources by hand, so nothing is missing or drifted.
- **Enforced by Kyverno:** untagged resources don't admit — the reality gate
  that keeps the spine complete.
- **Consumed at incident time:** alerts inherit the labels of what they fire
  on, so an incident opens already knowing its service, owning team, repo, and
  the PRs merged since the service was last healthy.

## GCP reference implementation mapping

| Role | GCP | (AWS equivalent, for the portability story) |
|---|---|---|
| Cluster | GKE | EKS |
| Tenant isolation unit | GCP project per env | AWS account per env |
| Pod → cloud identity | Workload Identity Federation | EKS Pod Identity / IRSA |
| Registry | Artifact Registry | ECR |
| Relational DB | Cloud SQL | RDS |
| Secrets backend | Secret Manager (+ ESO) | Secrets Manager (+ ESO) |
| DNS | Cloud DNS (+ Cloudflare public edge) | Route 53 |
| Edge WAF | Cloud Armor | AWS WAF |
| Crossplane providers | provider-upjet-gcp family | provider-upjet-aws family |

The Crossplane research in `research/crossplane-v2-2026.md` was verified against
the AWS provider family; the GCP family follows the same Upjet pattern — kind
coverage gets re-verified at build (tracked there as unverified).

## Open questions (ADRs pending)

- **Secrets posture:** sealed-secrets (encrypted in-repo, auditor-friendly,
  key-custody burden) vs ESO + Secret Manager (managed rotation, external
  dependency). Decide at build.
- **Compute mode:** GKE Standard vs Autopilot for the reference cluster.
- **Mesh posture** for a small-team platform: none / ambient / full — default
  is none until a concrete requirement lands.
