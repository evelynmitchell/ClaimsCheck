# Open Questions

Each question states the **default assumed in the plan**, so work can proceed without an
answer, and what changes if the answer differs.

---

## Q1 — Implementation stack

**Default:** Python 3.11+, SQLite, `uv` for packaging, `pytest`. Chosen for agent-tooling
ecosystem and the fact that the first receipt connectors (junit-xml, coverage) are
Python-native.

**If different:** TypeScript is the main alternative and would follow if the primary
surface turns out to be an editor extension rather than a CLI. Cost of switching after P1
is roughly a week; after P3, substantial.

---

## Q2 — First ingest source

**Default:** Claude Code / agent session transcripts, then PR threads.

**Why it matters:** Transcripts are dense, structured, and available in volume, which suits
the gold corpus. PR threads are sparser but have human claims and adjacent receipts, so
they show adoption value sooner. This choice sets which adapter and which annotation guide
get written first.

**Question:** Which traces do you actually have on hand and are willing to annotate? That
availability should probably decide this rather than the argument above.

---

## Q3 — Human review before entry, or write-then-review

**Default:** Agents write to the ledger freely; low-confidence extractions are quarantined
into a review queue and excluded from scoring until confirmed.

**Alternative:** Nothing enters without human confirmation. Much higher precision, and in
practice the queue never gets drained.

**This changes P1's schema** (a review-state column and its transitions), so it is the
question most worth answering before P1 starts.

---

## Q4 — Confidence representation

**Default:** Both — a discrete ladder rung (auditable, arguable) plus a continuous support
score (sortable, calibratable). The ladder is primary in every display.

**Alternative:** Ladder only. Simpler and harder to misuse, but gives up the calibration
study, which is the strongest evidence the whole approach works.

---

## Q5 — Local-first or shared service

**Default:** Local-first SQLite, one ledger per repo, committed or synced as the team
prefers.

**Open:** A claim asserted in one repo's session about another repo's behavior has no
natural home. Cross-repo claims are deferred to post-v1, but if that case is common for you,
the storage decision in P1 should anticipate it.

---

## Q6 — Trace retention and redaction

**Default:** Full traces stored locally with a redaction pass at ingest for obvious secrets;
spans keep quoted text so detections can cite exact wording.

**Open:** Do transcripts here contain anything (customer data, candid remarks about people)
that makes storing quoted text a problem? The alternative — storing hashes and offsets
without text — is implementable but makes every detection much harder to explain, and
explainability is what keeps the detectors trusted.

---

## Q7 — Whose claims are in scope

**Default:** Everyone's — human and agent alike, distinguished by `actor_kind`.

**Open:** There is a real social dimension. A tool that publicly tracks which colleague
over-claims most often will not be adopted twice. Options: agent claims only for v1;
personal-scope ledgers with only aggregates shared; or full transparency by team agreement.
**This is a values question, not a technical one, and it needs your answer rather than a
default.**

---

## Q8 — Blast radius derivation

**Default:** Derived from `subject_ref` plus files touched by supporting test runs, expanded
one hop through static imports.

**Open:** This is under-specified and probably the weakest part of the decay model. Coarse
radii make everything stale immediately; narrow ones miss real invalidations. Likely needs a
tuning study of its own in P4 — flagging it now rather than discovering it there.

---

## Q9 — What "expansion beyond software testing" means concretely

The plan sketches an order (testing → operations → design → planning) but the second step is
not designed. Worth knowing now: is there a specific second domain you have in mind? It
would change what stays generic versus what gets hardcoded for testing in P2–P4, and
generality bought speculatively is usually wasted.

---

## Q10 — Is there prior art here you want followed

Argumentation frameworks (Toulmin, IBIS), assurance cases (GSN), provenance standards
(PROV-O), and requirements traceability tooling all overlap this design. The plan uses none
of them directly, on the grounds that each carries formalism costs disproportionate to a v1.
If you want compatibility with any of them — particularly GSN, if this ever touches safety
cases — that constrains the data model and should be decided before P1.
