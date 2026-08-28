# ClaimsCheck

**A ledger for claims, evidence, and confidence — shared memory for collaborators separated
by role or by time.**

ClaimsCheck supports collaboration on a project — between people, between agents, and across
time — by tracking the claims made about it: what was asserted, where each assertion was
reused, what evidence stands behind it, and what that evidence actually supports.

The motivation is a pattern worth naming. A confident assertion gets made as **color
commentary** — an aside, an impression, not a position anyone intended to defend. It is
picked up and reused later, and by then nobody notices it was never grounded in a
requirement, a hypothesis, or evidence. Somewhere between the remark and its reuse it was
promoted from commentary to premise, without passing through any step that would have tested
it.

That origin is what makes the failure hard to catch. A wrong hypothesis announces itself as
something to be checked. An aside does not, and so nothing ever checks it — it acquires the
standing of a claim only in retrospect, by being used as one.

## The core signal

Two numbers are tracked separately for every claim:

| | Derived from | Grows when |
|---|---|---|
| **Support** | Evidence only | A receipt lands (test, telemetry, reproduction) |
| **Entrenchment** | Social weight only | The claim is restated, cited, or depended upon |

When entrenchment outruns support, the gap is **assumption debt** — belief the project
has taken on without paying for it. Surfacing that gap is the point of the system.

## Scope

Initial scope is **software testing**: claims like *"this path is covered"*, *"that flake
is a network timeout"*, *"this change is safe because X"*. These are chosen deliberately
because the receipts already exist and are machine-readable — test results, CI runs,
coverage data, git history, production telemetry. Once the loop is proven where ground
truth is cheap, it widens to contexts where it is not.

## Documents

- [`docs/PROJECT_PLAN.md`](docs/PROJECT_PLAN.md) — goals, phases, deliverables, risks
- [`docs/DATA_MODEL.md`](docs/DATA_MODEL.md) — entities and storage schema
- [`docs/CONFIDENCE_MODEL.md`](docs/CONFIDENCE_MODEL.md) — the ladder, scoring, decay, calibration
- [`docs/SEMANTICS.md`](docs/SEMANTICS.md) — terms, logical and causal relations, propagation
- [`docs/PATTERNS.md`](docs/PATTERNS.md) — the drift and entrenchment detector catalog
- [`docs/OPEN_QUESTIONS.md`](docs/OPEN_QUESTIONS.md) — decisions still needed

## Claims are structured, not text

A claim is built from **defined terms** and connected to other claims by **typed relations**.
Logical ones — `entails`, `negates`, `contradicts`, `specializes` — and causal ones, which
keep `precursor` ("X reliably precedes Y") distinct from `necessary` ("without X, no Y"),
because the first routinely gets restated as the second with nothing acquired in between.

Inference over that graph is deliberately lopsided: refutation propagates freely, support
propagates only downward and attenuated, and never upward. The rule is to enable what lowers
confidence first and what raises it last, if at all.

Relations are themselves claims, with their own evidence and confidence. A graph assembled
from casual remarks has to behave like casual remarks rather than like a proof.

## Decided so far

- **Python 3.11+ / SQLite**, local-first, CLI and MCP server as the primary surfaces.
- **Agents write freely**; low-confidence extractions are quarantined — stored and visible,
  but excluded from every score until confirmed.
- **Attribution is personal, propositions are shared.** Repetition is only countable across
  people, so claims and evidence are shared; *who said it* is scoped to the asserter, with
  per-claim pseudonyms elsewhere. Agent assertions are attributed openly.
- **The gold corpus spans three source types** — agent transcripts, PR/issue threads, and
  design docs/RFCs — so we learn early whether the model survives a receipt-poor domain.

## Status

Planning. No implementation yet. Everything in these documents is itself a claim at
ladder level **L0 (Asserted)** — see `docs/CONFIDENCE_MODEL.md` for what that means and
what it would take to raise it.
