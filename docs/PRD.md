# Product Requirement Document — Agentic Investment Research v1

**Version:** 1.0 (2026-10-12)
**Status:** Draft for review (this PR)
**Canonical inputs:** `PRODUCT_SPEC.md` (end-user product spec, this repo's README) and `V1_SPEC.md` (thin-slice build spec). This PRD translates those into buildable requirements scoped to v1.
**Privacy boundary (binding):** this repo contains no personal portfolio details — no positions, no dollar losses, no private tracker contents. Labeled evaluation sets are referenced only as "private" without their contents. All regression-test cases are framed as neutral historical cases (e.g., a hype-cycle premium top), never as personal history.

> **Disclaimer:** this tool is a research synthesizer, not financial advice. It never says "buy." It says: here is the evidence, here is what the price assumes, here is your invalidation trigger.

---

## 1. Problem statement

Retail investors don't lose money for lack of information — they drown in it. The real problem is the absence of a *decision process*: no systematic way to weigh evidence across sources, no contradiction detection, no position sizing, no invalidation trigger, no written record to learn from. So decisions get made on single-source conviction (one YouTuber, one headline) with full size and no exit plan.

This product delivers the analyst's *process*, not the analyst's *answer*: multi-source evidence with explicit weights, factor scores that show their work, contradictions flagged, hype labeled. Risk is unavoidable — no memo is complete without the decision box. Write-before-acting is enforced — the decision log creates the learning loop retail investing has never had.

The product succeeds when, after reading a memo, a user can (1) explain *why* the system thinks what it thinks, (2) state what would change the decision (a concrete invalidation trigger), and (3) size the position appropriately for their own risk.

## 2. Goals and non-goals

**Goals (v1):**
- One command turns a ticker into a cited investment memo with factor scores and a decision box.
- Every factual claim in every memo resolves to a real source passage (100% citation coverage).
- Cost discipline: a hard per-memo budget, enforced in code, with graceful degradation.
- Reproducible research: `--asof` replays history from cached filings so regression tests are deterministic.

**Non-goals (explicitly out of v1):**
- Web UI and the 7 end-user screens (v1 is CLI + markdown).
- Portfolio risk view, watchlist alerts, monitoring cron.
- Real-time intraday data; options data.
- YouTube/social ingestion (deferred to v1.1; sentiment in v1 is earnings-transcript tone only, weight-capped).

---

## 3. User stories

Personas: **P1 — Engaged novice investor** (real money at stake, intelligent, motivated, no professional training). **P2 — Builder/maintainer** (runs the pipeline, owns evals and releases).

| ID | Story | Persona |
|----|-------|---------|
| US-1 | As P1, I run `research TICKER` on a stock I just heard about and get a complete memo without needing to know how to read a 10-K. | P1 |
| US-2 | As P1, I see live progress while research runs ("Reading 10-K…", "Scoring quality factors…") so I trust the process before the answer. | P1 |
| US-3 | As P1, I read factor cards (Quality / Valuation / Financial Health / Regime / Sentiment) with a score, top evidence, and citations I can open to the exact source passage. | P1 |
| US-4 | As P1, I get a decision box with thesis, what-the-price-assumes, a concrete invalidation trigger (a price level), bear case, and a cap-aware size suggestion — risk is product, not a disclaimer. | P1 |
| US-5 | As P1, claims with no filing/transcript evidence are labeled UNVERIFIED rather than dropped or presented as fact — the hype filter must visibly work. | P1 |
| US-6 | As P1, I record my decision (buy / pass / watch, size, invalidation trigger) so the system holds my accountable record — write before acting. | P1 |
| US-7 | As P2, I replay history with `research TICKER --asof DATE` from cached filings so the hype-cycle regression test runs on a frozen past, not a moving present. | P2 |
| US-8 | As P2, per-memo spend is logged and a hard budget aborts a run gracefully with a partial memo + disclosure — cost discipline is enforced, not hoped for. | P2 |
| US-9 | As P2, I can run the full eval suite (`eval/`) and it gates release: hype-cycle regression, agreement case, hype-resistance, citation integrity, decision-box completeness. | P2 |

---

## 4. Functional requirements

### 4.1 CLI — entry point

**FR-1.** The product ships a single CLI: `research <TICKER> [--asof DATE]`.
- FR-1a. Ticker normalization and validation: accepts common ticker formats; rejects invalid/delisted tickers with a clear error before any spend.
- FR-1b. Default run uses latest data; `--asof DATE` replays from cached filings at that date (no new fetches for frozen history).
- FR-1c. Outputs: `output/<TICKER>/memo.md` (human-readable) and `output/<TICKER>/memo.json` (machine-readable, per §6 schema).
- FR-1d. Live progress output to the terminal naming the current stage ("Reading 10-K…", "Scoring quality factors…").

Acceptance criteria:
- [ ] `research AAPL` produces both output files for a valid ticker.
- [ ] `research NOTATICKER` fails fast with an error and spends $0.
- [ ] `--asof 2026-10-01` never triggers a live filing fetch; it reads only from cache (verifiable in the run log).
- [ ] Progress lines appear before the stage completes, in stage order.

### 4.2 Ingestion

**FR-2.** SEC EDGAR ingestion: latest annual (10-K) plus last 2 quarters (10-Q). Numbers via the companyfacts API (revenue, margins, FCF, cash, debt, D/E, retained earnings); section text from filing HTML via EDGAR archives — **10-K Items 1A (Risk Factors), 7 (MD&A), 8 (Financial Statements); 10-Q Part I Items 1 (Financial Statements) and 2 (MD&A), Part II Item 1A (Risk Factors)**.
- FR-2a. Filings cached locally; never re-fetch within the cache window.
- FR-2b. Fixture-based tests assert section extraction against known filings.

**FR-3.** Earnings transcripts: latest 2 quarters, sourced from Alpha Vantage `EARNINGS_CALL_TRANSCRIPT` (free tier, 25 req/day — verified live). The ingester abstracts the source behind an interface so the provider can swap later.
- FR-3a. Missing transcripts (or a transcript not yet available — post-call latency is unverified, ~1 day) degrade gracefully: the memo proceeds with the gap disclosed in `memo.md` ("No transcript available for Q2 2026"), never hidden.
- FR-3b. Quarter parameter is fiscal (`YYYYQM`); non-calendar-FY names (e.g., AAPL) require a fiscal-to-calendar quarter mapping.
- FR-3c. No paid fallback in v1: the chain is cache → Alpha Vantage → EDGAR 8-K disclosure.

**FR-4.** Market data: price history, 200DMA, 52-week high/low. No real-time intraday data in v1.

Acceptance criteria:
- [ ] `research AAPL --asof 2026-10-01` with a warm cache makes zero EDGAR calls (cache-window test).
- [ ] A ticker with a missing transcript still produces a complete memo with the gap disclosed.
- [ ] EDGAR section extraction passes against committed fixtures.

### 4.3 Factor agents

Each agent emits structured JSON with `score`, `max`, and `evidence[]` where every claim carries a citation. Qualitative inputs that cannot be auto-scored are emitted as `MANUAL_REVIEW` with a targeted prompt, never faked.

**FR-5. Quality agent** — the 12-category exceptional-business test.
- FR-5a. Quantitative categories auto-scored from filings.
- FR-5b. Qualitative categories → `MANUAL_REVIEW` with a targeted prompt for the user.
- FR-5c. Scored against the private 17-name labeled set: known-exceptional names must score high, known-rejects low (accuracy threshold, §7).

**FR-6. Valuation agent** — fine-tuned Graham IV: `IV = EPS × (7 + g) × (4.4 / bond yield)`.
- FR-6a. Normalized-EPS path for cyclicals (avoids peak-cycle EPS).
- FR-6b. Negative EPS → `N/A` by design, never a garbage number.
- FR-6c. IV% math verified against the private hand-checked IV sheet within tolerance.

**FR-7. Financial-health agent** — Buffett balance-sheet rules as pass/fail checks with numbers: cash > debt, D/E < 0.88, no preferred, retained-earnings growth.

**FR-8. Regime agent** — 200DMA position, weekly trend readout, staged-entry framing; bear-regime (below 200DMA on a weekly downtrend) excludes mean-reversion entries.

**FR-9. Sentiment agent** — earnings-transcript tone only in v1, weight capped at the floor in synthesis. YouTube/social deferred to v1.1.

Acceptance criteria:
- [ ] All five agents run on a fresh ticker and each emits cited JSON; any agent that cannot compute a score emits `MANUAL_REVIEW` or `N/A`, never a hallucinated score.
- [ ] Quality agent accuracy on the private labeled set meets the agreed threshold.
- [ ] Valuation agent reproduces the private hand-checked IV sheet within tolerance, including negative-EPS and cyclical-normalization edge cases.
- [ ] No agent contributes more than its capped weight to synthesis (weight-sum test).

### 4.4 Source hierarchy + hype filter

**FR-10.** Source hierarchy enforced at synthesis: filings > transcripts > market data > commentary. Weights sum to 1; social/YouTube capped at the floor (0 in v1 — deferred).
- FR-10a. Any claim without filing/transcript evidence is labeled `UNVERIFIED` in the memo — never silently dropped, never presented as fact.
- FR-10b. Contradiction detection: synthetic management-claims-vs-numbers cases are flagged with both sides cited.

Acceptance criteria:
- [ ] Unit test: weights sum to 1; social weight ≤ floor.
- [ ] Pipeline test: synthetic contradiction fixtures are flagged with both sides cited.
- [ ] E2E: a ticker with loud commentary bullishness but weak filings comes back with sentiment downweighted and labeled, never parroting the hype (hype-resistance test, §7).

### 4.5 Synthesis — outputs

**FR-11.** Synthesis produces `memo.md` containing, in order: factor cards (score + top evidence + citations), synthesis memo (thesis, bull/base/bear, what-the-price-assumes), decision box (§4.6), gap disclosures, cost footer.
- FR-11a. `memo.json` conforms to the §6 schema.
- FR-11b. Citation format `[source-type:doc-id#passage]` resolves to cached raw text — clicking through lands on the exact passage.
- FR-11c. Citation integrity: 100% of factual claims resolve to a real source passage. Zero hallucinated citations — the single most important e2e test.

Acceptance criteria:
- [ ] Every factual claim in a generated memo resolves to a cached passage (automated citation-integrity test).
- [ ] `memo.json` validates against the schema (§6); invalid output is a pipeline failure.
- [ ] Sampled claims verified against cited passages meet ≥ 95% precision target.

### 4.6 Decision box (mandatory)

**FR-12.** Every memo contains the decision box with exactly these fields; a memo without a complete decision box is invalid and must fail the run:
- `thesis` (1–2 lines)
- `bull` / `base` / `bear` cases
- `price_assumes` (what the price already assumes)
- `invalidation_trigger`: a concrete price level with a condition — never vague ("watch closely")
- `suggested_size_pct`: cap-aware (respects the single-name cap and speculative-sandbox cap)
- `bear_case`

Acceptance criteria:
- [ ] Completeness check: all six fields present, non-empty, on every memo; missing field = run failure.
- [ ] Invalidation trigger parses as a concrete level + condition (regex/struct check), not freeform advice.
- [ ] Size math unit-tested against caps (5% single-name, sandbox limits).

### 4.7 Cost accounting

**FR-13.** Per-memo spend logged to `cost: {usd, tokens}` in `memo.json` and printed in the memo footer.
- FR-13a. Hard per-memo budget (default TBD — open question §9; configurable via env/flag).
- FR-13b. Budget breach → graceful abort: run halts, partial memo is written with a prominent disclosure that it is incomplete and why.

Acceptance criteria:
- [ ] Over-budget run halts and writes a disclosed partial memo (budget-enforcement test).
- [ ] Cost accounting unit tests: aggregation correct across stages; budget check evaluated before each stage.

### 4.8 Decision log (write-before-acting)

**FR-14.** Append-only local JSONL decision log records user decisions: ticker, decision (buy/pass/watch), size, invalidation trigger, timestamp, memo reference.
- FR-14a. Log is file-based, local-first, gitignored (contains the user's private decisions — never committed to the repo).
- FR-14b. Recording is a separate explicit CLI step after reading the memo (e.g., `research record ...`), not automatic.

Acceptance criteria:
- [ ] Recorded decisions append to JSONL with all fields; the file is absent from git (gitignore test).
- [ ] No decision is ever auto-recorded — recording requires an explicit user action.

---

## 5. Non-functional requirements

- **NFR-1 Cost:** hard per-memo budget (default TBD, configurable); breach → graceful abort (FR-13).
- **NFR-2 Latency:** p50 < 10 minutes per ticker (measure; target, not promise).
- **NFR-3 Determinism:** same `--asof` + warm cache → same memo modulo model nondeterminism. Temperature 0 for extraction stages; model + prompt versions logged in `provenance`.
- **NFR-4 Secrets:** API keys via environment only. Never in the repo, docs, issues, or logs. Filings cache gitignored.
- **NFR-5 Privacy:** no personal portfolio data anywhere in the repo, docs, or issues (binding — see header).

---

## 6. Data contracts

`memo.json` top-level:

```json
{
  "ticker": "AAPL",
  "asof": "2026-10-01",
  "factors": [
    {
      "name": "quality",
      "score": 100,
      "max": 125,
      "evidence": [
        { "claim": "…", "source": { "type": "10-K", "doc_id": "…", "passage": "…" } }
      ]
    }
  ],
  "synthesis": {
    "thesis": "…",
    "bull": "…", "base": "…", "bear": "…",
    "price_assumes": "…",
    "mind_changers": ["…"]
  },
  "decision_box": {
    "thesis": "…",
    "bull": "…", "base": "…", "bear": "…",
    "price_assumes": "…",
    "suggested_size_pct": 2.0,
    "invalidation_trigger": "Weekly close below $X",
    "invalidation_condition": "…",
    "bear_case": "…"
  },
  "cost": { "usd": 1.23, "tokens": 45678 },
  "provenance": { "model": "…", "prompt_versions": { "…": "…" }, "fetched_at": "…" }
}
```

- Citation format: `[source-type:doc-id#passage]`, resolvable to cached raw text.
- Output layout: `output/<TICKER>/memo.md`, `output/<TICKER>/memo.json` (+ `run.log`, cached sources under `output/<TICKER>/sources/`).
- `decision-log.jsonl` lives at the repo/working root, gitignored.

---

## 7. Test strategy and release gates

### 7.1 Unit tests (`pytest`)
- IV formula math and edge cases (negative EPS → N/A; cyclical normalization; yield inputs).
- Source-weight application: weights sum to 1; social capped at floor.
- Citation parse/format round-trip.
- Per-memo cost accounting and budget enforcement.
- Ticker normalization and validation (invalid/delisted).
- Position-size math against caps (5% single-name, sandbox limits).
- Deduplication and recency-weighting on synthetic narrative sets.

### 7.2 Pipeline tests (key stages)
- EDGAR section extraction against committed fixtures: asserts the exact sections in FR-2 parse cleanly (10-K Items 1A/7/8; 10-Q Part I Items 1–2, Part II Item 1A) and extracted numbers reconcile with companyfacts values within tolerance.
- Transcript-missing degradation: gap disclosed, never hidden.
- Quality agent vs. private 17-name labeled set — accuracy threshold (exceptional high, rejects low).
- Valuation agent vs. private hand-checked IV sheet — tolerance-based.
- Contradiction detection on synthetic management-vs-numbers cases.
- Decision-box concreteness check (trigger is a level, never vague).

### 7.3 End-to-end evals (`eval/`, run on historical snapshots)
- **Hype-cycle regression:** `--asof` at a hype-cycle top (neutral historical case, e.g., a 2025 crypto-treasury premium top) → memo flags the collapsed premium / broken thesis and recommends pass-or-tiny size. If it can't catch a textbook top, it isn't ready.
- **Agreement case:** a name where quality and value lenses agree → constructive memo with intact risk framing — conviction without cheerleading.
- **Hype-resistance:** loud commentary bullishness, weak filings → sentiment downweighted and labeled, never parroted.
- **Citation integrity:** 100% of factual claims resolve to real source passages.
- **Decision-box completeness:** all six fields present on every memo.

### Eval quality rubric (dimensions every release must pass)

The e2e cases above test outcomes; this rubric defines what "green" means for memo quality. Three dimensions, each with a measurement:

- **Groundedness** (nothing unbacked): automated check — 100% of factual claims resolve to a real source passage; sampled citation precision ≥ 95% (claim text entailed by the cited passage), scored by a pinned judge model + human spot audit on release candidates. Failing either blocks release.
- **Accuracy** (the machinery is right): factor-agent outputs vs. private labeled oracles — quality agent meets the agreed accuracy threshold on the private 17-name set; valuation agent reproduces the hand-checked IV sheet within tolerance; financial-health pass/fails recomputed deterministically from filings with zero tolerance for mismatch.
- **Relevancy** (the memo serves the decision): judge-model rubric — after reading, can the reader size a position and name a concrete invalidation trigger (target ≥ 4/5)? Every factor card must carry at least one evidence item that bears on the decision box. Generic market commentary or filler sections fail the memo.

**Eval dependencies (release blockers, not nice-to-haves):** the eval suite cannot be "green" without (a) the private labeled sets existing and versioned, (b) frozen `--asof` fixtures for the regression cases, and (c) a pinned judge model + prompt version for the LLM-graded dimensions. These are prerequisites of the first release gate, owned by the eval-harness ticket.

### 7.4 Release gates (v1 is done when)
- [ ] `research AAPL --asof 2026-10-01` → `memo.md` + `memo.json`; all claims cited; decision box complete; cost under budget.
- [ ] `pytest` green; `eval/` suite green, including all regression cases.
- [ ] README documents install, configuration, and how to run; carries the not-financial-advice disclaimer.
- [ ] No secrets in the repo; no personal portfolio data in the repo, docs, or issues.

---

## 8. Out of scope (v1)

Web UI and the 7 end-user screens; portfolio risk view; watchlist alerts; monitoring cron; real-time intraday data; options data; YouTube/social ingestion (v1.1); multi-ticker compare view; alerts/notifications.

---

## 9. Open questions

1. ~~Earnings-transcript source (free vs. paid API).~~ — **Decided 2026-10-13:** Alpha Vantage free tier (`EARNINGS_CALL_TRANSCRIPT`, verified live), no paid fallback; chain = cache → Alpha Vantage → EDGAR 8-K disclosure.
2. LLM provider/model (config-driven; start with one).
3. Exact per-memo $ budget (default TBD).
4. Which 3 tickers anchor the demo + eval baseline.
5. Accuracy threshold for the quality-agent labeled set; tolerance for the valuation-agent IV sheet.

---

## 10. Ticketing plan (next step after this PRD)

Break this PRD into ordered GitHub issues, each leaving the repo in a working state, composing into end-to-end v1. Suggested ordering: repo skeleton + CLI shell → ingestion (EDGAR + cache) → market data → factor agents (one per ticket: quality, valuation, financial health, regime, sentiment) → source hierarchy + hype filter → synthesis + memo.json schema → decision box → cost accounting + budget → decision log → unit test backfill → pipeline tests + fixtures → eval harness + regression cases → README/docs + disclaimer → release gate checklist.
