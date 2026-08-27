# Confidence Model

Confidence is **computed, not declared**. A number nobody can trace back to receipts is
the disease, not the cure.

## The ladder

Adapted from requirements maturity. A claim's rung is the highest one with a receipt
attached — never an assessment, always an artifact.

| Rung | Name | What it takes | What it does not mean |
|---|---|---|---|
| **L0** | Asserted | Someone said it | Nothing. Zero evidential weight, however confidently phrased |
| **L1** | Sourced | Cited to a specific artifact somebody actually read — code location, doc, log line | That the source is right, only that it exists and says this |
| **L2** | Reproduced | Independently re-derived by a second party, or a manual run recorded | That it is automated or will stay true |
| **L3** | Tested | An automated check exists that fails if the claim is false | That the check runs, or runs on relevant changes |
| **L4** | Guarded | That check runs in CI on every change within the claim's blast radius | That it holds in production |
| **L5** | Observed | Production telemetry confirms it, with an alert if it stops | That it will hold under conditions never observed |

The right-hand column matters as much as the left. Each rung is routinely mistaken for the
one above it — "we have a test for that" (L3) read as "that cannot regress" (L4) is the
single most common overclaim in software testing, and it is exactly what this project
exists to catch.

The rung is a floor, not a score: a claim at L3 with one narrow test and a claim at L3 with
a thorough property test are not equally supported. The rung constrains support; evidence
quality sets it within that band.

## What counts toward a score

Only assertions and evidence in `review_state` `active` or `confirmed` contribute. Anything
`quarantined` is stored, visible in the trace, and worth **zero** — to support and to
entrenchment alike.

Excluding quarantined rows from *entrenchment* as well as support is the non-obvious half.
A weakly-extracted assertion that may not have been made should not inflate the repetition
count any more than it inflates the evidence, or a noisy extractor would manufacture
assumption debt out of its own errors.

## Two numbers, tracked separately

### Support ∈ [0, 1] — evidence only

Contributions from linked evidence, weighted by:
- **kind strength** — telemetry and tests over model judgment and code reading
- **scope match** — how well the receipt's `validity` covers the claim's context. A test
  passing on `python3.11/linux` supports a claim about `python3.9/windows` weakly at best
- **independence** — three receipts from one run are not three receipts
- **recency** against the claim's blast radius (see decay)

Refuting evidence subtracts, and does so faster than supporting evidence adds. One
reproducible refutation should sink a claim that ten weak confirmations floated.

### Entrenchment ∈ [0, 1] — social weight only

Contributions from:
- **assertion count**, with sharply diminishing returns
- **distinct actors**, weighted higher than repetition by one actor — though see
  single-source amplification in `PATTERNS.md`, since distinct actors restating one origin
  is not independence. Distinctness is computable from `actor_ref` without resolving any
  name, which is what lets entrenchment stay a shared, project-level number while
  attribution stays personal
- **modality escalation** — hedged → absolute over time
- **downstream dependence** — decisions, code, and other claims that cite this one
- **survival time** unchallenged

Entrenchment contributes **nothing** to support. That separation is the whole architecture;
if the two ever get summed into one headline number, the project has failed.

### Assumption debt

```
debt = max(0, entrenchment − support)
```

The headline metric. High debt means the project is leaning on something it has not paid
for. It is reported per claim, and aggregated per component so that a subsystem's total
unearned belief is visible.

Debt is not a bug count — some debt is rational, since not everything is worth proving.
The point is that it be *visible and chosen*, rather than accumulated invisibly.

## Decay

Evidence expires. A test result is a statement about a commit, a configuration, and a
moment, not a permanent fact.

Each claim has a **blast radius**: the set of artifacts whose change could invalidate it
(files, modules, config keys, dependency versions), derived from `subject_ref` and from
what the supporting evidence touched.

- When something in the blast radius changes, the receipts predating that change lose scope
  match, support drops, and the claim's status moves toward `stale`.
- Time-based decay applies only to receipt kinds where it is meaningful — telemetry
  observations age, a deterministic test result at a pinned commit does not.
- A stale claim is **not** refuted. It has simply stopped being supported, and the ledger
  says so rather than quietly keeping the old number.

Decay is why `Evidence.validity` is mandatory. Without it, this section is unimplementable
and confidence is permanently optimistic.

## Resolution

A claim resolves when a receipt settles it: a test that directly falsifies it, telemetry
that contradicts it, or a human attestation with a stated basis. Resolution supplies the
labels calibration needs, so the loop is built in P3 rather than deferred.

Many claims never resolve. The **unresolved fraction** is tracked as a first-class metric,
because silently dropping unresolved claims from the calibration set would bias it toward
exactly the claims that were easy to check.

## Calibration

The acceptance test for the whole system.

- **Reliability curve** — bucket claims by predicted support, plot against the fraction
  that survived. Target: within ±0.15 of diagonal in the 0.6–0.95 band.
- **Brier score** over resolved claims, tracked as a trend rather than a threshold.
- **Overconfidence asymmetry** — measured separately. Being over-confident is worse than
  being under-confident here, because the failure mode we exist to prevent is unearned
  certainty. The scoring is deliberately biased toward under-claiming.

If the curve is bad, the scoring rules are wrong and get reworked. Publishing a
badly-calibrated confidence number would make ClaimsCheck the most authoritative source of
exactly the problem it was built to solve.

## Presentation rule

**A confidence number is never rendered alone.** Every surface shows, at minimum:

```
support 0.82 · L3 Tested · 2 receipts · newest 4 commits ago · entrenchment 0.91 · debt 0.09
```

There is no compact view that shows the score without its basis. This is a hard constraint
on every interface — CLI, PR comment, MCP tool response, report — and it is deliberately
inconvenient.
