# Data Model

The ledger is **append-only**. Nothing is updated in place; corrections are new revisions
carrying a `supersedes` edge. An audit that cannot be trusted to show what was believed
*at the time* is worthless for studying how belief drifted.

## Entities

### Trace
A source of dialogue: an agent session transcript, a PR thread, an issue, a design doc
review, a chat log. Stored with a content hash so a re-ingest of the same material is
recognized rather than duplicated.

`id · kind · source_uri · content_hash · captured_at · participants · redaction_state`

### Span
A stable locator into a trace — message id plus character offsets. Every assertion points
at one. If we cannot say exactly where a claim was made, we cannot defend the record of it.

`id · trace_id · message_id · start · end · quoted_text`

### Context
The frame under which a claim is asserted: environment, component, version range, load
conditions, time window, audience. Contexts are structured key/value qualifiers, not prose,
so that two contexts can be compared for compatibility.

`id · qualifiers (json) · derived_from_span · parent_context_id`

**Why this is first-class:** context erosion is the main mechanism by which a true claim
becomes a false one. A claim true under `{env: ci, python: 3.11, payload: <1MB}` restated
without qualifiers is a different and probably false claim. Comparing contexts across
assertions of the same claim is what makes that detectable.

### Claim
A normalized proposition. One claim, many assertions.

`id · canonical_text · subject_ref · polarity · context_id · falsifier · status · opened_at`

- `subject_ref` — what the claim is about: a code path, a test, a service, a requirement.
  Links the claim to the artifacts whose change should invalidate it.
- `falsifier` — the observation that would show it false. Required. A claim with no
  articulable falsifier is recorded but flagged as unfalsifiable, which is itself the finding.
- `status` — `open · supported · contested · refuted · stale · retired`

### Assertion
One occurrence of a claim being made. **This is the entity that makes repetition countable.**

`id · claim_id · trace_id · span_id · actor · actor_kind · asserted_at · modality · context_id · extraction_confidence`

- `modality` — expressed confidence in the utterance itself: `hedged · plain · emphatic ·
  absolute`. Tracking this per assertion is what makes hedge decay measurable.
- `actor_kind` — `human · agent · tool · document`. Amplification by agents is worth
  distinguishing from amplification by people.
- `context_id` — the context *of this assertion*, which may be narrower or broader than
  the claim's. Divergence is the erosion signal.

### Evidence
A receipt. Not an opinion.

`id · kind · locator · observed_at · validity (json) · strength · reproducible · collected_by`

- `kind` — `test_run · coverage · telemetry · log · benchmark · static_analysis ·
  type_check · code_reading · document · human_attestation · model_judgment`
- `validity` — the scope in which this receipt holds: commit range, config, time window,
  environment. **Decay is impossible without this**, so it is required, not optional.
- `strength` — intrinsic weight of the kind, before scoping. A passing test beats a model
  judgment; both are recorded.

### Link
Evidence to claim, with a direction.

`id · claim_id · evidence_id · relation · weight · created_at`

`relation` — `supports · refutes · qualifies · irrelevant`

Links form a DAG. Cycles in it are the circular-support detector's entire job, so the
structure is queried as a graph, not just as rows.

### Score
A computed snapshot. Append-only time series, so confidence drift is visible.

`id · claim_id · computed_at · support · entrenchment · ladder_rung · staleness · inputs (json)`

`inputs` records exactly which evidence and assertions produced the numbers. A score that
cannot be replayed from its inputs is not auditable, and an unauditable confidence number
is the thing we are trying to eliminate.

### Detection
A pattern detector's finding.

`id · detector · claim_id · severity · detected_at · explanation · evidence_refs · dismissed_by · dismissed_reason`

Dismissals are recorded, not deleted — dismissal rate per detector is how we measure
precision in the field rather than only on the corpus.

## Relationships

```
Trace 1─* Span 1─* Assertion *─1 Claim 1─* Score
                        │                │
                        └─* Context ─────┘
                                         │
                            Claim *─* Evidence  (via Link)
                                         │
                            Claim 1─* Detection
```

## Storage

**SQLite, local-first.** Single file, easy to inspect, easy to version, no service to run.
The append-only property is enforced by triggers rejecting `UPDATE` and `DELETE` on the
core tables, and there is a test that attempts both.

Derived state (current status, latest score) lives in views over the log rather than in
mutable columns. Slower; correct by construction; the right trade at this size.

If a shared team ledger becomes necessary, the log ships to Postgres unchanged — the
append-only design makes that a replication problem rather than a rewrite. Deferring that
decision costs nothing now.

## Identity and deduplication

Claim identity is the hardest problem in the model and gets its own metric in P2.

- Claims are keyed by `(normalized proposition, subject_ref, context compatibility)`.
- Matching is conservative: **prefer opening a new claim over merging two.** Under-counting
  repetition weakens a signal; wrongly fusing two distinct claims fabricates evidence for
  one of them, which is the exact harm the project exists to prevent.
- Merges are recorded as an explicit `merged_into` edge, never by rewriting rows, and are
  therefore reversible when the match turns out to be wrong.
