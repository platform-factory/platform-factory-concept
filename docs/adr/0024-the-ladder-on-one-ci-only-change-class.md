# ADR-0024: The autonomy ladder on one CI-only change class: two locks hold the class, a frozen rule promotes it, and any escape returns it to L1

Status: Accepted · November 2026 · for the Go-module bump class on `svc-hello`, replaces the *mechanism* — not the decision — of ADR-0007's rung ("branch protection + CODEOWNERS on the class's paths") and of "changing a rung is itself a PR to the ruleset"; supersedes ADR-0007's "demotes one rung" (demotion ends at L1 from any merging rung), its "N set per class in the ruleset, tunable", its near-miss rule, and the order in which its shared-machinery freeze follows triage; defines L2 and L3 as ruleset values and keeps L3 as a definition only; decides the dependency bump ADR-0017's Consequences left to M4's walk and the GitHub App ADR-0021 §16 deferred to M4; moves C-12's remainder, which ADR-0021 §11 sent to the prod-tag-bump class at M4, past M4; gives ADR-0006's release pinning a mechanism for one skill; fixes ADR-0020's grade ceilings for C-15..C-22; the mechanism is untested at this date · from platform-factory-concept#23, #24, #27 and #38 and svc-hello#16, decided on 2026-11-11, before any M4 build command ran

## Context

ADR-0007 says a change class earns merge rights from its own track record and
loses them automatically. It names the rung's machinery in one line, "branch
protection + CODEOWNERS on the class's paths", and leaves the rest to M4: what
L2 and L3 are in those terms, where N lives, who applies a demotion, and what
an escape is for a class that deploys nothing. Four open issues promise this
ADR and M4's readiness note (platform-factory-concept#23, #24, #27 and #38),
and svc-hello#16 lists what `svc-hello` lacks to host a class at all. #38
asked M3's readiness note to take four decisions M3 and M4 share. It did not:
ADR-0021 §10 says "no GitHub App in M3" and §16 puts "a GitHub App (M4)" out
of M3. This ADR takes them.

What exists at this snapshot (`platform-factory-concept` `61dd2ba`,
`svc-hello` `312b18d`, `platform-knowledge` scaffold, `platform-factory/.github`
ruleset records through 2026-11-11):

- **L1 and L2 are the same rung today.** `svc-hello`'s ruleset
  `main-protection` (20204893) requires a pull request and zero approvals,
  has no required reviewers, no required status check and no bypass actor,
  and keeps `require_extra_approval_for_unattributed_changes: true` [C,
  read-only `gh api`, 2026-11-11]. M1 recorded why: with one human, a required
  approval blocks every merge, so "the count tightens when agent-authored PRs
  begin (M4) … This needs to be explicit in the M4 test design for
  C-15/C-16" [C, `docs/build-log/m1-spine.md:339-344`]. The org's two
  member accounts both belong to the maintainer. The one the maintainer's
  command line uses, and so the delegate too, is an organisation owner and
  the only member of `platform` and of `security`; the other is a plain
  member in no team. `platform` has write on `svc-hello` and
  `platform-knowledge` [C, read-only `gh api`, 2026-11-11].
- **Nothing in `svc-hello` can tell a good bump from a bad one.** There is no
  `*_test.go`. `validate.yml` runs `yaml`, `go vet and build` and `claims`,
  all advisory, on every `pull_request` and on `push` to `main` [C,
  `svc-hello` `.github/workflows/validate.yml:15-18`, `:26-29`, `:83-97`].
  The Dockerfile sets `GOFLAGS=-mod=mod`, copies `go.su[m]` as optional and
  runs `go mod tidy` in the build, so the committed checksums are not what
  ships [C, `Dockerfile:46`, `:55`, `:57`]. `go.mod` carries a dated
  "Verified 2026-09-16" comment [C, `go.mod:3-5`].
- **A pull request can pass its own required check.** On `pull_request`
  GitHub runs the workflow from the pull request's head; edge-config#19
  showed it (ADR-0023 §2). The guard ADR-0023 §2 decided was applied on
  2026-11-11, and edge-config#19 read BLOCKED afterwards [C,
  `platform-factory/.github` `rulesets/README.md`, steps 09 → 10].
- **A second non-human principal exists.** A GitHub App with slug `claude` is
  installed on every org repository since 2026-09-22, with
  `repository_selection: all` and `contents`, `pull_requests`, `workflows`,
  `actions`, `checks`, `issues` and `repository_hooks` write [C, read-only
  `gh api orgs/platform-factory/installations`, 2026-11-11]. "All" takes in
  repositories created later [I, not tried]. The Claude Code action's FAQ
  says the official Claude App has no workflow write [C,
  `anthropics/claude-code-action` `docs/faq.md:62`], so which App this is,
  or whether its permissions changed, is [unverified]. Every pull request in
  the org so far is authored by the maintainer's login.
- **The supply.** `svc-hello` pins pgx v5.11.0, x/text v0.29.0, x/sync
  v0.17.0 and `go 1.25.0` [C, `go.mod`]. On the Go module proxy on
  2026-11-11: x/text v0.30.0..v0.44.0 (v0.42.0 and later declare `go
  1.26.0`), x/sync v0.18.0..v0.25.0 (v0.23.0 and later declare `go 1.26.0`),
  pgx v5.12.0 (2026-10-31) and v5.13.0 (2026-11-11) [C, module proxy,
  2026-11-11]. Every x/text from v0.31.0 also raises x/sync, because x/text's
  own `go.mod` requires it; x/sync steps never move x/text; pgx v5.12.0 and
  v5.13.0 also require puddle v2.2.3 (pgx's connection-pool library), and
  `svc-hello` has v2.2.2 [C, observed in `golang:1.27.1-alpine`,
  2026-11-11]. All are releases that already exist, so a bump run through
  the class is a replay, not an independent live result.
- **`platform-knowledge` is an empty scaffold** with CODEOWNERS
  `* @platform-factory/platform` [C, `platform-knowledge` `README.md:3`,
  `.github/CODEOWNERS:1`].
- **The auto-mode classifier that runs this build's commands** refuses merges
  of blocked pull requests and edits that loosen a ruleset; tightening goes
  through [C, `docs/build-log/m3-approval-boundary.md`]. Every loosening in M4
  is therefore the maintainer's own act.

## Decision

**One change class, a Go-module bump on `svc-hello`, runs from L1 to L2 and
back. Two locks hold its boundary: a ruleset whose only difference between L1
and L2 is one approval count, and a check run from the default branch that
decides class membership by recomputing the change. The streak length, queue
and promotion rule are frozen in a file the agent cannot reach. Promotion is a
person's act; any escape returns the class to L1 at once, applied by an App
that can only raise approvals. The model that writes the change holds no
GitHub credential. The class deploys nothing, so M4 tests the ladder's
machinery with a governed non-human principal, not the safety of a class or
the capability of an agent.**

1. **The class: a Go-module bump on `svc-hello`, decided by meaning.** Files:
   `go.mod` and `go.sum` at the repository root, nothing else. One target
   module per pull request, already in `go.mod`, moved to a higher release tag
   within the same major version. Other modules move only as far as the
   target's own requirements force. No `go` or `toolchain` directive change; no
   `replace`, `exclude` or `retract`; no module added or removed; no
   pseudo-version or pre-release. Anything else is off-class. This decides
   ADR-0017's open item ("which kind of dependency bump M4 starts on").

   Membership is decided by recomputing the change, not by parsing its diff.
   A trusted job reads the tip of `main` when it runs, never the pull
   request's stated base. It takes that tip's `go.mod`, `go.sum` and every
   `.go` file, because `go mod tidy` counts test imports too. It runs `go get
   <module>@<version>` and `go mod tidy` for the queue's next item (below), in
   the pinned builder image (§6) with `GOTOOLCHAIN=local`. It then requires
   the pull request's two files, read at the event's head commit, to equal the
   result byte for byte. That one comparison decides each hard case. The
   forced x/sync with x/text, and puddle with pgx, are in the result. A
   downgrade visible only as a second changed line is not. Nor is an
   `// indirect` flip, a `go.sum` with appended instead of replaced lines, or
   a target other than the queue's [C, observed: on clean bumps `go mod tidy`
   after `go get` changes nothing further and keeps line order and both
   require blocks, 2026-11-11]. Whitespace, comment and line-ending edits are
   off-class too, which is accepted: nothing in the class needs them. `go
   get` downloads module sources from the proxy and does not run them. A
   proxy failure is inconclusive (§6), never a pass.

   The queue is the 25 single-minor steps that exist on 2026-11-11, in this
   order, because an x/text step queued before x/sync is done would move two
   modules: x/sync v0.18.0..v0.25.0 (8), x/text v0.30.0..v0.44.0 (15), pgx
   v5.12.0 and v5.13.0 (2). Releases that appear during the run are not added.
   The next item is read from GitHub's state, not from a file anyone updates:
   the first queue item whose version is above `main`'s and that no
   near-miss closed (§8). A reverted item is therefore queued again. The
   agent's workflow does not open an item younger than the minimum release
   age (§6); it waits.

2. **CI-only, and one human `go.mod` change before the run.** The class
   deploys nothing. `svc-hello`'s image is still built and pushed by hand
   (`m2-paved-road.md`, as the M2 walk recommended), so during the run `main`'s
   `go.sum` is not the deployed one. M4 needs no cluster for its window, which
   dissolves #24's open cost of keeping one up, and no CI image identity; that
   identity, deferred by M2 "to the M4 change class that needs it", moves again,
   with the prod-tag-bump class, to a later milestone. Three consequences are
   stated now. C-12's "PRs merged since the last healthy state", which
   ADR-0021 §11 sent to "the prod-tag-bump class at M4", is not run in M4, and
   neither is C-18's spine-attributed escape type: both need a deploy event to
   say when "healthy" was. C-25 is not run; its subject was removed (ADR-0017;
   the scoreboard's cell). Before the L1 rung is switched on, one human pull
   request raises the `go` directive to `1.26.0` and removes the dated comment
   from `go.mod`. With `go 1.25.0`, six of the 25 steps (x/text v0.42.0 and
   later, x/sync v0.23.0 and later) would need a directive change and be
   off-class [C, observed: `go get
   golang.org/x/text@v0.42.0` printed `go: upgraded go 1.25.0 => 1.26.0`].

3. **What a rung is, in ruleset terms.** A rung is the `minimum_approvals` of
   one required-reviewers entry for team `platform` on `go.mod` and `go.sum`
   (the *rung entry*), with everything else in the ruleset unchanged. A
   second entry, the *wall entry* (§4), covers every other path at one
   approval on every rung.
   - **L1**: the rung entry at 1. The agent writes; a person on `platform`
     approves before any merge.
   - **L2**: the rung entry at 0. "Requiring zero approvals means that the
     team will be added for visibility, but the team does not need to approve
     the request" [C, GitHub docs, "Available rules for rulesets", read
     2026-11-11]. The team is told; the agent may merge. That is ADR-0007's
     "human notified after", and the notified person is not asked to watch
     for anything: the gates decide (#23). A person who does object stops the
     class by raising the rung entry to 1 or disabling the agent's workflow;
     both are tightenings anyone on the build may make.
   - **L3**: no rung entry and no CODEOWNERS owner on the class's paths, so
     nobody is told. Kept as a definition, not built and not used in M4.
   - **L0**: reached only by designation, as framing commitment 4 says
     ("permanently manual" recorded as an ADR), never by demotion.

   The L1 → L2 change is one field, `minimum_approvals` 1 → 0, in the JSON
   recorded before and after; the readiness note fixes the whole L1 JSON.
   `svc-hello`'s CODEOWNERS (`* @platform-factory/platform`) keeps requesting
   review and blocks nothing, because `require_code_owner_review` stays
   false; it plays no part in the rung.
   `require_extra_approval_for_unattributed_changes` stays as it is and is
   recorded as inert: it adds an approval only to a non-zero bare count, which
   stays 0, and only for Copilot [C, same page: "This setting has no effect if
   the ruleset requires zero approvals"] (#27).

4. **Lock 1: the rung entry and the wall entry, in one ruleset with no bypass
   actor.** The wall entry, also for `platform`, covers every path except the
   class's two files at 1 approval, at every rung: patterns `*`,
   `.github/**`, `!/go.mod`, `!/go.sum`, in that order, and the rung entry is
   `/go.mod`, `/go.sum`. A pattern starting with `!` "will cause paths
   matching earlier patterns to *not* require approvals. Patterns are matched
   in order" [C, GitHub docs, same page; generally available since 2026-02-17
   per GitHub's changelog]. The patterns are anchored at the root, because in
   gitignore format, which the docs name, a bare `go.mod` matches at any depth
   and would take a nested `x/go.mod` out of the wall [C, gitignore
   documentation: a pattern with no slash matches at any depth; the REST
   description says "File patterns use fnmatch syntax" instead, so the two
   sources conflict; behaviour in rulesets not tried, probe P1].

   The pull request rule sets `dismiss_stale_reviews_on_push: true`, so an
   approval given at L1 does not survive a later push [C, same page: "the
   approving review is dismissed as stale"]. It sets
   `require_last_push_approval: false`. The docs describe that setting as
   requiring "an approval from someone other than the last person to push",
   and say nothing of it being inert at zero approvals; at L2 the agent App is
   always the last pusher and nobody approves [C, same page, read
   2026-11-11]. Stale dismissal is the guarantee. One window stays open and
   is stated: with that setting false, a push that lands on a person's
   non-class pull request just before the listed human approves would be
   approved unseen. No code path makes such a push and the model holds no
   token (§7); it is not a named test. Merges are squash
   only, and the merge step (§7) sets the commit title and message itself, so
   no agent-written text reaches `main`'s commit. `required_approving_review_count`
   stays 0: an App can submit an approving review, and whether it counts
   toward the bare count is [unverified], while an entry counts only
   approvals from members of a team with write.

   The wall closes a hole an author-only check leaves open. An App with
   `contents` write can push to any branch, write commits whose author and
   committer fields are free text, and merge a pull request through the API
   [C, GitHub App permission data, `server-to-server-permissions.json`,
   `PUT /pulls/{n}/merge` under `contents`]. So an App token could push to a
   human's pull request branch, or merge one, unless every path outside the
   class needs an approval it cannot give.

   The wall also makes the maintainer's own non-class pull requests on
   `svc-hello` need an approval from someone other than their author ("Pull
   request authors cannot approve their own pull requests" [C, GitHub docs]).
   M4 uses the maintainer's second account for that (§8): it approves, and
   applies `human-cleared` so that `class-scope` passes a pull request that
   changes neither class file (§6). So a human pull request after the wall
   needs no lift. A lift, the owner lowering an entry by hand, merging with
   the head commit pinned and restoring it, with each version in the history
   endpoint (ADR-0021 §10), is kept for faults and probe fallbacks only. A
   standing bypass actor was the other candidate. It is not used, because the
   login it would name is the owner account the maintainer's delegate also
   operates. `bypass_actors` stays empty, and the `exempt` mode, which "will
   not create" a bypass audit entry [C, REST description,
   `repository-ruleset-bypass-actor`], is never used.

   Read from the docs, not tried: how `*` and anchored patterns behave,
   whether one ruleset accepts two entries for the same team, whether stale
   dismissal applies to an entry's approvals, and whether an App's approval
   can satisfy a team entry. The readiness note's probes P1 and P3 try each
   on a throwaway repository, and fixes each fallback.

5. **Lock 2: `class-scope`, a check the pull request cannot edit.** A
   required status check, `class-scope`, pinned to GitHub Actions
   (integration 15368), in its own workflow in `svc-hello`, triggered by
   `pull_request_target` for `branches: [main]` with `types: [opened,
   synchronize, reopened, labeled, unlabeled]`; the default types leave out
   label events [C, GitHub docs, "Events that trigger workflows"]. That event
   runs "the workflow, and any subsequent `actions/checkout` call that does
   not specify a `ref`", from "the base repository's default branch, not from
   the pull request" [C, GitHub docs, "Securely using pull_request_target"].
   It never checks out or runs the pull request's code: it reads the two
   files at the event's head commit through the API as data, applies §1, and
   reads the class file from the controls repo's `main` (§7), recording that
   commit. The job has no job-level `if:` and no path filter, and every
   trigger recomputes the full verdict; a run that reaches no verdict fails.
   A skipped job would report success [C, GitHub docs, "Troubleshooting
   required status checks": "Successful check statuses are `success`,
   `skipped`, and `neutral`"]. The ruleset sets
   `strict_required_status_checks_policy: true`, so the head must contain
   `main`'s tip, and `class-scope` fails a head that does not.

   Required checks are looked up by head commit and name, not by pull request
   [I, from GitHub docs, "Troubleshooting required status checks": "Required
   checks must pass on the latest commit SHA", and "Troubleshooting rules":
   "Required status checks do not take workflow, matrix, or event trigger
   types into account"; probe P2's two-pull-request case tries it]. So a
   second pull request with the same head into a branch the
   agent made could otherwise lend the head a green check. The `branches:
   [main]` filter on `class-scope` and on `validate.yml`'s `pull_request`
   trigger (§6) means no run starts for a pull request into any other branch.

   It is author-blind: every pull request into `main` must be one on-class
   change matching the queue, whoever opened it, or carry a listed human's
   clearance (§6). That closes the forged-author and pushed-to-a-human-branch
   routes without trusting any author field. The pin to GitHub Actions does
   not stop a pull request that adds a workflow with a job of the same name,
   as M3 showed for `route files`; the agent App cannot push a workflow file
   (§7), and any other author meets the wall on `.github/**`.

   This departs from the 2026-10-27 review's design, which put `class-scope` in
   `validate.yml` on `pull_request` and passed it for non-App authors. On that
   event a pull request can edit the workflow that produces its own check
   (ADR-0023 §2), and an author test is defeated by the routes above.

   Read from the docs, not tried: which commit a `pull_request_target` check
   run attaches to, whether a red one blocks a merge, and whether a later
   green run replaces it. GitHub's default policy blocks
   `pull_request_target` in public repositories, in evaluate mode now and
   enforced from 2026-12-02 [C, same page, "Default policy"]; every org
   repository is public, so an explicit Actions policy allowing the event on
   the throwaway repository and `svc-hello` is an owner's step before probe
   P2. If P2 fails, the readiness note's fallback is a `pull_request` job in
   its own workflow, held by the agent App's missing `workflows` permission
   and the wall on `.github/**`. If no trusted check attaches and blocks, the
   L1 run goes ahead without one and the claims that need L2 are NOT REACHED,
   as the note fixes.

6. **The verifier: two required checks, three outcomes, and an inconclusive
   that blocks.** The required checks are `class-scope` (§5) and `go-test`, a
   new `validate.yml` job, whose `pull_request` trigger gains `branches:
   [main]`. `go-test` runs in the pinned builder image
   (`golang:1.27.1-alpine`, the Dockerfile's), with two Postgres service
   containers, one on trust authentication for the real `newPool` and one on
   SCRAM with a non-ASCII password, the only path that reaches x/text [C,
   observed, 10 of 10 prototype tests at baseline, after x/sync v0.25.0 and
   x/text v0.44.0, and after pgx v5.13.0, 2026-11-11]. In a container job the
   services are reached by name, and the tests connect as `postgres`, so no
   `psql` step is needed. It runs `go mod download`, `go mod verify`, `go
   vet`, `go build -mod=readonly`, the tests with any `--- SKIP` failing the
   job, and `go mod tidy -diff`. `go mod verify` stays for a modified module
   cache, and is not credited for a tampered `go.sum`: it exited 0 on one,
   while the read-only build, `go mod download` and `go mod tidy -diff` each
   exited 1 [C, observed, 2026-11-11]. A govulncheck step inside `go-test`
   writes its finding IDs to the job summary and never fails the job, because
   it exits 3 today on standard-library findings alone [C, observed]. The
   Dockerfile moves to `-mod=readonly`, a required `COPY go.mod go.sum` and no
   `go mod tidy` in the build. `go-test` runs the pull request's code, which
   is what a test is for; it has a read-only token and no secrets, and an
   off-queue module would run there before `class-scope` fails it. That is
   accepted and stated. Each check is made required only after it has failed
   once and passed once (ADR-0019 §4).

   The outcomes:
   - **pass**: both checks succeed.
   - **fail**: `class-scope` finds the change off-class, or `go-test` fails.
     `go-test` has no inconclusive outcome.
   - **inconclusive**: `class-scope` cannot decide. Reasons a person may
     clear: the target's release is younger than seven days by its proxy
     `.info` time; or the outcome was forced for C-20's test. Reasons nobody
     clears, which a re-run settles: the proxy or the sum database
     unreachable after the job's own retries, or the check's own failure. A
     clearance never passes a change §1 has not recomputed.

   Inconclusive is carried as a failing `class-scope` with a `needs-human`
   label, never as `neutral` or `skipped`. A person clears a clearable reason
   by applying `human-cleared`; the label event re-runs `class-scope`. A
   human label is in force from a listed human login's `labeled` event until
   a listed human login's `unlabeled` event; an App's removal is ignored and
   counted as an unknown. It counts only when the event carries no
   `performed_via_github_app` [I, from the REST issue-events schema; tried in
   probe P3], and only for the head commit it was applied to: on
   `synchronize`, `class-scope` removes it and a new head needs a new label.
   The agent App can apply labels too [C, permission data:
   `/issues/{n}/labels` under `pull_requests` write], which is why the actor
   is read from the issue's events, not from the label. Two more passes exist,
   both for pull requests a person opens. `human-cleared` lets `class-scope`
   pass a pull request that changes neither class file; the wall still needs
   the listed human's approval on it. And `escape:revert` in force lets it
   pass a pull request whose two files equal those at the parent of the latest
   class merge (§9). Clearing never touches the streak or the rung. At L1 the
   approval is still needed.

   C-20's forced inconclusive and C-18's injected post-merge failure are
   switched by two `svc-hello` repository variables, read by `class-scope` and
   by a `go-test` step that runs only on `push`. Setting a variable needs a
   permission the agent App does not hold [I, from the App permission list;
   C-21 tries it].

7. **Identity: two org-owned Apps, a controls repository, and a model with no
   GitHub credential.**
   - **The agent App** (working name `platform-factory-agent`): `contents`
     write, `pull_requests` write, `metadata` read; no `workflows`, no
     `administration`, no `checks`, no `statuses`, no `issues`. Installed on
     the throwaway probe repository and, once the L1 ruleset is active and
     recorded, on `svc-hello` only. Without `workflows` it cannot push a
     change under `.github/workflows/` [C, GitHub docs, "Choosing permissions
     for a GitHub App"; the push refusal text is from a user report]; without
     `checks` or `statuses` it cannot create a check run or a commit status
     [C, permission data].
   - **The ratchet App** (working name `platform-factory-ratchet`):
     `administration` write and `pull_requests` write on `svc-hello` only,
     installed at the same point. No narrower permission edits a repository
     ruleset, and `GITHUB_TOKEN` has no `administration` key [C, permission
     data; GitHub docs, `GITHUB_TOKEN` permissions]. That permission also
     lets it lower an entry, edit `bypass_actors`, disable or delete the
     ruleset, delete the repository, change its settings and Actions policy,
     and add collaborators and deploy keys [C, permission data,
     `administration`]. "Only raises approvals" is held by its code (§9,
     §10), not by GitHub, and its key sits in a GitHub environment only the
     detector and the merge step's first job use.
   - **The controls repository** (working name `platform-factory/factory-controls`,
     public), on which neither App is installed and which no `svc-hello` code
     can write: the class file, the agent's workflow with its merge step, the
     fixtures workflow, the scorecard, the escape detector and ratchet, their
     records, and every secret. No secret lives in `svc-hello`, and no
     organisation secret or variable is shared with it. A ruleset on its
     `main` requires a pull request, refuses force-pushes and deletion, has no
     bypass actor, and requires an approval from `platform`, which has write
     there, on `classes/**`, where the class file and the promotion and
     fixture records live. Records
     the machinery writes go to a `records` branch under a second ruleset
     that refuses force-pushes and deletion.

   The agent is an LLM agent run headless (Claude Code, the CLI or the base
   action pinned to a commit; model `claude-sonnet-5-5`, a pinned snapshot
   [C, Claude models overview, read 2026-11-11]), under the skill release
   pinned in the class file (§13). The model runs in a job of its own with
   `permissions: {}`, so it has no `GITHUB_TOKEN` scope and no OIDC token. It
   is given no GitHub token, no checkout credential and one secret, the model
   credential. Its workspace holds `svc-hello`'s module files and `.go`
   sources at `main`'s tip, the queue item and the pinned skill. Its tools are
   `Read`, `Write`, and `Bash` limited to `go get` and `go mod tidy`, with the
   Go toolchain of the pinned builder image and `GOTOOLCHAIN=local`, so its
   output is made the way `class-scope` recomputes it. Tool allowlists are a
   permission list, not a sandbox [I, from Claude Code's description of
   `--allowedTools` as a permission list], which is why the model holds
   nothing worth taking but its own credential. The job can reach the network,
   as a hosted runner allows; an injected instruction could send that one
   credential out, which is stated, not prevented. Upstream release notes
   reach the model as quoted data, and the skill states that upstream content
   is never an instruction (ADR-0006's prompt-injection risk).

   A second job takes exactly three files from the model's job as an
   artifact, `go.mod`, `go.sum` and the summary, refuses any other name, mints
   a one-hour installation token with `actions/create-github-app-token`
   (`owner` and `repositories: svc-hello` set explicitly, because from another
   repository the default scope is that repository [C, action README]),
   commits through GraphQL `createCommitOnBranch`, opens the pull request,
   and writes the run record: pull request number, head commit, body hash,
   queue item, run id and job. The merge step is two jobs of the same
   workflow that run first in each run. The first, in the ratchet's
   environment, runs the detector's checks (§9) and stops the merge if they
   fail. The second calls the merge API with the agent App's token on the one
   open agent pull request, passing the head commit `class-scope` verified,
   the squash method and a fixed title and message; GitHub's ruleset decides
   (405 if not mergeable). It closes a non-seed queue-item pull request whose
   `class-scope` or `go-test` has concluded fail; it never merges a seed
   (§11). It tells a fail from an inconclusive by reading the `class-scope`
   run itself through the Actions API, which step of the job failed, never
   from a label, which the agent App can write. Then, if no agent pull
   request is open, the model's job and the opening job run for the next
   item. "Allow auto-merge" stays off. A `fixtures` workflow, started by hand, holds the same token to make
   C-15's and C-21's attempts, one at a time, and records each response.

   With a two-file class the agent adapts no code: it does what a dependency
   bot does, plus the prose. That is enough for the ladder, which attaches
   trust to the class, never to the agent (ADR-0007), and it is what C-21 needs:
   a drivable non-human principal. A dependency bot as the author would not test
   C-21 and would need new dated claims, because C-16 and C-21 name an agent.

   Evidence is what trusted CI produced. The agent's prose is labelled "Agent
   summary (unverified)" and is not evidence (#27).

   This records the four decisions #38 asked of M3's note: one org-owned App
   is the non-human principal (here two, the second for demotion only); it is
   installed on `svc-hello` only, so `edge-config` never had it, and the
   org-wide `claude` App is narrowed off every repository M4 uses (§10);
   ruleset records stay in `platform-factory/.github` (ADR-0021 §10), with
   the ratchet's own records in the controls repo; and the rung mechanism is
   a required-reviewers entry (§3).

8. **Promotion: N = 20, frozen, and what the streak bounds.** The class is
   promotable after 20 consecutive merged class changes with no escape and no
   near-miss. A near-miss is a queue-item pull request, other than a seed,
   closed unmerged by anyone: by the merge step on a failed check, or by the
   reviewer under the readiness note's review rule. It is read from the close
   event, not from a label, so a close is never a free retry. A seeded defect
   (§11) is counted for the detection rate and neither counts nor resets. N,
   the queue, the reset rule and every number the readiness note fixes live
   in a versioned class file in the controls repo, not in the ruleset, which
   has no field for them, and are not tuned during M4. This supersedes
   ADR-0007's "N set per class in the ruleset, tunable".

   No risk ceiling is claimed. 20 is what a 25-step queue allows with five
   steps left for L2, so N is a mechanism choice. Zero failures in 20
   independent trials with perfect detection bounds the failure rate at 13.9%
   (one-sided 95%, #24); these trials are replays, not independent and not
   live, so the promotion is a proof of the mechanism, not of the class's
   safety. A class that deploys needs a ceiling and an N chosen against it;
   that is the prod-tag-bump class's decision. The rule's promotion odds and
   the bad merges each promotion is expected to let through are in the
   readiness note.

   The human login. The maintainer's second account becomes the only member
   of `platform` for the window, and the owner account leaves the team. No
   valid command-line credential for it is on the machine the delegate runs
   on: a stale token for it sat in that machine's keyring on 2026-11-11,
   already invalid, and the maintainer removes it before the team change. The
   maintainer's browser sessions are on the same machine, and the delegate
   does not use them, which is procedure. It is
   the one human login listed in the class file. So GitHub itself refuses the
   delegate's approvals on every entry, and `class-scope` ignores its labels.
   Admin hand edits (the promotion edit, a fixture edit, a lift) stay with
   the owner account, by procedure.

   Promotion is a person's act. At the 20th clean merge the scorecard job
   writes a promotion record to a branch and opens an issue; the delegate
   opens the pull request, which adds the record under `classes/`, links the
   scorecard at the streak's end and the ruleset diff it authorises. The
   listed human approves and merges it: that approval is the displaced
   approver's signature. The owner then applies the `minimum_approvals` 1 → 0
   edit by hand, recorded before and after. `platform`'s one member is the
   maintainer, who is also the L1 reviewer: the signature is a person's, not
   an independent one. Getting above L1 again after a demotion is a new
   promotion: a fresh streak of 20 and that signature. M4 makes at most two
   rung rises that are not earned, each a fixture record under `classes/`
   with the same signature and edit: the re-promotion after the first
   injection, and, if three queue items remain with no earned promotion, the
   entry to L2 that lets the L2 cases run. C-19 is graded on what happened in
   the second case.

9. **Demotion: any escape returns the class to L1, before attribution.**
   Escape types in M4:
   - (a) **failed verification after merge**: a merged class pull request
     whose merge commit has no `push: main` run of `go-test` whose first
     attempt succeeded within 30 minutes of the merge. A failed, cancelled or
     missing run all count, read per attempt; a re-run never clears one. A
     missing run covers `[skip ci]`, which suppresses `push` runs [C, GitHub
     docs, "Skipping workflow runs"];
   - (b) **revert for cause**: a merged pull request that reverts a class pull
     request and carries `escape:revert` in force from a listed human login.
     A person applies the label; that is the boundary of "automatic" for this
     type;
   - (c) **spine-attributed incident**: defined by ADR-0007, not run in M4 (§2).

   A detector in the controls repo runs every ten minutes, reads
   `svc-hello`'s workflow runs and their attempts, pull requests, label events
   and ruleset with the ratchet App's token, and matches type (a) and (b)
   events. It acts once per event, and only on events after the latest merged
   promotion or fixture record; each event it acts on goes to a ledger on the
   `records` branch. On one, the ratchet App sets the rung entry to 1, labels
   every open agent pull request `needs-human`, and writes the before and
   after ruleset JSON with its reason, the event's id. It acts on the trusted
   failure event, before anyone says why it happened; triage follows (#27).
   Its code only ever raises `minimum_approvals`, and checks before writing
   that the result differs from the live ruleset in that one field;
   otherwise it writes nothing and opens an issue in the controls repo.

   Scheduled runs can be delayed or dropped [C, GitHub docs, `schedule`], so
   the detector is not the only gate. The merge step runs the same checks
   before every merge, demotes first if it finds an unhandled event, and
   refuses to merge while `main`'s latest `push` run of `go-test` has not
   succeeded, while a ruleset difference stands (§10), or while the last
   complete detector run is older than 30 minutes. So a merge cannot land
   between an escape and its freeze. The minutes from the trusted event to
   the ruleset's new history version are measured. A bot outside the merge
   path with admin power over one repository's ruleset is the price of
   "automatic", and it is why C-15 is graded ADJUSTED.

   Demotion ends at L1 from any merging rung (#23). At L1 an escape resets the
   streak and the agent keeps writing; nothing demotes to L0. This supersedes
   ADR-0007's "demotes one rung". ADR-0007's freeze of every class that shares
   the escaped class's machinery stands, without the wait for triage, and is
   untested: M4 has one class.

10. **The ceiling above the ladder.** No class on `svc-hello` goes above L2 in
    M4, and the wall entry never goes below 1 approval. The maintainer sets
    both by hand. No promotion, scorecard value, agent action or evidence can
    change them: the agent App has no `administration`, and the ratchet's code
    only raises. That meets the prior-art obligation of a "fixed out-of-band
    ceiling above the top rung (who sets it, and that the ladder cannot)" [I,
    this repo's synthesis in `research/prior-art-landscape-2026.md:282-285`,
    from a preprint, confidence medium], with one limit stated: the ratchet's
    permission could lower the
    wall, so the ceiling is held by code.

    Each detector run therefore compares the live ruleset, read with the
    ratchet's token because `bypass_actors` "is only returned if the user
    making the API request has write access to the ruleset" [C, REST
    description, `GET /repos/{owner}/{repo}/rulesets/{ruleset_id}`], with the
    L1 JSON committed in the controls repo at the switch-on, whose hash is in
    the class file. It compares enforcement, conditions, every rule,
    `bypass_actors` and the list of rulesets on the repository, not
    timestamps. The L2 JSON is accepted only from a merged promotion or
    fixture record until the ledger's next demotion; from a demotion until a
    later record merges, only the L1 JSON is accepted. On any other difference
    it raises the rung entry to 1, records the difference as its reason and
    opens an issue, and the merge step refuses while the difference stands; it
    never loosens.

    The org-wide `claude` App sits outside this ceiling. The owner moves its
    installation to selected repositories that leave out `svc-hello`, the
    controls repo, the throwaway repository, `platform-factory/.github` and
    `platform-knowledge`, before either new repository exists. The L1 rung is
    not switched on while any App other than the two above holds write on
    `svc-hello` or the controls repo. The second obligation, "at least one
    non-agent-produced premise in every promotion decision", is met because
    every promotion in M4 is a person's approval and hand edit, on evidence
    from trusted CI and on seeds the agent did not choose; with one person in
    every seat, independence is by procedure.

11. **Seeded defects and shadow verdicts, so that zero is not blindness.**
    ADR-0020 names the problem for C-16. For a two-file class with §1's
    recompute, every defect in the diff that is known is stopped by a check:
    a downgrade, an unforced second module, a target other than the queue's,
    a directive change, a tampered or stale `go.sum`, a `replace`, an added
    module and an off-path file all fail `class-scope`, and the `go.sum`
    cases also fail `go-test`. No replayed release fails the suite, and a
    release that does cannot be made without a module that does not exist. A
    defect a check stops never reaches a review decision, so those cases are
    C-15's off-class attempts, not seeds.

    The four seeds are reviewer-facing: the diff is the correct queue item and
    the agent's summary misreports it, the defect class the M2 record shows
    twice [C, via #27: `docs/build-log/m2-paved-road.md`, "How it was
    authored, and what the gates caught"]. The readiness note lists six
    misreport types. The maintainer stores 32 random bytes as a controls-repo
    secret without displaying them. The opening job derives four positions
    and four types from an HMAC of that key, by a rule in the class file, and
    the key's SHA-256 is in the class file's first commit. A reveal job
    publishes the key at the close. Neither the delegate nor the reviewer sees
    the schedule. A seed is made by the same jobs as a real item, the opening
    job substituting the summary, and its logs print nothing that differs.
    The scorecard withholds per-pull-request rows until a pull request is
    closed or merged. A seed is never merged: after the reviewer's decision is
    recorded, the opening job labels it `seeded` and closes it. A seed whose
    head or body changed before the decision is void and reported as tampered.
    With one person, blindness is by procedure, not by access.

    On every L1 pull request the scorecard also records the shadow verdict:
    what "merge when the required checks are green" would have done, beside
    the person's decision (#24's checks-green baseline). No control arm, no
    second reviewer, no power analysis and no simulator.

12. **Packet and scorecard, minimal.** The packet is `class-scope`'s job
    summary, shown in its check run, plus the required checks, the shape
    `research/approval-packet-2026.md` proposes [I, that digest's Part 4 is
    labelled inference]. The agent App can edit and delete issue comments
    [C, permission data, `pull_requests` write], and cannot write a check run
    without `checks` [C, same]; a job summary is therefore out of its reach
    [I]. A pull request comment links to the summary and is not the packet.
    The packet holds: queue position and item, target old → new, forced
    modules, the `go.mod` change in words, the `go.sum` lines changed, release
    age, an upstream link, a link to `go-test`'s govulncheck findings, the
    verdict with its reasons, the class file's commit, the verifier's version,
    and the controls run and job that opened the pull request. The agent's
    prose sits apart under its heading. The research's admission backstop
    (Kyverno verifying attestations) has no counterpart here, because nothing
    is admitted. The reviewer applies `reviewing` on starting to read, and
    review minutes are the interval from that label to the review; the
    reviewer's own minutes and the packet sections used are entered in the
    review body in a fixed form beside it, because C-16 asks for them. The
    scorecard is one job in the controls repo that writes one JSON and one
    Markdown file from pull request metadata, check runs, review and label
    events, the run records and the class file. Every human-entered field is
    listed in the readiness note.

13. **Knowledge: one skill, pinned and revertible (C-22, shrunk).** One skill,
    `skills/go-module-bump/SKILL.md` in `platform-knowledge`, declares in its
    front matter what it depends on and the commit at which it was last
    verified. It is released by tag and pinned in the class file by tag and
    commit; the agent's workflow refuses a tag whose commit differs. The pin
    pull request in the controls repo is the gate ADR-0006 asks for ("Cutting
    or advancing a release is itself a gated change"), and the `classes/**`
    entry makes it need the listed human's approval: a tag is not a pull
    request, so the gate is on what the agent runs. A hotfix is a new tag and a
    pin pull request, the same path. One freshness check, a `platform-knowledge`
    workflow on a schedule and on manual dispatch, reads each declared
    dependency's file at the dependent repository's `main` through the API and
    fails when it has changed since the verified commit. Not built in M4: the
    question bank, its grader, gap tickets, `CLAUDE.md`, `standards/`,
    `questions/` and #27's comparison arms. Two stated departures: the skill
    serves one service, which fails `knowledge-as-code.md`'s test for central
    placement ("if two service repos would copy the same skill, it's
    platform-scope"), and it is placed centrally anyway because ADR-0006's gate
    and the release pin live there; and ADR-0006's `skills/**` approval is not
    enforced in `platform-knowledge`, whose ruleset requires a pull request
    and no approval [C, ruleset 20204896, read 2026-11-11].

14. **Ceilings, fixed now under ADR-0020.** C-15 **ADJUSTED**: its text says
    "branch protection + CODEOWNERS on its paths alone", and the class is held
    by required-reviewers entries, a trusted check and a demotion App (§3–§5,
    §9). C-16 **PARTIAL**: one reviewer who built the system, seeds blind by
    procedure, and seeds that are misreports only (§11). C-17 **PARTIAL**: six
    fields are human-entered by design. C-18 **PARTIAL**: the spine type is
    not run and the revert type rests on a person's label. C-19 **ADJUSTED**:
    N and the rung move out of the ruleset (§8), and the result is a mechanism
    proof. C-20: none. C-21 **PARTIAL**: one App identity carries the model's
    run, the seeds, the fixtures and the merge step, and the owner's admin acts
    share a login with the delegate. C-22 **PARTIAL** (§13). The readiness
    note gives each claim's test, pass line and falsifier.

15. **Out of M4.** L3; a second class; attestation and provenance;
    deploy-and-observe; the CI image identity (§2); `template-service`;
    framing commitments 2 to 5 (the scorecard shown back to each approver,
    break-glass tracked as a confidence metric, a first permanently-manual
    designation, game days); a `docs/design/m4` slice doc; the question bank
    (§13). C-12's remainder stays unrun and its PARTIAL stands; C-18's type
    (c) is a case not run, listed under C-18's PARTIAL; the precondition for
    both is a class with a deploy event. C-25 is not run.

## Consequences

- **The ladder becomes ruleset values, seven programs and two Apps.** A
  reader can check a rung by reading one JSON field. The programs:
  `class-scope`, `go-test`, the agent's workflow with its merge step, the
  fixtures workflow, the detector and ratchet, the scorecard, and the
  freshness check. Around them: two new repositories (the controls repo and
  a throwaway probe repository), three labels a person applies (`reviewing`,
  `human-cleared`, `escape:revert`), three that machinery applies
  (`needs-human`, and `fixture` and `seeded`, which nothing reads), and two
  repository variables. The cost is machinery beyond what
  C-15 names, a grade stated before the run.
- **Every loosening is the maintainer's hand, every tightening may be a
  machine's.** Promotion, the fixture rises, any lift and each clearance are
  a person's acts; demotion and the ruleset comparison are the ratchet's.
  That is ADR-0007's asymmetry made literal.
- **One person with two accounts, and a delegate on one of them.** The
  maintainer is author of the controls code, L1 reviewer and promotion
  signer. The delegate runs the build through the owner account's
  command-line credentials, so GitHub's record cannot tell its acts from the
  owner's; it holds no credential for the reviewer account (§8), so every
  approval and every label that counts is checkable from the record. The
  owner's admin hand edits remain by procedure, and the classifier refuses
  loosenings to the delegate.
- **The run is compressed.** At two agent runs an hour the queue can finish
  in a few sittings. Wall-clock is recorded, no soak is required, and an
  escape that would appear only later is not observable.
- **Alternatives not taken, one line each.** A dependency bot as author: does
  not test C-21. A bot feeding the agent: mixed authorship on one pull request.
  A person editing the ruleset on an escape: merges land until someone acts. A
  merge queue reading the scorecard: more machinery, and the merge REST call
  does not work with it. Auto-merge: not needed when the merge step asks and
  the ruleset decides. A standing bypass actor: the delegate's login (§4). An
  organisation required workflow as Lock 2: it needs organisation admin, runs
  on default activity types only, and a `pull_request` job in its own
  workflow gives the same guard more cheaply. Giving the model the App token:
  it would hold every power C-21 tries.
- **What it supersedes in earlier ADRs.** ADR-0007's rung mechanism, its
  "changing a rung is itself a PR to the ruleset" (promotion is a merged pull
  request in the controls repo plus a recorded hand edit), "demotes one rung",
  "N set per class in the ruleset, tunable", its near-miss rule and the
  triage-first order of its shared freeze; ADR-0017's open choice of M4's
  first bump; ADR-0021 §11's "at M4" for C-12's remainder and §16's deferred
  App. It gives ADR-0006's release pinning a mechanism. `docs/adr/README.md`
  carries each pointer; the earlier ADRs are not edited.
- **Still open, each with the step that settles it.** The GitHub behaviours
  above marked "read from the docs, not tried": probes P1 to P4, with the
  fallbacks in the readiness note. Whether `createCommitOnBranch` commits
  show as signed and attributed to the App, and whether `merged_by` names
  `<slug>[bot]`: probe P4; neither changes a decision, because the pull
  request author is the unforgeable field. Which App the org-wide `claude`
  installation is: the owner's read of the installation.
- **Not decided here.** A risk ceiling for any class (§8); N for any class but
  this one; who applies a shared-machinery freeze across classes; what the
  notified person at L2 does beyond being told and the stop in §3; the stale
  lines outside M4's files (the root README's Gateway sentence, the `.github`
  profile's "Where the build is", ADR-0001's "light review" line, which
  ADR-0006 revises for `skills/**` and `standards/`, and ADR-0016 §4's
  provider-bump sentence), left to a sweep.
- No grade of an earlier milestone changes, and C-12's PARTIAL stands.
