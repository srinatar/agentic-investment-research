# agentic-investment-research

**Institutional-grade investment research, democratized.** A multi-agent system that turns
a stock ticker into a cited investment memo with explicit risk framing — in minutes,
not weeks.

> Status: pre-v1. End-user spec defined; v1 build in progress.

## The problem

Retail investors don't lose money for lack of information — they drown in it. What's
missing is a *decision process*: no systematic way to weigh evidence across sources, no
contradiction detection, no position sizing, no invalidation trigger, no written record
to learn from. So decisions get made on single-source conviction (one video, one
headline) with full size and no exit plan.

## For whom

The engaged novice: real money at stake, intelligent, motivated, but without professional
training or 40 hours per ticker. Not day traders, not professionals with Bloomberg
terminals, not index-only investors.

## How it solves it

It delivers the analyst's *process*, not the analyst's *answer*:

- **Multi-source evidence, explicitly weighted** — SEC filings first, then earnings
  transcripts, market data, commentary, and social sentiment last. Every claim cited to
  a source passage you can open and verify.
- **Factor agents that show their work** — quality, valuation, financial health, price
  regime, and sentiment, each scored with its evidence on the table.
- **Signal from noise** — contradiction detection when management's story diverges from
  the numbers; a hype filter that labels unverified claims instead of parroting them.
- **Risk as product, not disclaimer** — no memo ships without a decision box: suggested
  position size, a concrete invalidation trigger, the bear case, and what the price
  already assumes.
- **Write before acting** — decisions get recorded with their invalidation triggers, so
  mistakes become learnable instead of repeatable.

It never says "buy." It says: here is the evidence, here is what the price assumes,
here is your invalidation trigger.

## What a memo contains

1. **Factor cards** — Quality / Valuation / Financial Health / Regime / Sentiment, each
   with a score, top evidence, and citations.
2. **Synthesis memo** — thesis, bull/base/bear, what the price assumes, what would
   change the mind.
3. **Decision box** — suggested size (cap-aware), invalidation trigger (a concrete
   level), bear case.

## Metrics we hold ourselves to

- 100% of factual claims cited to a real source passage; citation precision ≥ 95%
- Every memo draws on 3+ independent source types
- Decision readiness: the reader can state what would change the decision
- Cost per memo tracked against a hard budget — cost discipline is a feature
- Regression suite on historical cases must stay green on every release

## Roadmap

- **v1** — CLI: `research <TICKER>` → cited memo (`memo.md` + `memo.json`) with factor
  scores, decision box, and cost accounting. SEC EDGAR + earnings transcripts + market
  data. Eval harness with citation-integrity, factor-accuracy, and historical-regression
  tests.
- **v1.1** — Social/YouTube sentiment ingestion (downweighted by design), minimal memo
  viewer.
- **v2** — Watchlist with invalidation alerts, decision log with outcome tracking,
  portfolio risk view (concentration + thematic overlap).

## Run it (v1, when ready)

```bash
pip install -r requirements.txt
export LLM_API_KEY=...
python -m research AAPL --asof 2026-10-01
# → output/AAPL/memo.md + memo.json
```

## Tests

```bash
pytest            # unit + pipeline tests
python -m eval.run  # end-to-end outcome tests (citation integrity, factor accuracy,
                    # historical regression cases)
```

## Disclaimer

Research and educational purposes only. **Not financial advice.** This tool synthesizes
public information; it does not recommend securities transactions. Do your own diligence.

## License

MIT — see [LICENSE](LICENSE).
