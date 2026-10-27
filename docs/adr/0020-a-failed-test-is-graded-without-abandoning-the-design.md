# ADR-0020: A failed test is graded without abandoning the design, and a test that cannot discriminate gets a ceiling

Status: Accepted · October 2026 · adds three grades (PARTIAL, UNSUPPORTED, NOT REACHED) to the three of ADR-0008 and supersedes only ADR-0008's list of grades; adds grade ceilings, written in a readiness note before a milestone's first build command; applies from M3's readiness note on, so every earlier grade stands under ADR-0008's words · from platform-factory-concept#35, decided on 2026-10-27, before any M3 build command ran

## Context

ADR-0008 says "WRONG gradings are a deliverable, not an embarrassment", and
then defines WRONG as "abandoned — ADR required, published, not buried". It
defines nothing for a claim whose test failed and whose design was kept. That
case has to take one of the other two words, and the only one left is
ADJUSTED.

The register shows what this does. Eight claims are graded: five HELD, three
ADJUSTED, none WRONG. The three ADJUSTED grades are not alike:

- **C-01** asked for zero Terraform applies after M1 and got four, plus two
  changes that rode the rebuild. It was graded ADJUSTED on a restated rule
  (identity and reachability for a new capability are a declared Terraform
  crossing). ADR-0016 itself calls the zero target "wrong in a predictable,
  bounded way". Under the registered test that is a miss, and the vocabulary
  had no word for a miss that keeps the design.
- **C-07** is the precedent for a gate that fails its first live test and is
  fixed the same hour: ADJUSTED (ADR-0016, Consequences).
- **C-06** is ADJUSTED for a change of design, not of a defect.

With two outcomes available, "held" or "abandon the design", every partial
result is pushed into ADJUSTED, and the three words stop telling a reader
different things. Nothing here says a grade was dishonest; each is explained in
its build-log entry. The structure is the problem, and M3 and M4 make it worse.
Their tests are harder to score. C-09 run by one person shows that GitHub
honours a path-scoped rule, not that duties are separated
(platform-factory-concept#21). C-12 cannot test half of what it says, and
C-16 has no seeded defects, so zero near-misses would look the same as a
reviewer who sees nothing.

A second gap sits beside the first. A test can be unable to run: a precondition
(an identity, a gateway, a second reviewer) was not met inside the milestone.
Grading that as HELD, WRONG or ADJUSTED would all misreport it.

## Decision

**Six grades, each saying what happened to the claim and to the design. A
readiness note may fix a ceiling before the run for a claim whose test cannot
discriminate, and the close cannot grade above it.**

1. **The grades.** A claim is UNTESTED until a build-log entry with evidence
   moves it (ADR-0008, unchanged). The entry uses one of these, in this order:
   - **HELD.** The test ran and the claim held as written.
   - **ADJUSTED.** The claim held after a design change. A superseding ADR is
     linked.
   - **PARTIAL.** The test ran. The named cases held, and the named cases did
   not hold or did not run. The design is kept, and the entry lists each case
   that failed or did not run, with why.
   - **UNSUPPORTED.** The test ran and the data do not support the claim. The
     design is kept, and the entry records why it is kept. If the design
     changes, an ADR says so.
   - **WRONG.** The claim is false and the design resting on it is
     abandoned. An ADR is required.
   - **NOT REACHED.** A precondition of the test was not met inside the
     milestone, and the entry names it. The claim stays open for a later
     milestone, or an ADR retires it. Nothing about the design is said.

   HELD, ADJUSTED and WRONG keep the meaning ADR-0008 gave them. The three new
   words are the ones that were missing. PARTIAL and UNSUPPORTED are where a
   failed test with a kept design now goes, instead of ADJUSTED.

2. **Ceilings.** A readiness note may write, next to a claim and before the
   milestone's first build command, the highest grade the close can give it. It
   does so when the test as planned cannot tell the grade from the ones above
   it: for example, one person on both sides of a rule can show the rule is
   enforced and cannot show duties are separated. The order for this purpose
   is the order in item 1, HELD highest. A ceiling does not forbid a lower
   grade, and it does not forbid NOT REACHED, which says the test did not run.
   A ceiling is never raised after the run. If the test turns out better than
   planned, the entry says so beside the grade it is allowed to give.

3. **Scope.** The vocabulary applies from M3's readiness note on. M1, M2 and
   M2b keep ADR-0008's three words, and their grades stand; none is revisited
   under the new words. The scoreboard says so in one dated line. C-28 and the
   other M2b claims, which are still ungraded, are graded in the M2b entry
   under ADR-0008's words, as they were registered. A claim registered earlier
   and still ungraded when M3 closes — C-08 — is graded in M3's entry under
   these words, because the entry that grades it is M3's.

4. **What this supersedes.** Only ADR-0008's list of grades: the sentence that
   defines HELD, ADJUSTED and WRONG. The rest of ADR-0008 stands: the register
   before the build, entries append-only, grades earned by a linked build-log
   entry, one entry per milestone. No claim's text changes, and the register's
   rule that existing claims are never reworded applies to ceilings too: a
   ceiling is written in a dated note, not into the claim.

## Consequences

- **A failed test has a word that is not a design change.** A reader can tell
  C-07's "the gate failed and was fixed" (ADJUSTED, a change made) from "some
  cases failed and we kept the design" (PARTIAL). The register's credibility
  claim, that it reports what the design got wrong, can now be met without
  forcing a partial result to look like a repair.
- **Ceilings cost a sentence per claim, written while it is cheap to be
  honest.** The cost is that a ceiling can be wrong in the cautious
  direction. If the run turns out stronger than planned, the entry shows the
  evidence and still stops at the ceiling. That is the intent: the ceiling
  records what the test could discriminate, and a later milestone with better
  conditions can grade the claim again.
- **NOT REACHED is not a failure and not a pass.** It keeps a claim whose
  conditions were not met from being graded on evidence that does not exist.
  It can be a stop sign for a milestone that planned too much, and the entry
  names the precondition so the next plan can include it.
- **Two scoreboards read side by side.** Earlier rows use three words, later
  rows six. The one-line note under the scoreboard is the only bridge. Nobody
  maps an old ADJUSTED to the new words.
- **Not decided here.** Which later claims get a ceiling. Each readiness note
  decides its own. ADR-0021's note for M3 is the first.
