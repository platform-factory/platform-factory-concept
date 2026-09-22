# ADR index: what each decision later became

An ADR records what was decided and why, at the moment it was decided. Once
it is committed to `main` it is never edited afterwards — a change of mind is
a new ADR that supersedes it (`CLAUDE.md`, and ADR-0008: "existing claims are
never reworded or deleted — the same discipline as 'ADRs are superseded,
never edited.'"). Before that first commit an ADR is still correctable, and
the correction is disclosed rather than hidden: ADR-0017's status line is the
worked example, recording the two corrections it took before it was
committed. The rule keeps the record honest and makes it useless as a status
page: nothing in ADR-0012 tells you that ADR-0016 changed its count, because
ADR-0012 was not allowed to learn it.

So the pointer lives outside the decision. This repo has made that call once
already — the first post-close addendum in `../build-log/m2-paved-road.md`
(2026-09-17) says "ADR-0016 is not edited, so this is also where its
'unverified' item gets its answer." The table below generalises it: one row
per relationship that is currently recorded only on the superseding side. A
reader who lands on an early ADR should check here before trusting it.

Two kinds of later change are distinguished, because they mean different
things. **Superseded** means the decision itself was replaced. **Mechanism
replaced** means the decision stands and only the way it is implemented
changed — ADR-0017's own status line draws that line, replacing "the
*mechanism* — not the decision" of the ADRs it names. A third case is marked
where it occurs: a later ADR that contradicts an earlier sentence without
superseding it.

| ADR | Status | What later changed | Where |
|-----|--------|--------------------|-------|
| [ADR-0001](0001-repo-boundary-is-approval-boundary.md) | Accepted · July 2026 | `platform-roles` added to its repo list | [ADR-0018](0018-engine-permissions-are-a-list-in-platform-roles.md) |
| [ADR-0003](0003-database-claims-in-service-repo.md) | Accepted · July 2026 | Refined for GCP — the credential path it left open | [ADR-0013](0013-database-credentials-are-iam-identities.md) |
| [ADR-0003](0003-database-claims-in-service-repo.md) | Accepted · July 2026 | Mechanism replaced; the decision stands | [ADR-0017](0017-config-connector-engine-platform-rendered-forms.md) |
| [ADR-0005](0005-gcp-crossplane-reference-implementation.md) | Accepted · July 2026 | Second decision (Crossplane as the composition layer) and third consequence superseded | [ADR-0017](0017-config-connector-engine-platform-rendered-forms.md) |
| [ADR-0009](0009-corp-real-posture-over-cost.md) | Accepted · August 2026 | Access half superseded (2026-08-22); the site-to-site VPN and the "private endpoint once the VPN is live" were never built | [ADR-0011](0011-identity-gated-access-over-network-perimeter.md) |
| [ADR-0010](0010-image-plane-through-artifact-registry.md) | Accepted · August 2026 | The exception to its first decision that ADR-0013 §4 opened is widened | [ADR-0017](0017-config-connector-engine-platform-rendered-forms.md) |
| [ADR-0012](0012-system-is-the-unit-team-is-a-field.md) §4 | Accepted · September 2026 | Count and mechanism superseded | [ADR-0016 §1](0016-what-the-m2-build-changed.md) |
| [ADR-0012](0012-system-is-the-unit-team-is-a-field.md) §5 | Accepted · September 2026 | Contradicted, not superseded: "A team move therefore touches no cloud IAM" against the six bound things and three re-created IAM members recorded there | [ADR-0016 §1](0016-what-the-m2-build-changed.md) |
| [ADR-0012](0012-system-is-the-unit-team-is-a-field.md) §3, §6 | Accepted · September 2026 | Mechanism replaced | [ADR-0017](0017-config-connector-engine-platform-rendered-forms.md) |
| [ADR-0013](0013-database-credentials-are-iam-identities.md) §1, §5 | Accepted · September 2026 | Unconditioned Cloud SQL grants superseded by a per-System IAM Condition | [ADR-0016 §2](0016-what-the-m2-build-changed.md) |
| [ADR-0013](0013-database-credentials-are-iam-identities.md), the rest | Accepted · September 2026 | Stands, with three caveats the build made concrete: it depends on a provider fix, §5's human path is not built, the break-glass password is stale after a rebuild | [ADR-0016 §4](0016-what-the-m2-build-changed.md) |
| [ADR-0013](0013-database-credentials-are-iam-identities.md) §1–3, §6 | Accepted · September 2026 | Mechanism replaced | [ADR-0017](0017-config-connector-engine-platform-rendered-forms.md) |
| [ADR-0013](0013-database-credentials-are-iam-identities.md) §6 | Accepted · September 2026 | Its two built-in role names superseded for the Config Connector identity, from apply B; the two-role IAM Condition is kept | [ADR-0018](0018-engine-permissions-are-a-list-in-platform-roles.md) |
| [ADR-0014](0014-schema-denies-first-kyverno-covers-the-rest.md) §3 | Accepted · September 2026 | Group wildcards superseded by an enumerated reality gate | [ADR-0016 §3](0016-what-the-m2-build-changed.md) |
| [ADR-0014](0014-schema-denies-first-kyverno-covers-the-rest.md) §1–4 | Accepted · September 2026 | Mechanism replaced | [ADR-0017](0017-config-connector-engine-platform-rendered-forms.md) |
| [ADR-0015](0015-durable-resources-outlive-the-cluster.md) §6 | Accepted · September 2026 | Decisions confirmed, but §6's obligation was not met — only the parked rebuild ran | [ADR-0016 §5](0016-what-the-m2-build-changed.md) |
| [ADR-0015](0015-durable-resources-outlive-the-cluster.md) §1–2, §4 | Accepted · September 2026 | Mechanism replaced; §1's durable set extended to the database user | [ADR-0017](0017-config-connector-engine-platform-rendered-forms.md) |
| [ADR-0016](0016-what-the-m2-build-changed.md) §1, §3, §6 | Accepted · September 2026 | Mechanism replaced | [ADR-0017](0017-config-connector-engine-platform-rendered-forms.md) |
| [ADR-0016](0016-what-the-m2-build-changed.md) §7 | Accepted · September 2026 | Declared crossings extended by a Terraform root outside `platform-bootstrap` | [ADR-0018](0018-engine-permissions-are-a-list-in-platform-roles.md) |
| [ADR-0016](0016-what-the-m2-build-changed.md), "Unverified at decision time" | Accepted · September 2026 | The nested-group item is explained, not settled — Google Workspace filters external nested members out of an internal parent group, which accounts for the one negative result; whether IAM inherits the grant through a nested team group is still not positively tested | [m2-paved-road.md, first post-close addendum (2026-09-17)](../build-log/m2-paved-road.md#post-close-addendum-2026-09-17-the-external-account-refusal-explained) |
| [ADR-0016](0016-what-the-m2-build-changed.md), pre-publication correction | Accepted · September 2026 | The two commits behind it explained; no version on `main` ever carried the withdrawn text | [m2-paved-road.md, second post-close addendum (2026-09-21)](../build-log/m2-paved-road.md#post-close-addendum-2026-09-21-adr-0016s-pre-publication-correction) |
| [ADR-0017](0017-config-connector-engine-platform-rendered-forms.md) §1 | Accepted · September 2026 | Engine identity narrowed to four custom roles | [ADR-0018](0018-engine-permissions-are-a-list-in-platform-roles.md) |
| [ADR-0017](0017-config-connector-engine-platform-rendered-forms.md) §9 | Accepted · September 2026 | The third lock it looked for is supplied | [ADR-0018](0018-engine-permissions-are-a-list-in-platform-roles.md) |
| [ADR-0018](0018-engine-permissions-are-a-list-in-platform-roles.md) | Accepted · September 2026 | Current; nothing later yet | — |

ADR-0002, ADR-0004, ADR-0006, ADR-0007, ADR-0008 and ADR-0011 have no row of
their own, because nothing later touches them yet — ADR-0011 appears above
only as what changed ADR-0009.

Every row above is traced to the superseding ADR's own status line and
section headings — or, where the row adds detail those do not carry, to the
named section of the ADR or build-log entry in the **Where** column. Nothing
here is from memory. Where a row records a contradiction rather than a
supersession, that reading is this index's and is marked as such in the cell.
When a new ADR lands, its status line is the source for the rows it adds
here.
