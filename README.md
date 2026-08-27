# ClaimsCheck

**A ledger for claims, evidence, and confidence — so that repetition never substitutes for proof.**

An overly confident statement, repeated enough times, quietly becomes an assumption.
Nobody decides to believe it; it just stops being questioned. By the time it fails,
its origin is untraceable and everything built on top of it is suspect.

ClaimsCheck is a bookkeeping system that makes that failure mode visible. It records
every claim made in the course of engineering work, the **context** the claim was made
in, the **evidence** attached to it, and the **confidence** that evidence actually
justifies — then tracks how each of those changes over time.

The model is borrowed from requirements tracking, where a need travels from *want* to
*provided certainty* with test and telemetry receipts attached at each step. ClaimsCheck
applies the same discipline to the informal claims that fill dialogue traces, PR threads,
and design discussions.

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
- [`docs/PATTERNS.md`](docs/PATTERNS.md) — the drift and entrenchment detector catalog
- [`docs/OPEN_QUESTIONS.md`](docs/OPEN_QUESTIONS.md) — decisions still needed

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
