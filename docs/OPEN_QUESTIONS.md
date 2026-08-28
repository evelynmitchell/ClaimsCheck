# Open Questions

Resolved decisions are recorded here rather than moved out, so the reasoning behind them
stays next to the questions still open.

---

# Resolved

## Q1 — Implementation stack — **RESOLVED: Python + SQLite**

Python 3.11+, SQLite, `uv` for packaging, `pytest`. The first receipt connectors
(junit-xml, coverage) are Python-native, and the primary surfaces are a CLI and an MCP
server rather than an editor extension.

*Consequence:* P1 starts with a `claimscheck` package and a migration cut from
`DATA_MODEL.md`. Revisiting after P3 would be expensive; before P1 it is nearly free.

## Q2 — First ingest source — **RESOLVED: all three (transcripts, PR/issue threads, design docs)**

The gold corpus spans agent session transcripts, PR and issue threads, and design docs/RFCs.

*Consequence, stated plainly:* this is the more expensive answer and the plan absorbs the
cost rather than hiding it.

- P0 corpus stratifies across three source types, so 30–50 traces means roughly 12–16 of
  each rather than 40 of one. Annotation agreement is measured **per source type** — claim
  boundaries in an RFC are a different judgment than in a transcript, and one aggregate
  number would hide that.
- P2 builds three adapters instead of one. **Estimate rises from 3 to 4 weeks.**
- Design docs and RFCs are the awkward case: claim-dense but receipt-poor, so P3's evidence
  binding will serve them badly at first. They are still worth annotating from the start,
  because they are the sharpest test of whether the falsifier machinery generalizes past
  testing claims. Expect their claims to sit at L0/L1 for a long time, and treat that as a
  measurement rather than a defect.
- Order of implementation stays transcripts → PR threads → design docs, so the receipt-rich
  sources prove the loop before the receipt-poor one stresses it.

## Q3 — Entry policy — **RESOLVED: write freely, quarantine low-confidence**

Agents record claims in-loop without a human gate. Extractions below the confidence
threshold enter `quarantined` and are excluded from all scoring until confirmed.

*Consequence:* `review_state` is in the P1 schema from the first migration, and scoring
reads only `active` and `confirmed` rows. See `DATA_MODEL.md` § Review state.

The quarantine threshold itself is a tuning parameter, calibrated in P2 against the corpus:
set too high, the queue fills and nobody drains it, which is the failure mode this answer
was chosen to avoid.

## Q7 — Whose claims — **RESOLVED: personal ledgers, shared aggregates**

Attribution is personal; propositions and evidence are shared.

This needs care, because the naive reading breaks the system. Repetition can only be counted
across people — a claim asserted by six people is the exact case the project exists to
catch — so the claim, assertion, and evidence layer must be **shared**. What is scoped is
**who said it**.

The resulting rule:

| Layer | Visibility |
|---|---|
| Claims, evidence, links, scores | Shared. Assumption debt is a project fact |
| Assertion existence, count, modality, timing | Shared. Repetition stays countable |
| Assertion *attribution* to a named person | Visible in that person's own scope only; elsewhere a stable per-claim pseudonym |
| Actor-level detections (your hedge decay, your fast climbs) | Personal scope only |
| Component-level and project-level aggregates | Shared |

Agent-authored assertions are attributed openly — `actor_kind = agent` carries no social
cost, and agent drift is what we most need to see.

*Consequence:* pseudonyms are stable **per claim**, not globally. Globally stable ones would
be trivially de-anonymized by cross-referencing timestamps against commit history, which
would quietly reintroduce the leaderboard this decision exists to prevent. Per-claim
pseudonyms still let a detector say "four distinct actors, one source" without saying who.
See `DATA_MODEL.md` § Attribution scoping.

## Q5 — Attribution storage — **RESOLVED: split store**

Two SQLite databases: `ledger.db` shared, `attribution.db` per person and held only by that
person. Joined via `ATTACH` when both are present.

*Consequence, and the part that is easy to get wrong:* the shared ledger cannot hold a stable
per-person id with a private mapping to names. That is a global pseudonym, and correlating
assertion timestamps against commit history de-anonymizes it without ever needing the mapping.
So the shared ledger stores the per-claim value directly, `actor_ref = HMAC(actor_secret,
claim_id)` — keyed rather than plainly hashed, because a team is small enough to enumerate.

Accepted costs: no global actor statistics (distinctness exists only within a claim), and
losing `attribution.db` permanently anonymizes that person's history even to themselves. See
`DATA_MODEL.md` § The split store.

Cross-repo claims remain deferred to post-v1.

## Q11 — Semantic layer formality — **RESOLVED: lightweight typed edges**

Relations are rows; propagation is explicit rules in ordinary code. No Datalog or OWL engine,
no consistency checking, no formal semantics.

*Consequences:*

- Propagation is **bounded** (default depth 3, recorded in the score's inputs) rather than
  transitive closure. Edge accuracy compounds: at 0.9 precision per edge, a five-hop inference
  is right about 60% of the time while looking exactly as authoritative as a one-hop one.
- The system will not detect logical inconsistency beyond the contradiction detector, and will
  miss inferences a real logic would catch. That is the accepted trade.
- **The schema stays reasoner-compatible.** Every relation is expressible as a triple with
  attributes, so exporting to an engine later needs no migration. Revisit only with evidence
  of the explicit rules failing — which makes Q10 (prior-art compatibility) answerable the
  same way: borrow the shape, defer the formalism.

See `SEMANTICS.md` §§ Propagation is bounded, No reasoner.

---

# Still open

Each states the **default assumed in the plan**, so work proceeds without an answer. **None
of these block P1's migration** — Q5 and Q11, which did, are settled above.

## Q4 — Confidence representation

**Default:** Both — a discrete ladder rung (auditable, arguable) plus a continuous support
score (sortable, calibratable). The ladder is primary in every display.

**Alternative:** Ladder only. Simpler and harder to misuse, but gives up the calibration
study, which is the strongest evidence the whole approach works.

## Q6 — Trace retention and redaction

**Default:** Full traces stored locally with a redaction pass at ingest for obvious secrets;
spans keep quoted text so detections can cite exact wording.

**Open, and now interacting with Q7:** quoted text is attribution by another route — a
verbatim quote identifies its author to anyone who was in the room, regardless of
pseudonymization. If personal scoping is to mean anything, quoted text in cross-scope views
needs either paraphrase or suppression, and both weaken the explainability that keeps
detectors trusted. Worth deciding deliberately rather than discovering in P6.

Design docs and RFCs raise this less (they are already shared artifacts); transcripts raise
it most.

## Q8 — Blast radius derivation

**Default:** Derived from `subject_ref` plus files touched by supporting test runs, expanded
one hop through static imports.

**Open:** the weakest part of the decay model. Coarse radii make everything stale
immediately; narrow ones miss real invalidations. Likely needs a tuning study of its own in
P4 — flagged now rather than discovered there.

## Q9 — What "expansion beyond software testing" means concretely

The plan sketches an order (testing → operations → design → planning) but the second step is
not designed. Including design docs in the P0 corpus (Q2) partially pre-empts this — we will
have annotated data from a receipt-poor domain before committing to one. Still worth knowing
whether you have a specific second domain in mind, since generality bought speculatively is
usually wasted.

## Q10 — Prior art to stay compatible with

The semantic layer moves this from a nice-to-have to a real fork. Argumentation frameworks
(Toulmin, IBIS), assurance cases (GSN), and provenance standards (PROV-O) all model roughly
what `SEMANTICS.md` now models, and each has a documented history of the formalism becoming
the work.

Q11 answers most of this: the design borrows the *shape* (typed relations, evidence-backed
edges) and stays deliberately informal, with the schema kept expressible as triples so an
export remains possible.

**What is still open:** whether to commit to a *standard serialization* — GSN or PROV-O — as
an output format. That is a smaller decision than adopting the formalism, and it no longer
blocks P1. It becomes live if this ever touches safety cases, where GSN compatibility is
sometimes a requirement rather than a preference.

