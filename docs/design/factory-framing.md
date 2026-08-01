# Explaining the Factory

Design doc, July 2026. The articulation layer: how to explain the
platform-plus-factory pattern so it lands with every role it touches. The
technical docs (`platform-pattern.md`, `knowledge-as-code.md`) say what the
system *is*; this doc says how to *tell* it — in the README, the launch blog
post, a talk, or an executive review. It ends with the design commitments the
framing generates, because a good frame is not decoration: it tells you what to
build.

Origin: two framings first used in 2019 to articulate an infrastructure vision
at fintech scale — start from personas, and explain automation maturity through
the levels of autonomous driving. Both survive the agentic era. Each needs
exactly one update, and the analogy has become *more* accurate, not less: in
2019 it was a metaphor, because Kubernetes decides nothing novel. Now there is
an actual driver-like actor making judgment calls — the agent — and the driving
world's hard-won lessons transfer almost mechanically.

## The subject of automation moved

The 2019 ladder ran: racking-and-stacking (SSH, hand-built systemd units) →
virtualization → cloud → containers → managed Kubernetes. Each rung automated
more of the *machines*. That ladder topped out around 2020 — GitOps plus
operators plus managed Kubernetes means the machines now largely drive
themselves.

The residual toil was never machine operation. It is the knowledge work around
the machines: deciding, writing, reviewing, patching, responding. The factory
model (ADR-0007: agents author, review, and test changes; humans set intent and
gate) automates *that* — the 2026 ladder climbs the engineering work itself.
Same ladder shape, new subject.

One usage change matters as much as the subject change. In 2019 the ladder was
a **maturity model**: one number for the org, and the implied goal was "climb."
In 2026 it is a **risk policy**: every change class sits on its own rung,
"what level are we at?" has no answer, and some classes should stay at
human-approves *forever* — a correct policy choice, not immaturity. A maturity
model says climb; an autonomy policy says set the dial per domain and move it
on evidence.

## Borrow the driving concepts, not the numbers

The automotive standards body (SAE) defines driving automation levels 0–5; the
ladder in ADR-0007 is L0–L3 with different meanings. Lining the numbers up
confuses readers — borrow the four concepts instead, which map one-to-one onto
the ladder's machinery:

1. **The operational design domain → the change class.** A robotaxi is not
   "level 4"; it is level 4 *in Phoenix, in good weather* — autonomy is always
   scoped to a bounded domain, never granted globally. The factory equivalent:
   trust attaches to a change class (a kind of diff, on defined paths), never
   to an agent, enforced by branch protection and CODEOWNERS. The org is a
   portfolio of per-class dials, not a single level.
2. **Disengagement reports → the scorecard.** Regulators require autonomous
   vehicle operators to publish every instance of a human taking over; that
   public track record is what expansion decisions are made on. The per-class
   scorecard (changes, escapes, near-misses) is the same instrument: promotion
   evidence, visible to the people being asked to let go.
3. **Permits earned slowly, pulled overnight → the asymmetric ratchet.**
   Driverless permits are expanded city-by-city over years of evidence and
   suspended immediately after a single serious incident (California did
   exactly this to one operator in 2023). ADR-0007's ratchet is the same
   shape: promotion is a deliberate human decision; demotion is automatic and
   instant.
4. **The vigilance trap → the rubber-stamp queue.** The driving world's
   hardest lesson: humans are terrible at passively monitoring mostly-reliable
   automation — attention decays until the one moment it matters, which is why
   serious operators skipped the "human as standby fallback" level entirely.
   The ops version is ADR-0007's opening observation: a review rule past a
   certain volume becomes a suggestion. No rung on the ladder is "human
   passively monitors"; a class either gets genuine review (L1, kept honest by
   the approval packet) or earns machine-gated autonomy. Camping in between is
   where the incident happens.

## The confident handover

The 2019 articulation of the end state: *the ultimate goal is to take away the
steering wheel, brakes, and accelerator — not because the user isn't trusted,
but because the user confidently hands control over and focuses on the
objective: getting from A to B.*

This line aged the best, because it names the trust direction correctly. The
2026 discourse asks "can we trust the agent?" — guardrails, sandboxes,
human-in-the-loop. The actual adoption bottleneck is the mirror image: **does
the human feel confident enough to let go?** These are different problems. An
agent can be objectively reliable while humans hover over every PR; or humans
capitulate and rubber-stamp what they shouldn't. The factory's machinery — the
approval packet, the scorecard, the ladder — is best understood as a
**confidence-calibration instrument for the human**: it makes letting go a
rational act with evidence behind it, instead of either vibes or surrender.

Three refinements the intervening years add:

- **Ordering is load-bearing.** Confidence first, then the wheel comes out.
  Remove controls by fiat before confidence exists and people route around the
  platform — shadow IT in the 2010s, shadow pipelines and rubber-stamps now.
  The ladder encodes the ordering: every class starts at L1, and the humans
  whose review is being removed are the ones who sign its removal.
- **The end state has exactly two controls, not zero.** The robotaxi passenger
  keeps the destination input and a pull-over button — and the pull-over
  button is a *request the system executes safely*, not a brake pedal, because
  a panicked human slamming brakes mid-freeway is itself dangerous. The
  factory equivalent: humans keep "state intent" and "halt this," and halting
  is a controlled stop the machinery performs — ADR-0007's
  drain-in-flight-changes-back-to-the-human-queue *is* the safe pull-over.
  Hand-editing production mid-incident is the slammed brake. Blog-ready form:
  **the goal state has two controls — say where, and pull over.**
- **There are two handovers, and the second is the hard one.** The developer
  handing the driving to the factory is the 2019 handover, one level up. The
  platform team handing *supervision* to the gates is the safety driver
  getting out of the car — which, notably, was the actual milestone in
  Phoenix, not the car merely driving well. That is the L2→L3 rung, and the
  displaced-approver rule is the safety driver signing their own exit.

The framing also implies the correct success metric, measurable as revealed
preference — watch what people do, not what they say. Count the change classes
humans have *voluntarily* promoted; watch break-glass and manual-override
usage. If people keep grabbing the wheel, there is no autonomy, whatever the
ruleset says. The scorecard is the trust UI (ADR-0007); override usage is the
confidence UI.

## The personas, one seat up

Persona-first articulation — start from what the developer, the operator, the
SRE, and security each need, and design the platform as a product for them —
predates "platform as product" becoming orthodoxy, and it survives unchanged as
a method. What changes is the cast. The rhyme to name explicitly: the
datacenter→cloud transition did not eliminate sysadmins; it moved them one
abstraction up, into SRE and platform engineering. The factory transition does
it again — **every role moves from operating the system to governing the thing
that operates it.**

| Persona | 2019 job | Factory job | What they need from the platform |
|---|---|---|---|
| Developer | Ship features on the paved road | Author of intent, editor of agent output | Precise intent channels; verification they can trust; fast feedback on agent work |
| Operator / platform engineer | Run the machines | Design the factory: change classes, gates, ladder ruleset; run the agent fleet the way ops ran server fleets | Levers that are PRs (rungs, rulesets, Compositions); the scorecard |
| SRE | Reliability, error budgets, toil reduction | Same machinery, new budget: escape rate gates autonomy promotion the way error budgets gate deploys; incident response where the first responder is an agent | The metadata spine; three-outcome gates; escape attribution |
| Security | Perimeter and least privilege for humans | Identity for non-humans: which principal — human or agent — may take which action in which domain, on what evidence | The displaced-approver seat; provenance and attestation; a complete audit trail |

### The fifth persona: the agent

In 2019 the personas were the customers of the automation. In the factory, one
of the customers *is* the automation — and it has needs like any persona:
machine-readable context, its own identity and scoped credentials,
deterministic interfaces, an evidence trail for everything it does. This
yields the one-sentence justification for the entire knowledge layer:
**knowledge-as-code is the documentation written for the agent persona.**
Humans skim prose and fill gaps with judgment; the agent persona gets context
as versioned, tested, owner-gated files (`knowledge-as-code.md`), because for
this persona documentation quality is an operational parameter, not a
nice-to-have.

## Judgment is an envelope model

The relatable framing for the judgment question — the one a skeptical reader
will raise about "take away the steering wheel":

**Judgment is a calibrated internal model of the vehicle's envelope — its
size, its dynamics, its limits — built only by operating with consequences,
never by reading the spec sheet.** Everyone has the felt version: nobody
*computes* whether they can make a lane change; they know, because of a
thousand repetitions with feedback. The car's dimensions are printed in the
manual. Nobody parallel parks off the manual.

Each driving instance has an exact engineering twin:

- **Knowing how big your car is → knowing your blast radius.** The senior
  engineer who winces at a "one-line" config change because it fans out to
  forty services is doing what a driver does sliding past a parking-garage
  pillar. Juniors misjudge the size of the vehicle: the shared library makes
  it a semi, and they think they're in a coupe.
- **Knowing how it turns → knowing reversibility and response.** How fast the
  system answers the wheel: deploy latency, rollback cleanliness, which
  maneuvers can be steered out of mid-turn and which are one-way doors.
  Dropping a column is a turn that cannot be unwound.
- **Knowing whether it will make the gap → headroom and window judgment.**
  Will the migration finish inside the maintenance window; will capacity
  absorb the spike; is the gap in traffic big enough for this maneuver.

Aviation is thirty years ahead on exactly this problem — autopilot took the
controls decades ago — and learned that hand-flying judgment atrophies under
automation. Its training culture named the failure ("children of the magenta
line," after the flight-director line pilots follow instead of flying the
aircraft), and its worst outcome is the canonical case study: an autopilot
disconnect mid-storm handed control to a crew whose manual-handling model had
decayed, and they held a stall input to the ocean. That is the vigilance trap
with a body count. Aviation's countermeasures map onto factory machinery
one-to-one:

| Aviation | Factory |
|---|---|
| Mandatory hand-flying time | Change classes deliberately kept at L0/L1 |
| Simulators | Game days |
| Type ratings — certified per aircraft, because envelope judgment does not transfer between airframes | Approval and promotion rights held per domain; CODEOWNERS is quietly a type-rating registry |
| Anonymous near-miss reporting | Escape and near-miss triage (ADR-0007) |

Two consequences worth stating plainly:

- **The scorecard closes the human feedback loop.** A repetition only builds
  judgment if the outcome comes back: an approver must eventually learn
  whether the approval was right. With that loop, L1 review is a rep; without
  it, review is a ritual that builds confidence without calibration — worse
  than no practice at all.
- **Knowledge-as-code is the envelope, externalized.** The agent needs the
  same three knowings — blast radius, reversibility, headroom — and gets them
  from the metadata spine and per-class done-criteria: the car's dimensions
  written down instead of learned by curb rash. This gives ADR-0007's
  "no checkable done-criteria, no autonomy" its plain-English gloss: **if
  nobody can write down the size of the car, nobody — human or agent — should
  be driving it fast.** And it produces a tidy residue: the classes whose
  envelope cannot be written down are exactly the ones that stay manual, which
  is exactly where human judgment keeps being built. The ladder, applied
  honestly, leaves the right practice behind. The honest caveat: aviation
  found the leftover manual work eventually got too *thin* and had to mandate
  practice anyway — which is the argument for game days as a first-class
  institution rather than an afterthought.

## What the framing commits the design to

A frame that changes nothing is marketing. This one generates commitments:

1. **Public-facing docs are organized by persona**, and the agent is a
   first-class persona in them. The README and blog articulate the pattern as
   "what each of the five personas gets."
2. **The scorecard surfaces outcomes back to the humans who approved**, not
   only per-class aggregates — the human feedback loop is a designed feature,
   because it is what keeps L1 review a rep instead of a ritual.
3. **Break-glass and manual-override usage is tracked as the confidence
   metric**, alongside the scorecard's trust metrics. Persistent
   wheel-grabbing on a promoted class is a signal on par with an escape.
4. **Some change classes will be designated permanently manual** — for
   envelope reasons (criteria cannot be written down) or judgment reasons
   (this is where humans keep their reps). Record the first such designation
   as an ADR; "permanently L1" must be a visible decision, not a default.
5. **Game days become a first-class factory institution** — the simulator, on
   the calendar, not an afterthought. ADR pending when designed.

## Real-world anchors to verify before publication

Per the repo's research standard, the anecdotes above are working knowledge
[I], to be verified against primary sources before the blog ships:

- SAE J3016 levels 0–5 and the operational-design-domain definition [I]
- Driverless (no safety driver) public service in Phoenix beginning ~2020;
  in-app rider "pull over" control [I]
- California DMV suspension of one operator's driverless permits, effective
  immediately, October 2023 [I]
- "Children of the magenta line" — a late-1990s American Airlines training
  lecture on automation dependency [I]
- Air France 447 (2009) final accident report: crew manual-handling decay
  under automation as contributing factor [I]
- California DMV annual autonomous-vehicle disengagement reports, public [I]
