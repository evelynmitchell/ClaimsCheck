# ClaimsCheck — Project Plan

Status: **Draft for review**
Scope of this document: what we are building, why, in what order, and how we will know
it works.

---

## 1. Problem

A statement made confidently and repeated often becomes load-bearing without ever being
checked. This is not a reasoning failure by any single participant; it is an accounting
failure across a conversation. The mechanics are consistent:

1. **Origination.** Someone asserts something with more confidence than their evidence
   supports — often reasonably, as a working hypothesis.
2. **Context stripping.** The claim was true under qualifiers ("on the CI image", "for
   small payloads", "as of last week"). Each restatement drops a qualifier, because
   qualifiers are tedious to carry.
3. **Hedge decay.** "I think this is covered" → "this is covered" → "obviously that's
   covered".
4. **Provenance collapse.** After a few hops, the claim's support is "we've been saying
   this for a while." The single unverified origin is no longer reachable.
5. **Load bearing.** Decisions accumulate on top of it. The cost of it being wrong rises
   monotonically while the evidence for it stays at zero.
6. **Failure.** It breaks, expensively, and the post-mortem finds that nobody ever
   actually checked.

Agents make every step of this faster. They produce fluent, confident prose at volume,
carry claims across sessions through summaries, and readily restate a prior conclusion as
an established fact. The problem is not new but its rate is.

## 2. The insight we are building on

Requirements engineering already solved a version of this. A requirement does not go from
"wanted" to "done" by assertion; it climbs a defined ladder, and each rung demands a
specific artifact — a specification, a design, a test, a passing run, a telemetry signal
in production. The maturity of a requirement is not an opinion, it is the highest rung
with a receipt attached.

ClaimsCheck applies that ladder to informal claims, and adds one thing requirements
tracking does not need: **separate accounting for evidence and for social weight.**

- **Support** rises only when a receipt lands.
- **Entrenchment** rises when a claim is restated, cited, or built upon.

Requirements tracking assumes entrenchment is zero, because requirements are not
persuasive by repetition. Claims in dialogue are. So the gap between the two — **assumption
debt** — is the primary output of the system, and everything else exists to compute it
honestly.

## 3. Goals

**G1.** Every claim in a trace is recorded with its context, anchored to the exact span
where it was said, and never silently mutated afterward.

**G2.** Claims restated across time and across traces are recognized as the *same claim*,
so repetition is counted rather than mistaken for independent confirmation.

**G3.** Each claim carries an explicit **falsifier** — the observation that would show it
false — and, where possible, the automated check that implements it.

**G4.** Confidence is computed from evidence with declared rules, decays when its evidence
goes stale, and is auditable back to the receipts.

**G5.** The system is **calibrated**: among claims it rated 0.9, close to 90% survive
contact with reality. This is the acceptance test for the whole project.

**G6.** Recurring failure and success patterns (context erosion, circular support,
single-source amplification) are detected and reported without a human going looking.

**G7.** Bookkeeping cost is low enough that it happens by default — agents write to the
ledger in-loop, not in a retrospective annotation session nobody schedules.

## 4. Non-goals

- **Not a truth oracle.** ClaimsCheck reports what evidence exists and what it supports.
  It does not adjudicate contested claims, and never marks one refuted on its own judgment
  alone — only on a receipt.
- **Not general-purpose fact-checking.** No open-web claim verification. The evidence
  space is the project's own artifacts.
- **Not a gate.** It does not block merges in v1. Nothing kills adoption faster than a
  noisy new required check.
- **Not a replacement for code review or testing.** It is bookkeeping *about* those
  activities.
- **Not an autonomous adjudicator.** Terminal verdicts on contested claims need a human or
  a deterministic receipt, never an LLM's opinion.

## 5. Meta-risk (stated up front)

ClaimsCheck can fall to its own failure mode. A confidence score displayed in a dashboard
is exactly the kind of statement that gets repeated until assumed. Two mitigations are
structural, not advisory:

1. **Every ClaimsCheck output carries its own evidence level.** A confidence number is
   never rendered without the ladder rung and receipt count that produced it. There is no
   view of the score alone.
2. **The system's claims about itself live in the ledger.** Assertions in these planning
   documents are ingested as L0 claims with falsifiers attached. If we cannot make our own
   claims climb the ladder, that is the first and most useful finding.

## 6. Initial scope: software testing

Testing claims are the right beachhead because ground truth is cheap and mechanical:

| Claim shape | Receipt that settles it |
|---|---|
| "This path is covered" | Coverage report for the named lines at a commit |
| "That flake is a network timeout" | Failure-log classification across N reruns |
| "This change is safe" | The named tests passing, plus a stated blast radius |
| "This endpoint is idempotent" | A property test that replays the request |
| "This is fixed" | A regression test that fails on the parent commit and passes on the fix |
| "Latency is fine under load" | A telemetry query against a stated window and percentile |

Every row has a falsifier we can automate. Where the loop works, we learn the mechanics of
extraction, linking, decay, and calibration under conditions where we can measure whether
we are right. Only then do we widen.

Design docs and RFCs are in the P0 corpus from the start even though P3 will bind evidence
to them poorly. That is deliberate: they are the sharpest available test of whether the
falsifier machinery generalizes beyond domains where receipts are cheap, and we would rather
learn that in week two than in month five.

**Expansion order** (deferred, sketched for direction): testing claims → operational and
incident claims (telemetry receipts) → architecture and design claims (weaker receipts,
stronger need for explicit falsifiers) → cross-team planning claims. Each step trades
receipt availability for reach, and each needs its own calibration study before being
trusted.

## 7. System shape

```
  traces                  ledger                        surfaces
 ─────────               ────────                      ──────────
 transcripts ─┐      ┌─ claims ────┐                ┌─ CLI queries
 PR threads  ─┼─ ▶ ──┤  assertions ├── ▶ ── scoring ┼─ PR / report view
 issues      ─┤      │  contexts   │       + decay  ├─ MCP tools (in-loop)
 commits     ─┘      │  evidence   │                └─ detector alerts
                     └─ links ─────┘                        │
                            ▲                               │
                            └──── receipts ◀── CI, coverage, telemetry, git
```

Five stages, each independently testable:

1. **Ingest** — normalize a source into a trace with stable, addressable spans.
2. **Extract** — claims, contexts, expressed confidence, and speaker, each anchored to a span.
3. **Link** — resolve each new assertion to an existing claim or open a new one; attach
   evidence to claims.
4. **Score** — compute support, entrenchment, ladder rung, and staleness.
5. **Detect & surface** — run pattern detectors; report.

Stages 1, 3 (evidence side), 4 and 5 are deterministic and unit-testable. Stage 2 and the
claim-matching half of stage 3 are model-driven and need an eval harness with a gold set.
Keeping that boundary sharp is a design constraint, not an implementation detail: it is
what lets us say the scoring is correct even while extraction is still improving.

## 8. Phases

Estimates assume roughly one engineer's sustained attention. Each phase ends with a
demonstrable artifact, and each is independently useful if the next never happens.

### P0 — Specification and gold corpus *(2 weeks)*
- Ratify the data model, confidence ladder, and detector catalog in this repo.
- Assemble the **gold corpus**: 30–50 real traces hand-annotated with claims, contexts, and
  expected confidence. This is the most valuable artifact in the project and the one most
  likely to be skipped under time pressure. It is not skippable.
- **Stratified across three source types** — agent session transcripts, PR/issue threads,
  and design docs/RFCs — roughly 12–16 each rather than 40 of one.
- Write the annotation guide; measure inter-annotator agreement on a 10-trace overlap,
  reported **per source type**. Claim boundaries in an RFC are a different judgment call
  than in a transcript, and one aggregate number would hide a weak type behind a strong one.
- **Exit:** corpus committed, agreement ≥ 0.7 on claim boundaries *in each source type*,
  model docs reviewed.

### P1 — Ledger core *(2 weeks)*
- Python 3.11+ / SQLite, packaged with `uv`, tested with `pytest`.
- Append-only store: claims, assertions, contexts, evidence, links, scores. Content-addressed
  IDs; no destructive updates, ever. Corrections are new revisions with a supersedes edge.
- `review_state` in the first migration, since scoring must read only `active` and
  `confirmed` rows from the beginning.
- Attribution split: `actor_ref` as an indirection rather than a name, so the personal-scope
  boundary is structural rather than bolted on later.
- CLI: `claim add|show|list`, `evidence add`, `link`, `export`.
- Manual entry only. Deliberately: the ledger must be useful to a human with no extraction
  at all, or we will not be able to tell whether later failures are storage or extraction.
- **Exit:** a human can record and query a claim's full history; the store round-trips and
  the append-only property has a test that tries to violate it.

### P2 — Extraction *(4 weeks)*
- Three trace adapters, built in receipt-richness order: agent transcripts, then PR/issue
  threads, then design docs/RFCs. The receipt-rich sources prove the loop before the
  receipt-poor one stresses it.
- Claim, context, and modality extraction with structured output and span anchoring.
- Claim normalization and linking (is this the same claim as one we've seen?).
- Quarantine threshold tuning: the confidence level below which an extraction is stored but
  not scored. Too high and the queue fills and never drains, which is the failure the
  write-freely policy was chosen to avoid; quarantine depth is monitored, not just set.
- **Eval harness** against the gold corpus: precision/recall on extraction, accuracy on
  linking, reported per source type.
- **Exit:** extraction P/R ≥ 0.7 on the held-out split, linking accuracy ≥ 0.8, and a
  regression run that gates changes to the prompt or model.

### P3 — Evidence binding *(3 weeks)*
- Receipt connectors: test results (junit-xml first), coverage, CI run metadata, git history.
- Falsifier proposal: for each claim, generate the observation that would refute it, and
  where it maps onto an existing or writable test, say which.
- Evidence scoping: every receipt records the commit range, configuration, and time window
  it is valid for. Without this, decay cannot work.
- **Exit:** a claim can be walked from assertion to a specific passing test at a specific
  commit, and back.

### P4 — Confidence and decay *(2 weeks)*
- Ladder placement, support and entrenchment scoring, staleness decay on blast-radius change.
- **Calibration report**: reliability curve of predicted confidence against resolved outcomes,
  plus Brier score.
- **Exit:** calibration report runs over the corpus; the curve exists and is honest even if
  it is bad. A bad curve is a finding, not a failure.

### P5 — Pattern detection *(2 weeks)*
- Implement the detector catalog (`docs/PATTERNS.md`): context erosion, hedge decay, circular
  support, single-source amplification, silent contradiction, orphan entrenchment.
- Precision-first tuning. A detector below 0.8 precision on the corpus ships off by default;
  false alarms are how this class of tool dies.
- **Exit:** detectors run over the corpus with per-detector precision reported.

### P6 — In-loop integration *(3 weeks)*
- MCP server exposing `record_claim`, `attach_evidence`, `check_claim`, `list_open_claims`
  so an agent writes to the ledger while working and consults it before restating.
- Session-end hook: extract from the just-finished transcript automatically.
- PR surface: a comment summarizing new claims, assumption debt, and detector hits on the diff.
- **Exit:** a full session records its claims with no human bookkeeping step.

### P7 — Dogfood and calibration study *(4 weeks, overlapping)*
- Run against this project's own development for a month.
- Publish: calibration curve, assumption-debt trend, detector precision, and an honest
  account of what the system missed.
- **Exit:** a go/no-go decision on widening scope, made from the data rather than from
  enthusiasm.

**Total to a defensible v1: roughly 18 weeks**, of which P0 and P7 are the two most often
cut and the two that determine whether any of it is true. The extra week over the original
estimate is the cost of covering three source types rather than one — bought deliberately,
in exchange for knowing early whether the model generalizes past receipt-rich domains.

## 9. Success criteria

The project succeeds if, at the end of P7:

1. **Calibration.** Reliability curve within ±0.15 of diagonal in the 0.6–0.95 band.
2. **Recall on what matters.** In a retrospective over known incidents in the dogfood
   period, ≥ 60% of the load-bearing wrong assumptions were in the ledger with support
   materially below entrenchment *before* the failure.
3. **Precision.** Detector alerts run at ≥ 0.8 precision, judged by the developers who
   received them.
4. **Cost.** Median added latency per session under 5 seconds; no manual bookkeeping step
   in the default path.
5. **Adoption signal.** Someone other than the author consults the ledger unprompted to
   settle a question. This is the softest criterion and the most diagnostic.

Failing (1) or (2) means the model is wrong and should be reworked, not shipped wider.
Failing (3) means tuning. Failing (5) means the surfaces are wrong, whatever the numbers say.

## 10. Risks

| Risk | Why it bites | Mitigation |
|---|---|---|
| Extraction quality is mediocre | Everything downstream inherits the error | Gold corpus in P0; extraction confidence is itself recorded and propagates; low-confidence extractions are quarantined for review rather than scored |
| Alert fatigue | The tool gets muted in week two and never unmuted | Precision-first gating; detectors ship off by default until they clear 0.8; at most one PR comment per PR |
| Over-formalization | Ceremony exceeds value, people route around it | Zero mandatory human steps in the default path; the CLI stays useful standalone |
| Ledger becomes an authority | Our own failure mode, applied to us | Scores never render without receipts; system self-claims live in the ledger |
| Claim identity is genuinely hard | Same-claim detection is the linchpin for counting repetition, and paraphrase is subtle | Measured explicitly in P2 with its own metric; conservative default (prefer a new claim over a wrong merge — under-counting repetition is safer than fusing distinct claims) |
| Transcript privacy | Traces contain credentials, customer data, candid remarks | Local-first store; redaction pass at ingest; retention policy set in P1, not later |
| Design-doc claims never leave L0 | Receipt-poor by nature; a third of the corpus may stay unscoreable | Expected, and treated as a measurement of where the falsifier machinery stops working rather than as a defect. Their calibration is reported separately, never pooled with testing claims |
| Attribution scoping leaks | Per-claim pseudonyms defeat timestamp correlation, but quoted text does not | Cross-scope views paraphrase or suppress quotes; the explainability cost is real and tracked as an open question (Q6) |
| Calibration needs resolved outcomes | Confidence cannot be scored until claims resolve, and many never do | Resolution loop built in P3, not P7; explicitly track the never-resolved fraction as its own metric rather than dropping it |

## 11. Immediate next steps

Q1, Q2, Q3 and Q7 are settled (see `docs/OPEN_QUESTIONS.md`). Two of the remainder gate P1's
migration and should be answered before code is written:

1. **Q5 — attribution storage shape.** Split store (shared ledger + per-person attribution
   database) or encrypted `actor` fields. Decides whether `actor_ref` is a column or a
   cross-database key. Blocking for P1.
2. **Q6 — quoted text in cross-scope views.** A verbatim quote identifies its author however
   well the ids are pseudonymized. Paraphrase, suppress, or accept the leak. Blocking for
   the P6 surfaces, and cheaper to decide before spans are designed.
3. Nominate the seed traces — roughly a dozen of each source type.
4. Write the annotation guide, with per-source-type sections, and run the agreement check.
5. Cut the P1 schema from `docs/DATA_MODEL.md` into a migration.
