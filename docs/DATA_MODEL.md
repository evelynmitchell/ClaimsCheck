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

`id · canonical_text · term_bindings · subject_ref · polarity · context_id · falsifier · status · opened_at · review_state · origin_kind`

- `subject_ref` — what the claim is about: a code path, a test, a service, a requirement.
  Links the claim to the artifacts whose change should invalidate it.
- `falsifier` — the observation that would show it false. Required. A claim with no
  articulable falsifier is recorded but flagged as unfalsifiable, which is itself the finding.
- `status` — `open · supported · contested · refuted · stale · retired`
- `origin_kind` — the register of the claim's *first* assertion. See § Register and origin.
- `term_bindings` — the term versions this claim's canonical text resolves against. A claim
  whose bound term has been superseded is flagged for re-binding, never silently
  re-interpreted under the new definition.

### Assertion
One occurrence of a claim being made. **This is the entity that makes repetition countable.**

`id · claim_id · trace_id · span_id · actor_ref · actor_kind · asserted_at · modality · register · context_id · extraction_confidence · review_state`

- `modality` — expressed confidence in the utterance itself: `hedged · plain · emphatic ·
  absolute`. Tracking this per assertion is what makes hedge decay measurable.
- `register` — what kind of speech act this occurrence was. See § Register and origin.
- `actor_kind` — `human · agent · tool · document`. Amplification by agents is worth
  distinguishing from amplification by people.
- `actor_ref` — an indirection, not a name. See § Attribution scoping.
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

### Term
A precise, reusable definition of a concept that claims are built from. Versioned and
append-only: a definition change is a new version with a `supersedes` edge, never an edit,
and claims bind to a term *version*.

`id · label · definition · synonyms · subject_ref · defined_by · version · supersedes`

`defined_by` points at what grounds the definition — a spec, a code artifact, a test that
operationalizes it. A term defined only by prose sits at L0 exactly as a claim does.

Claims reference terms rather than bare words, which is what makes linking a tractable
problem instead of a paraphrase-matching one. See `SEMANTICS.md` § Terms.

### Relation
A typed, directed edge between two claims — logical or causal.

`id · from_claim_id · to_claim_id · kind · polarity · context_id · claim_id`

- `kind` — logical: `negates · contradicts · entails · equivalent · specializes ·
  presupposes`; causal: `necessary · sufficient · contributory · enabling · precursor`
- `polarity` — whether the edge is **asserted** or **refuted**. A refuted `necessary` edge is
  a recorded negative ("checked; X is not a necessary precursor of Y") and must stay
  distinguishable from no edge at all ("nobody has checked").
- `claim_id` — **the edge's own claim record.** Relations are claims: they carry a register,
  evidence, a ladder rung, and decay like anything else, and propagation strength is gated on
  that confidence. A graph built from casual remarks must behave like casual remarks, not
  like a proof. See `SEMANTICS.md` § Edges are claims too.

## Review state

Agents write to the ledger freely; nothing waits on a human. What protects the ledger from
weak extraction is not a gate but a **quarantine**.

`review_state` — `active · quarantined · confirmed · rejected`

- Extractions at or above the confidence threshold enter `active` and score normally.
- Extractions below it enter `quarantined`: stored, queryable, visible in the trace, and
  **excluded from every score and detector**. They are on the record without being counted.
- A human confirming one moves it to `confirmed`; rejecting moves it to `rejected`. Both are
  new revisions, never edits.
- `rejected` rows stay in the store. Rejection rate per extractor version is how we measure
  extraction quality in the field rather than only on the corpus.

The threshold is tuned in P2. Set too high, the quarantine becomes the review queue nobody
drains — the exact failure the write-freely policy was chosen to avoid — so quarantine depth
is monitored as a health metric in its own right.

## Attribution scoping

Attribution is personal; propositions and evidence are shared. The split is load-bearing,
because repetition is only countable **across** people, so the claim and assertion layer
cannot be partitioned per person.

`actor_ref` therefore resolves differently depending on who is asking:

- **In the asserter's own scope** — resolves to their identity.
- **Anywhere else** — resolves to a pseudonym derived from `(actor, claim_id)`.

Per-claim, not global. A globally stable pseudonym would be de-anonymized in an afternoon by
correlating assertion timestamps against commit history, which would silently reinstate the
attribution leaderboard this design exists to prevent. Per-claim pseudonyms still support
the query that matters — *"four distinct actors, all tracing to one unverified source"* —
without naming anyone.

Agent assertions (`actor_kind = agent`) are attributed openly. An agent carries no social
cost from being shown to over-claim, and agent drift is the signal we most want visible.

Quoted span text is attribution by another route: a verbatim quote identifies its author to
anyone who was present. Cross-scope views must paraphrase or suppress it, at a real cost to
detector explainability. This is unresolved — see `OPEN_QUESTIONS.md` Q6.

## Register and origin

Modality captures how firmly something was said. **Register** captures something different
and, for the pattern that motivates this project, more important: what kind of statement it
was in the first place.

`register` — `requirement · hypothesis · finding · commentary · question`

- **requirement** — a stated need or constraint
- **hypothesis** — offered as something to be checked, announcing its own provisionality
- **finding** — presented as resting on evidence, whether or not it does
- **commentary** — an aside, an impression, color; not a position anyone intended to defend
- **question** — raised rather than asserted

A claim's `origin_kind` is the register of its first assertion.

**Why this is a field and not a note.** The failure this project targets is not primarily a
wrong hypothesis — a hypothesis announces itself as unsettled and invites the checking that
would settle it. The failure is a statement that was not making a claim at all being reused
as though it were: commentary promoted to premise by nothing more than a later citation. That
promotion is invisible unless the original register was recorded at the time it was said,
because by the point anyone thinks to look, the aside and the finding are indistinguishable
in the transcript.

Register is per assertion, not per claim, because it changes across restatements — and that
change is the signal. An assertion at `commentary` followed by an assertion of the same claim
at `finding`, with nothing in between, is the pattern itself, caught in the act.

Register is also the reason `question` is in the list. A question that gets restated as a
finding is the same promotion with an even weaker starting point, and it is common enough in
agent transcripts to be worth naming separately — a model asked to consider whether X holds
will readily summarize the discussion as X holding.

## Relationships

```
Trace 1─* Span 1─* Assertion *─1 Claim 1─* Score
                        │                │
                        └─* Context ─────┤
                                         │
                            Claim *─* Evidence  (via Link)
                            Claim *─* Claim     (via Relation — logical and causal)
                            Claim *─* Term      (via term_bindings)
                                         │
                            Claim 1─* Detection
```

Relation is drawn as a claim-to-claim edge, but every Relation also *has* a Claim of its own
(`Relation.claim_id`), which is what lets an edge be evidenced, scored, and decayed.

## Storage

**Python 3.11+ with SQLite, local-first.** Single file, easy to inspect, easy to version,
no service to run.
The append-only property is enforced by triggers rejecting `UPDATE` and `DELETE` on the
core tables, and there is a test that attempts both.

Derived state (current status, latest score) lives in views over the log rather than in
mutable columns. Slower; correct by construction; the right trade at this size.

### The split store

SQLite has no access control, so the attribution boundary cannot be enforced within one
file. **Two databases:**

- `ledger.db` — shared. Claims, assertions, contexts, evidence, links, relations, terms,
  scores, detections. Everything except who said what.
- `attribution.db` — per person, held only by that person. Their identity secret, and
  whatever local notes they keep about their own assertions.

Joins across the boundary use SQLite's `ATTACH`, so a person working with both files queries
them as one database. Someone holding only the shared file sees a complete, queryable ledger
with pseudonymous actors.

### How `actor_ref` is derived

The obvious implementation is wrong in a way worth stating, because it looks fine: storing a
stable per-person id in the shared ledger and keeping the id-to-name mapping private. That
gives every assertion by one person the same value across the whole ledger — a global
pseudonym, de-anonymizable in an afternoon by correlating assertion timestamps against commit
history. The private mapping does not help, because the correlation never needs it.

So the shared ledger stores the **per-claim** value directly:

```
actor_ref = HMAC(actor_secret, claim_id)
```

- **Keyed, not a plain hash.** `H(name, claim_id)` would be trivially reversible: a team has
  a handful of members, so an attacker enumerates the names and compares. The secret is what
  makes the space unsearchable, and it lives only in `attribution.db`.
- **Within a claim**, one person's assertions share a value — repetition stays countable, and
  `distinct actors` for entrenchment is a `COUNT(DISTINCT actor_ref)` needing no identity.
- **Across claims**, the values are unlinkable. This is the property being bought, and it is
  bought by giving something up (below).
- **Only the owner can recompute it**, for any claim, using their secret. Their own
  personal-scope detectors — hedge decay, fast climb — run inside their scope and are
  impossible to run from outside it.
- **Agents are exempt.** `actor_kind = agent` stores a plaintext id, discriminated by that
  column. An agent carries no social cost from being shown to over-claim.

### What this costs

Two things, both accepted deliberately:

1. **No global actor statistics.** "How many distinct humans have asserted anything in this
   project" is not answerable, because distinctness exists only within a claim. The
   aggregates that matter — assumption debt per component, detector precision — do not need
   it. Single-source amplification survives because it traces *provenance* through relation
   and evidence edges, not actor identity across claims.
2. **Losing `attribution.db` is unrecoverable.** Without the secret, a person permanently
   loses the ability to recognize their own assertions; the shared ledger keeps working and
   their history in it becomes anonymous even to them. There is no recovery path by design,
   so the secret needs backing up like a key, because that is what it is.

If a shared team ledger becomes necessary, the log ships to Postgres unchanged — the
append-only design makes that a replication problem rather than a rewrite. Deferring that
decision costs nothing now.

## Identity and deduplication

Claim identity is the hardest problem in the model and gets its own metric in P2.

- Claims are keyed by `(term-annotated proposition, subject_ref, context compatibility)`.
  Resolving the words to term versions first is what makes this better-posed than paraphrase
  matching — and makes its failures predictable, since a mismatch points at a specific term.
- Matching is conservative: **prefer opening a new claim over merging two.** Under-counting
  repetition weakens a signal; wrongly fusing two distinct claims fabricates evidence for
  one of them, which is the exact harm the project exists to prevent.
- Merges are recorded as an explicit `merged_into` edge, never by rewriting rows, and are
  therefore reversible when the match turns out to be wrong.
