# Semantics: Terms, Logic, and Causation

The model as first drafted treats a claim as text with metadata. That is enough to count
repetition and attach receipts, and not enough for anything that depends on what a claim
*means*. Three capabilities were missing, and their absence left parts of the design either
hand-waved or unimplementable:

| Missing | What it left broken |
|---|---|
| Precise reusable definitions | Claim identity was string similarity. Two claims using "covered" differently looked the same; one claim restated in new words looked different |
| Logical relations | The silent-contradiction detector had no way to know two claims conflict. Refuting evidence stopped at the claim it was attached to |
| Causal relations | "Downstream dependence" in entrenchment had no structure to read. Blast radius fell back to file proximity |

This document adds the three layers. It also states where each one is dangerous, because a
semantic layer is the most reliable way to make a system like this confidently wrong.

---

## 1. Terms — the ontology layer

A **Term** is a precise, reusable definition of a concept that claims are built from.

`id · label · definition · synonyms · subject_ref · defined_by · version · supersedes`

Claims reference terms, not just words. `Claim.canonical_text` keeps a term-annotated form,
so "this path is covered" resolves to a specific sense of *covered* rather than the word.

**Definitions drift, and that is a distinct failure from claim drift.** "Covered" means line
coverage to one person and branch coverage to another. "Flaky" means nondeterministic to one
and environment-dependent to another. When a term's meaning shifts, every claim built on it
silently changes proposition while keeping its support — the receipts stay attached to a
claim that no longer says what it said.

So terms are versioned and append-only like everything else, and a definition change creates
a new version with a `supersedes` edge rather than an edit. Claims bind to a term *version*.
A claim whose term has been superseded is flagged for re-binding, not silently re-interpreted.

`defined_by` points at what grounds the definition — a spec, a code artifact, a test that
operationalizes it. **A term defined only by prose is at L0 the same way a claim is.** The
strongest form of a definition is one with an executable referent: *covered* meaning "appears
in the branch-coverage report for this commit" is a definition that can settle disputes,
where "exercised by the tests" is not.

This is what makes claim linking tractable. Matching on `(term-annotated proposition,
subject_ref, context)` is a far better-posed problem than matching on paraphrase, and it fails
in more predictable ways.

---

## 2. Logical relations

Typed, directed edges between claims. Each carries its own evidence and confidence — see § 4.

| Relation | Meaning | Sound inference it licenses |
|---|---|---|
| `negates` | X and ¬X — exactly one holds in a given context | Support for one is refutation of the other |
| `contradicts` | Cannot both hold, without being strict negation | Support for one weakens the other |
| `entails` | If X then Y | ¬Y ⟹ ¬X (refutation flows backward, at full strength) |
| `equivalent` | Same proposition, different phrasing | Evidence is shared in both directions |
| `specializes` | X is a narrower case of Y | Support for X supports Y weakly; support for Y does **not** support X |
| `presupposes` | X is only meaningful if Y holds | ¬Y makes X ill-formed, not false |

`specializes` is worth noting: **context erosion is an unlicensed move from a specialized
claim to its generalization.** What was a heuristic in `PATTERNS.md` becomes a graph
operation — the eroded restatement is the parent node, asserted with the child's evidence.

### Propagation rules

The inference this licenses is deliberately lopsided:

- **Refutation propagates freely.** ¬Y ⟹ ¬X up an entailment edge is modus tollens, sound and
  unconditional. Contradiction and negation likewise reduce support.
- **Support propagates downward only, and attenuated.** X entails Y and X is supported ⟹ Y is
  supported, at no more than X's own support and reduced by the edge's confidence.
- **Support never propagates upward.** Y holding does not make X hold. That is affirming the
  consequent, and it is the single most likely way this layer would manufacture confidence out
  of nothing. It is prohibited structurally, not by convention.

**Derived support is marked derived and is never a receipt.** A claim's ladder rung is set by
receipts attached to it, never by inference from a neighbour. A claim can have high derived
support and still sit at L0, and the display shows both — inference can tell you where to
look, but it cannot pay for a rung.

The ordering principle for turning this on: **enable the inference that lowers confidence
first, and the inference that raises it last, if at all.** Refutation propagation makes the
system more skeptical and is safe to ship early. Support propagation makes it more confident
and every error in the graph inflates a score somewhere downstream.

---

## 3. Causal relations

Separate from logical relations, because "X causes Y" and "X implies Y" fail differently and
get overclaimed differently.

| Relation | Meaning |
|---|---|
| `necessary` | Without X, no Y |
| `sufficient` | X alone produces Y |
| `contributory` | X raises the likelihood of Y; neither necessary nor sufficient |
| `enabling` | X makes Y possible without driving it |
| `precursor` | X reliably precedes Y; causality not established |

`precursor` earns its place by being the honest floor. Most causal claims in engineering
dialogue begin as "we noticed X before Y" and are restated later as "X causes Y" — an
observation promoted to a mechanism, with nothing acquired in between. Without a distinct
`precursor` type there is nowhere to record the weaker thing, so it gets recorded as the
stronger one at the moment of first entry, and the promotion becomes invisible.

The same distinction handles Evelyn's case directly: *X is a precursor to Y but not a
necessary precursor* is `precursor` plus an explicit refutation of `necessary` — a recorded
negative, not an absence. **Absence of an edge and a refuted edge must be distinguishable**,
or "nobody has checked" and "somebody checked and it does not hold" collapse into the same
silence, which is the project's core failure mode wearing a different hat.

### What this fixes

Causal edges give **blast radius** a principled basis, which `OPEN_QUESTIONS.md` Q8 flagged as
the weakest part of the decay model. Invalidation propagates along `necessary` and `enabling`
edges rather than along file adjacency: if X is a necessary precursor for Y and X's support
decays, Y's support decays with it. Import-graph proximity becomes a fallback for claims with
no causal edges, not the primary mechanism.

They also make entrenchment's **downstream dependence** computable. What decisions and claims
actually rest on this one is an in-edge count over the graph, not a guess.

---

## 4. Edges are claims too

The load-bearing constraint on all of the above.

"X entails Y" is itself an assertion somebody made, with a register, a context, and evidence
that may or may not exist. A system that treats its claims as uncertain while treating the
graph connecting them as ground truth has smuggled certainty in through the edges — and it
will produce confident, well-formed, wrong conclusions, with a provenance trail that looks
impeccable.

So relations are stored as claims: same table shape, same ladder, same register, same decay.
An edge asserted once in conversation and never checked sits at L0 and propagates almost
nothing. **Propagation strength is gated on edge confidence**, which means a graph built from
casual remarks behaves like a graph of casual remarks rather than like a proof.

This also yields the causal analogue of commentary promotion, and it is the detector this
whole layer most justifies: an edge first asserted as `precursor` or `contributory`, later
asserted as `necessary` or `sufficient`, with no evidence acquired in between. Same promotion
the project was built around, moved from the node to the edge.

---

## 5. Cost, and where this goes wrong

Formal semantics is where projects of this kind reliably die. Assurance cases (GSN),
argumentation frameworks (Toulmin, IBIS), and semantic-web provenance all have the same
history: the formalism becomes the work, the ontology becomes a committee, and the tool stops
being used by anyone with a deadline.

Three specific hazards:

1. **Edge extraction is harder than claim extraction.** Recognizing that one claim entails
   another, from dialogue, is a genuinely difficult inference. P2 needs edge-level precision
   and recall as separate metrics, and low-precision edges must not propagate at all.
2. **A wrong edge is worse than no edge.** It moves confidence, silently, through a mechanism
   that looks rigorous. This is why propagation is gated on edge confidence and why support
   propagation ships last.
3. **Ontology sprawl.** A term layer invites definition of everything. Terms should be created
   only when a definitional dispute actually occurs or a claim cannot be stated without one —
   demand-driven, never up-front.

**Recommended posture: adopt all three as data model immediately, as detection early, as
inference late.** Recording edges is cheap and they are claims like any other. Querying them
for contradictions and promotions is low-risk and surfaces things for humans to judge. Letting
them move confidence numbers is the part that can make the system actively worse than not
having it, and it should be earned — with the refutation direction first, since it can only
make the system more skeptical.
