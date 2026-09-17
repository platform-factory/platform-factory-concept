# ADR-0014: The XRD schema denies first; Kyverno covers what a schema cannot say

Status: Accepted · September 2026 · defines the messages C-07(b) records

## Context

C-07(b)'s test: a claim requesting an oversized instance and a wrong region
is denied at admission; the data is "denial messages as seen by the
developer." The claim text accepts either mechanism ("Kyverno/Composition
bounds"), but the two produce different messages, arrive at different
moments, and fail in different ways — so the test cannot be written until
one is chosen. The M2 readiness walk also noted that Kyverno is on M2's build
list but not installed.

Three places can say no, and the developer meets them in this order:

1. **In the pull request.** `crossplane resource validate` checks a claim
   offline against the XRD, including its CEL rules, with no cluster
   involved [C, Crossplane CLI reference, 2026-09-14]. This is the only gate
   that fires *before merge*.
2. **At the API server.** An XRD's `openAPIV3Schema` supports `enum`,
   `pattern`, bounds, and `x-kubernetes-validations` CEL rules with a custom
   `message` [C, same source]; Kubernetes rejects the object with that
   message before it is stored.
3. **In an admission webhook.** Kyverno validates any kind, including XRs and
   managed resources, and can express what a schema cannot: rules across
   objects, per-namespace budgets, "nobody but Crossplane creates this
   kind" [C, research digest].

A fourth place is not a gate at all. A Composition's function pipeline runs
*after* admission; a Composition that "refuses" a value leaves the claim
stored, NotReady, with a condition the developer has to go find. That is the
worst of the four experiences and it is what "Composition bounds" would mean
in practice.

The GitOps shape matters for what the developer sees. Nobody on the paved
road runs `kubectl apply`; the claim is merged and Argo CD applies it. A
rejection at (2) or (3) therefore surfaces as an Argo sync failure carrying
the API server's message, on a claim that is already in `main`.

## Decision

**Constraints that a schema can express live in the schema. Kyverno is for
the rest, and it validates only. Nothing relies on a Composition to say no.**

1. **The `Database` XRD is intent-level and closed.** Region is an `enum` of
   the platform's allowed regions; size is a class (`S`, `M`, `L`) the
   Composition maps to a machine type — the schema never carries a `tier`
   string a developer could set to something enormous. Engine version is an
   enum of supported majors. Anything not in the schema is rejected as an
   unknown field. This is ADR-0012's principle applied to the second XRD:
   the denial for "oversized" is that the field cannot say it.

2. **CEL rules carry the cross-field constraints the schema's types cannot:**
   for example, that a `L` size is not allowed with a non-production tier,
   or that backups cannot be disabled for anything above `S`. Each rule has
   a `message` written for the developer, not the platform team.

3. **Kyverno is installed in M2, validate-only, for three rules the schema
   cannot express:**
   - the **reality gate** behind "app teams create XRs only": deny creation of
     raw managed resources (`*.gcp.m.upbound.io`, `*.gcp.upbound.io`) in any
     tenant namespace by anything other than Crossplane's own identity. Team
     RBAC already omits those kinds; this is the belt to that suspender, and
     ADR-0001's rule that an intent gate is paired with a reality gate;
   - a **per-System budget**: at most N `Database` claims in one namespace;
   - the **metadata spine** pre-check that M3 will need — deferred, but the
     install is the same.
   Kyverno's `failurePolicy` is `Fail` for the reality gate and scoped to the
   kinds it names, so Kyverno being down blocks managed-resource creation
   and nothing else. No mutation policies in M2 (see ADR-0013's
   consequences for why).

4. **The gate order is the contract.** A bad claim is caught in CI; the API
   server is the backstop for a claim that skipped CI; Kyverno is the
   backstop for a claim the schema could not judge. "Denied at admission"
   in C-07(b) means denied at (2) or (3), *and* the test also records what
   (1) said about the same file, because (1) is where the developer is
   supposed to meet it.

5. **The test records every message verbatim, from every surface.** For the
   wrong-region claim (an `enum` denial), the oversized claim (an unknown-
   field or `enum` denial), and one CEL rule: the CLI's output, the API
   server's response, and the Argo CD sync-status text. For the raw-MR
   claim: Kyverno's message through the same three surfaces. Predicted
   shape, written before the run so the build cannot improve it after the
   fact: the API server names the field and the allowed values
   (`spec.region: Unsupported value: "…": supported values: …`); CEL denials
   carry the rule's `message`; Kyverno denials carry the policy and rule
   name plus its message [I — Kubernetes and Kyverno conventions, confirmed
   by the run].

## Consequences

- **The cheapest gate does most of the work.** An `enum` costs one line,
  is checked offline in CI, and produces the clearest message. The build
  should be suspicious of any constraint it reaches for Kyverno to express
  before checking whether the schema could.
- **Schema tightening has a cost the research already recorded:** XRD schema
  changes need a Crossplane pod restart [C, research digest]. Allowed
  regions and sizes therefore change rarely and deliberately, which is the
  right pressure.
- **CI exists earlier than the plan said.** `crossplane resource validate`
  in the service repo's PR check is M4-shaped work (the factory's change
  classes need it) arriving in M2, because it is the first gate the
  developer meets. For M2 it may be run by hand and recorded as such; a
  minimal GitHub Action is the follow-up, not a prerequisite.
- **Argo CD gets a new failure mode to read.** A claim rejected at the API
  server leaves the service's Application OutOfSync with the message in its
  sync status. That is worse than a red PR check and better than a claim
  that quietly never becomes Ready. The Argo message is one of the three
  surfaces C-07(b) records precisely because it is the one a developer sees
  when CI was skipped.
- **Kyverno's images are on `ghcr.io`,** so they pull through the existing
  ADR-0010 remote with no new layer-0 change; C-23's zero-egress result is
  expected to hold and will be re-checked when it is installed.
- **Validate-only keeps Argo CD honest.** The known Kyverno / Argo CD
  friction is mutation versus drift detection [C, research digest]. With
  no mutating policies, Argo's diff sees exactly what git says, and the
  `ServerSideDiff` change is not needed yet.
- **The Composition is not a guardrail, and the build must not pretend it
  is.** If a value reaches a function that cannot handle it, that is a
  schema bug to fix at the schema, not a condition message to polish.
