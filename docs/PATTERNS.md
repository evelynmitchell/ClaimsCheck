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

**False-positive mode.** Qualifiers legitimately dropped because they were genuinely
established elsewhere, or omitted as obvious from shared context.

### 2. Hedge decay
**Signal.** Modality escalates across assertions — `hedged → plain → emphatic → absolute` —
with no new supporting evidence in between.

**Why it matters.** Confidence rising while evidence stays flat is the pure form of the
problem. The "no new evidence" condition is what makes it a finding rather than an
observation, since escalation *after* a receipt lands is correct behavior.

**False-positive mode.** Evidence gathered outside the ledger, or a summarizer flattening
hedges as a stylistic artifact rather than a belief change.

### 3. Circular support
**Signal.** A cycle in the evidence DAG — a claim's support traces back to itself, usually
through a document that cited it or a summary that restated it.

**Why it matters.** Feels like the best-supported claim in the system. Is worth zero.
Agent-generated summaries make this dramatically more common, since a summary asserting a
claim is easily ingested later as a source for it.

**False-positive mode.** Nearly none, and the reason this detector is worth building first.
Cycles are structural facts about the graph.

### 4. Single-source amplification
**Signal.** Many assertions by many distinct actors, all provenance-traceable to one
unverified origin.

**Why it matters.** Entrenchment scoring rewards distinct actors as a proxy for
independence. This is the case where that proxy is wrong, and it must be corrected for
rather than merely reported — a claim flagged here has its actor-diversity bonus removed.

**False-positive mode.** Genuine independent confirmation that happens to cite the same
convenient source afterward.

### 5. Orphan entrenchment
**Signal.** Assumption debt above threshold and rising, with downstream dependence — a
claim that has become load-bearing while never leaving L0/L1.

**Why it matters.** This is the headline detector. It fires *before* the failure, which is
the only time the information is worth anything.

**False-positive mode.** Claims that are genuinely not worth proving. Dismissal with a
reason is the right outcome, and dismissal rate here is a health metric for the threshold.

### 6. Silent contradiction
**Signal.** Two claims in the ledger with incompatible content and compatible contexts, both
carrying non-trivial support.

**Why it matters.** Usually means a context qualifier is missing from one of them — the
contradiction is the symptom, the unstated scope is the bug.

**False-positive mode.** Context incompatibility that the qualifier model failed to
represent, which is the same weakness measured in P2.

### 7. Stale guard
**Signal.** A claim at L3/L4 whose supporting test no longer executes the code in its
subject reference — skipped, deleted, or made vacuous.

**Why it matters.** The claim keeps its rung while the receipt has quietly stopped being one.
Requires cross-referencing test-run receipts against coverage, which is why it lands in P5
rather than earlier.

**False-positive mode.** Coverage tooling gaps, especially with dynamic dispatch.

### 8. Unfalsifiable claim
**Signal.** No falsifier could be articulated for a claim, or the proposed falsifier is
unobservable.

**Why it matters.** Not necessarily wrong — often just vague. Either way it cannot climb
the ladder, so the finding is "sharpen this or accept it stays at L0 forever."

**False-positive mode.** Falsifier generation failing on a claim that is in fact falsifiable.

---

## Success patterns

Worth detecting too. A tool that only reports problems trains people to read it as noise,
and the successes tell us which practices actually move claims up the ladder.

### 9. Fast climb
A claim reaching L3+ within a short window of first assertion. Identifies the practices and
the people that convert assertions into receipts efficiently.

### 10. Honest retraction
An asserted claim later refuted by its own author with evidence, and superseded rather than
quietly dropped. The healthiest thing in the ledger; it should be visible and credited.

### 11. Qualifier addition
Assertions that *narrow* context over time — the inverse of erosion, and the signature of
a claim being sharpened by contact with reality.

---

## Detector implementation rules

1. **Explain by citation.** Every detection names the specific assertions, spans, and
   evidence rows that triggered it. No detection says only "this looks over-confident."
2. **Precision gate.** A detector below 0.8 precision on the gold corpus ships disabled.
3. **Dismissals are data.** Recorded with a reason, never deleted; field dismissal rate is
   the real precision measurement.
4. **One comment per PR.** Detections aggregate into a single surface. Per-detection
   notifications are how this class of tool gets muted in its first week.
5. **Detectors are claims too.** Each detector's precision claim lives in the ledger, with
   the corpus run as its receipt.
