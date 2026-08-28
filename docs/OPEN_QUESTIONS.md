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

---

# Still open

Each states the **default assumed in the plan**, so work proceeds without an answer.

## Q4 — Confidence representation

**Default:** Both — a discrete ladder rung (auditable, arguable) plus a continuous support
score (sortable, calibratable). The ladder is primary in every display.

**Alternative:** Ladder only. Simpler and harder to misuse, but gives up the calibration
study, which is the strongest evidence the whole approach works.

## Q5 — Local-first or shared service

**Default:** Local-first SQLite, one ledger per repo.

**Now sharper, given Q7:** "personal ledgers, shared aggregates" implies a visibility
boundary, and SQLite has no access control. Two workable shapes:

1. **Split store** — a shared ledger file plus a per-person attribution file that only that
   person holds. Keeps local-first, at the cost of joins across two databases.
2. **Attribution encrypted at rest** — one file, `actor` fields encrypted per-person.
   Simpler operationally, but a shared key is a fiction and a real one needs key management.

Default is (1). This should be settled before P1's migration, since it decides whether
`actor` is a column or a foreign key into a separate store.

Cross-repo claims remain deferred to post-v1.

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

## Q10 — Prior art to stay compatible with — **now live**

The semantic layer moves this from a nice-to-have to a real fork. Argumentation frameworks
(Toulmin, IBIS), assurance cases (GSN), and provenance standards (PROV-O) all model roughly
what `SEMANTICS.md` now models, and each has a documented history of the formalism becoming
the work.

The current design borrows the *shape* of these (typed relations, evidence-backed edges) while
staying deliberately informal: no reasoner, no consistency checking, no standard serialization.
If compatibility with any of them matters — particularly GSN, if this ever touches safety
cases — that constrains the schema and should be decided before P1's migration.

## Q11 — How formal should the semantic layer be? **(new, and the significant one)**

There is a real fork here, and it is the decision most likely to determine whether the project
survives contact with a deadline.

1. **Lightweight typed edges (the current default).** Relations are rows with a kind, a
   polarity, and a confidence. Propagation is a handful of explicit rules. No reasoner, no
   consistency checking, no formal semantics. Cheap, debuggable, and will miss inferences a
   real logic would catch.
2. **A real reasoner** — Datalog, or OWL with an off-the-shelf engine. Gives sound
   transitive inference, consistency checking, and detects contradictions the rule-based
   version cannot. Costs a formalization discipline that every claim must then satisfy, and
   makes extraction much harder: the model must emit well-formed logic, not just a plausible
   edge.
3. **Probabilistic graphical model** — a proper joint distribution over claims. The
   theoretically right answer for calibration, and it needs structure and parameters nobody
   has, from a graph extracted by a language model with unknown edge accuracy.

**Recommendation: (1), with the schema kept compatible with (2).** The failure mode this
project exists to catch is unearned confidence, and (2) and (3) both produce confident,
well-formed output from uncertain inputs — with a provenance trail that looks impeccable. Get
calibrated with explicit rules first; the reasoner is a P8-or-later question, and only if the
rules demonstrably miss things that matter.
