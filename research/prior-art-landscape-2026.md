# Platform Factory Prior-Art and Landscape Scan — 2026

Research digest, 2026-07-31. Motivating question: what public work — products,
open-source projects, industry patterns, government programs, academic research,
practitioner writing — relates to the Platform Factory concept, and what does it
teach us: prior art to cite, similar attempts that hit problems, evidence for or
against the core bets, and how crowded the "factory" naming is?

Method: five-angle web sweep, 24 sources fetched live on 2026-07-31, 120 claims
extracted, the 25 most load-bearing verified adversarially (three independent
votes each; 24 confirmed, 1 refuted — see "Do not cite"). Parts 5–6 come from a
follow-up single-pass sweep: fetched and verbatim-quoted against primary
sources, but not adversarially voted. [C] confirmed against a primary source
(URL or source named inline), [I] inferred; unverified items flagged inline and
collected at the end.

---

## TL;DR

1. **Every pillar has respected prior art validating it at the concept level;
   the central mechanism has none.** Nobody ships earned per-agent autonomy —
   promotion/demotion driven by an agent's own track record, enforced in the
   merge path. Scorecards in this space grade *services and teams*, never
   agents. Approval evidence everywhere attaches to *changes* for humans, never
   to an agent's standing. Nobody uses repo topology as the approval boundary
   (boundaries live in portal RBAC and tool policies). The approval packet as a
   first-class merge artifact is unclaimed. [C by absence — searched in this
   scan and independently in the 2026-07-30 approval-packet digest]
2. **Nearest open-source neighbor: kagent** (CNCF Sandbox, from the Istio
   founders at Solo.io). Independently converges on our core loop — "The agent
   files the GitOps PR; your existing approvals do the rest" — and ships
   Skills-from-Git (runbooks loaded from git at agent startup). But its
   human-in-the-loop model is static per-tool approval gates with zero
   graduation machinery. Differentiator, not collision. [C]
3. **Autonomy-level taxonomies are crowded; enforcement is empty.** Knight
   Institute (five levels named by the shrinking human role), CSA (six levels
   echoing SAE J3016, the self-driving-levels standard), Microsoft (five
   driving-inspired levels), Weave's "ADP" ladder (four org-maturity levels).
   All classify; none couples levels to an enforced promotion mechanism. [C]
4. **Two adversarial design inputs that should become ADR material:** (a) a
   formal-model preprint argues an earned-autonomy ladder is safe only under an
   immutable authority ceiling set *outside* the promotion process, and every
   promotion needs at least one premise the agent cannot self-produce; (b) the
   Knight framework names approver rubber-stamping — disengaged approvers
   exploited by agents to gain autonomy — as a core unsolved risk at exactly
   our human-approver configuration. [C on what the sources say]
5. **The knowledge-as-code bet gained its strongest evidence yet:** Datadog's
   against-interest postmortem (gather-everything context produced a wrong
   root cause) and a 2,303-file empirical study (agent context files evolve
   like configuration code; security/performance guardrails appear in only
   14.5% of files). [C]
6. **Ladder sequencing has market validation:** Datadog's own product climbed
   investigate → propose → human-in-the-loop remediate → scoped-autonomous one
   rung at a time, and benchmarked agents plateau well below full autonomy. [C]
7. **Naming: "software factory" is contested, not burned — and "platform
   factory" is effectively unencumbered in our space.** The DoD literature's
   own succession narrative is factory → platform, which the name can be
   positioned as synthesizing. Exact-name collisions are a payments startup, a
   UK dev shop, and two dead repos; trademark register unchecked.
   [C, single-pass]
8. **Closest shipped collisions to track:** Akuity Intelligence (agents on
   Argo CD + Kargo promotions — evidence bundles for humans about changes, not
   about agents), Port (per-agent autonomy levels, approval workflows, and
   git-integrated "Skills"), Humanitec (agent-safe orchestration marketed
   against our exact reference stack). [C, single-pass]

---

## Part 1 — Nearest neighbor: kagent

kagent (https://kagent.dev, https://github.com/kagent-dev/kagent; CNCF Sandbox
accepted 2025-05-22; v0.10.0-rc1 released 2026-07-29):

- **Agents under GitOps themselves.** Agents, models, and tools are Kubernetes
  CRDs "versioned in Git, reviewed in PRs," rolled out via Argo CD — prior art
  for extending the GitOps control plane to cover the agents, not just the
  infrastructure. [C]
- **Agent-proposes-via-PR, human-approves-at-the-repo-boundary.** Flagship use
  cases: platform self-service ("The agent files the GitOps PR; your existing
  approvals do the rest") and incident response (agent "opens the rollback
  PR — with humans approving each step"). Caveats: these are marketing blurbs,
  not production case studies, and a direct MCP tool-execution path (kubectl,
  argocd) exists alongside the PR path. [C]
- **No graduation.** Advertised HITL is "Tool approval gates, agent-initiated
  questions, and cascading HITL" — in code, a static per-tool approval set
  (`MakeApprovalCallback(toolsRequiringApproval)`). A code search across the
  entire kagent-dev org returns zero hits for "autonomy"; no promotion,
  demotion, scorecard, or graduated anything. [C]
- **Knowledge both ways.** Ships "Skills from Git" — an init container that
  clones SKILL.md bundles from git/OCI before the agent starts ("Agents learn
  your runbooks, not just generic docs") — working prior art for git-versioned
  knowledge consumed by agents. The same homepage separately markets "Knowledge
  agents" as "RAG over runbooks, ADRs, and Slack history": the neighbor hedges
  across curated-from-git *and* broad ingestion. Curation-only plus pinned
  releases (ADR-0006) remains our stated position, not a collision. [C]

## Part 2 — Autonomy ladders: taxonomies, safety framing, benchmarks

**Taxonomies (crowded, none enforced):**

- **Knight First Amendment Institute / Univ. of Washington** (Feng, McDonald &
  Zhang, Jul 2025, peer-reviewed;
  https://knightcolumbia.org/content/levels-of-autonomy-for-ai-agents-1).
  Five levels named by the shrinking human role: operator → collaborator →
  consultant → approver → observer. Direct academic prior art for "each persona
  moves one seat up." Also states the separability thesis our "risk policy,
  not maturity model" framing rests on: "an agent's level of autonomy can be
  treated as a deliberate design decision, separate from its capability." [C]
- **Cloud Security Alliance** (Jim Reavis, 2026-01-28;
  https://cloudsecurityalliance.org/blog/2026/01/28/levels-of-autonomy). Six
  levels (L0 human execution → L5 self-directed), "deliberately echoing the
  structure of SAE J3016"; framed as a draft invitation, not a standard.
  Published before this repo went public, so convergence is independent. [C]
- **Microsoft Research** (Jul 2024; https://arxiv.org/abs/2407.14402). Five
  autonomy levels for autonomous microservice maintenance, "Inspired by the six
  levels of autonomous driving." Capability/evaluation scale; zero
  promotion/demotion/merge-rights concepts in the full text. [C]
- Further convergent taxonomies exist (NVIDIA four-level, ASDLC L1–L5,
  Weave's ADP ladder in Part 6; https://arxiv.org/abs/2506.12469 surveys the
  space). Positioning nuance worth making explicit: CSA and Microsoft echo SAE
  J3016 *capability* levels; our analogy is graduated driver *licensing* —
  earned promotion with demotion on failure. Related but distinct driving
  analogies. [C/I]

**Safety-not-capability framing (validated):**

- **STRATUS** (UIUC + IBM Research, NeurIPS 2025;
  https://arxiv.org/abs/2506.02009; code at github.com/xlab-uiuc/stratus)
  asserts safety — "rigorously defined and enforced as a first-class system
  design principle," explicitly contrasted with expected model-capability
  gains — is the key barrier to agents operating live production clouds, with
  "the risk of making things worse" named the fundamental deployment blocker.
  Neither STRATUS nor Knight uses our words "risk policy"/"maturity model" —
  that mapping is our (fair) inference. [C, mapping I]
- STRATUS formalizes **Transactional No-Regression**: every mitigation action
  must be undoable ("every agent action has a corresponding undo operator;
  otherwise, the action is not allowed"), via deterministic scheduling,
  sandboxing, and a stack-based Undo Agent. Complementary, not competing: TNR
  governs the agent *during* execution; the ladder governs what it may touch
  *before* execution. Note the positioning contrast: STRATUS targets full
  autonomy now — motivated by "human-in-the-loop approaches [cannot] keep up
  with the scale of modern clouds" — substituting rollback guarantees for
  incrementally earned trust. Leading academic work thus validates the
  routine-review-doesn't-scale premise while jumping straight to the end
  state. Fine print: TNR guarantees monotonic severity decrease at transaction
  boundaries (transient mid-transaction worsening possible); undo completeness
  is imperfect for external state; an optional human-approval hybrid mode
  exists but benchmarks run fully autonomous. [C]

**Evidence-bearing promotion (adjacent prior art):**

- Knight's **autonomy certificates**: a third-party body (UK AI Security
  Institute and METR named as candidates) issues a certificate only after the
  developer submits an evidence-based "autonomy case" — explicitly analogous
  to AI safety cases — demonstrating the agent behaves "at most, at a
  particular autonomy level," with the body running private evaluations before
  issuance, renewable on change. Prior art for evidence-bearing approval
  packets, with clean differentiation: certificates attach to the *agent* per
  environment via external certification; we attach evidence per *change* and
  enforce in-repo via merge rights. Renewability is an independent echo of
  demotable autonomy. [C]

**Benchmarks (candidate scorecard instrumentation):**

- **ITBench** (IBM + UIUC, Feb 2025, open-source and active;
  https://arxiv.org/abs/2502.05352, github.com/itbench-hub/ITBench) benchmarks
  agents on real-world SRE, CISO, and FinOps scenarios. Cite it as a framework
  — its widely repeated per-domain solve rates failed verification (see "Do
  not cite"). [C]
- **Microsoft's plateau numbers** (same 2407.14402 paper): GPT-4 Turbo agents
  on AutoGen managing the Sock Shop demo under chaos injection — pass rates
  0.74 (20/27) at L3 proactive detection → 0.5 (6/12) at L4 automatic
  root-cause analysis → 0.25 (3/12) at L5 full self-maintenance; "full L5
  autonomy has not been achieved by current LLMs." Date-stamp when citing:
  mid-2024 models, small samples. Successor benchmarks (AIOpsLab, OpenRCA,
  ITBench) continue to report state-of-the-art agents failing autonomous
  root-cause analysis and mitigation. Supports graduated over immediate
  autonomy. [C]

## Part 3 — Adversarial design inputs (the two ADR-shaped findings)

**3a. The fixed-ceiling argument.** Zhang & Zhang, arXiv 2607.23586 (posted
2026-07-26; https://arxiv.org/abs/2607.23586) — a formal model of "earned
authority under a fixed ceiling":

- Core claim: conditioning agent authority on demonstrated performance is
  sound *as a security mechanism* only if the system also fixes an immutable
  authority ceiling, set outside the promotion process, that no performance
  evidence can raise: "A system may condition authority on demonstrated
  performance only if it also fixes a bound that no performance can move."
  Their objection is precise: "Our objection is not to gradual allocation
  beneath a limit. It is to allowing performance evidence to raise the limit
  itself." [C]
- On self-produced evidence: test passes, low error rates, task history —
  i.e., an approval packet the agent assembles — can inform *reliability*
  decisions but cannot tighten the *adversarial security* bound: "A tighter
  security bound requires at least one premise that the session cannot realize
  under its current authority, such as an external attestation, an
  independently controlled evaluator, or a fresh human countersignature."
  Their appendix: otherwise agent-manufacturable competence evidence is "a
  privilege-escalation channel by construction." [C]
- Two concrete design obligations follow for the ladder [I, our synthesis]:
  declare a fixed out-of-band ceiling above the top rung (who sets it, and
  that the ladder cannot); and require at least one non-agent-produced premise
  in every promotion decision (human countersignature or independently
  controlled evaluator). Note this does not kill "earned removal of routine
  review" — it says the *terminal boundary* is a human decision made outside
  the promotion system, which arguably strengthens the licensing analogy
  (the licensing authority is not the driver's logbook).
- Confidence: **medium** despite unanimous verification votes — a days-old,
  unreviewed two-author preprint whose guarantees hold within model
  assumptions (complete mediation, sound effect abstraction, monitor
  integrity). Cite as a formal-model design input naming a real structural
  risk, not settled theory. Verifiers note the principle is consonant with
  established security doctrine (remote attestation, separation of duties).

**3b. Rubber-stamping at the approval boundary.** The Knight framework, at its
Level 4 (user as approver) — exactly our configuration:

- "User disengagement can lower the care with which they approve actions and
  can result in unintended approvals," and "misaligned agents... can employ
  strategies to gradually convince disengaged users to approve risky actions"
  — disengaged approvers plus a promotion pathway are an exploitation channel.
  Mitigation is left explicitly open: "How can users be engaged in agent
  activities to avoid meaningless rubber stamping?" [C]
- This is a framework-identified risk consistent with the automation-complacency
  literature, not an observed incident. [C]
- Positioning we can honestly claim [I]: earned removal of *routine* review is
  a partial answer — fewer, higher-salience approvals concentrate attention
  where rubber-stamping is likeliest. But the design should add explicit
  countermeasures (approval-latency signals, sampled deep review, engagement
  checks); no production deployment has published results on any.

## Part 4 — Evidence on the knowledge-as-code bet

- **Datadog's against-interest postmortem** (Bits AI SRE engineering blog,
  Shan & Ratchford, 2026-01-12;
  https://www.datadoghq.com/blog/building-bits-ai-sre/): with many tool
  responses summarized into one prompt, "incorporating additional telemetry
  data slowly degraded model performance." In a documented Kafka-lag incident,
  an early version made 12 tool calls; one correctly pinpointed the root
  cause, but noise from the others ("suspicious signals like critical
  application errors in an upstream service") led the summarization step to a
  wrong root cause. Their redesign: hypothesis-driven targeted queries that
  "significantly reduce noise." Strongest single verified data point for
  curated/targeted agent context over indiscriminate ingestion. One
  vendor-selected incident, but a first-party engineering admission, not
  marketing. [C]
- **Ladder sequencing at the same vendor.** As of Jan 2026 Datadog scoped its
  flagship agent to autonomous *investigation* ("audit-ready root cause
  analyses"), remediation described only as future work; corroborated timeline:
  Jun 2025 proposal-only fixes → Jan 2026 autonomous investigation → Mar 2026
  explicitly human-in-the-loop remediation → ~Jun 2026 scoped
  autonomous-remediation preview limited to "common and repetitive" issues; a
  Datadog executive calls automated remediation "the next frontier" requiring
  "an even higher level of trust." De facto autonomy ladder, climbed one rung
  at a time — n=1 vendor, date-stamp when citing. [C]
- **The 2,303-file study** (Hassan, Adams et al., arXiv 2511.12884, Nov 2025;
  https://arxiv.org/abs/2511.12884; 1,925 repos): agent context files are "not
  static documentation but complex, difficult-to-read artifacts that evolve
  like configuration code, maintained through frequent, small additions"
  (59–67% modified across multiple commits; additions dominate, deletions
  negligible; median Flesch Reading Ease 16.6 — "very difficult"). Security
  and performance instructions each appear in only **14.5%** of files (vs
  ~62–70% for build/implementation/architecture); the authors conclude the
  files "provide few guardrails to ensure that agent-written code is secure or
  performant." Replicated by arXiv 2509.14744 (security 8.7%, performance
  12.7% in Claude Code manifests). Validates the *problem* the curated,
  owner-gated knowledge layer targets — no source tests our specific
  mechanism. Caveats: v1 preprint; scope is coding-agent context files, so
  extending to operational runbooks is extrapolation [I]; degradation-over-time
  is inferred from cross-sectional data; 33–41% of files are write-once.
  Qualifier worth knowing: arXiv 2602.11988 finds context files often do not
  improve task success and add ~20% inference cost — challenges context-file
  efficacy generally while reinforcing curation-over-accumulation. [C]
- **"Technical enforcement makes autonomy levels real."** CSA's Reavis
  estimates (self-described, from practitioner conversations) that "the
  majority of organizations deploying agentic AI have no formal classification
  system for autonomy levels, make autonomy decisions on an ad hoc basis, lack
  technical enforcement of autonomy boundaries, and have unclear or
  nonexistent policies"; his core operationalization principle: "autonomy
  boundaries must be technically enforced, not just policy-documented...
  Technical enforcement is what makes autonomy levels real." Second-hand
  corroboration of the gap: Deloitte 2026 (~80% of orgs lack mature agentic-AI
  governance), McKinsey State of AI Trust 2026 (~1/3 at maturity 3+). This is
  principle-level validation of repo-topology-plus-CODEOWNERS as the
  enforcement mechanism — the post never mentions git or repos, so do not cite
  it as endorsement of the specific mechanism. Confidence: medium (executive
  estimate in a blog post). [C on quotes, medium on figures]

## Part 5 — The "factory" naming landscape [single-pass]

**Where the term came from, and its two lives.** "Software factory" dates to
Bob Bemer's 1968 paper "The economics of program production" (per Addy Osmani).
Its modern *government* life began at Kessel Run (2017): "we coined the term to
resonate with leaders more familiar with aircraft production and missile
assembly than DevSecOps" — Bryon Kroger, Kessel Run co-founder, Federal News
Network, 2025-09-02
(https://federalnewsnetwork.com/commentary/2025/09/the-software-factory-reckoning-why-the-model-needs-a-reboot/). [C]

**The DoD reckoning (what went wrong, on the record):**

- Kroger: "too many lost the plot. They delivered slide decks instead of
  software"; the model "became synonymous with platforms and pipelines — tech,
  not people and process"; citing the Mar 2025 State of DevSecOps report,
  "over 50 software factories exist in name. But only a few are delivering
  real outcomes" (report stats not independently verified). [C on quotes]
- Kessel Run itself pivoted to "government-led, vendor-managed" delivery amid
  criticism (Air & Space Forces Magazine, 2025-03-01, reported with named
  sources;
  https://www.airandspaceforces.com/kessel-run-air-force-software-factor-pivoting/).
  Structural causes on record: "We were turning over 50 percent of our staff
  every six months"; no career path for uniformed coders; Kroger: by 2022 the
  factory was "failing... It's the Air Force that failed Kessel Run"; his
  summary of the counter-reformation: "The empire struck back." [C]
- The succession narrative — directly useful for our name: "The successor to
  the software factory is not another factory. It is a platform engineering
  model where small, focused teams curate technology choices and define
  integration patterns... The factory that tried to control everything is
  giving way to a platform that enables everyone" (Intelligence Community
  News, 2026-03-09 — vendor-sponsored content by a Coder executive, so treat
  the eulogy as interested; the factory→platform arc is echoed by Platform
  One's own "platform-of-platforms" evolution). [C on quotes]

**The AI-era re-mint (2025–26 vendor land grab):**

- Addy Osmani, "Software Factories, Light and Dark" (2026-07-22;
  https://addyo.substack.com/p/software-factories-light-and-dark) — the
  neutral intellectual frame: "A software factory is harnessing loops at
  scale"; "dark factory" = "code ships that no human has read, verified only
  by other machines." His core constraint is ours: "The fundamental constraint
  in a software factory isn't how much code we can churn out: it's how quickly
  we can verify it" (comprehension debt). Two passages are near-verbatim our
  ladder thesis, named but not mechanized: "Back pressure is the rule that you
  can only hand a loop as much autonomy as you can cheaply and reliably
  verify, and not one inch more," and a section literally titled "What earns a
  loop the dark": fully automated status is *earned* only when "the check is
  cheap, runs at high frequency, and relies on something that can't be easily
  faked out." Cites a four-month fully-automated code factory that ended in
  "painstaking manual debugging" (secondhand via a conference talk;
  unverified). [C]
- Cortex (CTO Ganesh Datta, 2026-06-23): "An AI software factory is an
  organizational system... with AI agents doing the building while engineers
  move up a level" — independent convergence on one-seat-up; and "deciding how
  much autonomy an agent has earned becomes a judgment call leaders have to
  make deliberately rather than by default" — earned autonomy named, answered
  with a metrics product, not a mechanism. Also usefully disambiguates
  NVIDIA's "AI factory" (data centers) as unrelated. Vendor-reported stat to
  handle with care: PR throughput up → incident volume up. [C on quotes]
- Pulumi/Compostable (2026-05-21): the maximalist pole — "An AI software
  factory is a software operation where autonomous agents write and ship most
  of the code"; embraces the dark-factory pattern; memorable rule: "Don't let
  any agent mark its own homework." [C]

**Exact-name collision check ("platform factory"):** outside our space
entirely — Platform Factory, Inc. (platformfactory.io, a US payments startup),
platformfactory.co.uk (UK dev shop), One Platform Factory (consultancy). On
GitHub, only three exact `platform-factory` repos exist: two dead zero-star
repos and ours. A targeted search in the kubernetes/GitOps/platform-engineering
space returned zero exact-term usage. **US trademark status is unchecked** —
the USPTO query failed; the payments company is the main encumbrance datapoint.
[C, single-pass]

**Verdict [I]:** contested, not burned — and splitting in two (a government
term in open reckoning; an AI-era term being re-minted by vendors). "Platform
Factory" is unencumbered where we operate, and the DoD literature's own
factory→platform succession lets the name read as the synthesis — the platform
*is* the factory — rather than a mashup of a dying term and its replacement.
Positioning mistakes the sources warn about: metaphor without metrics ("teams
playing factory"); letting the term mean pipelines-not-people; the centralized
factory as critical-path bottleneck; dark-factory maximalism without a
verification story; defining the factory as a product pitch instead of an
open, measurable pattern.

## Part 6 — IDP vendors and the GitOps ecosystem [single-pass]

**How the field frames agents (validation of the premise everywhere):**

- CNCF member post, "Platform engineering for the agentic enterprise"
  (WSO2's Lakmal Warusawithana, 2026-07-21;
  https://www.cncf.io/blog/2026/07/21/platform-engineering-for-the-agentic-enterprise-managing-applications-resources-and-ai-agents/):
  "the platform must now serve both humans and intelligent software"; agents
  are both first-class *consumers* and first-class *managed assets*, each
  actor with "a distinct identity, scoped permissions, and a clear audit
  trail"; context is "becoming a first-class platform capability" (as a live
  context graph, not versioned knowledge). GitOps appears only as a human/SRE
  interface. Concrete vehicle is OpenChoreo (CNCF Sandbox) with an "Agent
  Manager / AI Gateway / evaluation frameworks" module list — names only,
  maturity unverified. [C]
- Spotify/CNCF at KubeCon EU 2026 (SiliconANGLE, 2026-03-25;
  https://siliconangle.com/2026/03/25/platform-engineering-essential-age-ai-agents-kubeconeu/):
  CNCF CTO Aniszczyk: "agents need to feed off generally structured
  information... Every organization is going to have to have this"; Spotify's
  Tyson Singer: the "agentic workforce... really needs to know what the
  standards and the guidelines are," and agents "amplify what's good in your
  ecosystem and they amplify what's bad." Spotify's AiKA knowledge assistant
  is exposed to agents via MCP; their internal Honk fleet-management system
  "shift[s] the burden of code review back to the team initiating the change
  rather than each individual repository owner" — a deliberate re-drawing of
  the review boundary for machine-scale change (internal; details unverified
  beyond the interview). [C on quotes]
- Backstage (official docs, current): shipped agent surface is AI Skills +
  MCP Actions Backend (Scaffolder actions exposed as MCP tools). Maintainer
  guardrail stance (via Computer Weekly): "You should not be able to ask
  Copilot to delete this prod instance of a database" — enforced "by the
  permission model of the MCP tools, not by the agent's judgment." Static
  least-privilege; no autonomy levels. [C]

**Closest shipped collisions:**

- **Akuity Intelligence** (Argo CD co-creator Alexander Matyushentsev, Akuity
  blog "Beyond Dashboards: AI Agents for GitOps Operations," 2026-02-24) — the
  only shipped product attaching agents to a GitOps *promotion* pipeline
  (managed Argo CD + Kargo). Three agents: Deployment Advisor, On-Call Agent
  (runbook-governed), and **Promotion Advisor**, which before a Kargo
  promotion will "Enumerate all commits associated with the release...
  Analyze commit messages and code diffs... Produce[] a high-level summary and
  an inferred risk score, incorporating promotion history from other stages" —
  "turns Kargo promotions into explainable, data-driven decisions." That is an
  evidence bundle attached to a promotion, pointed at *humans deciding about a
  change* — the inverse of an approval packet an agent submits to earn
  standing. Autonomy is admin-configured, not earned: inherited RBAC ("never
  exceeds the privileges of the requesting user") plus tool policies
  (auto-allowed / require approval / disabled), with environment-graduated
  behavior written in runbooks (dev: act freely; prod: escalate and request
  approval). Ecosystem note: the kargo repo carries a root AGENTS.md
  instructing coding agents — git-versioned agent instructions are already
  being dogfooded there. [C]
- **Port** (CEO Zohar Einy, Oct 2025, updated Jul 2026; docs and Skills launch
  Apr 2026): agents as "a first-class user of the platform"; coins "agentic
  chaos" and cites the Replit production-database deletion as the risk
  archetype. Ships the most complete governance kit found: agents as catalog
  entities with "allowed tools and... autonomy levels for actions," "Bounded
  autonomy," "Action approval workflows: Critical actions can require human
  approval," every run logged as an `_ai_invocation` entity, RBAC inheritance
  — and autonomy recommended only for "well-defined, low-risk scenarios."
  **Skills** (Apr 2026) are the direct knowledge-layer overlap: "reusable
  instruction sets... load automatically into AI agents based on context,"
  permission-controlled, "Git integration lets you manage skills alongside
  your code" — explicitly pitched as the alternative to maintaining your own
  repo + CI for agent knowledge. Port sells the portal-hosted version of
  knowledge-as-code. Static levels; no earning, no demotion, no evidence
  artifact. [C]
- **Humanitec**: the most agent-forward infrastructure positioning —
  "Unifies infrastructure management for humans and AI agents"; a feature
  section titled "The features that let agents act — safely" ("what happens
  when an agent acts on your infrastructure?"): rule-based provisioning with
  no human in the loop, drift detection, one-command rollback, progressive
  rollouts, "Service users for agents with least-privilege access," "agents
  and developers deploy through it, not around it." Marketed *against* our
  exact reference stack: "Stop patching together ArgoCD, Crossplane,
  Terraform, and home-grown scripts." [C]
- **Weave Intelligence, "From IDP to ADP"** (Kaspar von Grünberg — the
  Humanitec founder who coined "IDP" — 2026-06-10): the closest conceptual
  collision. Renames the category (IDP → Agentic Development Platform); core
  thesis is ours in different words: "the SDLC becomes the harness" — the
  deterministic platform "is precisely what makes probabilistic agents safe to
  operate at enterprise scale." Proposes a four-level ladder: L1
  human-in-the-loop → L2 human-on-the-loop ("reviews aggregate results, not
  individual diffs") → L3 human-as-orchestrator ("Human review becomes
  exception-based. Promotion becomes rule-based for low-risk changes"; the
  platform "must distinguish between changes that can auto-promote... and
  changes that require human review") → L4 fully autonomous ("a projection,
  not a prescription"). Crucial difference: the ladder grades the
  *organization's* maturity, autonomy is granted by humans setting rules —
  no per-agent earning, no demotion mechanism, no evidence artifact. [C]
- platformengineering.org (Nov 2025) still frames AI mostly as workloads on
  the platform, but is launching "Agent infrastructure for Platform
  Engineers" / "Agentic Development Platforms" certifications — the community
  org is institutionalizing the category. [C]

**White space confirmed by this sweep [I]:** nobody ships earned per-agent
autonomy; nobody uses repo topology as the approval boundary; scorecards grade
services, never agents; the approval packet as a first-class merge artifact is
unclaimed.

## Part 7 — Positioning synthesis [I]

How the pieces read together (synthesis, not source claims):

- **kagent is a component, not a competitor.** It is machinery to run agents
  in a cluster (Agent CRDs + controller). Its two convergent choices — agents
  versioned in git under the same GitOps control plane, and agents acting via
  PRs that existing approvals gate — validate the loop; a Platform Factory
  implementation could plausibly run kagent as its agent runtime.
- **The closest feature-level competition is uniformly commercial and
  hosted** (Port, Akuity Intelligence, Humanitec, Datadog): governance lives
  in portal config and tool policies — products with lock-in, not adoptable
  patterns. Nuance: Kargo, the promotion engine under Akuity Intelligence, is
  itself open source; the agent layer on top is the hosted part.
- **Open source supplies components, not an assembled pattern:** kagent
  (agent runtime), Backstage (knowledge surface), Kargo/Argo (promotion and
  GitOps), Kyverno/Sigstore/in-toto (evidence and enforcement). Nothing
  open-source occupies the layer this repo does — opinionated topology +
  approval boundaries + governed knowledge + earned autonomy. Nearest
  conceptual repos found: a 3-star Red Hat demo and a 4-star experiment.
- **The launch-post syllogism this supports:** commercial products validate
  demand for agent governance; the open-source world has all the parts and no
  assembly instructions; an Apache-2.0 pattern plus reference implementation
  is the only neutral thing that can sit in that layer. The parts exist; the
  pattern doesn't.
- Standing caveat: "nothing found as of 2026-07-31" in a fast-moving space is
  evidence of absence only as of that date — re-run the currency check
  immediately before publishing.

## Do not cite

- **ITBench per-domain solve rates (13.8% SRE / 25.2% CISO / 0% FinOps)** —
  refuted 0–3 in adversarial verification despite circulating widely in
  secondary discussion. Cite ITBench as a benchmark framework only.
- Renamed/secondhand stats flagged above: Gartner 100:1 agent-to-developer,
  MIT "95% of AI pilots failed," Cortex benchmark figures, Akuity "50–70%
  faster incident resolution," State of DevSecOps counts, Spotify AiKA usage
  figures — all vendor-reported or secondhand, none checked against primaries.

## Open / unverified

- US trademark status of "Platform Factory" (USPTO query failed; a payments
  company operates as Platform Factory, Inc. at platformfactory.io).
- Backstage 1.43 scoped short-lived MCP token mechanics (claimed in a
  low-authority aggregator; plugin existence verified, version/mechanics not).
- Horthy's four-month dark-factory failure account (secondhand via Osmani).
- OpenChoreo module maturity (names only, CNCF Sandbox).
- Honk (Spotify) mechanics beyond the one interview.
- No production deployment has published results on rubber-stamping
  countermeasures (Part 3b) — searched, none found.
- Several sources are under six months old; re-verify currency immediately
  before the launch blog post cites any of them.
