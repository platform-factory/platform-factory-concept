# Approval Evidence and Earned Autonomy — 2026 State of the Art

Research digest, 2026-07-30. Motivating question: in a software factory where
agents write, review, and test the changes, the human gate moves to approval
time — so (a) what evidence (**the approval packet**) can be put in front of an
approver so they can responsibly approve without reading the code, and (b) what
does the industry know about letting change classes *graduate* from
human-approved to auto-merged? Load-bearing claims verified against primary
sources on 2026-07-30 by two research agents; [C] confirmed against a primary
source (URL inline), [I] inferred, unverified items flagged.

---

## TL;DR

1. **Evidence formats are settled and GA.** in-toto Statement v1 + DSSE
   signing, SLSA Provenance v1 (spec v1.2 "Approved"), vetted predicates for
   test results / vulnerability scans / SBOMs, all signed via Sigstore. GitHub
   Artifact Attestations GA since June 2024. [C]
2. **Enforcement is production-grade and already in this platform's stack.**
   Kyverno (CNCF Graduated, v1.18) verifies Sigstore/GitHub attestations at
   admission with arbitrary predicate-field checks — the reality gate for
   "no evidence, no deploy" needs no new tooling here. [C]
3. **The human approval surface is the industry-wide weak link.** Everyone can
   produce and enforce evidence; almost nobody shows it to the approver.
   GitHub's native "Review deployments" screen shows a ref and logs — no
   evidence panel. [C]
4. **The coherent 2026 pattern is PR-as-approval-surface:** the human approves
   by merging a PR whose *required status checks are the evidence* (GitOps
   Promoter's model), optionally compressed into one signed machine verdict
   (SLSA's Verification Summary Attestation — explicitly designed for
   "delegate the complex policy decision to a trusted verifier"). [C]
5. **Earned autonomy is whitespace.** Renovate is the only live example of
   confidence-driven automerge, it's per-package not per-actor, and the
   confidence→automerge loop is paywalled. Keptn (the canonical quality-gate
   project) was CNCF-archived Sept 2025. And an explicit negative result:
   **no vendor, standards body, or platform has published a trust-tier scheme
   under which AI-agent PRs merge with less review as track record accrues.**
   Every agent product ships static admin toggles. [C by absence, searched]

---

## Part 1 — Evidence: formats and enforcement

- **SLSA v1.2** (current, "Approved"; v1.1 retired). Build levels L1–L3;
  provenance is an in-toto predicate recording builder identity, build type,
  parameters, resolved dependencies. Verification model: pin trusted builder
  identities, verify signature, check parameters against expectations.
  https://slsa.dev/spec/v1.2/build-provenance [C]
- **SLSA VSA (Verification Summary Attestation)** — the "approve without
  reading" primitive: a trusted verifier attests it evaluated the artifact +
  its attestations against a policy → one signed `PASSED|FAILED` +
  `verifiedLevels`. https://slsa.dev/spec/v1.2/verification_summary [C]
- **in-toto attestation framework v1** — Statement (subject digests +
  predicate) in a DSSE envelope. Vetted predicates relevant to a packet:
  **Test Result** (`test-result/v0.1`), **Vulnerabilities** (`vulns/v0.2`),
  **Simple Verification Result** (`svr/v0.2` — lightweight policy-evaluation
  evidence), SBOMs (SPDX/CycloneDX).
  https://github.com/in-toto/attestation/blob/main/spec/predicates/README.md [C]
- **GitHub Artifact Attestations** — GA June 2024. Actions builds sign SLSA
  Provenance v1 via Sigstore (`actions/attest@v4`); `gh attestation verify`
  checks it; GitHub ships Helm charts (Sigstore policy-controller + org
  `ClusterImagePolicy`) for cluster enforcement. Private repos use GitHub's
  Sigstore instance (no public transparency log). **Not available on GitHub
  Enterprise Server** (docs versioned cloud-only). [C]
  https://docs.github.com/en/actions/concepts/security/artifact-attestations
- **Kyverno v1.18 (CNCF Graduated)** — `verifyImages` supports
  `type: SigstoreBundle`, keyless GitHub OIDC identities, and **JMESPath
  conditions over arbitrary predicate JSON** (documented example checks SLSA
  provenance from GitHub Actions); new CEL-based `ImageValidatingPolicy` is
  Stable in 1.18. This is the packet's reality gate: an image without passing
  attestations does not admit.
  https://kyverno.io/docs/policy-types/cluster-policy/verify-images/sigstore/ [C]
- **cosign v3** (GA) — signs/attaches attestations to images as OCI 1.1
  referring artifacts; built-in CUE/Rego evaluation of predicates on verify.
  Sigstore **policy-controller is still 0.x** ("actively under development")
  — even though GitHub builds on it; prefer Kyverno as the admission verifier
  here [I]. https://docs.sigstore.dev/policy-controller/overview/ [C]
- **GCP-native parallel:** Binary Authorization enforces attestations at
  GKE/Cloud Run deploy time (enforcement GA; continuous validation Preview);
  Cloud Deploy has first-class per-target release approvals.
  https://cloud.google.com/binary-authorization/docs/overview [C]

## Part 2 — The approval surface (where the human actually looks)

- **GitHub environments** (native): required reviewers (max 6 listed, only ONE
  must approve — a quorum of one [C]), wait timers, branch restrictions,
  admin-bypass on by default. The approve screen shows the ref, logs, and a
  comment box — **no evidence panel**.
  https://docs.github.com/en/actions/managing-workflow-runs-and-deployments/managing-deployments/reviewing-deployments [C]
- **Custom deployment protection rules** (GitHub Apps): still public preview.
  The App receives a webhook per gated deploy, may inspect anything (checks,
  artifacts, attestations, external systems), then approves/rejects — and may
  post **up to 10 Markdown "status reports" (≤1024 chars each)** onto the run:
  GitHub's only native way to put machine-gathered evidence in front of the
  approver. [C]
  https://docs.github.com/en/actions/deployment/protecting-deployments/creating-custom-deployment-protection-rules
- **Argo CD** (v3.4 stable): manual sync + health + diff is the native
  approval; **Source Hydrator** (alpha 3.4 → beta 3.5) pushes fully rendered
  manifests to git pre-sync so the reviewer sees exactly what will hit the
  cluster — packet-relevant: the diff the human approves becomes the *real*
  diff. [C]
- **GitOps Promoter** (argoproj-labs, v0.34, pre-1.0): environment promotion
  via PRs; gating = SCM **commit statuses** (e.g. `argocd-app-health` from the
  running env, `security-scan` on the candidate); prod sets `autoMerge: false`
  so **a human approves by merging a PR whose required checks are the
  evidence**. The cleanest structural match to this platform's PR-everything
  model. https://github.com/argoproj-labs/gitops-promoter [C]

## Part 3 — Earned autonomy and promotion gating

- **Renovate Merge Confidence** — the only live earned-autonomy mechanism.
  Four PR badges (Age / Adoption / Passing / Confidence) computed from
  crowd-sourced test+adoption data; change classes expressed as `packageRules`
  matchers (update type, dep type, package name) each with its own
  `automerge`; `minimumReleaseAge` adds time-based trust. But
  **confidence-as-automerge-condition (`matchConfidence`) requires a paid Mend
  key (private beta)** — OSS users get evidence for humans + class-based
  automerge, not the closed loop. Two design gotchas worth stealing:
  platform automerge needs ≥1 required status check or it can merge on red;
  maintainer guidance is "automerge only where you'd click merge without
  reading changelogs." https://docs.renovatebot.com/merge-confidence/ [C]
- **Dependabot** — no native automerge key; the pattern is an Actions workflow
  gating on update-type metadata then `gh pr merge --auto`. Compatibility
  scores still exist but only for security updates. [C]
- **Keptn — CNCF-archived 2025-09-03** (maintainer company stepped back).
  Its v1 quality-gate *design* remains the canonical scored gate: weighted
  SLO objectives, key-objectives that must pass regardless of score, relative
  ("≤ +10% vs last run") and absolute thresholds, and a three-outcome result
  — **pass / warn ("where a manual approval might be needed") / fail**.
  https://github.com/keptn/spec/blob/master/service_level_objective.md [C]
- **Argo Rollouts v1.9** (CNCF Graduated) — AnalysisRuns gate canary steps on
  metric expressions (Prometheus, CloudWatch, Datadog, Job, Web, +8 more);
  outcomes **Successful / Inconclusive (pause for human) / Failed (auto-abort,
  canary weight back to zero)**; `dryRun` metrics = shadow-mode evaluation
  before a metric is allowed to gate. [C]
- **Kargo v1.11** (GA since Oct 2024, Akuity) — promotion as product: Freight
  (immutable artifact set) **earns downstream availability** by passing
  verification (Argo Rollouts AnalysisTemplates) + `requiredSoakTime` in
  upstream stages; per-stage `autoPromotionEnabled` (auto test/uat, manual
  prod); manual approval is the documented hotfix bypass and is recorded on
  the Freight. Deliberate RBAC: promotion policy lives at project level so
  stage-edit rights ≠ promote rights. https://docs.kargo.io [C]
- **Flagger, Harness CV** — same shapes: metric-gated progressive delivery
  with webhook human-gates; Harness adds ML-baselined anomaly detection
  (1σ/2σ/3σ sensitivity) as the verdict. [C]

### The negative result (and the whitespace it marks)

Searched explicitly: **no published scheme exists where agent-authored PRs
gain merge autonomy from accrued track record.** What ships instead [C]:

- **Copilot coding agent**: treated as an outside contributor — CI doesn't run
  until a human clicks "Approve and run workflows"; requester's approval
  doesn't count toward required review. Static admin opt-outs, no graduation.
- **OpenAI Codex**: PR summaries must cite terminal logs/test outputs
  (verifiable evidence practice); tiered *execution* permissions, not merge
  rights.
- **claude-code-action**: progress comment with task checkboxes; cannot
  approve, cannot merge; short-lived repo-scoped token.
- **Devin**: same branch protections as humans; distinctive evidence practice
  — **annotated video of it exercising the app** attached post-PR ("verify
  the changes work without pulling the branch").
- Adjacent taxonomies (UW autonomy-levels paper, Sourcegraph L0–L5) are
  capability/HCI frameworks, not merge-gating schemes; OpenSSF's 2025 AI-
  assistant guide mandates human review always, no tiers.

Two patterns recur across everything above and belong in the ladder design:
promotion is earned **per-artifact** (Kargo Freight) or **per-change-class**
(Renovate rules) — never per-agent [C]; and every mature gate is
**three-outcome** (pass / inconclusive→human / fail), not binary [C].

## Part 4 — What the packet looks like here [I]

All inference — this maps the verified prior art onto this platform's repos.
Common spine for every packet: link to the intent artifact + its
machine-checked "done means" results + a plain-English summary of what changed
and why. Then per repo, the evidence differs because the risk differs:

| Repo / change | Packet on top of the spine | Reality gate |
|---|---|---|
| Service repo, prod tag bump | Image provenance (built from this commit by CI — GitHub attestation); test-result attestation; canary/preview AnalysisRun result | Kyverno `verifyImages` at admission |
| `edge-config` `external/**` | Exposure diff (what becomes reachable), folder↔field consistency check, selected WAF policy + DNS zone, hostname-schema check | Kyverno denies edge objects from app namespaces |
| `platform-config` | Blast radius: which Compositions change, count of existing claims affected, Kyverno policy dry-run report, rendered-manifest diff (hydrator pattern) | Argo sync + admission |
| `systems` | Tenant diff: quota, RBAC bindings, project mapping | Composition schema |
| `platform-knowledge` release | Diff since last tag; question-bank eval results on the new release | Fleet pins tags (ADR-0006) |

Delivery mechanism, v1 [I]: no new tooling — the packet is a **PR comment +
required status checks** (GitOps Promoter's shape), with attestations verified
by Kyverno at admission as the backstop. A custom GitHub App deployment
protection rule (status reports on the approval screen) is the later upgrade;
it's still public preview.

## Open / unverified

- SLSA v1.2 release date; BuildEnv track (draft only).
- Whether Dependabot compatibility scores are reliably populated in practice.
- Kyverno PolicyReport surfacing details (well-known, not re-fetched).
- Ladder promotion/demotion rules are a design decision, not a research
  question — nothing to copy; the whitespace is the point (ADR pending).
