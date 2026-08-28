# Pattern Catalog

Detectors run over the ledger, not over raw text. That is what makes them cheap, testable,
and explainable — each one is a query with a threshold, and each names the rows that
triggered it.

Every detector must state its **false-positive mode** up front. A detector nobody can
predict the misfires of will be muted rather than tuned.

---

## Failure patterns

### 1. Context erosion
**Signal.** Assertions of one claim lose context qualifiers over time — the later
assertion's context is strictly broader than the originating one.

**Why it matters.** The most common way a true claim becomes false. Nobody lied; the
qualifiers just got tiring to repeat.

*Example: "the retry logic is safe **for idempotent handlers**" → "the retry logic is safe".*

**Formal shape.** With the semantic layer this stops being a heuristic: erosion is an
unlicensed move along a `specializes` edge, asserting the parent claim on the child's
evidence. Support propagates from specialized to general only weakly and never in reverse
(`SEMANTICS.md` § Propagation rules), so the gap is computable rather than estimated.

**False-positive mode.** Qualifiers legitimately dropped because they were genuinely
established elsewhere, or omitted as obvious from shared context.

### 2. Hedge decay
**Signal.** Modality escalates across assertions — `hedged → plain → emphatic → absolute` —
with no new supporting evidence in between.

**Scope.** Personal. Escalation is a habit, and naming whose habit it is turns the tool into
a scoreboard. The claim-level fact (this claim hardened without evidence) is shared; the
actor-level pattern is not.

**Why it matters.** Confidence rising while evidence stays flat is the pure form of the
problem. The "no new evidence" condition is what makes it a finding rather than an
observation, since escalation *after* a receipt lands is correct behavior.

**False-positive mode.** Evidence gathered outside the ledger, or a summarizer flattening
hedges as a stylistic artifact rather than a belief change.

### 3. Commentary promotion
**Signal.** A claim whose `origin_kind` is `commentary` or `question` now being asserted at
register `finding`, or cited as evidence for another claim, with no receipt acquired in
between.

**Why it matters.** This is the pattern the project was built around, caught at the moment it
happens rather than after the failure. The statement was never a claim when it was made — it
was an aside — so it never attracted the scrutiny a claim attracts, and it arrives at
premise-hood having skipped every step that would have tested it.

Unlike hedge decay, which tracks a claim getting firmer, this tracks a claim changing
*category*. The two often co-occur but are independent: an aside can be promoted to a finding
while staying hedged, and that is if anything harder to notice.

**False-positive mode.** A remark that was casual in phrasing but grounded in real knowledge
the speaker did not bother to cite — expertise expressed informally. Register extraction will
also confuse `commentary` and `finding` on terse utterances, which makes this detector's
precision a direct function of P2's register accuracy. Measure the two together.

### 4. Definitional drift
**Signal.** A term used with materially different definitions across assertions, or a term
whose definition was superseded while claims bound to the old version kept their support.

**Why it matters.** A distinct failure from claim drift and easier to miss. When a definition
shifts, every claim built on it silently changes proposition while its receipts stay attached
— the evidence now supports something the claim no longer says. "Covered" meaning line
coverage and "covered" meaning branch coverage produce identical sentences and different
facts.

**False-positive mode.** Legitimate refinement of a definition that does not change which
claims hold under it. Distinguishing a narrowing that matters from one that does not requires
knowing the claims involved, so this detector is noisy on terms with many bindings.

### 5. Causal promotion
**Signal.** A relation first asserted as `precursor` or `contributory`, later asserted as
`necessary` or `sufficient`, with no evidence acquired in between.

**Why it matters.** Commentary promotion moved from the node to the edge, and the single
detector that most justifies the semantic layer. Engineering causal claims routinely begin as
"we noticed X before Y" and are restated as "X causes Y"; an observation becomes a mechanism
with nothing acquired to license it. Downstream this is expensive, because causal edges drive
blast radius and therefore what the system thinks needs re-checking.

**False-positive mode.** A genuine strengthening whose evidence landed outside the ledger, and
edge-extraction confusion between `contributory` and `necessary` on hedged phrasing — the same
weakness as register extraction, on a harder judgment.

### 6. Circular support
**Signal.** A cycle in the evidence DAG — a claim's support traces back to itself, usually
through a document that cited it or a summary that restated it.

**Why it matters.** Feels like the best-supported claim in the system. Is worth zero.
Agent-generated summaries make this dramatically more common, since a summary asserting a
claim is easily ingested later as a source for it.

**False-positive mode.** Nearly none, and the reason this detector is worth building first.
Cycles are structural facts about the graph.

### 7. Single-source amplification
**Signal.** Many assertions by many distinct actors, all provenance-traceable to one
unverified origin.

**Signal detail.** Computed over `actor_ref` distinctness, which needs no name resolved —
the detector fires identically inside and outside a personal scope.

**Why it matters.** Entrenchment scoring rewards distinct actors as a proxy for
independence. This is the case where that proxy is wrong, and it must be corrected for
rather than merely reported — a claim flagged here has its actor-diversity bonus removed.

**False-positive mode.** Genuine independent confirmation that happens to cite the same
convenient source afterward.

### 8. Orphan entrenchment
**Signal.** Assumption debt above threshold and rising, with downstream dependence — a
claim that has become load-bearing while never leaving L0/L1.

**Why it matters.** This is the headline detector. It fires *before* the failure, which is
the only time the information is worth anything.

**False-positive mode.** Claims that are genuinely not worth proving. Dismissal with a
reason is the right outcome, and dismissal rate here is a health metric for the threshold.

### 9. Silent contradiction
**Signal.** Two claims joined by a `negates` or `contradicts` relation, with compatible
contexts, both carrying non-trivial support.

Before the semantic layer this detector had no way to know two claims conflict and was a
statement of intent rather than a query. The relation edge is what makes it implementable.

**Why it matters.** Usually means a context qualifier is missing from one of them — the
contradiction is the symptom, the unstated scope is the bug.

**False-positive mode.** Context incompatibility that the qualifier model failed to
represent, which is the same weakness measured in P2.

### 10. Stale guard
**Signal.** A claim at L3/L4 whose supporting test no longer executes the code in its
subject reference — skipped, deleted, or made vacuous.

**Why it matters.** The claim keeps its rung while the receipt has quietly stopped being one.
Requires cross-referencing test-run receipts against coverage, which is why it lands in P5
rather than earlier.

**False-positive mode.** Coverage tooling gaps, especially with dynamic dispatch.

### 11. Unfalsifiable claim
**Signal.** No falsifier could be articulated for a claim, or the proposed falsifier is
unobservable.

**Why it matters.** Not necessarily wrong — often just vague. Either way it cannot climb
the ladder, so the finding is "sharpen this or accept it stays at L0 forever."

**False-positive mode.** Falsifier generation failing on a claim that is in fact falsifiable.

---

## Success patterns

Worth detecting too. A tool that only reports problems trains people to read it as noise,
and the successes tell us which practices actually move claims up the ladder.

### 12. Fast climb
A claim reaching L3+ within a short window of first assertion. Identifies the practices and
the people that convert assertions into receipts efficiently.

### 13. Honest retraction
An asserted claim later refuted by its own author with evidence, and superseded rather than
quietly dropped. The healthiest thing in the ledger; it should be visible and credited.

### 14. Qualifier addition
Assertions that *narrow* context over time — the inverse of erosion, and the signature of
a claim being sharpened by contact with reality.

---

## Detector implementation rules

1. **Explain by citation.** Every detection names the specific assertions, spans, and
   evidence rows that triggered it. No detection says only "this looks over-confident."
2. **Scope by subject, not by author.** Detections about a *claim* (orphan entrenchment,
   circular support, silent contradiction) are shared — they are facts about the project.
   Detections about an *actor's habits* (hedge decay, fast climb) surface only in that
   actor's own scope, aggregated and pseudonymous elsewhere. Agent actors are exempt:
   `actor_kind = agent` detections are shared openly.
3. **Precision gate.** A detector below 0.8 precision on the gold corpus ships disabled.
4. **Dismissals are data.** Recorded with a reason, never deleted; field dismissal rate is
   the real precision measurement.
5. **One comment per PR.** Detections aggregate into a single surface. Per-detection
   notifications are how this class of tool gets muted in its first week.
6. **Detectors are claims too.** Each detector's precision claim lives in the ledger, with
   the corpus run as its receipt.
