# ADR-0006: Agent-executed knowledge gets heavier gates and release pinning

Status: Accepted · July 2026

## Context

ADR-0001 placed knowledge in its own repo, `platform-knowledge`, with light
review, on the grounds that it has many contributors and zero deploy blast
radius. That reasoning held while knowledge was something humans *read*.

The factory direction changes what knowledge *is*: agents load `standards/` as
the rules they must obey and execute `skills/` as procedures. The moment an
agent executes a document, that document's effective blast radius is everything
the agent can do — fleet-wide, since the scaffolding and maintenance agents run
across every service repo. A wrong or maliciously injected skill is no longer a
stale doc; it is bad code on the widest deployment path in the platform, and
the lightest-gated repo becomes the highest-leverage attack surface (prompt
injection via docs is the supply-chain attack for agent fleets).

Applying ADR-0001's own rule: the approver of agent instructions is whoever is
accountable for what agents do fleet-wide — the platform and security teams,
not "many contributors."

## Decision

Gate `platform-knowledge` by folder, matching each folder's executable blast
radius:

- `standards/**` — CODEOWNERS requires **security + platform** approval (these
  files are what agents are told they must obey, including security and
  compliance rules).
- `skills/**` — CODEOWNERS requires **platform** approval (these files are what
  agents execute).
- `questions/**` and `adr/**` — light review, as before (read by graders and
  humans; zero execution risk).

Anyone may still *draft* freely — the gate is on merge, not on contribution.

**Production agents consume a tagged release of `platform-knowledge`, never
HEAD.** Cutting or advancing a release is itself a gated change. A bad merge
therefore cannot reach the running fleet on merge alone; it must also survive
the release gate. Local/developer agents may track HEAD — they are supervised
by the human driving them.

## Consequences

- Contribution friction rises on `skills/` and `standards/`; accepted, because
  the alternative is an unreviewed instruction path into every agent. The
  low-friction on-ramp remains `questions/` (capturing what needs answering
  stays cheap).
- Emergency knowledge fixes require a release cut; the fast path (tag from a
  hotfix branch) gets documented in the repo's own runbook.
- The pinned-release model gives incident response a new lever: rolling the
  fleet's knowledge back is a one-line revert of the release pointer.
- The PR-audited inventory of "what agents are told" (already claimed as a
  compliance feature in the knowledge-as-code doc) becomes real: the release
  tag is the auditable, reproducible answer to "what instructions was the
  fleet running on date X?"
