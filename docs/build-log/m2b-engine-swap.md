# M2b: Engine swap — closed 2026-10-31 (opened 2026-09-19; built 2026-09-27 to 2026-10-29; tested 2026-10-19, 2026-10-27, 2026-10-29 and 2026-10-31)

> **Status: closed 2026-10-31.** Every session has run. Every claim is graded
> except C-32, which is UNTESTED because its test did not run. The grades are
> the decisions of the maintainer's delegate on the data, under the
> maintainer's instruction to close M2b. They use ADR-0008's words (HELD,
> ADJUSTED, WRONG, UNTESTED for a claim not run), with scope words where a cell
> carries a scope, not ADR-0020's six, which start at M3.
>
> **How to read the markers.** `[C, source]` read or observed; `[I]` inferred;
> `[unverified]` not checked. All times are UTC unless a git timestamp is
> quoted with its offset. A path under `evidence/` names one of the build's
> raw files (logs, test output, audit and policy reads), kept with the
> operator outside the repo. They hold IAM policies, audit entries and
> personal addresses, so they are not published. Every fact cited from one
> is quoted in this entry.
>
> **Grades in one table** (reasoning under "Claims graded").
>
> | Claim | Grade, 2026-10-31 | What it rests on, and what is open |
> |---|---|---|
> | C-26 | HELD | Session A reconstructed from audit logs, live notes lost; the object-by-object diff is done; S3's updates closed the two live observations |
> | C-27 | HELD | data reconstructed; "healthy while the cloud disagreed" not determinable |
> | C-28 | ADJUSTED, superseding ADR = [ADR-0022](../adr/0022-what-the-m2b-build-changed.md) §1–2 | two gate defects fixed in the build; S3's team-member cases ran; the `/status` write is admitted and accepted (ADR-0022 §5) |
> | C-29 | HELD | forward leg reconstructed, S1, S3; `postgres` refused the rebuilt Secret as ADR-0017 §8 predicted, which is not a design change; the draft's 2026-10-27 rule is withdrawn in the entry |
> | C-30 | HELD, scoped | falsifier not met; "fewer recipe lines" missed; CPU 73 to 217 m unregistered and ungraded |
> | C-31 | HELD | the five grants, the return build, the scratch tenant and Google's refusals in S3; the desk test on 2026-10-31 refused an edit to its own role and a `roles/viewer` grant |
> | C-32 | UNTESTED | the test did not run; the only billing export holds no rows for the project |
> | C-33 | HELD, scoped | (a)–(d) held; (e) held only under a deviation; (x) a counter-example outside the registered cases; ADR-0022 §3–4 |

## Readiness and provenance

**What was registered, and when.** C-26..C-32 were registered on 2026-09-19,
2026-09-21 and 2026-09-22 and C-33 on 2026-09-24, all before any M2b build
command (register, "M2b — Engine swap"). The register records that C-28 (b),
(b2), C-29, C-30 and C-31's Data were reworded on 2026-09-21 and 2026-09-22
before its first commit. No claim text has changed since. The first M2b build
commit is `platform-bootstrap` e7e180c, 2026-09-27T06:58:59-07:00.

**What was run, and how good the instrument was.** Seven sources feed this
entry. They are not equal, and the grades below weigh them differently.

| Source | When | What it is | Standing |
|---|---|---|---|
| Session A | 2026-10-19 22:01Z to 2026-10-20 04:35Z | The cutover session: C-26, C-27, C-28 (a) (b) (b2) (c), C-29's forward leg, C-31's control, three late cycles | **Reconstructed on 2026-10-27 from Cloud Audit Logs, because the session's live notes were lost.** Not observed live. |
| C-33 rehearsal | 2026-10-27, cases 20:04Z to 23:20Z (setup earlier) | Local k3d cluster, Argo CD v3.4.6, Kyverno 1.19.1, six Config Connector CRDs, no engine | Observed, with a log per case. Run by an agent under the operator's direction. |
| S1 | 2026-10-27 19:49:15Z to 20:36:53Z | The reverse leg of C-29: `up` onto the `crossplane` branch (dd6730c), then `down` | Observed; `cycle.sh` output and `cycle-results.tsv` rows |
| Desk work | 2026-10-27 | C-31's subset check, audit-versus-role-files check, Apply B init, plan and reads | Observed; read-only |
| IAM poller | from 2026-10-27 19:57:06Z, last sample read 2026-10-29 06:55Z, one gap (C-31 Data) | Project and registry policies every ten seconds | Observed; covers S1 and S3, not Session A |
| S3 | 2026-10-29, start 05:34:08Z, after Apply B (05:29Z) | The return build on `main` (`e8eedeb`) onto the narrowed roles: `up` 32m40s = 1960 s, 13/13 Healthy; the `psql` login, the team-member cases and C-30's count; C-31 (a) and (b); `down` as cycle 8 with one re-run, which `cycle.sh` labels cycle 9 (Surprise 18); `park` of three instances, 167 s, at 06:37Z to 06:39Z, before the cluster was gone | Observed; `s3-*.log`, `cycle-results.tsv`, `s3-engine-audit.tsv`. The cluster was destroyed by 06:50:11Z, and all three instances read STOPPED/NEVER after it (`evidence/s3-after-down.txt`) |
| Desk test | 2026-10-31, 14:50:06Z to 14:50:22Z | C-31's must-succeed list and refusals, impersonating the engine, run by the operator | Observed; two overlapping copies, identical verdicts (C-31 Data) |

**What the audit log cannot show.** This is the limit on every Session A
number. Cloud API reads are not logged (data-access logging is on only for IAM
Credentials and OS Login), so "the engine's first read" does not exist as a
datum. Artifact Registry `SetIamPolicy` entries carry no policy body, so a
repository's policy at a past moment is not determinable, only the times of
writes. Kubernetes audit is at Metadata level, so Application status and
conditions are mostly absent. Kubelet transitions are not audited, so a Pod's
Running time is bracketed, not read. Commands typed inside the jump box are
not Cloud Audit events. The ten-second cloud-side poll the register asks for
left no record from Session A; the project-level substitute is exact (full
policy bodies in every project `SetIamPolicy`), the repository-level one does
not exist. Cloud Logging also redacts the engine's email in messages (the
denial text reads "co...r@pl...m"); the unredacted text comes from the
Kubernetes `events` log. [C, `evidence/session-a/reconstruction.md` §0 and §9]

**Where a test ran differently from its registered form.** Collected here so
no grade has to hunt for them.

1. Session A's data are reconstructed, not observed (all of C-26, C-27, C-28
   (a) (b2) (c), C-29 forward, C-31 control).
2. C-33 (e): the registered step, scaling Kyverno's admission controller to
   zero, did not produce the failure the prediction describes. A second
   attempt used a deviation (below).
3. C-33 (d): the register does not name the field; two forms ran.
4. C-33 (c): the claim was removed as `databases: {}`, not by deleting the
   entry's file.
5. C-33: Argo CD came from upstream manifests at v3.4.6, not the platform's
   Helm chart 10.2.2; Kyverno ran chart 3.9.1 (app v1.19.1), not 3.9.2, which
   `platform-config` `main` carries from 2026-10-27T19:56Z; neither `systems`
   nor `platform-config` CI ran.
6. C-29's subset desk check was due before the first leg. The only record in
   the evidence is dated 2026-10-27, after both legs.
7. C-30's Crossplane side was counted on 2026-10-27 from the `crossplane`
   branch, not "from the last M2 cycle" as registered; and its line reading was
   fixed on 2026-10-28, after the 2026-10-23 desk count had been seen.
8. Session A's `up` wall-clock has no single logged event and no
   `cycle-results.tsv` row; the file holds no row between 2026-09-17 and
   2026-10-27. [C, `cycle-results.tsv`]
9. C-29's `psql` login was made after S3's return build, not after the forward
   leg, as registered.
10. The Policy Analyzer read (ADR-0019 §5, which puts it before the session)
    was first made at 06:00:40Z, 26 minutes into S3.
11. The scratch tenant `svc-scratch` reused `svc-onboard`'s service repo, which
    caused two manual interventions (C-31 (a)).

## Built

Times are PR merge times in UTC unless stated.

**`platform-config`.** Config Connector 1.156.0 in place of Crossplane
(`config-connector/`), the `system` and `claims` charts, the `systems`
ApplicationSet carrying ADR-0019 §1's four settings, two Kyverno ClusterPolicies
(`deny-raw-cloud-resources`, `keep-abandon-annotation`), and
`scripts/check_charts.py` with refusal fixtures in `tests/charts/cases.yaml`.
PRs #26 (the M2b Applications, 2026-10-06), #27 (`62b9451`, the gate refuses a
`kubectl` edit, 2026-10-07), #28 and #30 (C-27's naming-rule break and
restore, 2026-10-19 22:41:01 and 23:27:03), #29 (`5da7d4a`, the gate lets the
garbage collector finish a delete, 2026-10-19 23:09:58), #31 (the `crossplane`
branch reads itself, 2026-10-20 04:45:07), #32 (M2b becomes `main`, 2026-10-20
04:45:12), #33 (`a9e2bdd`, a team cannot write Kyverno policy, and Kyverno
1.19.2, 2026-10-27 19:56:22). The Crossplane engine stays rebuildable from the
long-lived `crossplane` branch (dd6730c).

**`platform-bootstrap`.** PR #26, `e7e180c`: the Config Connector identity, one
Workload Identity binding, the Cloud Asset API, and `cycle.sh` on two engines.
Additive, merged 2026-09-30 15:05Z. `cycle.sh` records the revision it built
from; the S1 rows carry "root at crossplane (dd6730c)" (ADR-0017 §12).

**`platform-roles`.** New repo. Four custom roles, 21 permissions: Sql 8,
Registry 5, ServiceAccounts 6, ProjectIam 2 (each file read on 2026-10-27).
A role-file check with refusal fixtures (21 of 21 pass, per the 2026-10-23
review [C, verdict.md]). PR #1 merged 2026-10-06 16:36Z. Apply A (the roles,
plus the four broad built-in grants) ran before Session A: the project policy
in Session A's 22:32:49Z `SetIamPolicy` body already lists all four custom
roles and the four broad roles for `config-connector@`. Apply A ran on
2026-10-16: the four `CreateRole` calls at 03:04:44.435Z–03:04:44.703Z and the
project `SetIamPolicy` granting the engine its roles at 03:04:46.439Z, all by
the operator [C, Admin Activity audit log, read 2026-10-28].
Apply B is a commit, `b766d81` on branch `apply-b` (2026-10-27T13:21:48-07:00),
deleting `terraform/grants-broad.tf`. It merged as `platform-roles#4`
(`08178fb`) at 2026-10-27 20:37:53Z. **It was not applied as of 23:28Z
2026-10-27**: the poller still shows the four broad roles on the engine
identity at 23:34:56Z, and at 00:03Z on 2026-10-28 the engine still held its
four broad roles plus `roles/compute.viewer` [C, poller; desk read]. It was
applied on 2026-10-29 at 05:29:10Z to 05:29:25Z (C-31 Data).

**`systems`.** The retired-name list and the check (#14, 2026-09-27); the
third tenant `svc-onboard` from one file (#15, 22:32:06 on 2026-10-19); C-27's
two moves (#16 22:36:01, #17 23:22:37). In S3 on 2026-10-29: #19 (offboards
`svc-onboard`, 06:14:30Z), #20 (the scratch tenant `svc-scratch`, one file,
06:19:04Z) and #21 (removes it and adds the name to the retired list,
06:33:14Z) (C-31 Data). The branch `c31-onboard-svc-scratch` (`e9d224c`) held
the scratch file earlier [I: that #20 is its content].

**`svc-hello`, `svc-onboard`.** `svc-hello` gains `claims.yaml` with `main` and
a temporary `second` (#18, 22:32:28), the C-28 (b) boundary objects (#20,
#21), C-28 (c) (#22 02:53:57 and #23 02:56:21 on 2026-10-20), and a claims
check that reads `platform-config` `main` (#24). `svc-onboard` is a stock
nginx placeholder (3415d1c), with a temporary refused claim (#1, #2). In S3
`svc-scratch` reused its repo: `svc-onboard#3` added the claim (06:17:44Z),
`#4` repointed the manifests (06:25:34Z) and `#5` restored the repo (C-31
(a)). `svc-hello#25` removed the `second` claim (06:21:16Z).

**What the build did not add that the design lists.** The fourth read of the
reality gate (Policy Analyzer at the organisation, ADR-0019 §5) has no record
in the evidence from before S3; Apply B's "reads before" file holds two of the reads and the
plan (see C-31). The Policy Analyzer read was first made on 2026-10-29 at
06:00:40Z, 26 minutes into S3 and after Apply B: `gcloud asset analyze-iam-policy
--organization=545291611456 --identity=serviceAccount:config-connector@…`,
**with neither `--expand-roles` nor `--analyze-service-account-impersonation`**.
The operator's organisation-level access was enough, and the result says
"fully explored". It returns **five bindings, all attached to
`//cloudresourcemanager.googleapis.com/projects/platform-factory-ref`**:
- `platformEngineProjectIam`, under the condition
  "only-cloudsql-connect-roles", expression
  `api.getAttribute('iam.googleapis.com/modifiedGrantsByRole',
  []).hasOnly(['roles/cloudsql.client','roles/cloudsql.instanceUser'])`;
- `platformEngineRegistry`;
- `platformEngineServiceAccounts`;
- `platformEngineSql`;
- `roles/compute.viewer`.

There are none on the organisation, a folder, or any other resource [C,
`evidence/c31/policy-analyzer-1.txt` and `.json`]. The read at S3's end,
06:35:34Z, shows the same five bindings, all on the project
(`evidence/c31/final-reads.txt`). A third read, at 06:51:55Z, used
`--expand-roles --analyze-service-account-impersonation`
(`evidence/c31/policy-analyzer-2.txt`):
- fully explored, five bindings, all on the project, and no identity reachable
  by impersonation (`identities []` on every binding);
- the expanded permissions of each custom role equal that role's definition
  (analyzer-only and role-only sets empty for all four);
- `platformEngineServiceAccounts` carries `iam.serviceAccounts.setIamPolicy`,
  the route ADR-0018's Consequences names.

## Data

### C-26 — The tenant files survive the engine

**Bytes changed in existing tenant files.** `systems/tenants/svc-hello.yaml` is
byte-identical between the M2 close (`1df1030`, 2026-09-17) and the end of
Session A (`a79a910`); C-27 changed it and put it back. `svc-ledger.yaml` was
rewritten on 2026-09-21 (`8c7b0cb`, "Tenant docs catch up": 18 lines added, 40
removed), **a comment-only change: the non-comment lines are identical** to the
file M2 ran. So the swap changed zero bytes, and a literal count of the file
between M2 and the cutover is not zero. [C, `git diff` on `systems`]

**The third tenant.** `systems#15` added `tenants/svc-onboard.yaml` and nothing
else in `systems`, `platform-config` or `platform-bootstrap` (commit body of
`80f1c87`). Its service repo, a stock nginx placeholder, was created earlier,
as `svc-ledger`'s was in M2. Merged 22:32:06. [C, audit log]

| Event | Time | Seconds after merge |
|---|---|---|
| ApplicationSet creates Application `svc-onboard-system` | 22:32:40.572 | 34.6 |
| Namespace `svc-onboard` created | 22:32:41.576 | 35.6 |
| First cloud write (`CreateRepository`) | 22:32:45.136 | 39.1 |
| GSA created | 22:32:45.267 | 39.3 |
| Workload Identity binding | 22:32:46.635 | 40.6 |
| Project grants for the GSA (`cloudsql.client`, `cloudsql.instanceUser`) | 22:32:49.515 | 43.5 |
| Repository writer grant | 22:32:55.965 | 50.0 |
| Pod created | 22:33:50.679 | 104.7 |
| Container started (`events`) | 22:33:55 | 109 +/- 1 |
| Pod Ready (endpoint moves to `addresses`) | 22:34:00 to 22:34:01 | 114 to 115 |

Merge to a running workload: **1m49s to 1m54s**, against M2's 4m08s to Ready.
This is one sample of a stock nginx, and about 35 seconds of it is the time
Argo CD took to notice the merge; M2's number held about three minutes of the
same poll. The gap from namespace (35.6 s) to Pod (104.7 s) is not explained by
the log. The Application was patched at 22:32:43 and 22:33:20 and the first
Deployment write is 22:33:50; what it waited on is not determinable. A
readiness-probe failure at 22:33:55 is normal start-up.

**Objects present in M2's tenant surface and absent now, and the reverse.**
The desk diff was made on 2026-10-27/28: M2's surface is what the System and
Database Compositions at `3bfc72d` compose; the present surface is `helm
template` of `charts/system` and `charts/claims` at `platform-config` `main`
(`e8eedeb`), in Argo CD's value-file order, for `svc-hello` (19 objects from
the system chart, 6 from the claims chart for claim `main`) and `svc-ledger`
(19, and an empty claims render), with `systems` `a79a910` and `svc-hello`
`76a6294`. 17 composed objects at M2 plus the XR equals the M2 inventory in
C-05's data. The cloud side is the same set (below); the differences are all
in the cluster and in labels. Classes, as the diff file gives them: (a) engine
machinery; (b) a change a team can see or do; (c) follows a named decision in
ADR-0017 or ADR-0019 [I: the diff file does not define the letters; this
gloss is read from its use]. [C, [`m2b-c26-surface-diff.md`](m2b-c26-surface-diff.md)]

*Present in M2, absent now (7):*
1. The `System` XR per tenant, the Application `systems` that applied the
   files, and the `Database` XR from `k8s/database.yaml`. (a). The claim is
   replaced by `claims.yaml`, read by `<name>-claims` (ADR-0017 §4, §5, §12).
2. RoleBinding `<name>-crossplane-edit`. (b) and (c). ADR-0017 §7: the team
   holds the built-in `admin` alone; §7 also records that `cnrm-admin`
   aggregates into `admin`, which widens what the team can do on Config
   Connector kinds.
3. Secret `main-admin` as a rendered object. (a). Created at runtime by the
   seed Job (ADR-0017 §5).
4. The `privateIp` key in ConfigMap `main-connection`. (c), **with a visible
   consequence**: a consumer of the key loses it (ADR-0017 §5, the Job looks
   the address up at run time).
5. The GCP label `managed-by: crossplane` (SQL instance, registry). (c);
   ADR-0017 §8, "the one label that goes".
6. AppProject blacklist entries `*.upbound.io` and `*.crossplane.io`. (a);
   replaced by `*.cnrm.cloud.google.com` (ADR-0017 §4).
7. The team name in the Google service account's description. (b),
   cosmetic; no ADR found.

*Present now, absent in M2 (7):*
1. Application `<name>-system` per tenant (from the ApplicationSet) and
   Application `<name>-claims`, even for `svc-ledger`, which has no claim.
   (a) and (c); ADR-0017 §3, §4.
2. ServiceAccount, Role (secrets get/create) and RoleBinding `database-seed`
   in every tenant namespace, claim or not. (b)/(c); ADR-0017 §5. The ADR
   marks which chart owns the account "[unverified]".
3. Job `main-seed-<hash>`. (c); ADR-0017 §5.
4. AppProject blacklist entries `argoproj.io` (ADR-0017 §4) and
   `policies.kyverno.io` (`platform-config#33`; ADR-0021 Context). (b) and
   (c): a tenant can no longer sync those kinds.
5. `directory.exclude: database.yaml` on the tenant's Application. (c);
   ADR-0017 §12.
6. Namespace annotations `cnrm.cloud.google.com/project-id` and `Prune=false`.
   (a)/(c); ADR-0017 §8, ADR-0019 §1.
7. Cloud side only: Config Connector adds `managed-by-cnrm: true` to cloud
   labels. (a).

Everything else in the 17 composed objects is present on both sides with the
same names, roles, members and limits (the diff file's Tables A and B give the
rows). Object names that changed kind only (`ProjectIAMMember` to
`IAMPolicyMember`, `RegistryRepository` to `ArtifactRegistryRepository`, `User`
to `SQLUser`) are counted as present on both. The cross-check with Session A
found no cloud object beyond the render for `svc-onboard`, and the live project
and repository policies match the render. Two live observations the files do
not explain:
- `svc-hello-main` and the three registries still carry the cloud label
  `managed-by: crossplane` (with `managed-by-cnrm: true`), though the chart
  drops it. They are the M2 objects, but the cause is not determinable from the
  files. **After the return build (S3, 2026-10-29 06:07:57Z), `svc-hello-main`'s
  `userLabels` are `database: main`, `managed-by-cnrm: true` and `system:
  svc-hello`. `managed-by: crossplane` is gone.** The engine's update at
  05:52:33Z sent the chart's labels (`evidence/c29-update-audit.json`), and
  that update removed it. The three registries were updated by the engine at
  the same time (three `UpdateRepository`, status 0).
- The three Google service accounts' live descriptions still read "Workload
  identity for System X (team T). Composed by the System XR; do not edit by
  hand.", while the chart renders "Rendered by the platform; do not edit by
  hand." and the chart text has been unchanged since `90a522d`. `svc-onboard`'s
  account was created by Config Connector at 22:32:45 and shows the old text,
  so this is not only a survivor of M2. **After the return build, all three
  descriptions read "Workload identity for System X. Rendered by the platform;
  do not edit by hand."**, the chart's text. The engine's three
  `PatchServiceAccount` calls during S3's `up` (status 0) rewrote them. The
  forward leg's engine had not rewritten the descriptions it adopted. Why it
  did not is not determinable from the record. On S3's build it did, and the
  render and the cloud now agree [C, `evidence/s3-tests.txt`,
  `evidence/s3-engine-audit.tsv`].

### C-27 — Ownership still moves with a YAML edit

**Move 1: `systems#16`, payments to checkout, merged 22:36:01.** One field in
one file (commit body of `2d17c38`). Argo CD deleted three objects at 22:39:53
(`svc-hello-sql-instance-user-team-payments`, `svc-hello-sql-client-team-payments`,
`svc-hello-registry-writer-payments`), 232.5 s after the merge, and created
three at 22:39:55, 234.1 s. **Resources re-created: 3.** The project policy
showed payments removed at 22:39:56.999, **236.0 s after the merge**. Checkout
already held both Cloud SQL roles (the grants of `svc-ledger` and `svc-onboard`),
so adding checkout wrote nothing and the log shows only the REMOVE. M2's
baseline was 1m55s. Of the 236 s, 232.5 s is the time before Argo CD acted;
Config Connector took about 3.5 s from the delete to the cloud write. The
repository policy for `svc-hello` is **not determinable** (no body logged):
two writes bracket the change, at 22:39:53.9 and 22:39:58.0.

**`platform-config#28`, merged 22:41:01: the naming rule broken** so that the
registry writer's name no longer carries the team. Argo CD deleted
`svc-hello-registry-writer-checkout` at 22:43:44.807 and created
`svc-hello-registry-writer` at 22:43:46.172. Changing the object name did not
fail: same member, new name.

**Move 2: `systems#17`, checkout back to payments, rule still broken, merged
23:22:37.** Argo CD deleted the two SQL grants at 23:25:49 (192.5 s) and created
their payments replacements at 23:25:50. The project policy showed checkout out
and payments in at 23:25:53.184, **196.2 s** after the merge. The registry
writer was refused at 23:25:50.579, the Kubernetes API's answer to Argo CD's
patch, code 9:

> IAMPolicyMember.iam.cnrm.cloud.google.com "svc-hello-registry-writer" is invalid: spec.member: Invalid value: "group:payments@thecloudgeek.io": member is immutable

The same patch was refused six times (23:25:50.579, 23:26:30.236, 23:27:32.005,
23:29:34.260, 23:33:36.072, 23:38:38.303), the spacing doubling from 39.7 s to
302.2 s. At each of those times the other five `svc-hello` objects patched
successfully. [C, audit log]

**The restore did not heal it.** `platform-config#30` restored the rule at
23:27:03. The next retry, 29 s later, still failed, and so did the two after
it. The only operator write in the window is a `kubectl` patch of the
Application `svc-hello-system` at 23:40:00.174 by the operator (what it
patched is **not determinable**, the body is not logged). 1.4 s later Argo CD
deleted the old object, 777.2 s after the restore merge, and created
`svc-hello-registry-writer-payments` at 23:40:02.525. The last repository
policy write of the session is 23:40:06.129, **1049.1 s (17m29s)** after the
`systems#17` merge. Resources re-created in move 2: 3, one of them only after
the operator's patch. [I: the retries kept the revision the failed sync
started with, so the restore was not read until the operation was cleared;
this is a reading of the pattern, not an observation.]

**Cloud side.** The repository policy today reads `roles/artifactregistry.writer:
group:payments@thecloudgeek.io` only, but today's read follows later writes
(the Crossplane run of 2026-10-27 20:11Z) and is not a session-end snapshot.
During the 1049 s the registry-writer object kept its old member, because the
patch that would have changed it was refused [I]. So for about 14 minutes
(23:25:50 to 23:40:02) the file said payments and the repository probably said
checkout [I; the repository policy is not readable from the logs].

**Whether anything reported healthy while the cloud disagreed**, M2's
baseline "yes": **not determinable** (Application status is not logged). **Whether
the broken rule fails loudly:** the API server refused it with a message that
names the field. Whether the Application showed a failed sync is not in the
record.

**Absent grants another System still wanted (move 2).** Both of checkout's
Cloud SQL roles left the project policy at 23:25:53.184594, while `svc-ledger`
and `svc-onboard`, both owned by checkout, still declared them (their
IAMPolicyMember objects existed throughout; no delete or create in either
namespace after 22:32).

| Grant | Removed | Re-added | Absent for |
|---|---|---|---|
| `roles/cloudsql.instanceUser` `group:checkout` | 23:25:53.184594 | 23:27:02.260370 | **69.076 s** |
| `roles/cloudsql.client` `group:checkout` | 23:25:53.184594 | 23:30:14.604402 | **261.420 s** |

The re-adds are Config Connector writes with no accompanying Kubernetes write;
they are its periodic re-reconcile, and the object that triggered each is not
attributable. Payments was absent from the project policy by design from
22:39:56.999 to 23:25:53.184 (2756.2 s). The move-1 run found no interval in
which another System lacked a grant it wanted. [C, `iam-policy-diffs.txt`]

### C-28 — Guardrails replace review without a custom API

**(a) PR to usable database: `svc-hello#18`, merged 22:32:28.** Config
Connector applied the claim objects at 22:33:59 (91.2 s), and the SQLInstance
waited nine seconds for the `second-admin` Secret, which the seed Job created at
22:34:07.739. The Cloud SQL create call is at 22:34:08.350 (100.4 s) and its
operation ended at 22:44:20.304 (712.3 s); the instance object was Ready at
22:45:07 (759.9 s). `CREATE_DATABASE` ran at 22:46:09 and `CREATE_USER` at
22:46:10 (both by `config-connector@`, from `gcloud sql operations list`), and
the `second-grant` Job completed at 22:46:27, **839 s = 13m59s** after the
merge, against M2's 17m45s. The Cloud SQL create is 612.0 s of that.
"Usable" here is the Job's completion event; that an application connected to
`second` is **not determinable**. **Manual commands: none found** in the
operator's Cloud SQL or Kubernetes writes between 22:26 and 23:40; the two
jump-box sessions (OS Login at 22:09:23 and 22:27:39) both end before the
merge, and what ran inside them is not a Cloud Audit event. The instance was
created with `deletionProtectionEnabled: true`, labels `database=second,
system=svc-hello`. The Postgres log shows `postgres` getting past
authentication from 22:44:21, which reads as the Secret's password being
accepted by the new instance [I; successful logins are not logged].

**(b) Refusals.** What was recorded, by surface.

*The tenant's AppProject (Argo CD).* `svc-hello#20` (merged 22:54:18) put a raw
SQLInstance and two Argo CD Applications, with and without
`metadata.namespace: argocd`, in `svc-hello/k8s/`. The commit body of the
removal (`1264524`, authored 15:56:45-07:00) records the refusals: "resource
argoproj.io:Application is not permitted in project svc-hello" (twice) and
"resource sql.cnrm.cloud.google.com:SQLInstance is not permitted in project
svc-hello", and that none of the three was created. [C, commit body; the
Application status that carried the text is not in the audit log.]

*The claims Application's status.* `svc-onboard#1` (merged 22:57:26) carried a
size L claim with `tier: critical` and `system.tier: critical` set in
`claims.yaml`, in a System whose tenant file leaves `tier` out. The removal's
commit body (`d4cbb03`) records `svc-onboard-claims` showing ComparisonError,
"at '/databases/main/size': 'not' failed", and rendering nothing.

*Kyverno, as the operator with `kubectl` on 2026-10-19, 22:24:32.110 to
22:24:44.573: 13 denials, code 3*, from `validate.kyverno.svc-fail`, policy
`deny-raw-cloud-resources`: creates of a SQLInstance, an IAMPolicyMember and an
ArtifactRegistryRepository named `by-hand`; patches of `svc-hello-main`,
`svc-hello-sql-client-team-payments` and the `svc-hello` repository (seven
patches); deletes of the same three kinds. Rule `no-cloud-objects-by-hand-in-tenant-namespaces`
for create and patch, `no-cloud-object-deletes-by-hand-in-tenant-namespaces`
for delete. Create message, verbatim:

> admission webhook "validate.kyverno.svc-fail" denied the request: resource SQLInstance/svc-hello/by-hand was blocked due to the following policies deny-raw-cloud-resources: no-cloud-objects-by-hand-in-tenant-namespaces: Cloud resources are not made by hand here. Ask for a database in your service repo's claims.yaml, and the platform's chart creates the Cloud SQL instance for you, with its identity, grants, labels and lifecycle. A resource made directly would be one the platform cannot see, bill or clean up. See charts/claims/README.md in platform-config.

Delete message: "Cloud resources are not deleted by hand here. To remove a
database, take its entry out of your service repo's claims.yaml; the cloud
instance is kept, and deleting it for real is a platform runbook step (ADR-0015
§5)."

*Not recorded from Session A:* a wrong region, an oversized request, the
cross-field rule, an extra key, a string crafted to inject YAML, a claim over
the per-System limit, an unknown key under `environment`, a hyphenated claim
name, and the `tier` variants with a tenant file that states `tier: standard`
or leaves it out and `claims.yaml` setting `system.tier: critical` (one of
these, the leaves-it-out variant, is the `svc-onboard` case above). These are
covered by `platform-config`'s render fixtures (`tests/charts/cases.yaml`, 144
checks at 6949999, 152 at a9e2bdd [C]), which refuse each with a message naming
the field. That is a chart-render refusal at the desk, not the PR check,
the Application status and Kyverno surfaces the register asks for.
**All of them also ran through a real PR.** svc-hello#19 (closed unmerged,
2026-10-19) put one case per commit, so the `claims.yaml against the platform's
charts` check ran once on each. The ten (b) cases each **failed** the check, and
each job log names the field:

| Commit | Case | The check's refusal |
|---|---|---|
| `89ad411` | 01, wrong region | `/databases/main/region`: value must be one of 'us-central1', 'us-east1' |
| `a9c1996` | 02, oversized | `/databases/main/size`: value must be one of 'S', 'M', 'L' |
| `afabedb` | 03, cross-field (L without critical) | `/databases/main`: 'allOf' failed, missing property 'tier' |
| `d42f41a` | 04, extra key | `/databases/main`: additional properties 'ipv4Enabled' not allowed |
| `3ef5a08` | 05, YAML injection | `/databases/main/engine`: value must be one of 'POSTGRES_16', 'POSTGRES_17' |
| `dce1bdd` | 06, over the claim limit | `/databases`: maxProperties: got 3, want 2 |
| `cb151ee` | 07, unknown `environment` key | `/environment`: additional properties 'sqlFlags' not allowed |
| `0275612` | 08, hyphen in a claim name | invalid propertyName 'api-main' (pattern `^[a-z][a-z0-9]{1,33}$`) |
| `f4a192c` | 09, size L in a standard System | `/databases/main/size`: 'not' failed |
| `259be28` | 10, size L, claims.yaml says critical | `/databases/main/size`: 'not' failed |

The two (b2) commits, `65f98b6` (`environment.projectID` set) and `ed72e3b`
(`system.tier: critical` set), **passed**, as predicted (C-28 (b2), below). The
message quality is uneven. The cross-field case names the missing `tier`.
Cases 09 and 10 say only "'not' failed", which names the field and not the
rule. [C, check runs on each commit, read 2026-10-29,
`evidence/c28-pr-surface.txt`]

*Not recorded from Session A:* an edit by anyone that removes or changes the abandon
annotation; a team member (rather than the operator) doing any of the
`kubectl` cases; what a developer can read of a refusal in Kyverno's policy
reports. S3 recorded all three (table below). The 13 operator denials were refused at admission by the two rules
named above; the request bodies are not logged, so whether any patch targeted the
annotation is **not determinable**.

**S3, 2026-10-29 06:07:41Z to 06:07:52Z.** These cases ran on the return build,
as a team member: `kubectl --as=c28b-member@thecloudgeek.io
--as-group=payments@thecloudgeek.io --as-group=system:authenticated`
[C, `evidence/s3-tests.txt`].

| Case | Door | Result |
|---|---|---|
| Create a raw `SQLInstance` `c28b-raw` | team member | **refused** by Kyverno, `deny-raw-cloud-resources` / `no-cloud-objects-by-hand-in-tenant-namespaces`, with the message quoted above |
| Label `svc-hello-main` | team member | **refused**, same rule |
| Delete `svc-hello-main` (server dry run) | team member | **refused**, `no-cloud-object-deletes-by-hand-in-tenant-namespaces` |
| Patch `svc-hello-main`'s `/status` (server dry run) | team member | **admitted**: `auth can-i patch sqlinstances --subresource=status` says `yes`, and the patch returns `patched`. RBAC's `admin` covers the subresource, and Kyverno is not asked |
| Create a `NamespacedMutatingPolicy` (server dry run) | team member | **refused by RBAC**: `can-i` says `no`, and the create is `Forbidden` (`a9e2bdd` holds) |
| Remove the abandon annotation (server dry run) | cluster admin | **refused**, by `deny-raw-cloud-resources` and by `keep-abandon-annotation` / `abandon-stays` |
| Change it to `none` (server dry run) | cluster admin | **refused**, same two rules |
| Policy reports for `svc-hello` | developer read | readable: seven reports, one per cloud object, each `PASS 1` |

The removal and change cases ran as the operator, because the team member's
create and edit are already refused on the first rule, before
`keep-abandon-annotation` would be reached. The operator holds no exemption,
so both rules fire.

*Gate defects found during M2b, fixed, not refusals.* `62b9451` (2026-10-06):
the gate admitted a `kubectl` edit of a cloud object. `5da7d4a` (Session A,
2026-10-19 23:09Z): the gate refused the garbage collector's update, so "four
pruned IAMPolicyMembers stayed terminating after their cloud grants were gone,
and Kyverno's log showed the garbage collector refused on each retry" (commit
body). See Surprises 1 and 2.

**(b2) The two things predicted not to be refused.**
*A StorageBucket.* The operator created `platform-factory-ref-c28b2-never` in
namespace `svc-hello` at 22:25:10.187 (code 0; Kyverno did not block it).
Config Connector tried `storage.buckets.create` six times, 22:25:10.589 to
22:25:19.818, every one code 7, `granted: false`. The k8s `events` text,
verbatim:

> Update call failed: error applying desired state: summary: googleapi: Error 403: config-connector@platform-factory-ref.iam.gserviceaccount.com does not have storage.buckets.create access to the Google Cloud project. Permission 'storage.buckets.create' denied on resource '//storage.googleapis.com/projects/_/buckets/platform-factory-ref-c28b2-never' (or it may not exist)., forbidden

Admitted by the API server, never actuated: the bucket does not exist
(`gcloud storage buckets describe` returns 404) and no successful
`storage.buckets.*` entry exists. The object was deleted at 22:25:23.878 with
no cloud delete call. This ran with the broad roles bound (the register's
caveat: "ungated" is not "unpermitted"); `cloudsql.admin` and
`artifactregistry.admin` carry no `storage.buckets.create`.
*A `claims.yaml` that sets a real platform value* (`environment.projectID`,
`system.tier: critical`) with the rendered output byte-identical with and
without it. No record of it was made at the time. **The comparison was run at
the desk on 2026-10-29:** svc-hello#19's base `claims.yaml` (`01d746c`) and the
two (b2) commits (`65f98b6`, `ed72e3b`) were each rendered through
`charts/claims` with Helm 3.19.4. The values were the claims file, then the
`valuesObject` that `charts/system` writes for svc-hello (systems `a79a910`),
in namespace `svc-hello`. At platform-config `6949999` and again at `e8eedeb`,
all three renders were **byte-identical**: 13,562 bytes, six objects, one
SHA-256 (`7185439395fac149…`). Both PR checks passed on those commits (above).
So a team's `claims.yaml` that sets `environment.projectID: another-project`
or `system.tier: critical` changes nothing. [C, `evidence/c28-b2-render.txt`]

**(c) Remove and restore `second`: `svc-hello#22` merged 02:53:57,
`svc-hello#23` merged 02:56:21 on 2026-10-20.** Run with two claims on the
System, as ADR-0019 §2 orders, so it tested a prune and not the empty-render
guard. Argo CD deleted the Kubernetes objects at 02:54:09.480 to .511 (12.5 s
after the merge): `second-connection`, both Jobs, `svc-hello-second-app`,
`svc-hello-second-iam-user` and `svc-hello-second`. The deletion defender
returned status 10 (conflict) once per object at 02:54:09.642 to .658, then
succeeded at .718 to .747. **The instance survived**: no
`cloudsql.instances.delete` appears anywhere in the session; `svc-hello-second`
still shows `createTime` 2026-10-19T22:34:08.440Z; the Cloud SQL operation list
holds nothing between `CREATE_USER` (22:46:10) and the operator's parking
update (03:10:04) except a system backup. **Adoption: 119.6 s** after the
restore merge (SQLInstance `UpToDate` at 02:58:20.608); 118.8 s of that is the
time before Argo CD's first reapply, and Config Connector took under 0.5 s. No
cloud write was needed or made. M2's 66 s is not comparable for that reason.
The new `second-grant` Job completed at 02:58:24.

### C-29 — The cutover is a rebuild

**The `down` on Crossplane** that begins the forward leg was M2's own: cycle 6
`down` on 2026-09-17 (3-argocd 47 s, 2-cluster 527 s, total 577 s) and `park`
(34 s). The instance then sat parked for 32 days. [C, `cycle-results.tsv`] That
`down` is not M2b data.

**Forward leg (Session A), cluster create call 22:01:17.096.**
Reconstructed. The engine identity held the four custom roles and the four
broad roles together (22:32:49Z policy body), as the register requires.

| Step | Time | After create call |
|---|---|---|
| Cluster create done | 22:08:37.545 | 440.4 s (7m20s) |
| `primary` pool created | 22:11:59.164 | 642.1 s (10m42s) |
| Argo CD installed (first operator Kubernetes write) | 22:12:54.465 | 697.4 s |
| Config Connector operator `UpToDate` | 22:16:20 | 902.9 s |
| First engine token | 22:16:28.194 | 911.1 s (15m11s) |
| First logged engine cloud write | 22:17:02.874 | 945.8 s (15m45.8s) |
| `svc-hello-main` update complete | 22:29:54.675 | 1717.6 s (28m37.6s) |

| Durable resource | What the engine did | Evidence | Re-created |
|---|---|---|---|
| Cloud SQL `svc-hello-main` | One full-body `instances.update`, intents `UPDATE_MAINTENANCE_VERSION` and `START_INSTANCE`, op 12m51.8s | `createTime` 2026-09-16T16:54:08.673Z unchanged; no create or delete op | No |
| `svc-hello-main-app` (SQLDatabase), `svc-hello-main-iam-user` (SQLUser) | `UpToDate` 22:31:04 and 22:31:08, no cloud op | no `CREATE_DATABASE` or `CREATE_USER` this session | No |
| AR repos `svc-hello`, `svc-ledger` | `UpdateRepository` 22:18:50.168, 22:18:50.937 | createTime 2026-09-16 | No |
| GSAs `svc-hello`, `svc-ledger` | `PatchServiceAccount` 22:18:49.862, 22:18:52.255 | uids match today's list | No |
| Project grants, Workload Identity bindings, repository policies | No engine write in the leg | first project `SetIamPolicy` is 22:32:49 | Nothing removed |

**Non-durable resources the engine could not acquire: none found.** The only
`UpdateFailed` events are the six bucket denials. `DependencyNotFound` and
`DependencyNotReady` events are transient (`Error: secret "main-admin" not
found` at 22:16:58, resolved by 22:17:10). **Seconds any grant was absent from
the cloud policy: 0** by the project policy bodies; the repository policy is
not determinable, but no Artifact Registry `SetIamPolicy` falls in the leg.
**Manual interventions found: none**, apart from the operator's Helm install of
Argo CD and the deliberate tests. **`up` wall-clock:** no single event defines
it and no `cycle.sh` row exists; the table above is what the log has. The
M2 baseline, 35m31s, is not matched by a comparable forward-leg number.

**`psql` after the forward leg.** The one login the register asks for was not
made on the forward leg. **It was made after S3's return build:**
- **The login (2026-10-29 06:08:20Z).** A pod in `svc-hello` ran `psql -h
  10.60.0.3 -U postgres` with `PGPASSWORD` from `main-admin` and SSL
  required. Result: **`FATAL: password authentication failed for user
  "postgres"`**, pod `Failed` (`evidence/s3-c29-psql.txt`). The first attempt
  at 06:07:35Z was refused by `svc-hello`'s ResourceQuota before it started:
  the test pod named no requests or limits. That was the test's own defect,
  fixed on the re-run.
- **The Secret.** The seed Job minted a new `main-admin` at 05:52:31Z, because
  the Secret was absent after the rebuild.
- **The update.** The engine's update of `svc-hello-main` was logged at
  05:52:33Z. Its logged body carries `settings.activationPolicy: ALWAYS` and
  the chart's settings, and no `rootPassword`. Audit logs may redact
  credentials, so the absence is not evidence either way.
- **The GRANT Job.** It logged the same refusal, then "grant already in place
  for svc-hello@platform-factory-ref.iam on "app"", and completed. It needs
  the password only when a grant is missing (ADR-0017 §8).

This settles, for the one update observed, ADR-0017 §8's [I]: the live
password was left alone, as the ADR expected; the logged update body shows no
`rootPassword`, and audit logs may redact it. The `main-admin` Secret a
rebuild mints does not open `postgres` on an adopted instance. What Session
A's log holds: the `main-seed` Job started at 22:16:47 (the Secret
was absent after the rebuild, so it generated a new value) while the instance
kept the password from 2026-09-16; the `main-grant` Job completed at 22:29:34,
10 seconds after Postgres reported ready, which is consistent with an accepted
login but does not prove one (S3's Job log shows the Job completing after a
refused login when the grant is already in place, so its completion is not
evidence of an accepted one [I]); and `postgres.log` shows exactly one failed
authentication on that instance, "db=postgres,user=postgres FATAL: password
authentication failed for user "postgres"" at **22:31:29.746**, whose origin is
**not determinable**. The new `svc-hello-second` accepted its Secret's
password. Session A's one failed login at 22:31:29 is consistent with the GRANT Job's
password attempt, as S3's Job log shows [I].

**Return build (S3), 2026-10-29, `main` `e8eedeb`.** `up` 32m40s (1960 s),
13/13 Applications Synced and Healthy, adopted 5 and re-created 0 with the
creation times unchanged, no manual intervention in the leg, all engine calls
status 0 (C-31 (a) gives the table). **Seconds any grant was absent: 0** in
the leg on the poller's evidence: from 05:29:32Z (Apply B) to 06:16:17Z the
project policy's Cloud SQL and Artifact Registry bindings did not change; the
first change is Surprise 17's offboarding grant loss, after the leg [C,
`evidence/iam-poll.jsonl`, scanned for this entry]. `down` needed one re-run
(Surprise 18) and does not park (Surprise 16).

**Late cycles (2026-10-20 03:09Z to 04:35Z).** Teardown of the first cluster:
`DeleteCluster` called 03:04:28.343 (done 03:09:44.515), life 18,191 s
(5h03m11s); both instances parked at 03:10:04 and 03:11:42 by the operator.
Cycles 1 and 2 failed to create the cluster (below). Cycle 3, a Crossplane
rebuild: create call 03:47:00.423, first Crossplane cloud call 04:05:10.201
(1089.5 s after the create call). Crossplane acquired the three repositories
(`UpdateRepository`) and three GSAs (`PatchServiceAccount`), made no `Create*`
or `Delete*` call, and woke `svc-hello-main` (`START_INSTANCE`, op 04:05:56.985
to 04:20:09.508, 14m12.5s). Its Database and User objects tried a replacing
update ("refuse to update the external resource because the following update
requires replacing it: cannot change the value of the argument "project" from
"" to "platform-factory-ref"") and then 13 creates of database `app`, each
refused with "Invalid request since instance is not running". Whether they
reached Ready is **not determinable** from the logs; the last Crossplane
event is 04:19:14. This cycle ran before PR #31 merged, so it is not the
registered reverse leg; S1 is.

**Reverse leg (S1), 2026-10-27, `crossplane` branch, root at dd6730c.**
`cycle.sh up` started 19:49:15Z. [C, `s1-reverse-leg-up.log`, `cycle-results.tsv`]

| Layer | Seconds |
|---|---|
| 2-cluster (cluster 10m51s, node pool 1m25s) | 740 |
| 3-argocd | 129 |
| verify (11/11 Applications Synced and Healthy, 3/3 tenants have Applications) | 1218 |
| **`up` TOTAL** | **2106 (35m06s)** |

M2's baseline was 35m31s. Durable resources, from `cycle.sh`: "adopted: 5
[cloudsql/svc-hello-second@2026-10-19T22:34:08Z cloudsql/svc-hello-main@2026-09-16T16:54:08Z
registry/svc-hello@2026-09-16T16:53:10Z registry/svc-ledger@2026-09-16T17:05:14Z
registry/svc-onboard@2026-10-19T22:32:45Z], recreated: 0". Every `createTime`
matches the value recorded independently in Session A, so the adopted/created
line is not the resumed-`up` baseline artifact the 2026-10-23 review warned of.
**Two caveats on that line.** It lists the five instances and registries and
no GSA, so GSA adoption on the reverse leg is not evidenced by S1 (Session A's
cycle 3 shows `PatchServiceAccount`, not create). And Crossplane declares no
`second`, so "adopted" for `svc-hello-second` means it survived untouched, not
that Crossplane acquired it. **Seconds any grant was absent: 0** on the
poller's evidence: 166 samples inside 20:01:35Z to 20:36:43Z, mean spacing 12.8
s, longest 19 s, no change in the project's Cloud SQL, Artifact Registry and
custom-role bindings or in the three registries' policies; the same holds for
all 1012 samples to 23:34:56Z. The poller does not read the GSAs' own IAM
policies or any database user. **Manual interventions: none recorded** in the
two logs; `cycle.sh` does not count interventions, so this is the absence of
a recorded one, not a measured zero. **Non-durable resources not acquired:** none
reported. The engine identity held the four custom and four broad roles through
S1 (poller). `down`: 3-argocd 91 s, 2-cluster 627 s (node pool 4m55s, cluster
5m26s), **720 s (12m00s)**; run 20:26:25Z to 20:36:53Z. S1's cluster
wall-clock, first apply to cluster destroyed: 47m38s.

**`down` did not park, and the instances ran unparked for 3h15m49s.**
`cycle.sh down` does not run `park`; September's cycle 6 ran `down` then
`park` as two commands. After S1's `down` ended at 20:36:53Z nothing parked the
instances. `svc-hello-main` ran unparked from 20:36:53Z until 23:52:42Z, when
the operator ran `cycle.sh park` by hand: `cycle-results.tsv` rows "7 park":
`svc-hello-second` stopped in 11 s, `svc-hello-main` in 55 s, total 67 s; log
`evidence/s1-park.log`. [C, `cycle-results.tsv`, `s1-park.log`] This is the
"session left running" that C-32's prediction names, in miniature and uncosted
(see C-32). It also means S3's "parked instance woken" starts from a parked
instance only because of that manual command.

**Cycle numbering.** Today's `cycle-results.tsv` rows are labelled cycle 6
(`up`) and cycle 7 (`down`, then `park`) because `cycle.sh` numbers cycles from
the TSV and Session A's rows were never recorded there. September's cycle 6
had a `down` and a `park` and no `up`, so today's `up` continued it; the label
does not mean the two are the same cycle. [C, `cycle-results.tsv`]

### C-30 — The new stack is smaller

**Crossplane side, counted 2026-10-27 20:20:24Z during S1, `crossplane` branch dd6730c, three tenants** [C, `evidence/c30/crossplane-s1.txt`]:

| Count | Crossplane |
|---|---|
| Engine pods | 11 (`crossplane`, `crossplane-rbac-manager`, three Functions, `provider-family-gcp` and five providers) |
| Memory, `kubectl top` working set, summed | 1196 Mi (190 + 19 + 17 + 18 + 15 + 150 + 149 + 171 + 149 + 160 + 158); CPU 73 m |
| CRDs, engine (`crossplane.io`, `upbound.io`, `platform.thecloudgeek.io`) | 114 (112 + 2) |
| CRDs, whole cluster | 174 |
| Config Connector CRDs | 0 |

The pods had been up 13 to 16 minutes and the count preceded `up`'s final
verify by four minutes. The figures are working-set usage, not requests. The
Crossplane requests were not captured, and none are needed: the Config
Connector side is counted on the same basis, `kubectl top` working set.

**Config Connector side, counted 2026-10-29 06:07:10Z during S3, `main` `e8eedeb`, three tenants** [C, `evidence/c30/cc-s3.txt`]:

| Count | Crossplane (S1) | Config Connector (S3) |
|---|---|---|
| Engine pods | 11 | **10**: `cnrm-controller-manager-0`, `cnrm-deletiondefender-0`, `cnrm-resource-stats-recorder`, six `cnrm-webhook-manager` replicas, `configconnector-operator-0` |
| Memory, `kubectl top` working set, summed | 1196 Mi | **971 Mi** (252 + 197 + 34 + 71 + 70 + 90 + 71 + 92 + 71 + 23) |
| CPU, summed (unregistered, ungraded) | 73 m | 217 m |
| CRDs, engine | 114 | **273** (265 resource kinds + 8 `core.cnrm.cloud.google.com`) |
| CRDs, whole cluster | 174 | **333** |
| The other engine's CRDs | 0 | 0 (`crossplane.io`, `upbound.io`, `platform.thecloudgeek.io`) |

- **Tenant surface.** It is S1's: `svc-hello`, `svc-ledger`, `svc-onboard`.
  `svc-hello` carried `main` **and `second`**, which S1 did not. `svc-scratch`
  had not been added. So the Config Connector side has one more instance and
  its three objects than S1's.
- **Timing.** The pods had been up 18 to 21 minutes. The count followed `up`'s
  verify by 21 s.
- **The webhook.** Six of the ten pods are `cnrm-webhook-manager` replicas,
  set by the operator's autoscaler; the manifest figure is 6 pods in all, with
  two webhook replicas. **The pod margin is one pod** (11 to 10).
- **CPU, unregistered.** 73 m to 217 m is one `kubectl top` sample each, taken in
  different phases: S1's mid-`up`, four minutes before its verify finished;
  S3's 21 s after verify. The register asks for no CPU figure. It is reported
  beside the grade, ungraded, and not buried.
- **The manifest figures.** The 2026-10-23 review's figures were **6 pods and
  273 CRDs**: ADR-0017's 265 resource CRDs and eight of its own, six pods
  requesting about 2.3 GiB [C for the manifests, not a measurement]. S3
  measures 273 CRDs as predicted, and 10 pods using 971 Mi.

**Recipe, schema and policy lines (desk): the reading fixed on 2026-10-28.**
The register's dated note of 2026-10-28 ("before S3 counts the Config Connector
side, C-30's two open readings are fixed") reads the line count narrowly:
recipe is the two Compositions against the templates of `charts/system` and
`charts/claims`; schema is the two XRDs against the two `values.schema.json`;
policy is `kyverno/policies/`; lines are non-comment with template comments
stripped. **The note says the reading was chosen after the 2026-10-23 desk
count had been seen on both sides**, and that it is the reading less favourable
to the prediction. The recount below follows it. [C, [`m2b-c30-concept-lists.md`](m2b-c30-concept-lists.md) §7]

| Category (narrow, graded) | Crossplane `3bfc72d` | Config Connector `6949999` | Config Connector `main` `e8eedeb` |
|---|---|---|---|
| Recipe | 633 | 654 | 656 |
| Schema | 297 | 383 | 383 |
| Policy | 144 | 153 | 153 |
| **Total** | **1074** | 1190 | **1192** |

Main against Crossplane: +118 lines, **+11.0%**. **The 2026-10-23 figure for
`6949999`, 1206, included `Chart.yaml` (16 lines) in chart meta; the fixed
reading counts templates only, which gives 1190 (re-run and confirmed).** The
only counted file that changed since `6949999` is `charts/system/templates/argocd.yaml`
(+2 non-comment lines, the `policies.kyverno.io` rows from `a9e2bdd`). Recipe
alone: 633 to 656. **The line half of the prediction ("fewer recipe lines")
missed.** S3 ran `e8eedeb`, the commit counted, so no re-run is needed.

The 2026-10-23 wider rows, beside the graded ones and **ungraded**
(counted at `6949999`, before `a9e2bdd`):

| Set (ungraded) | Crossplane | Config Connector | Change |
|---|---|---|---|
| Wide: recipe incl. platform and provider plumbing | 834 | 692 | -17.0% |
| Wide: recipe + schema + policy + engine Applications | 1424 | 1313 | -7.8% |
| Narrow, plus ADR-0018's roles and Terraform (Apply B state) | 1074 | 1321 | +23.0% |
| Narrow, plus the role-file checker | 1074 | 1550 | +44.3% |

With template comments left in (hash-only), the narrow recipe core read 850 to
832 (-2.1%) on 2026-10-23.

**Concepts: the desk list, made by the 2026-10-28 method.** A reader follows
`svc-hello` with one database claim from tenant file and claim to cloud
resources. Two kinds of item are listed once, by name, with the file where the
reader first meets it: every API kind met on the path, and every mechanism that
is not a kind but that a file on the path cites as the reason an object is
shaped, ordered, named, kept or refused as it is. Shared items are counted on
both sides. Repos are not items; ADR-0018's three additions are one each. The
lists are in full in [`m2b-c30-concept-lists.md`](m2b-c30-concept-lists.md) §2 and §3 (names, files and lines);
the totals are:

| | Kinds | Mechanisms | Total | With ADR-0018's three additions |
|---|---|---|---|---|
| Crossplane (`3bfc72d`) | 28 | 36 | **64** | n/a |
| Config Connector (`main` `e8eedeb`) | 20 | 36 | **56** | **59** |

Shared with each other: 11 kinds and 20 mechanisms. Engine-specific: Crossplane
17 kinds and 16 mechanisms (33); Config Connector 9 kinds and 16 mechanisms
(25); the 8-point gap is entirely there. ADR-0018's additions are the eighth
repo (`platform-roles`), the custom roles with one IAM Condition, and a second
home for Terraform. **59 is lower than 64 by 5 (7.8%); without the additions
the margin is 8 (12.5%).** The prediction, "still fewer, by a smaller margin",
holds on the desk count, and the falsifier's concept condition is not met.

**Items a reader might dispute, and their largest swing** [I, my sum of the
listed items]: the three `cnrm.cloud.google.com/*` annotations are one item
each (bundled, Config Connector falls by 2). Against the prediction: the
Crossplane-form `Database` kind still present in `svc-hello` but skipped by the
tenant Application would add 1 kind and 1 mechanism to Config Connector (+2);
`crdWatcher`, still set in Config Connector's `apps/kyverno.yaml`, would add 1;
excluding engine-install kinds (`Provider`, `DeploymentRuntimeConfig`,
`ConfigConnector`) takes 2 from Crossplane and 1 from Config Connector. The
worst case in that direction is Config Connector 62 - 1 = 61 against Crossplane
64 - 2 = 62, still lower. Other items run the other way (dropping the two
mechanisms that exist only because of `a9e2bdd` and the 2026-10-06 rehearsal
takes 2 from Config Connector; `ImageConfig`, if included, adds 1 to
Crossplane). No listed swing makes 59 reach 64. The remaining disputes
(Terraform as a shared mechanism; the parallel "how the form is enforced"
items) move both sides together or are a matter of how many concepts one is.

**Time to diagnose, beside the claim and ungraded (ADR-0019, Consequences):**
minutes from noticing each failure to knowing its cause were not recorded
during Session A. The one case the record allows: the `member is immutable`
refusal at 23:25:50.579 was named in the refusal itself; the restore that did
not heal took 777.2 s and an operator patch.

### C-31 — The engine runs on narrow roles; what they leave out is the third lock

**Permission counts.** Before: the four built-in roles hold 182 (cloudsql.admin), 69
(artifactregistry.admin), 19 (iam.serviceAccountAdmin) and 9
(resourcemanager.projectIamAdmin) permissions on 2026-10-27 (`gcloud iam roles
describe`), **279**; ADR-0018 wrote 173, about 70, 19 and 9, about 270. The role
files hold 8, 5, 6 and 2, **21**. After Apply B, `gcloud iam roles describe`
reads the four custom roles as 8, 5, 6 and 2, **21** (2026-10-29 05:29Z,
`evidence/apply-b/reads-after.txt`).

**Desk, before narrowing (done 2026-10-27).** Each role file is a subset of the
built-in it replaces: "platformEngineSql: 8 perms vs roles/cloudsql.admin (182):
not in built-in -> none (subset)", and the same line, "none (subset)", for
Registry (5 of 69), ServiceAccounts (6 of 19) and ProjectIam (2 of 9). [C,
`evidence/c31/desk-subset.txt`] **The register put this check before the first
leg; the evidence for it is dated after both.** Today's built-ins have grown
(cloudsql.admin was 173), so a subset today follows from one earlier only if
no permission was withdrawn. **No earlier record was found.**
`platform-roles`' CI check, the one gate that ran before the leg
(2026-09-27, 2026-09-30 and 2026-10-06, run 37497043602), checks the role
files' grammar and the grants allowlist. It does not compare the files with
the built-ins, and platform-roles#1's body describes no such comparison. So
the subset check the register put before the first leg was not run then, and
`desk-subset.txt` is its first record.

**Write permissions the first session's audit log shows the engine using, and
whether the files have them.** From admin-activity entries for
`config-connector@`, 22:17Z to 23:40Z on 2026-10-19 [C, `desk-used-vs-files.txt`]:
`artifactregistry.repositories.create`, `.setIamPolicy`, `.update`;
`cloudsql.instances.create`, `.update`; `iam.serviceAccounts.create`,
`.setIamPolicy`, `.update`; `resourcemanager.projects.setIamPolicy`: all in a
role file. **Missing from every file: `artifactregistry.repositories.delete`,
used once, by the C-31 control** (below), as the Apply B commit body says. So
the desk found **none among the writes the admin-activity log can see.** The
register predicted the desk would find at least one. **That half of the
prediction missed, in the favourable direction: zero missing permissions found
by what ran** (the admin-activity scan, blind to reads and to
`cloudsql.databases.create` and `cloudsql.users.create`; the return build; the
scratch tenant's onboarding, an instance create, a database, a user, service
accounts and project IAM, and its offboarding, a service-account delete; the
desk script's must-succeed list has not run). **Zero found is not a proof that
21 permissions suffice.** The scan has two blind spots: `cloudsql.databases.create`
and `cloudsql.users.create` ran (operations `CREATE_DATABASE`, `CREATE_USER`,
both by `config-connector@`) but are not admin-activity entries, so the scan
could not see them (both are in `platformEngineSql`); and reads are unlogged, so
a `list` or `get` the engine makes that no file holds cannot show.

**Control (first session, broad roles).** Namespace `c31-scratch` created by
the operator with `kubectl` at 22:22:04.220; `ArtifactRegistryRepository`
`c31-control` created 22:22:07.127; the cloud create call was 22:22:07.823
(status 0, completed 22:22:08.529); `UpToDate` 22:22:18; the object deleted
22:22:20.510; cloud `DeleteRepository` 22:22:20.824, **status 0**, completed
22:22:20.964. **The cloud delete succeeded**, user agent `kcc/1.156.0`,
the repository lived 12.4 s in the cloud, and the Kyverno gate did not stop
the operator's delete [I: the namespace carries no system label, so ADR-0017
§7's rule does not match it]. Namespace
deleted 22:22:29.160. `gcloud artifacts repositories list` shows no
`c31-control` today. That the object carried no abandon annotation is **not
determinable** (request bodies are not logged); the cloud delete succeeding is
consistent with it.

**Whether the first real project grant passes the condition.** In Session A
Config Connector made the project grants (22:32:49.515 ADD, 22:39:56.999
REMOVE, and the 23:25:53 to 23:30:14 writes), all within `hasOnly(['roles/cloudsql.client',
'roles/cloudsql.instanceUser'])`, and all succeeded, while the only path to
`resourcemanager.projects.setIamPolicy` the identity had was bindings under
that condition (the broad `projectIamAdmin` grant carried the same one). [I:
the logged request bodies do not show conditions, so this rests on the
status-0 results and the policy as the poller reads it.] Whether the custom
role alone passes it is the remainder. **After Apply B it does.** The first
project grant through Config Connector on the custom roles alone was the
scratch tenant's, at 2026-10-29 06:19:15.549Z: one `SetIamPolicy` on
`projects/platform-factory-ref` by `config-connector@`, status 0. Its binding
deltas:
- ADD `roles/cloudsql.client` for `serviceAccount:svc-scratch@`;
- ADD `roles/cloudsql.instanceUser` for `group:checkout@thecloudgeek.io`;
- ADD `roles/cloudsql.instanceUser` for `serviceAccount:svc-scratch@`.

All three are inside `hasOnly(['roles/cloudsql.client',
'roles/cloudsql.instanceUser'])` [C, `evidence/c31/first-project-grant.json`].
The offboarding removed them again, and no project grant names `svc-scratch@`
after it. On `main` these grants carry no IAM condition of their own. C-24's
per-instance condition is M3's.

**Apply B, prepared, not applied (reads before, 2026-10-27 20:27:07Z).**
`terraform init` succeeded; `plan`: "Plan: 0 to add, 0 to change, 4 to destroy",
the four `engine_broad_*` members. The identity held the four custom roles
(ProjectIam under the condition titled "only-cloudsql-connect-roles"), the four
broad roles and `roles/compute.viewer`; the engine service account's own policy
held one binding, `roles/iam.workloadIdentityUser` for
`[cnrm-system/cnrm-controller-manager]`. The Policy Analyzer read is absent (first made 2026-10-29 06:00:40Z,
Built above).
**The commit body of `b766d81` says "Applied with the cluster down" in the past
tense. At merge (`platform-roles#4`, `08178fb`, 2026-10-27 20:37:53Z) Apply B had
not been applied, and it was still unapplied at 00:03Z on 2026-10-28: the
engine still held its four broad roles plus `roles/compute.viewer`** [C,
poller; desk read]. The record was written before the event it reports.
**Applied 2026-10-29 05:29:10Z to 05:29:25Z** from `platform-roles` `main`
(`08178fb`), by the operator: "Apply complete! Resources: 0 added, 0 changed, 4
destroyed", rc 0 (`evidence/apply-b/apply.log`). The poller saw the project
policy change between 05:29:20Z and 05:29:32Z. An empty commit on
`platform-roles` `main` records the correction to `b766d81`. The first attempt,
at 05:27:22Z, stopped before planning because the operator's
application-default credentials had expired (`invalid_rapt`). Nothing changed
then. **Reads after:**
- `terraform plan -detailed-exitcode`: exit 0, "No changes".
- The project policy for the engine: exactly `platformEngineProjectIam`
  (condition "only-cloudsql-connect-roles"), `platformEngineRegistry`,
  `platformEngineServiceAccounts`, `platformEngineSql` and
  `roles/compute.viewer`. None of the four broad built-ins, and no
  `roles/viewer`.
- The engine service account's own policy: only
  `roles/iam.workloadIdentityUser` for `[cnrm-system/cnrm-controller-manager]`.

**Poller gap.** The IAM poller wrote nulls from 2026-10-28 07:35:42Z to
2026-10-29 05:26:21Z, because the operator's gcloud credentials had expired.
The cluster was down and no build command ran in that window, so the gap
hides no build activity. It also covers no IAM change by this project's
tooling: the next good read, at 05:26:21Z, matches the last before the
gap.

**The desk script as the engine: NOT RUN.** The script is
`c31_engine_desk.sh`: it impersonates `config-connector@`, needs no cluster,
and needs a temporary Token Creator grant, which the run removes.
- **First run (2026-10-29 05:30:55Z).** The temporary grant was added at
  05:29:34Z, and impersonation worked after 90 s. The script then stopped on a
  quoting error in the script itself, before any call: `--format=value(...)`
  unquoted. The safety pass found no drift: `platform-roles` plan exit 0, no
  `roles/viewer`. The grant was removed at 05:31:01Z, and the engine service
  account's policy was back to Workload Identity only
  (`evidence/c31/desk-grant-run1.txt`, `desk-after-run1.txt`).
- **Second run.** The quoting was fixed, verdict lines were added and the
  logic was checked with read-only operator calls. Run by the delegate, it was
  refused by the session's auto-mode classifier (a temporary IAM grant).
- **Third run, 2026-10-31, by the operator.** Two copies of the script
  overlapped. The first, at 14:40:37Z, met expired credentials, so its deletes
  and its grant failed, but its wait loop kept polling. The second, at
  14:44:27Z after a fresh login, made the grant at 14:48:34Z. Impersonation
  began working about 90 s later, so both copies ran the test: 14:50:06Z to
  14:50:19Z and 14:50:09Z to 14:50:22Z. The first copy removed the grant at
  14:50:21Z. Its test finished before that removal, so it is the clean record,
  and the second copy's verdicts are identical. Both wrote to the same
  `desk-after.txt`, so each copy's log is kept separately
  (`evidence/c31/desk-20261031-runA-full.txt`, `-runB-full.txt`).

The results, 12 verdicts of 12:

| Attempted as the engine | Result |
|---|---|
| Describe instance, registry, service account; read the project policy | succeeded |
| Describe a database, list users | admitted by IAM; 400 "instance is not running", because the instance is parked. The reads themselves ran in S3 |
| Delete an instance | refused, `PERMISSION_DENIED` |
| Delete a database, delete a user | refused, HTTP 403, naming no permission |
| Delete a registry | refused, naming `artifactregistry.repositories.delete` |
| `gcloud iam roles update platformEngineSql --add-permissions=cloudsql.instances.delete` | refused, naming `iam.roles.get`: the command reads the role first, so the update call itself was never reached |
| Grant itself `roles/viewer` on the project | refused, "Policy update access denied" |

Afterwards, read as the operator: the engine holds the same five grants, no
`roles/viewer`, `platformEngineSql` still has 8 permissions, and the service
account's policy is back to Workload Identity only. The script's own drift
check did not work: `terraform plan` exited 1 in both copies, because the
application-default credentials had expired. Re-run at 14:51:23Z with an
operator token, it reports no changes (`evidence/c31/desk-plan-after.txt`).
[C, `evidence/c31/desk-20261031-runA-full.txt`, `desk-grant.txt`]

The cluster half follows.

**(a) The return build, S3, 2026-10-29: cycle 7 `up` from `main` (`e8eedeb`)
onto the narrowed roles.**
- **Rebuild:** 2-cluster 612 s, 3-argocd 116 s, verify 1222 s, **total 1960 s
  (32m40s)**, 0 manual interventions.
- **Applications:** 13/13 Synced and Healthy, and 3/3 tenants with
  Applications.
- **Durable resources:** adopted 5, re-created 0:
  - `cloudsql/svc-hello-main@2026-09-16T16:54:08Z`;
  - `cloudsql/svc-hello-second@2026-10-19T22:34:08Z`;
  - `registry/svc-hello@2026-09-16`, `registry/svc-ledger@2026-09-16`,
    `registry/svc-onboard@2026-10-19`.
- **The parked instances.** Both were woken by the engine's own
  `cloudsql.instances.update` (`activationPolicy: ALWAYS`): `svc-hello-main`
  05:52:33Z to 06:04:52Z, and `svc-hello-second` 05:52:29Z to 06:05:40Z.
- **Engine calls during `up`:** all status 0. Four instance updates, three
  `PatchServiceAccount` and three `UpdateRepository`. No object reported a
  permission error.
- **Beside C-29's forward leg:** the same five durable resources. Session A's
  tenants (`svc-onboard`, and `svc-hello`'s `second`) are in the table, and
  `svc-hello`'s move to checkout and back is visible only as its unchanged
  service account.
- **Comparison.** The forward leg has no `cycle.sh` row (see C-29). M2's
  baseline is 35m31s and cycle 6's Crossplane `up` 2106 s.

[C, `cycle-results.tsv`, `evidence/s3-up.log`, `evidence/s3-engine-audit.tsv`]

**The scratch tenant, onboarded from one file, then offboarded.**
- **Offboarding `svc-onboard` first**, because `svc-scratch` reuses its repo.
  `systems#19` was merged at 06:14:30Z. It deleted nothing by itself: after
  60 s all three Applications and the namespace were still there. The
  person's `argocd app delete --core svc-onboard-system` ran at 06:16:09Z,
  with no claims step because there was no database. The namespace and
  Applications were gone by 06:17:11Z. The engine deleted the service
  account `svc-onboard@`: `DeleteServiceAccount` by `config-connector@` at
  06:16:17Z, status 0 (`evidence/s3-engine-audit.tsv`). The operator's
  `gcloud iam service-accounts describe` afterwards returned
  `PERMISSION_DENIED: Permission 'iam.serviceAccounts.get' denied on resource
  (or it may not exist)` (`evidence/c31/a-offboard-svc-onboard.txt`). That read
  is the account being gone, not an operator permission fault. The registry
  `svc-onboard` stays, abandoned, for a person's `gcloud` delete (ADR-0015 §5).
  `systems#19` (`a406cef`) put `svc-onboard` on the retired list, in the same
  change that removed `tenants/svc-onboard.yaml`.
- **Onboarding.** `svc-onboard#3` added the claim (06:17:44Z). `systems#20`,
  the one tenant file, was merged at **06:19:04Z**. Then:
  - +12 s: the engine's `cloudsql.instances.create`;
  - +14 s: the namespace;
  - +8m42s: the workload Running;
  - 06:30:21Z: the instance create done (11m05s);
  - 06:31:24Z: the database Ready;
  - **06:31:32Z: the GRANT Job complete, "granted svc-scratch@…iam on
    "app"", 12m28s after the merge**;
  - by 06:31:38Z: all three Applications Healthy.

  **Two manual interventions, both caused by reusing another tenant's repo.
  Neither would occur for a tenant with its own repo [I].**
  1. The repo's manifests hard-code `namespace: svc-onboard`, and the
     `svc-scratch` AppProject refused them: "namespace svc-onboard is not
     permitted in project 'svc-scratch'". That is the tenant boundary working.
     `svc-onboard#4` repointed them at 06:25:34Z.
  2. "terminate svc-scratch's retrying operation (pinned to 7b0bc6a; main is
     f3742c6)" (the evidence's own words, 06:27:37Z). Argo CD kept retrying the
     failed sync at the revision it started with, so the fix was not picked up,
     and the operation was terminated by hand. This is Argo CD's retry-pinning,
     the second occurrence after Session A's move 2 (Surprise 7).

  While the 11-minute create ran, the instance object showed
  `DependencyNotFound` ("Secret svc-scratch/main-admin was not found"), a
  stale condition. The Secret had existed since 06:19:17Z.
- **Offboarding `svc-scratch`.** `systems#21` removed the file and added the
  name to `retired.yaml` in one change, merged 06:33:14Z. After 30 s nothing
  had been deleted. Then the person's two steps:
  - `argocd app delete svc-scratch-claims` at 06:33:47Z: gone at 06:33:55Z,
    with the instance abandoned;
  - `argocd app delete svc-scratch-system` at 06:33:58Z: the namespace and
    Applications gone by 06:34:53Z.

  The engine deleted `svc-scratch@`, and no project grant names it
  afterwards. **Left behind:** instance `svc-scratch-main` (RUNNABLE,
  deletion protection on) and registry `svc-scratch`. The operator deleted
  both with `gcloud` on 2026-10-31 (ADR-0015 §5), and the name then came off
  the retired list (ADR-0017 §9).
- **`svc-onboard`'s repo restored:** `svc-onboard#5`.
- **`svc-hello-second`, abandoned and left running.** Its claim was removed in
  `svc-hello#25` (merged 06:21:16Z); the SQLInstance object was gone by
  06:21:24Z and the cloud instance stayed RUNNABLE, deletion-protected. The
  person's `gcloud sql instances delete` at 06:21:47Z was refused with a 400,
  "The instance is protected. Please disable the deletion protection and try
  again.", and turning protection off was refused to the delegate by the
  session's classifier ("Irreversible Deletion"; noted in `evidence/s3-cleanup-second.txt`). `cycle.sh park` stopped it at 06:37Z. The operator
  turned protection off and deleted it on 2026-10-31. It was C-32's "left running" until it was parked, and
  M3's C-24 depends on it. [C, `evidence/s3-cleanup-second.txt`,
  `evidence/s3-park.log`]

[C, `evidence/c31/a-offboard-svc-onboard.txt`, `a-onboard-svc-scratch.txt`,
`a-offboard-svc-scratch.txt`]

**(b) Refused by Google, S3, 2026-10-29.** Namespace `c31-scratch` had no
System label and carried the project annotation. In it, a raw
`ArtifactRegistryRepository` `c31-scratch-repo` and a raw `SQLDatabase`
`c31-scratch-db` on `svc-hello-main` were created by the operator with
`kubectl`, with no abandon annotation. Both were Ready by 06:09:59Z, and the
repository's cloud `createTime` was 2026-10-29T06:09:48.649397Z. `kubectl
delete` on both ran at 06:10:01Z.
- **The repository:** eight `DeleteRepository` calls by `config-connector@`
  from 06:10:01.9Z to 06:11:05.7Z, each **status 7**, "Permission
  'artifactregistry.repositories.delete' denied on resource
  '//artifactregistry.googleapis.com/projects/platform-factory-ref/locations/us-central1/repositories/c31-scratch-repo'
  (or it may not exist)".
- **The database:** "Delete call failed: error deleting resource: [{0 Error
  when reading or editing Database: googleapi: Error 403: The client is not
  authorized to make this request., notAuthorized []}]". The database
  existed, so this is a refusal, not a missing name. **The message names no
  permission**, where the registry's error names
  `artifactregistry.repositories.delete`. The register asks whether the error
  named the permission, so this is a registered datum (Surprise 19).
- **Both objects stayed** with `deletionTimestamp` set, finalizer
  `cnrm.cloud.google.com/finalizer` and condition `DeleteFailed`. **Whether
  the namespace hangs was not tested**: the namespace was not being deleted
  while the objects were stuck. The objects were released at 06:12:01Z and the
  namespace was deleted at 06:13:07Z, after. The cloud read
  after 90 s showed both still present, the repository's `createTime`
  unchanged.
- **The release.** The abandon annotation was added at 06:12:01Z. Both
  objects were gone at 06:12:14Z, **13 s**, and both cloud resources were
  still present. Config Connector's event for each says "Successfully
  deleted", for an object it abandoned (Surprise 22; `evidence/c31/b-part2.txt`).
- **Then the person's deletes.** `gcloud artifacts repositories delete` and
  `gcloud sql databases delete` both succeeded at 06:12:16Z to 06:12:20Z, and
  the reads after return NOT_FOUND and 404. The namespace was deleted at
  06:13:07Z.

The "minutes from a fix's apply to the engine's next successful call" datum
is not applicable: no fix was needed. An instance delete was not attempted, as
registered ((b) names the registry and the database). [C,
`evidence/c31/b-part1.txt`, `b-part2.txt`]

**After narrowing.**
- **Permission errors for the engine identity:** none apart from (b)'s
  intended delete refusals. That holds at the desk and in the cluster.
  Between 05:34Z and 06:35Z the engine made:
  - 4 `cloudsql.instances.update` and 2 `cloudsql.instances.create`;
  - 4 `CreateRepository`, 3 `UpdateRepository`, 2 `SetIamPolicy` on
    repositories and 8 refused `DeleteRepository`;
  - 1 `CreateServiceAccount`, 3 `PatchServiceAccount`, 2
    `DeleteServiceAccount` and 2 `SetIAMPolicy` on service accounts;
  - 4 `SetIamPolicy` on the project.

  Every one is status 0 except the refused deletes.
- **`platform-roles` applies needed after Apply B: 0.** The plan read "No
  changes" at 05:29Z and again at S3's end.
- **Grants on the identity not in the repo: none.** At 06:35:34Z the project
  policy shows exactly the four custom roles and `roles/compute.viewer`. The
  service account's own policy shows only the Workload Identity binding.
  Policy Analyzer at organisation scope shows the same five bindings, all on
  the project (06:35:34Z, no expansion flags; with `--expand-roles
  --analyze-service-account-impersonation` at 06:51:55Z, Built above).

[C, `evidence/c31/final-reads.txt`, `evidence/s3-engine-audit.tsv`]

**Terraform applies M2b caused, for C-01's tally of declared crossings:** the
additive layer-0 change (`e7e180c`), Apply A, Apply B. The layer-0 change
created `config-connector@` on 2026-10-07 at 20:51:19.550Z (`CreateServiceAccount`
by the operator); Apply A ran on 2026-10-16 at 03:04:44–46Z; Apply B ran on
2026-10-29 at 05:29:10–25Z [C, Admin Activity audit log, read 2026-10-28;
`evidence/apply-b/apply.log`].

### C-32 — Rebuild cost is measured, not estimated

**The export cannot be tied to a session.** The billing export in BigQuery
dataset `thecloudgeek.billing` holds tables only for billing account
`015588-C94B87-3A50AF`, with no rows for `platform-factory-ref`. The project
bills to `0174CB-E6957B-042C42`. This is the mismatch the M1 entry recorded on
2026-08-28 ("the export must be keyed to billing account ...; the pre-existing
table in that dataset covers a different account"), called there "a one-query
follow-up, not a missing premise". It has stayed unrepaired through M2 and M2b.
[C for the account IDs, as read on 2026-10-27; the evidence directory holds no
query transcript.] **No figure is published here**: nothing in this entry is
a cost.

What can be stated without the export is wall-clock, and only that:

| Session | Cluster present | Note |
|---|---|---|
| Session A, first cluster | 22:01:17Z to 03:04:28Z (create call to delete call), 5h03m11s; delete done 03:09:44 | `svc-hello-main` woken (update done 22:29:54); `svc-hello-second` created 22:34:08 |
| Cycles 1 and 2 | create calls 03:16:55 and 03:23:00 failed after 223.0 s and 212.3 s; delete calls 03:21:24 and 03:45:24 | No cluster usable |
| Cycle 3 (Crossplane) | 03:47:00 to the delete call 04:28:29.894, 41m29s; delete done 04:33:59.849 | Instance woken 04:05:57 |
| S1 | 19:49:15Z to 20:36:53Z, 47m38s | |
| S3 | start 05:34:08Z; cluster destroyed by 06:50:11Z (`evidence/s3-after-down.txt`, `cycle-results.tsv` cycle 9) | `park` of three instances, 167 s, at 06:37Z to 06:39Z, before the cluster was gone (Surprise 18). `svc-hello-second` was abandoned and deletion-protected, and `svc-scratch-main` was a leftover; the operator deleted both on 2026-10-31 (C-31 Data). Not costed |
| Parked instances | `svc-hello-second` stopped from 03:10:04 on 10-20; `svc-hello-main` stopped 04:34:31.877 | After S1: `svc-hello-main` **ran unparked 20:36:53Z to 23:52:42Z (3h15m49s, about 3h16m)** until the operator's `cycle.sh park` (67 s); not costed |

The unparked instance is the prediction's "session left running", in miniature
and uncosted: 3h15m49s of an instance the plan had parked, with no figure for
what it cost. What is needed, and is the human's to do: enable a billing export
for `0174CB-E6957B-042C42`. The M1 entry's expectation (US multi-region
backfills the current and previous month) has not been checked for this
dataset, and a backfill reaching 2026-10-19/20 and 2026-10-27 is [unverified].
Everything the register asks for after that (list-price and net spend per
session, the implied hourly rate against ADR-0009's roughly $0.50, the
parked-instance line items, the split by the `system` label) follows from it or
does not exist.

### C-33 — A removed, renamed or broken file deletes no tenant by itself; a Kyverno outage recovers with one manual sync per failed Application

Local rehearsal, 2026-10-27. k3d v5.8.3, k3s v1.33.6, one node; Argo CD v3.4.6
from upstream manifests; Kyverno chart 3.9.1 (app v1.19.1) from
`platform-config` 6949999; the six Config Connector 1.156.0 CRDs the charts use,
no engine; a git server inside the cluster (the system schema pins
`gitBase` to `^https://`). Three tenants as registered: `svc-hello` and
`svc-ledger` with one claim each, `svc-onboard` with none. Method: a
checker diffed name, UID and `deletionTimestamp` of the ApplicationSet, every
Application and AppProject, every system-labelled namespace and every Config
Connector object against a baseline after each case. Argo CD noticed a pushed
commit 1 to 5 minutes later (about ten for (d2)). Deviations of setup: URLs
repointed to the in-cluster server; the Kyverno registry override removed;
tenant workloads in ImagePullBackOff throughout (private images), unrelated to
the cases. [C, `evidence/c33/REPORT.md` and logs]

**(a) The ApplicationSet.**
*a1, remove `apps/systems.yaml`* (pushed 20:04:58): root OutOfSync; the
ApplicationSet OutOfSync with `requiresPruning=True`, finalizer intact, not
deleting; 12 of 12 Applications kept their UIDs. Nothing deleted. Restored
20:06:33.
*a2, rename it to `systems-renamed`* (20:10:10): the new set Degraded, "error
while wrapping using MutateFn: Object argocd/svc-hello-system is already owned
by another ApplicationSet controller systems"; it adopted nothing; the old set
stayed healthy. Nothing deleted. After the revert the stray set stayed
(`Prune=false`) and a person ran `kubectl delete applicationset
systems-renamed`; it owned nothing.
*a3, `kubectl delete applicationset systems`, background cascade* (20:19:00):
root's self-heal re-created it at 20:19:01 (new UID); 12 of 12 Applications kept
their UIDs, owner references re-pointed. The stricter variant, with root's
auto-sync off (delete at 20:20:49): the set stayed deleted 60 s, 12 of 12
Applications survived, none deleting, owner references removed. Nothing deleted
before a person's step.
*a4, `charts/system` stops rendering the Namespace* (20:23:21): three `-system`
Applications OutOfSync; the Namespace `requiresPruning: True`, result
PruneSkipped "ignored (no prune)"; namespaces Active with the same UIDs. Nothing
deleted. (A first attempt at 20:23:10 committed nothing: the editing script
raised.)

**(b) `tenants/` and offboarding.**
*b1, rename `tenants/` to `tenants-renamed/`* (20:32:35): the ApplicationSet
logged "generated 3" and then "generated 0 applications" from 20:35:13 every
three minutes; no Application deleted (`create-update`); `ErrorOccurred` False.
The three `-system` Applications went to sync Unknown, ComparisonError, for
`svc-hello`: "Failed to load target state: failed to generate manifest for
source 1 of 2: rpc error: code = Unknown desc = Manifest generation error
(cached): failed to execute helm template command: failed running helm: `helm
template . --name-template svc-hello-system --namespace svc-hello --kube-version
1.33.6 --values <path to cached source>/tenants/svc-hello.yaml … --include-crds`
failed exit status 1: Error: open <path to cached source>/tenants/svc-hello.yaml:
no such file or directory". Nothing deleted. Restored 20:45:05, all Synced in
189 s.
*b2, offboard `svc-ledger` per ADR-0019 §1* (PR `f5572fb` removing its file and
adding it to `retired.yaml`, pushed 20:48:44; `systems` CI not run). Before the
person's step: `svc-ledger-system` sync Unknown with the same ComparisonError,
the checker clean, "generated 2 applications". The person's step, 20:56:45,
`argocd app delete svc-ledger-system --core --yes`. After it exactly
`svc-ledger` went: three Applications, the AppProject, the Namespace and ten
Config Connector objects; the Namespace Terminating at +5 s, gone by +20 s (the
Namespace's `Prune=false` did not block the offboard). Revert: back in about 20 s,
Synced within 61 s, with new UIDs.

**(c) Claims.**
*c1, the only claim removed as `databases: {}`* (21:01:02): `svc-hello-claims`
OutOfSync, condition SyncError, "Skipping sync attempt to
[1263ecb4fb701d3702f5ce3385cfb0093cbe1ef4 7859f43d03838808a7f5b15f73270bd0c1f657af]:
auto-sync will wipe out all resources", six rows `requiresPruning` (ConfigMap,
two Jobs, SQLInstance, SQLDatabase, SQLUser), about 75 s after the push.
*c2, `claims.yaml` renamed `claims.yml`* (21:08:05): the same condition and rows
(revision 54ea783), about 245 s. Nothing deleted.
*c3, manual `argocd app sync svc-hello-claims --prune --core`* (state pushed
21:15:32, `ed904c1`; sync at 21:20:48): pruned the two Jobs, the ConfigMap, the
SQLUser, the SQLDatabase and the SQLInstance, the only checker difference; then
Synced and Healthy, no condition.
*The tenant with no `claims.yaml`* (`svc-onboard-claims`): Synced, Healthy, no
conditions, no resources, through every case including (e).

**(d) A tenant file with a required field missing.** The register does not name
the field; two ran.
*d1, `metadata.name` removed from `tenants/svc-onboard.yaml`* (21:26:33): the
ApplicationSet `ErrorOccurred=True`, "failed to execute go template {{
.metadata.name }}-system: template: base:1:12: executing "base" at
<.metadata.name>: nil pointer evaluating interface {}.name"; "unable to generate
applications", then "generated 2 applications" at 21:29:14; the set and root
Degraded. No Application deleted; checker clean. Restored 21:30:44.
*d2, `spec.size` removed* (21:32:25): `svc-onboard-system` Unknown,
ComparisonError, "…Error: values don't meet the specifications of the schema(s)
in the following chart(s): system: - at '/spec': missing property 'size'"; about
ten minutes to detect; the set Healthy; nothing deleted. Restored 21:44:14.

**(e) A Kyverno outage.**
*Attempt 1, as registered* (admission controller scaled to 0 at 21:48:30,
`kyverno` Application auto-sync on): self-heal did **not** restore it over 180
s, because the chart renders `replicas:` empty and Argo CD does not own
`spec.replicas`; so the repeat with auto-sync off was not needed. **No write was
refused.** On shutdown Kyverno removed its resource, policy and exception
webhook configurations; only cleanup and TTL configurations remained. At 21:52:44
a `svc-hello` claim change merged and namespace `svc-onboard` was deleted, and it
went at once; self-heal re-created it. Under the register's own rule (e)
counts only if at least one write was refused, so this attempt does not count.
Kyverno was back 21:53:35 after a revert (`221ee71`).
*Attempt 2, a deviation* (to approximate a crashed Kyverno): the eight webhook
configurations (five validating, three mutating) were saved while healthy;
at 21:58:20 the admission controller was scaled to 0, the Pod waited out, and
the saved configurations re-applied. `validate.kyverno.svc-fail` is
`failurePolicy: Fail` on the gated groups. Refused hand write, 21:58:41
(`kubectl annotate sqlinstance svc-hello-main c33-probe=1`):

> Error from server (InternalError): Internal error occurred: failed calling webhook "validate.kyverno.svc-fail": failed to call webhook: Post "https://kyverno-svc.kyverno.svc:443/validate/fail?timeout=10s": no endpoints available for service "kyverno-svc"

At 21:58:42 a `svc-hello` claim change merged (`backups` true to false,
1364496) and `kubectl delete namespace svc-onboard --wait=false` ran (eight
Config Connector objects). The Namespace went Terminating with
NamespaceDeletionContentFailure "ContentDeletionFailed: Failed to delete all
resource types, 3 remaining: Internal error occurred: failed calling webhook
"validate.kyverno.svc-fail": … no endpoints available for service
"kyverno-svc"" (three times). No other commit reached any source during the
wait (heads platform-config 1263ecb, systems 8c1e568, svc-hello 1364496,
svc-ledger 423611b, svc-onboard 2d07309). At 22:44:03, 45 minutes after the
merge, all three blocked Applications had used the block: `svc-hello-claims`
started 22:00:07, failed 22:37:37, **37.5 minutes**, retry 10 of 10 (the register
predicted about 38); `svc-onboard` and `svc-onboard-system` started 21:58:52 and
failed 22:36:22, 10 of 10. Messages: `svc-hello-claims`: "one or more
synchronization tasks completed unsuccessfully, reason: Internal error occurred:
failed calling webhook "validate.kyverno.svc-fail": … no endpoints available for
service "kyverno-svc" (retried 10 times)." (SQLInstance, SQLDatabase, SQLUser).
`svc-onboard`: "services "svc-onboard" is forbidden: unable to create new content
in namespace svc-onboard because it is being terminated (retried 10 times)";
`svc-onboard-system`: the same for resourcequotas, configmaps and rolebindings.
**Only `svc-hello-claims` is a webhook refusal; the other two are self-heal
writing into a terminating namespace.**
*Recovery.* Kyverno scaled to 1 at 22:44:10, ready 22:44:31; **the namespace
finished terminating by itself at 22:44:51**, 41 s after scale-up, with the eight
Config Connector objects removed. The three Applications stayed Failed and
OutOfSync on the same revisions; at 22:54, six minutes later, nothing had
recovered. Manual `argocd app sync --core` at 22:54:34 (`svc-onboard-system`),
22:54:55 (`svc-onboard`) and 22:55:15 (`svc-hello-claims`), **each succeeded at
the first attempt: three syncs for three failed Applications.** The order
mattered: `svc-onboard-system` re-creates the namespace the workload
Application needs.

*The mechanism, read from source on 2026-10-27/28.* Kyverno v1.19.2 deletes its
webhook configurations on graceful shutdown only when the admission-controller
Deployment is deleted, not found, or has `spec.replicas == 0` [C, kyverno
v1.19.2 `pkg/webhooks/server.go` `Stop` → `cleanup`, `pkg/utils/runtime/utils.go`
`IsGoingDown`; kyverno#6554 closed 2023-05-25, re-confirmed in 1.19.1 on
2026-10-13]. No opt-out flag exists in v1.19.2. A rolling update or node drain
keeps `spec.replicas` above zero and should not trigger the deletion [I, not
run]. This explains attempt 1: the registered step is the scale-to-zero case.

**(x) An extra case, outside the registered test: `metadata.name` edited
inside the file** (answers `platform-config#34`). `metadata.name` in
`tenants/svc-onboard.yaml` was changed from `svc-onboard` to `svc-onboard2`,
the file not renamed; merged 23:02:49 (`b97e568`); `svc-onboard2-system`
appeared about 160 s later. At 23:12:46, with no person's step: gone were the
Applications `svc-onboard` and `svc-onboard-claims`, the AppProject
`svc-onboard`, all eight Config Connector objects (1 IAMServiceAccount, 6
IAMPolicyMember, 1 ArtifactRegistryRepository), the ResourceQuota, both
RoleBindings, the ConfigMap, and the `database-seed` and `svc-onboard`
ServiceAccounts. Left in namespace `svc-onboard`: the Namespace (`Prune=false`,
PruneSkipped), and the Deployment, Service, ReplicaSet and Pod, now owned by
nothing. The old `svc-onboard-system` was not pruned and now tracked
`svc-onboard2`'s objects too, with SharedResourceWarning ("AppProject/svc-onboard2
is part of applications argocd/svc-onboard2-system and svc-onboard-system").
The tenant had no claim, so no durable object was at stake; with one, the
abandon annotation would have kept the cloud instance and the Kubernetes
objects would have gone the same way. Restore: reverted 23:12:57, old objects
back after about seven minutes with new UIDs; then `argocd app delete
svc-onboard2-system --core --yes` (23:19:49) **cascaded into the re-created
`svc-onboard` namespace as well**, because the two Applications shared
tracking; self-heal re-created it; the leftover `svc-onboard2` namespace was
deleted by hand. The 2026-10-23 review had found the same route in the
render ("An in-place metadata.name edit prunes the old tenant") and called it a
coverage gap in C-33's text, not a falsification. [C, `08-x-rename-inside-file.log`]

**Not run.** The engine's finalizer updates during an outage (no engine, as
registered); Kyverno's auto-sync was never turned off (not needed);
`systems` and `platform-config` CI.

## Claims graded

Every grade is the delegate's decision on the data, dated 2026-10-29, for the
maintainer to confirm at the close. S3 has run. C-31 waits on the operator's
desk test; C-32 is UNTESTED.

### C-26 — The tenant files survive the engine

**Grade: HELD** (2026-10-31; Session A reconstructed from audit logs, live
notes lost). Cell words: "the two M2 files drove Config Connector with no
change for the swap (non-comment bytes identical; `svc-ledger.yaml`'s comments
rewritten 2026-09-21); the third tenant was one file, running in 1m49s–1m54s
(one stock-nginx sample, poll phase favourable); same cloud set, seven
cluster-side differences each way, each engine machinery or a named
ADR-0017/0019 decision; the leftover `managed-by: crossplane` label and
"Composed by the System XR" descriptions were rewritten by the engine's S3
updates."


The claim: "The files in `systems/tenants/` drive the new engine without edit
and materialize the same tenant surface; a new tenant is still one file." The
two files M2 ran drove Config Connector at Session A with no change made for
the swap, and the third tenant was one 25-line file with nothing else changed
in `systems`, `platform-config` or `platform-bootstrap`. It was running in
1m49s to 1m54s. The object-by-object diff is done: the cloud side is the same
set, and in the cluster seven differences each way are classed as engine
machinery or as following a named ADR-0017/0019 decision (C-26 Data). Two of
them touch what a team can see or do: the `<name>-crossplane-edit` binding is
gone and `cnrm-admin` aggregates into `admin`, and the `privateIp` key left
ConfigMap `main-connection`. Neither makes the tenant files drive the engine
differently, and neither is a failure of the claim, but they are visible
consequences of the swap and are recorded here.
**What keeps this from being cleaner:** the Session A data are reconstructed
from audit logs, and the live notes were lost. "Zero bytes" is true of the
swap and not of the file: `svc-ledger.yaml`'s comments were rewritten on
2026-09-21 (non-comment lines identical). The merge-to-workload figure is one
sample on a stock nginx whose poll phase was favourable; it is not evidence
that the engine is faster. Two live observations were open before S3 and are closed: the cloud label
`managed-by: crossplane` and the service account descriptions "Composed by the
System XR" survived on live objects. Both were fixed by the engine's S3 updates
(05:52:33Z for the labels, three `PatchServiceAccount` calls for the
descriptions; C-26 Data). No counter-evidence was found.

### C-27 — Ownership still moves with a YAML edit

**Grade: HELD** (2026-10-31; reconstructed). Cell words: "1 file, 3
re-created per move; project policy agreed at 236.0 s and 196.2 s (M2 1m55s;
232.5 s of it is Argo CD's poll); the broken rule failed loudly at the API
server ("member is immutable", six retries) and the restore did not clear the
retrying sync, an operator patch did, 777 s later; repository policy and
Application status not readable from the record, so "healthy while the cloud
disagreed" is not determinable."

The claim: "Changing `spec.owner.team` re-creates the three team-bearing cloud
grants through Argo CD apply-and-prune, and the cloud agrees"; and a broken
naming rule "fails loudly, as a failed sync". Files touched 1; resources
re-created 3, both moves; the project policy showed the new team and not the
old at 236.0 s and 196.2 s after the merge.
**"Fails loudly, as a failed sync":** six API-server refusals naming the field,
plus 777 s of failing retries that an operator patch of the Application
cleared, are a failed sync. The Application's own status is inferred (the audit
log has no record of it).
**The misses, stated:** 236.0 s is slower than M2's 1m55s, but 232.5 s of it
is the time before Argo CD acted, and the cloud write followed the delete by
about 3.5 s. The restore (`platform-config#30`) did not heal the failure: the
retries kept failing for 777 s; this is outside the claim but it is what
"loud" costs. The repository policy could not be read at any time, so "the
cloud agrees" is exact for the project policy and only bracketed for the
repository: the last write came 1049.1 s after the move. **"Healthy while the
cloud disagreed" is not determinable** (Application status is not logged), so
M2's baseline "yes" has no comparison. The shared-grant hazard of M2
(surprise 14) reproduced: `svc-ledger` and `svc-onboard` lost checkout's two
Cloud SQL roles for 69.1 s and 261.4 s, until Config Connector's periodic
re-reconcile re-added them. It is the subject of C-24 at M3 and does not count
against C-27.

### C-28 — Guardrails replace review without a custom API

**Grade: ADJUSTED** (2026-10-31). Superseding ADR:
[ADR-0022](../adr/0022-what-the-m2b-build-changed.md) §1–2. Cell words: "two
gate defects fixed in the build (`62b9451`: updates admitted; `5da7d4a`: the
garbage collector refused); after them every registered refusal held, S3 as a
team member included; a `/status` write is admitted by RBAC and unseen by
Kyverno, accepted in ADR-0022 §5; PR→usable 13m59s, no manual command found;
the instance survived its prune, adopted in 119.6 s; the
`NamespacedMutatingPolicy` hole was found by the 2026-10-23 review, not by this
test, and closed by `a9e2bdd` (RBAC door tried in S3, AppProject door not)."

*Four parts, as the template requires.*

**The prediction as registered and how it failed.** C-28 has no "Falsified if";
its test says each listed change "is refused", "deletion protection holds",
and (c) the instance survives and is adopted. The reality gate, the part that
refuses the `kubectl` path, had two defects. First, until `62b9451` it
admitted an edit of a cloud object: ADR-0017 §7 says Kyverno denies "create,
update and delete", and the rehearsal on 2026-10-06 found the update was let
through (changing a database's machine tier, or adding a label), because both
rules said `allowExistingViolations: true`. Second, `5da7d4a`: after that fix
the update rule refused every identity but Argo CD and the engine's two
writers, including Kubernetes' garbage collector, so in Session A four pruned
IAMPolicyMembers stayed Terminating after their cloud grants had gone. This is
the shape of C-07's miss at M2: the gate admitted, or blocked, the opposite of
what it exists for.

**The revised mechanism.** Both rules say `allowExistingViolations: false`,
and `scripts/check_charts.py` gained a ninth check that every deny-without-
conditions says so (fails on the old policy, passes on the new). The update
rule's exclusions now include `system:serviceaccount:kube-system:generic-garbage-collector`,
which is the name GKE presents. ADR-0008 requires a superseding ADR for
ADJUSTED; it is [ADR-0022](../adr/0022-what-the-m2b-build-changed.md) §1 (the update rule's exempt
identities, superseding ADR-0017 §7's list for UPDATE) and §2 (the
`allowExistingViolations: false` rule).

**What the revised mechanism was observed to do.** (a) 13m59s PR to usable
against 17m45s, no manual command found. (b) Seen refused: the three AppProject
cases by name; the size-L-critical-in-a-standard-System claim in the claims
Application's status; 13 `kubectl` creates, patches and deletes of three kinds
at admission, each with a message that says what to do instead; and, in the
render fixtures, every claim-schema case. (b2) A StorageBucket admitted by the
API server and never actuated: six `403`s, nothing created. (c) The instance
survived the prune and was adopted in 119.6 s with no cloud write.

**What S3 settled, and what stays unresolved.** The `kubectl` cases ran as a
team member in S3 (C-28 Data): create, edit and delete refused, a
`NamespacedMutatingPolicy` create refused by RBAC, the `/status` write
admitted, and seven policy reports readable, each `PASS 1`. The
abandon-annotation removal and change ran as the operator, because a team
member's edit is refused on the first rule; both rules fired, the operator
holding no exemption. The (b2) byte-identical check was
run at the desk on 2026-10-29: identical. The claim-schema refusals also ran
through a real PR, svc-hello#19, ten cases, each refused by the check. What a
developer can read of a refusal in Kyverno's policy reports is not recorded.
The `/status` of a cloud object is not sent to Kyverno at all (`62b9451`'s
body: the webhook is registered for resources, not subresources); ADR-0022 §5
accepts that gap and states it; S3 showed the admission (RBAC `can-i` says
`yes`, and the patch returns `patched`).
Outside the registered test, the 2026-10-23 review showed offline that a tenant
could write a Kyverno `NamespacedMutatingPolicy` and rewrite what Argo CD
applies, which defeats this claim's boundary without a node escape; `a9e2bdd`
(merged 2026-10-27 19:56Z, Kyverno 1.19.2, `policies.kyverno.io` added to the
AppProject blacklist, `rbac.roles.aggregate.admin: false`) closes both doors; the RBAC door for a `NamespacedMutatingPolicy` was tried in
S3 (refused), and the AppProject-blacklist door for `policies.kyverno.io` was
not tried. **That defect was found by an
outside review, not by this claim's test**, and Session A ran against the
unfixed platform.

*S3 outcome.* Two cases were added to the S3 script: a team member's patch of
`svc-hello-main`'s `/status` (ADR-0022 §5) and a team member creating a
`NamespacedMutatingPolicy` (`a9e2bdd`). Every refusal the register lists held,
with the abandon-annotation cases run as the operator; the `NamespacedMutatingPolicy`
create was refused by RBAC; the `/status` write was admitted, and ADR-0022 §5
accepts it. Nothing was admitted that the design cannot refuse, so
ADJUSTED stands with the two fixes recorded and no third. The `/status` case
does not change the grade; it decided ADR-0022 §5's wording.

### C-29 — The cutover is a rebuild

**Grade: HELD** (2026-10-31). Cell words: "forward leg (reconstructed),
reverse leg (S1) and return build (S3): every durable resource adopted, none
re-created, no grant absent, no manual intervention recorded in the legs; `up`
35m06s (S1), 32m40s (S3), M2 35m31s. `postgres` refused the rebuilt
`main-admin` Secret, as ADR-0017 §8 predicted: the break-glass password is
stale after a rebuild (ADR-0016 §4), the reset is a person's `gcloud` step
(ADR-0018 §2), the application path is unaffected. The draft's 2026-10-27 rule
"refused → ADJUSTED" was mis-specified and is withdrawn in the entry, not
applied. `down` does not park (Surprise 16), and S3's `down` needed one re-run
(Surprise 18)." Surprise 17 records a team grant absent for 156 s to 176 s,
during S3's offboarding steps, which come after the legs.

The claim: "Switching engines costs one `down` and one `up`, re-creates nothing
durable, and is reversible the same way." Forward: seven durable resources
checked (one instance with its database and user, two repositories, two service
accounts), **none re-created**, adopted by update calls; no grant absent from the
project policy; no permission failure; no non-durable resource refused.
Return build (S3): `up` in 32m40s, five durable resources adopted and none
re-created, 13 of 13 Applications Synced and Healthy, no grant absent.
Reverse (S1): `up` in 35m06s, five durable resources "adopted 5, recreated 0"
with unchanged creation times, 11 of 11 Applications Synced and Healthy,
**no grant absent in 166 ten-second samples**, `down` in 12m00s.
**What keeps it from being clean:** the forward leg is reconstructed, its `up`
wall-clock has no definition, and its manual-intervention count is "none found"
from a log that cannot see the jump box. The reverse-leg intervention count is
the absence of a recorded one. S1's durable list does not include the service
accounts. The desk subset check the register required before the first leg
has no pre-leg record. One failed `postgres` login on `svc-hello-main` at
22:31:29.746 is not determinable from the log (see the [I] below). **`down` did not park:** `svc-hello-main` ran
unparked from 20:36:53Z to 23:52:42Z, until the operator ran `cycle.sh park`
by hand (C-29 Data).

**The `psql` login (S3, 06:08:20Z) was refused:** `FATAL: password
authentication failed for user "postgres"`. That is the outcome ADR-0017 §8
predicted, for the one update observed (its logged body shows no
`rootPassword`, and audit logs may redact it). The consequence was decided
before S3: the break-glass `postgres` password is stale after a rebuild
(ADR-0016 §4), its reset is a person's `gcloud sql users set-password`
(ADR-0018 §2), and the application path uses IAM auth and is unaffected. The
stale Secret stays in the cell.

**2026-10-29: the rule written on 2026-10-27 is withdrawn, not applied.** This
section's earlier text read: "The rule: accepted → HELD; refused → ADJUSTED,
with ADR-0022 §6." It was backwards, and the texts it was written from, all
earlier than S3, show it. ADR-0017 §8 says that the generate-once seed
"never changes the live password" is "[I until C-29]": a refusal is the
outcome the ADR predicted. ADR-0016 §4 already says "The break-glass password
is stale after a rebuild" and that "the application path is unaffected (it
never uses a password); the break-glass path needs one `gcloud sql users
set-password`". ADR-0018 §2 puts the reset with a person's `gcloud` command.
So the consequence of a refusal was decided before S3, and it is not a design
change. The rule sent the predicted outcome to ADJUSTED and the outcome the
ADR did not expect (the new Secret accepted, the live password changed) to
HELD. It is withdrawn here openly. The grade is made on the claim's text:
nothing durable was re-created, and the legs are reversible the same way.
Optional reading, not relied on: Session A's one failed login at 22:31:29 is
consistent with the GRANT Job's password attempt [I].

### C-30 — The new stack is smaller

**Grade: HELD, scoped** (2026-10-31). Cell words: "falsifier not met: concepts
64→56, 59 with ADR-0018's three additions. Pods 11→10 (six of the ten are
webhook replicas set by the operator's autoscaler; manifest figure 6), working
set 1196→971 Mi, engine CRDs 114→273, more as predicted. Missed: "fewer recipe
lines" — recipe 633→656, recipe+schema+policy 1074→1192 (+11.0%) on the narrow
reading fixed 2026-10-28, chosen as the reading less favourable to the
prediction. Beside the grade, unregistered: CPU 73→217 m, one `kubectl top`
sample each."

Prediction as registered: "fewer pods, fewer recipe lines — and more CRDs, by a
wide margin. Concepts: still fewer on the new engine, by a smaller margin than
the swap alone would give." "Falsified if ... 'less of the reader' ... the
total with them is not lower than Crossplane's."
**The rule, as written before S3.** HELD only if S3 shows pods fewer AND memory
(working set, same basis as S1's 1196 Mi) lower AND the concept total with the
additions (59) lower than Crossplane's (64). WRONG if S3's memory is higher, or
if a disputed concept item, resolved, makes 59 reach 64 (it cannot by the listed
swings; C-30 Data). All three conditions are met. The cluster half carried an
ADR-0017 warning in its own Consequences: Config Connector "may well cost more
than it saves on a small cluster." On memory it did not.
**What is in.** Crossplane 11 pods, 1196 Mi, 114 engine CRDs; Config Connector
10 pods, 971 Mi, 273 engine CRDs, measured in S3 on `e8eedeb`, the counted
commit, so no line re-run is needed. **The pod margin is one pod**, and six of
the ten are `cnrm-webhook-manager` replicas set by the operator's autoscaler;
the manifest figure is 6. The CRDs rose from 114 to 273, by a wide margin, as
predicted. Concepts: Crossplane 64, Config Connector 56, 59 with ADR-0018's
additions. **The line half of the prediction missed:** recipe 633 to 656, and
recipe + schema + policy 1074 to 1192 (+11.0%), under the narrow reading fixed
on 2026-10-28, after the 2026-10-23 desk count had been seen, and chosen as the
reading less favourable to the prediction. It held under the wide readings,
which are ungraded. A line-count miss alone does not decide the claim (the
falsifier names only the reader).
**Beside the grade, unregistered:** CPU rose from 73 m to 217 m. Each is one
`kubectl top` sample, taken in different phases (S1 mid-`up`; S3 21 s after
verify). It is reported ungraded and is not buried.

### C-31 — The engine runs on narrow roles; what they leave out is the third lock

**Grade: HELD** (2026-10-31). Cell words: "Apply B 05:29Z with the cluster down; the
identity holds exactly the five grants (project policy; Policy Analyzer at the
organisation, five bindings, all on the project, impersonation analysis
included); return build adopted 5, re-created 0, 13/13 Healthy, no engine
permission error; the first project grant passed the condition on the custom
role alone (06:19:15Z); raw registry and database deletes refused by Google
(403), objects stuck on the finalizer until the abandon annotation, 13 s, as
predicted — the registry error named the permission, the database error did
not; scratch tenant onboarded from one file to a usable database in 12m28s,
with two manual interventions caused by reusing another tenant's repo, then
offboarded, name on the retired list; desk script: self-edit and `roles/viewer`
refused (2026-10-31). Prediction missed in the favourable direction: zero missing
permissions found by what ran. The subset check was first recorded after both
legs, not before the first. The engine deleted `svc-onboard`'s and
`svc-scratch`'s service accounts on offboarding, which ADR-0018 §2 keeps on
purpose and ADR-0015 §1 does not count as durable."

What decided it: the desk test (C-31 Data, third run). Its must-succeed list
passed, and all six must-be-refused items were refused. One refusal is narrower
than the claim's wording: the edit to its own role stopped at `iam.roles.get`,
before any update call. The engine holds no `iam.roles.` permission in any of
its grants (Policy Analyzer, S3), so the update would be refused too [I].

What is in: the subset check; the audit scan (none found among the writes the
admin-activity log can see, except the control's delete); the control (a cloud
delete succeeded under the broad roles); Apply B applied with the cluster down
(05:29Z) and the five-grant read; the return build; the scratch tenant; the
registry and database deletes refused by Google; the desk script.
*HELD* if S3 shows all of the following. The engine identity holds exactly the
five grants (met). The desk script passes its must-succeed list and is refused on
every must-be-refused item (met, 2026-10-31). The return build adopts all and
re-creates none (met). A scratch tenant onboards from one file and offboards
(met; two manual interventions, caused by reusing another tenant's repo). The raw
registry and the raw SQLDatabase deletes are refused by Google and the cloud
objects remain (met). Any permission errors are tied to a denied call in the
audit log and fixed by a `platform-roles` change that stays inside the list
(none occurred). Per the register's prediction, "the refused deletes leave
their objects stuck until the annotation is added" is the expected form, not a
requirement; it occurred.
*WRONG* if any one of the register's "Falsified if" items occurs: the engine
identity deletes a durable resource; an adopted resource is re-created; the
golden path needs a permission the design forbids (a durable delete, anything
under `iam.roles.`, an unconditioned `projects.setIamPolicy`); `roles/viewer`
appears in the project policy; the engine's edit to its own role succeeds; or
a failure can neither be tied to a denied call nor reproduced with the four
broad grants restored. None of these was observed. The self-edit and the `roles/viewer` grant were
both attempted on 2026-10-31 and refused.
*Not a falsification, but recorded as a miss:* the prediction "one or two
permissions are missing on the first try" missed, in the favourable direction:
zero missing permissions found by what ran: the admin-activity scan (blind to
reads and to `cloudsql.databases.create` and `cloudsql.users.create`), the
return build, the scratch tenant's onboarding (an instance create, a database,
a user, service accounts, project IAM) and offboarding (a service-account
delete), and the desk script's must-succeed list, if run. Zero found is not a
proof that 21 permissions suffice.
**What S3 cannot show.** The register's own limit and ADR-0018 §5's apply:
the lock binds only the engine identity, not a human Owner, and not an engine
under hostile control; `iam.serviceAccounts.setIamPolicy` is project-wide and
unconditioned in `platformEngineServiceAccounts`, which the 06:51:55Z Policy
Analyzer read shows (Built). The 2026-10-23 review's point, that the legacy
Crossplane identity still holds the broad roles and that the README's "cannot
delete a database or a registry, whatever happens in the cluster" is stated
unconditionally, bears on how HELD is worded, not on whether the test passes.

### C-32 — Rebuild cost is measured, not estimated

**Grade: UNTESTED, not WRONG** (2026-10-31). Cell words: "the test did not
run: the only billing export is keyed to billing account 015588-C94B87-3A50AF
and holds no rows for the project, which bills to 0174CB-E6957B-042C42; the
2026-08-28 M1 follow-up was never done. Read literally both "Falsified if"
clauses are met; they are read as written for an export that exists, and
ADR-0009's $0.50/h stands as a published, unmeasured estimate. No figure in the
entry. Graded by a dated post-close addendum if an export on 0174CB… returns
rows for 2026-10-19/20, 10-27 and 10-29." The backfill is [unverified].

The register: "Falsified if the export cannot be tied to a session — no
session's spend can be isolated from it — or if a published cost figure still
has to come from an estimate after M2b closes." **Read literally, both clauses
are met.** No session's spend can be isolated, because there is no export for the
project. ADR-0009's roughly $0.50/h is a published figure that still comes from
an estimate after the close. **The reading:** the falsifier presupposes an
export that exists and is keyed to the project. The instrument was never
enabled, so the test did not run and has no result to falsify. The grade reads
the clauses as written for that case, and says so openly so the maintainer can
overrule it. UNTESTED is the word for a test that did not run; WRONG would grade
ADR-0009's estimate on data that does not exist. The prediction (an hourly rate
within an order of magnitude of $0.50; a parked day costing little but not
nothing; the largest surprise a session left running) was not tested. One piece
of it did occur and is uncosted: `svc-hello-main` ran unparked for 3h15m49s
after S1 (C-29 Data; Surprise 16), and S3's abandoned `svc-hello-second`
stayed running until it was parked (C-31 Data). If an export is enabled and
returns rows for the dates, the grade is derived from data by a dated
post-close addendum, as C-02's was.

### C-33 — A removed, renamed or broken file deletes no tenant by itself; a Kyverno outage recovers with one manual sync per failed Application

**Grade: HELD, scoped** (2026-10-31), with the cell words:
"(a)–(d) held in every registered case; (e) held only under a deviation
(webhooks re-applied after scale-down) — the registered step fails open,
because Kyverno deletes its webhooks when its Deployment is scaled to zero
(Surprises); (x) an in-place metadata.name edit deleted the old tenant's
Applications and Config Connector objects, outside the registered cases, a
counter-example to the headline; systems' [name-matches-file] check would have
been red, is not a required check, and did not run; ADR-0022 §3–4. Kyverno
1.19.2 and systems CI not rehearsed."

The claim has two halves. **(a)–(d):** "None of these deletes a tenant's
Application, namespace or claim objects without a person's step." Held in every
registered case: the ApplicationSet file removed and renamed, deleted with
`kubectl` under the background cascade, the chart dropping the Namespace;
`tenants/` renamed; a tenant file removed (nothing until the person's
`argocd app delete`, then exactly the one tenant); the last claim removed and
`claims.yaml` misnamed (both "auto-sync will wipe out all resources", the manual
prune completing the removal); a tenant file with a field missing, in two forms.
The prediction that the tenant with no `claims.yaml` reports Synced and Healthy
held. **(e):** the register's step does not do what its prediction assumes.
Scaling the admission controller to zero removes Kyverno's resource webhooks,
so no write is refused (attempt 1, which by the register's rule does not
count); Kyverno's source shows why (C-33 Data (e)). Attempt 2 is a deviation:
the webhook configurations were re-applied after scale-down to approximate a
crashed Kyverno. Under it the prediction held in both parts: the namespace
finished terminating by itself 41 s after Kyverno returned, the three blocked
Applications stayed Failed on the same revision until a manual sync, and each
took exactly one (three for three). The retry block ran 37.5 minutes, against
about 38 predicted. A deviation is not the registered test, so (e) is a recorded
deviation: it supports the prediction only for an unreachable Kyverno whose
webhooks remain registered. For a clean scale-to-zero the observed behaviour is
the opposite of the prediction: nothing is blocked and the gate is open
(Surprise 4).
**Case (x) is not in the registered cases and the register's "Falsified if"
names only (a)–(d) and (e).** By that text the claim is not falsified. By its
headline ("a removed, renamed or broken file deletes no tenant by itself") the
in-file rename is a counter-example the guards do not cover, and the grade
carries it as scope: ADR-0022 §4 reads the headline with that exception, and the
choice of guard is `platform-config#34`'s. The 2026-10-27 issue comment on
`platform-config#34` says (x) "does not change C-33's grade"; the scoped cell
records the counter-example beside the grade.
**A consequence, not decided here:** the fail-open finding needs a dated M3
readiness-note entry before C-11's first cluster case, adopting a remedy or
accepting the window with the fact stated (ADR-0022 §3).
Not tested: the engine's finalizer updates during an outage; `systems` CI (so
ADR-0019 §3's retired-name check has no refusal on record); Kyverno 1.19.2, now
on `main`, which the rehearsal did not run.

### C-25 — note at M2b's close (the register asks for it)

C-25 ("A provider release with the sql.User fix removes the manual database-user
step", ADR-0016 §4) has lost its subject: ADR-0017 retires the provider.
**Proposed scoreboard cell:** "UNTESTED — subject removed by ADR-0017 (M2b entry,
2026-10-27); not an M4 claim; not graded." One observation, not a grade: on the
engine that replaced it, the IAM database user `svc-hello-second-iam-user` was
created by `config-connector@` at 22:46:10.133Z (Cloud SQL `CREATE_USER`,
completed 22:46:10.272Z) with no manual command found in the operator's
audit entries, so the manual step C-25 was about does not exist on Config
Connector. The README's milestone table still lists C-25 under M4.

### C-03 — note at M2b's close

C-03's carried-forward hands-on checks, a GCS bucket and Cloud DNS records,
**never ran on the provider C-03 names** and now will not: ADR-0017 retires it
from the reference path. Recorded here as never tested by hand, not carried to
M3. The grade cell stays **HELD** for the kinds M2 composes (unchanged); add
"GCS bucket and Cloud DNS records: never hands-on created; not carried forward
(M2b entry, 2026-10-29)". Two things that look like tests and are not:
`provider-gcp-storage` and `provider-gcp-dns` were installed and Running on S1
and in Session A's cycle 3, but no Composition composes either kind
(`crossplane/compositions/` holds `system` and `database` only); and the
`StorageBucket` of C-28 (b2) is a Config Connector kind, denied and never created.

### C-08

C-26 tests C-08's idea on the new engine. C-08 itself stays UNTESTED; the M3
readiness note pre-commits it to NOT REACHED at M3's close.

### What the grades add up to

Grades, 2026-10-31: HELD C-26, C-27, C-29 and C-31; HELD, scoped C-30 and C-33;
ADJUSTED C-28, via [ADR-0022](../adr/0022-what-the-m2b-build-changed.md); UNTESTED C-32 (the test did
not run). Four of
the grades with data rest in part on reconstructed Session A data (C-26, C-27,
C-28, C-29's forward leg), and C-33's HELD carries a deviation and a
counter-example in its scope. C-29's draft rule of 2026-10-27 was
mis-specified and is withdrawn in its entry. The honest summary is that the
engine swap worked in the places it was tested, that the build found two gate
defects, one unguarded deletion route and one fail-open Kyverno shutdown that
the design had not, that the new engine is smaller on pods, memory and
concepts and larger on CRDs and lines, and that the one measurement the
experiment promised about money cannot be made yet.

## Research verified

[unverified] items resolved this milestone.

- **ADR-0019's five unverified items (C-33):** a renamed ApplicationSet leaves
  the old one's Applications alone and the new set Degraded, "already owned by
  another ApplicationSet controller systems"; the ApplicationSet's finalizer
  keeps its Applications through a background delete (a3's stricter variant); the
  Namespace's `Prune=false` holds through a chart change and does not block
  offboarding (a4, b2); an Application empty from the start is Synced and
  Healthy; a Kyverno outage blocks, and recovers, as the Prediction says,
  but only when the webhooks stay registered (Surprise 4). [C, c33 logs]
- **ADR-0019 §1's reading of Argo CD v3.4.6** (the per-ApplicationSet policy is
  honoured when the controller sets none; `create-update` never deletes; a
  failing render is not auto-synced; the "auto-sync will wipe out all resources"
  guard): each matched the observed behaviour in the rehearsal. [C]
- **ADR-0017 §10:** an edit to an IAMPolicyMember's member is refused by the API
  server: "spec.member: Invalid value: ... member is immutable". [C, Session A,
  23:25:50.579]
- **ADR-0017 §7 [unverified]: which identities the garbage collector presents on
  GKE.** `system:serviceaccount:kube-system:generic-garbage-collector`
  (commit body of `5da7d4a`, from Kyverno's log). The namespace controller's
  identity on GKE is still not recorded, but the delete rule's need is
  resolved by S3: the `svc-onboard` namespace, under the gate, terminated by 06:17:11Z, and so did `svc-scratch`'s by 06:34:53Z
  (`evidence/c31/a-offboard-svc-onboard.txt`, `a-offboard-svc-scratch.txt`);
  on k3s the namespace finished terminating under the delete rule in every case.
- **ADR-0017 §7: the deletion defender removes its finalizer with an update.**
  Observed: status 10 (conflict) once per object, then status 0, at 02:54:09.
- **ADR-0017 §12:** `cycle.sh` records the revision it built from ("root at
  crossplane (dd6730c)"). [C]
- **ADR-0017 §8's [I] on `rootPassword`, resolved by S3 for the one update
  observed.** `postgres` refused the rebuilt `main-admin` Secret (06:08:20Z);
  the live password was left alone. The logged update body shows no
  `rootPassword`, and audit logs may redact it. [C, `evidence/s3-c29-psql.txt`]
- **ADR-0018 §5's [I], resolved by S3:** a refused delete fails stuck, with
  condition `DeleteFailed`, and the abandon annotation releases it, 13 s. [C,
  `evidence/c31/b-part1.txt`, `b-part2.txt`]
- **ADR-0018's [I] that the IAM condition passes for the custom role alone,
  resolved by S3:** the first project grant on the custom roles, at
  06:19:15.549Z, was inside `hasOnly(['roles/cloudsql.client',
  'roles/cloudsql.instanceUser'])` and succeeded. [C,
  `evidence/c31/first-project-grant.json`]
- **ADR-0019 §5, resolved by S3:** the Policy Analyzer read works, at
  organisation scope, with the operator's access, plain (06:00:40Z) and with
  `--expand-roles --analyze-service-account-impersonation` (06:51:55Z). [C,
  `evidence/c31/policy-analyzer-1.txt`, `policy-analyzer-2.txt`]
- **Still unresolved:** ADR-0017 §7's [I] that labels stripped by hand come
  back at the operator's next reconcile (not tested); `roles/viewer` refused to
  the engine, which stays [I] until the desk test runs (C-31).

## Surprises

1. **The garbage collector was refused its own cleanup (`5da7d4a`).** Argo CD
   prunes in the foreground, so every pruned object carries a
   `foregroundDeletion` finalizer, which Kubernetes' garbage collector removes
   with an UPDATE. After `62b9451` the update rule refused every update but Argo
   CD's and the engine's, so four pruned IAMPolicyMembers stayed Terminating
   after their cloud grants had gone. ADR-0017 §7 had seen the delete rule's
   need and not the update's. Found in Session A, fixed live ("Approved by Ronak
   in Session A").
2. **The gate let an edit through (`62b9451`).** Found on 2026-10-06 in the local
   rehearsal, before any cluster: a deny with no conditions plus
   `allowExistingViolations: true` admits every update, because Kyverno judges
   the old object too and lets through what "was already refused before". A
   create has no old object and a delete is not an update, which is why only
   edits slipped. The M2 pattern (C-07: the gate admitted the exact thing it
   exists to deny) repeated on the new engine.
3. **Google Compute stock-outs.** Cycles 1 and 2 on 2026-10-20 each failed at
   cluster creation, code 14, "Try a different location, or try again later:
   Google Compute Engine does not have enough resources available to fulfill
   request: us-central1.", after 223.0 s (03:16:55 to 03:20:38) and 212.3 s
   (03:23:00 to 03:26:32). Cycle 3 succeeded; from the first create call to the
   third's completion is 37m24s against 7m20s for a normal one. After cycle 2
   failed the delete call came 1132 s later; why is not in the log [I: the
   operator was elsewhere]. Nothing ran, so no tenant was touched. S1 on 2026-10-27
   did not hit it. It is a hazard for any session on one zone, not a finding
   about the engine.
4. **Scaling Kyverno to zero removes its webhooks; the gate fails open.** On
   shutdown Kyverno deletes its resource, policy and exception webhook
   configurations. Nothing was refused: a namespace delete and a claim change
   went straight through (C-33 (e) attempt 1). ADR-0017's Consequence, "an
   outage now blocks every write to the gated kinds", is true only if the
   webhooks stay registered; a clean stop makes the gate disappear, and with it
   the reality gate for the window. ADR-0019 §6's two remedies address the
   blocked case. Source read on 2026-10-27/28 explains it: Kyverno v1.19.2
   deletes its webhook configurations on graceful shutdown only when the
   admission-controller Deployment is deleted, not found, or has
   `spec.replicas == 0` [C, `pkg/webhooks/server.go` `Stop` → `cleanup`,
   `pkg/utils/runtime/utils.go` `IsGoingDown`; kyverno#6554 closed 2023-05-25,
   re-confirmed in 1.19.1 on 2026-10-13]; no opt-out flag exists in v1.19.2; a
   rolling update or node drain should not trigger it [I, not run]. The finding
   needs a dated M3 readiness-note entry before C-11's first cluster case
   (ADR-0022 §3).
5. **Argo CD's self-heal did not restore a scaled-down Kyverno.** The chart
   renders `replicas:` empty, so `spec.replicas` is not Argo CD's to restore. The
   register's "repeat with auto-sync off" branch was not needed.
6. **Renaming `metadata.name` inside a tenant file deletes the old tenant (case
   (x), answers `platform-config#34`).** The old tenant's Applications, AppProject
   and Config Connector objects went with no person's step; the Namespace and
   the workloads survived, owned by nothing; and the two Applications shared
   tracking, so cleaning up the new one cascaded into the restored old one. The
   ADR-0019 guards cover the file's name and presence, not the identity written
   inside it. The 2026-10-23 review had predicted it from the render; the
   rehearsal observed it. A fix was proposed there (`guard.yaml` fails unless
   `.Values.metadata.name == .Release.Namespace`) and is not decided here.
7. **A restored rule does not un-stick a sync that is already retrying.** After
   `platform-config#30` the retries kept failing against the broken state for 777 s
   until the operator patched the Application. The failing patch also left the
   registry writer's repository policy on the old team for about 14 minutes.
   **Second occurrence, S3:** `svc-scratch`'s sync kept retrying at the revision
   it started with (`7b0bc6a`, not `main`'s `f3742c6`) after its cause was
   fixed, and the operation was terminated by hand at 06:27:37Z (C-31 (a)).
8. **Argo CD's poll is most of every latency.** Merge to first Argo CD action: 34.6
   s (C-26), 91 s (C-28 a), 118.8 s (C-28 c), 192.5 s and 232.5 s (C-27), and
   1 to 5 minutes in the rehearsal. The engine's part is seconds. The register's
   comparisons with M2's numbers (4m08s, 1m55s, 66 s) are comparisons of poll
   phases as much as of engines. A webhook from the repos would remove the
   noise; the evidence records none.
9. **The shared-grant hazard of M2 recurs on the new engine.** Moving one team
   removed a Cloud SQL role that two other Systems still wanted, for 69.1 s and
   261.4 s, and Config Connector re-added it on its periodic re-reconcile, not
   in reaction to the removal. C-24 (M3) is the registered answer.
10. **Reverting an ApplicationSet rename leaves the stray set** (`Prune=false`
    holds it); a person deletes it. This is the guard working, with a cleanup
    step the runbook does not list.
11. **The instrument for Session A was the log.** Its live notes were lost; the
    reconstruction recovers writes to the millisecond and cannot recover
    reads, Application status, repository-policy contents or jump-box commands.
    The ten-second poller exists since 2026-10-27 and covers S1 and what follows.
    A claim's test is a requirement on the instrument (the M1 pattern, surprise
    9) and this is its fourth instance, after C-23's logs, C-02's spend and
    C-32's export.
12. **`cycle.sh` left gaps the evidence has to fill.** Session A produced no
    `cycle-results.tsv` row. Today's rows label the `up` "cycle 6" and the
    `down` and `park` "cycle 7" because `cycle.sh` numbers cycles from the TSV;
    September's cycle 6 had a `down` and a `park` and no `up`, so today's `up`
    continued it. `down` did not park, and the `park` row came by hand 3h15m49s
    later (Surprise 16).
13. **The audit-versus-files scan cannot see two of the permissions it
    checks.** `cloudsql.databases.create` and `cloudsql.users.create` do not
    produce admin-activity entries; a scan for "write permissions the files
    lack" is blind to exactly the Cloud SQL sub-resources.
14. **A tenant could write the platform's policy** (found by the 2026-10-23
    review, fixed by `a9e2bdd`): the Kyverno chart's default aggregated
    `create/update/delete` on Kyverno's namespaced policy kinds into `admin`,
    which every team holds, and the AppProject blacklisted `kyverno.io` but not
    `policies.kyverno.io`. Kyverno 1.19.1 also carried an unpatched Critical
    advisory (GHSA-jcw5-3wcg-fqx5). Not found by any registered test.
15. **A record written in the past tense before the event.** `platform-roles`
    `b766d81`'s body says "Applied with the cluster down". At merge
    (`platform-roles#4`, `08178fb`, 2026-10-27 20:37:53Z) Apply B had not been
    applied, and at 00:03Z on 2026-10-28 the engine still held its four broad
    roles plus `roles/compute.viewer`. The correction is an empty commit on
    `platform-roles` `main` once Apply B is applied (C-31 Data).
16. **`down` does not park, and an instance ran unparked for 3h15m49s.**
    `svc-hello-main` ran from 20:36:53Z (S1 `down` end) to 23:52:42Z, when the
    operator ran `cycle.sh park` by hand (67 s). It is C-32's "session left
    running" in miniature, uncosted, and it is not in the claim C-29 states
    ("one `down` and one `up`").
17. **Offboarding one System removed a grant its team's other System still
    renders, on the narrowed engine too.** This is M2's shared-grant hazard
    (M2 surprise 14, C-24's subject). On `main` the team grants carry no
    condition, so one cloud binding serves every System of the team.
    - Offboarding `svc-onboard` (team checkout) removed `group:checkout`'s
      `roles/cloudsql.client` and `roles/cloudsql.instanceUser` at
      06:16:19Z, though `svc-ledger`, also checkout, still renders both.
      `cloudsql.client` came back at 06:18:55Z, after **156 s**, when Config
      Connector re-asserted `svc-ledger`'s member. `instanceUser` came back
      at 06:19:15Z, after **176 s**, with `svc-scratch`'s grant.
    - Offboarding `svc-scratch` removed both again at 06:34:07Z. They were
      still absent at teardown, so the cloud policy holds no Cloud SQL grant
      for `checkout` until the next `up` re-asserts `svc-ledger`'s members.
    - The poller (10 s) saw the same windows. `svc-ledger` has no database,
      so nothing broke.

    This is the baseline C-24 measures in M3, with the per-instance
    condition of ADR-0021 §14 [C, the engine's `SetIamPolicy` binding deltas;
    `evidence/iam-poll.jsonl`].
18. **S3's `down` failed at its first step, and `park` ran before the cluster
    was gone.** The run was `cycle.sh down` followed by `cycle.sh park`.
    - **The failure (06:36:40Z).** `down` failed after 40 s:
      "helm_release.argocd: Error uninstalling release … failed to delete
      release: argo-cd". Every Argo CD workload was already deleted, and
      Helm's release record was left in status `uninstalling`. Kyverno's
      admission log shows no refusal in that window. The cause is not
      determined.
    - **The park.** `park` ran next, against a cluster that still had Config
      Connector running and three instances. It stopped `svc-hello-second`,
      `svc-hello-main` and `svc-scratch-main` (55 to 56 s each). With the
      engine still up, its `activationPolicy: ALWAYS` could undo the park on
      `svc-hello-main`.
    - **The intervention.** `down` was re-run by hand (06:40:32Z to
      06:50:11Z). The instances were read STOPPED/NEVER right after it (and
      again at 07:02:18Z, `evidence/s3-after-down.txt`), so the early park had
      held and no second park was needed. `cycle.sh` labels the re-run as a new
      cycle (9), because `down` takes the next number. It is cycle 8's
      `down`.

    The failed `down` is a C-02 manual intervention. [C, `evidence/s3-down.log`,
    `s3-down-2.log`, `s3-park.log`, `cycle-results.tsv`]
19. **Google's refusal of a database delete names no permission.** The raw
    registry's error says "Permission 'artifactregistry.repositories.delete'
    denied". The raw database's says "The client is not authorized to make this
    request." with `notAuthorized []`, and no permission. A person reading the
    condition cannot tell which role is missing. The register asks whether the
    error named the permission, so this is a datum, not a fault. [C,
    `evidence/c31/b-part1.txt`]
20. **Offboarding left the cloud lopsided.** The engine deleted the service
    accounts `svc-onboard@` and `svc-scratch@` (it holds
    `iam.serviceAccounts.delete`, in ADR-0018 §2's list on purpose), and kept
    the registry and the instance, which it cannot delete. After a clean
    offboarding the cloud holds a registry (and for `svc-scratch` an instance)
    whose service account is gone, until a person's `gcloud` delete. [C,
    `evidence/s3-engine-audit.tsv`, `evidence/c31/a-offboard-*.txt`]
21. **A tenant's ResourceQuota refused C-29's test pod.** The `psql` pod named no
    requests or limits, and `svc-hello`'s quota refused it at 06:07:35Z ("must
    specify limits.cpu ... requests.memory"). The test's own defect, fixed on
    the re-run; the quota did its job. [C, `evidence/s3-tests.txt`, first
    section]
22. **Config Connector says "Successfully deleted" for an object it abandoned.**
    The event on both raw objects in C-31 (b) reads "Deleted ... Successfully
    deleted" after the abandon annotation released them, while both cloud
    resources were still present. A reader of the event alone would believe the
    cloud object is gone. [C, `evidence/c31/b-part2.txt`]
23. **A stale `DependencyNotFound` condition stayed on an instance for the whole
    create.** `svc-scratch-main` showed `False DependencyNotFound` ("Secret
    svc-scratch/main-admin was not found") at +141 s through +588 s of an
    11-minute create, though the Secret had existed since 06:19:17Z. The
    condition had cleared by +720 s, when the create finished. [C,
    `evidence/c31/a-onboard-svc-scratch.txt`]

## ADR errata found by the build

ADRs are superseded, never edited, so the correction lives here. Each says
whether it looks to need a superseding ADR.

- **ADR-0017 §7 — the update rule's exempt identities omit the garbage
  collector.** §7 lists Argo CD's controller, `cnrm-controller-manager` and
  `cnrm-deletiondefender` as the identities Kyverno lets through, and says the
  *delete* rule must let the namespace controller and garbage collector through.
  The update rule needs the garbage collector too (Surprise 1). The decision
  stands. Superseded by [ADR-0022](../adr/0022-what-the-m2b-build-changed.md) §1–2, because
  C-28's ADJUSTED rests on it.
- **ADR-0017 §7 — "denies create, update and delete" was not what was built until
  `62b9451`.** The policy's text as the ADR describes it was correct; the
  `allowExistingViolations` default made it untrue for update. No ADR change, a
  build fix. The ADR also does not mention that a write to `/status` is not sent
  to Kyverno at all; `62b9451` left that for a decision "for M2b's close",
  which [ADR-0022](../adr/0022-what-the-m2b-build-changed.md) §5 takes (the gap is accepted and
  stated; S3 showed the admission).
- **ADR-0017 Consequences — a Kyverno outage "now blocks every write to the gated
  kinds".** True only while the webhooks stay registered: right for a Kyverno
  that is unreachable with its webhook configurations registered (C-33 (e)
  attempt 2), wrong for a clean stop, because Kyverno v1.19.2 deletes its
  webhook configurations on graceful shutdown when the admission-controller
  Deployment is deleted, not found, or has `spec.replicas == 0` [C, `pkg/webhooks/server.go`
  `Stop` → `cleanup`, `pkg/utils/runtime/utils.go` `IsGoingDown`; kyverno#6554
  closed 2023-05-25, re-confirmed in 1.19.1 on 2026-10-13]. No opt-out flag
  exists in v1.19.2. Superseded by [ADR-0022](../adr/0022-what-the-m2b-build-changed.md) §3;
  the remedy is C-11's, in M3.
- **ADR-0019 Decision, headline — "No removed, renamed or broken file ... deletes
  a tenant without a person's step."** True of file names and presence; false of
  `metadata.name` edited inside a file (case (x)). The Context never names the
  route. Read with the exception in [ADR-0022](../adr/0022-what-the-m2b-build-changed.md) §4; the
  choice of guard is `platform-config#34`'s. The register's C-33 text has the
  same gap.
- **Register, C-33 (e) — "Scale Kyverno's admission controller to zero".** The
  registered step does not create the failure its prediction is about
  (Surprise 4). The register is append-only and the text stays; the correction
  lives here, with the deviation that was run.
- **ADR-0018 Context and §2 — "about 270" permissions and "cloudsql.admin, 173".**
  On 2026-10-27 `gcloud iam roles describe` gives 182, 69, 19 and 9: 279. Google
  grew the built-in role after the ADR; the 21 permissions and the subset
  property are unchanged. No ADR change.
- **ADR-0018 §4 — the subset check "before the first leg".** The only record is
  from after both legs (process, not text). Not an erratum to the ADR; a gap in
  the build's evidence.
- **ADR-0019 §5 — the Policy Analyzer read "before the session".** It was first
  made mid-session, at 06:00:40Z, 26 minutes into S3 (process, not text). Not
  an erratum to the ADR; a gap in the build's evidence.

## What is pending, in one place

- **The billing export** for `0174CB-E6957B-042C42`, enabled by the human, then
  a dated post-close addendum for C-32 if it returns rows for 2026-10-19/20,
  10-27 and 10-29.
- **The tag `m2b-close`, its release and the Software Heritage save.**

Done by the operator on 2026-10-31: the desk test; the deletes of instances
`svc-hello-second` and `svc-scratch-main` and registry `svc-scratch`. The
abandoned registry `svc-onboard` stays for ADR-0015 §5's runbook, as ADR-0019
§1 says. Done with this close: the dated M3 readiness-note entry on the Kyverno
fail-open finding, before C-11's first cluster case (it accepts the window,
with the fact stated).

Done at the desk on 2026-10-29: the C-28 (b2) render record, the PR runs of the
claim-schema cases, and Crossplane's requests (not needed, because both sides
are counted as working set). The empty correction commit on `platform-roles`
`main` for `b766d81` is made (C-31 Data).
