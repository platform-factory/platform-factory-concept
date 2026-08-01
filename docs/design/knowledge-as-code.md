# Knowledge as Code

Design doc, July 2026. The platform's second half: a curated knowledge layer that
makes both humans and agents productive on the paved road — designed against the
observed failure mode of "central brain" tools.

## Governing idea

**Treat knowledge exactly like code.** It is born via PR, changed via PR, has
owners, breaks when the world changes, and therefore needs CI and a test suite.
The approval boundary applies to knowledge too.

The anti-pattern this replaces: the ingest-everything central brain — point a
search index or an LLM at every doc, wiki, and Slack channel and hope. That
category fails on stale content: the index can't tell current from obsolete, so
answers regress to confidently wrong. Curated, versioned, owner-gated knowledge
is the category that works; the market has converged the same way (commercial
skills-registry products with versioning, policy gating, and evals — "evals are
to skills what unit tests are to code").

## Lifecycle

### Create

- Knowledge is captured **at the moment work happens**, never in doc sprints.
- Decisions → ADRs. How-tos → **skills**, written for dual consumption: the same
  skill file serves production agents and a developer's local coding agent.
- The agent drafts the knowledge PR; a human gates it. (Writing it down is the
  cheap part; the gate is where curation happens.)
- Prefer knowledge that **is** the system — GitOps YAML, scaffold templates,
  schemas — which structurally cannot go stale. Prose is reserved for the *why*.

### Update (freshness as CI)

- Every prose file declares its dependencies (a repo, a decision, a version).
  When a dependency changes, CI flags the file for review — **stale docs become
  failing checks** instead of silent rot.
- CODEOWNERS per file/folder.
- **ADRs are superseded, never edited** — the why-trail must survive
  reorganizations and personnel changes.

### Improve (the question bank)

- **The question bank is the regression suite for knowledge.** Seed it with the
  questions only tribal knowledge can currently answer — which simultaneously
  de-risks the people who are single points of failure.
- Periodically run an agent against the bank and grade the answers; failures
  become gap tickets.
- Make consumption mandatory via the golden path: knowledge nobody consumes
  rots invisibly, but a wrong skill fails *loudly* when the scaffolding agent
  uses it. Three distinct states worth tracking: published, covered by evals,
  and actually activated in use — installed is not the same as used.

## Where knowledge lives

**Rule: knowledge lives in the repo whose approval boundary covers the thing it
describes.** Hybrid layout:

- **`platform-knowledge` (central, its own repo).** Config repos have few
  approvers, heavy review, and deploy blast radius; knowledge has many
  contributors, light review, and zero deploy risk — the approval-boundary rule
  itself says these are different repos. Structure:

  ```
  platform-knowledge/
  ├─ CLAUDE.md          entry contract for agents
  ├─ skills/            scaffold-service, add-route, add-database, …
  ├─ adr/               platform-wide decision records
  ├─ standards/         org-mandated rules agents must load (incl. security/
  │                     compliance rules — a Git-native, PR-audited inventory
  │                     of what agents are told is itself a compliance feature)
  └─ questions/         the question bank, foldered by domain
  ```

  Test for what's central: if two service repos would copy the same skill, it's
  platform-scope.

- **Per-repo:** each service carries only its own CLAUDE.md + runbook.
  Co-location is the strongest staleness defense — same PR, same reviewer — and
  matches how coding agents auto-load repo context.

- **`template-service` seeds the per-repo skeleton** plus a pointer to central —
  knowledge is born with each service, never retrofitted.

- **Growth path:** new domains are new CODEOWNERS-gated folders under `skills/`
  and `questions/` — expansion falls out of the directory layout, no new
  architecture.

## Agents operate through the same gates

Agents build via PRs only — they never hold cluster credentials. Every mutation
an agent makes lands as a reviewable diff behind the same CODEOWNERS and
admission rules as a human's. A transferable lesson from watching autonomous
fleet-style systems: **autonomy scales with the quality of the external
acceptance contract** (a strong eval/test corpus), not with the model. The
question bank and the golden-path evals *are* that acceptance contract here.

Two later ADRs harden this section for the factory model: agent autonomy is
earned per change class and revoked automatically (ADR-0007, the autonomy
ladder), and the knowledge agents execute is itself heavily gated and
release-pinned (ADR-0006) — because instructions to an agent fleet carry the
fleet's blast radius, not a document's.

In persona terms (`factory-framing.md`): the agent is the platform's fifth
persona, and this layer is that persona's documentation — for the agent,
documentation quality is an operational parameter, not a nice-to-have.

## First artifacts

The ADRs in `docs/adr/` are the first knowledge entries — this repo dogfoods the
lifecycle from commit one: everything in the knowledge layer got there through a
PR.
