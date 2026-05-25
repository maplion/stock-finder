# Feature Landscape — Stock Finder

**Domain:** Personal-use stock-research aggregator (desktop)
**Researched:** 2026-05-25
**Initial tickers:** MU (semiconductor cyclical) + IRM (data-center / records-storage REIT)
**Confidence:** HIGH (table stakes are well-established conventions; differentiators flagged at MEDIUM)

---

## How to Read This Document

Three categories, in priority order for v1 scope:

1. **Table Stakes** — must exist or the tool fails its core value ("one place to look")
2. **Differentiators** — what makes this noticeably better than free public dashboards
3. **Anti-Features** — explicit Out of Scope items with the reason logged so they cannot quietly creep back in

Every feature includes: complexity (S/M/L/XL), dependencies, and REIT vs common-stock applicability.

**Complexity legend**
- **S** = 1–2 days end-to-end (UI + data + glue)
- **M** = 3–7 days
- **L** = 1.5–3 weeks
- **XL** = 1 month+ (model training, evaluation infra, etc.)

---

## 1. Table Stakes — By Panel

### 1.1 Tracked-Tickers / Watchlist Shell

| Feature | Description | Complexity | Dependencies | Applies To |
|---|---|---|---|---|
| Ticker list (add/remove) | User-editable list, persisted locally (JSON or SQLite). Starts seeded with MU + IRM. | S | None | Both |
| Ticker validation on add | Reject unknown symbols by probing yfinance/lookup before saving. | S | Ticker list | Both |
| Per-ticker route / dashboard view | URL or tab per ticker; same panel layout for all. | S | Ticker list | Both |
| Sector tag on ticker | Auto-classify (e.g., `REIT`, `Semiconductor`, `Default`) on add; drives which fundamentals template renders. | S | Ticker list | Both (drives REIT extension) |
| Manual refresh button per panel | On-demand fetch is the model; no auto-refresh. Show "Last fetched at HH:MM". | S | Per panel | Both |
| Global "refresh all panels for this ticker" action | Single click → fan out to all data sources. | S | Per panel | Both |

**Why this is table stakes:** without a watchlist + per-ticker view, there is no app — every other panel hangs off this.

---

### 1.2 Fundamentals Panel — Common (applies to MU, IRM, and any future ticker)

Source of truth: most recent 10-Q / 10-K via SEC EDGAR or yfinance `info` + `financials`. Show TTM and most recent quarter side by side.

| Metric | Why it matters | Complexity | Dependencies |
|---|---|---|---|
| Revenue (TTM, latest Q, YoY %) | Top-line trajectory | S | Data fetch |
| Gross margin (TTM, latest Q) | Pricing power; critical for semis | S | Data fetch |
| Operating margin (TTM, latest Q) | Operating leverage | S | Data fetch |
| Net income (TTM, latest Q) | Bottom-line | S | Data fetch |
| GAAP EPS + non-GAAP EPS (if reported) | Street tracks non-GAAP; show both | S | Data fetch |
| Free cash flow (TTM) | Quality of earnings | S | Data fetch |
| Cash & equivalents (latest balance sheet) | Liquidity | S | Data fetch |
| Total debt (latest balance sheet) | Leverage | S | Data fetch |
| Net debt / EBITDA | Leverage normalized | S | Data fetch |
| Current ratio | Short-term solvency | S | Data fetch |
| Shares outstanding + diluted count | Dilution watch | S | Data fetch |
| Dividend per share + yield + payout ratio | Income perspective | S | Data fetch |
| P/E (trailing & forward), P/S, EV/EBITDA | Valuation snapshot | S | Data fetch |
| Revenue & EPS trend mini-sparklines (8 quarters) | Trend at-a-glance | M | Historical fetch |

**UI guidance:** Two-column "Common Metrics" block, grouped: Profitability / Liquidity & Leverage / Valuation / Return to Shareholders. Show absolute + YoY % delta. Red/green tint on deltas.

---

### 1.3 Fundamentals Panel — REIT Extension (auto-shown for IRM, any `sector=REIT` ticker)

REITs report on a different basis. P/E and net income are misleading for REITs because of large depreciation on real assets. Required additions:

| Metric | Why it matters for IRM | Complexity | Source |
|---|---|---|---|
| **FFO** (Funds From Operations, TTM + latest Q) | The REIT equivalent of net income; strips real-estate depreciation | S | Press release / 8-K (REIT companies disclose in earnings) |
| **AFFO** (Adjusted FFO, TTM + latest Q) | Closer to true distributable cash; preferred for dividend coverage | S | Press release / 8-K |
| **FFO per share / AFFO per share** | Per-share basis for valuation | S | Press release |
| **P/FFO and P/AFFO** | REIT-appropriate valuation multiples (use these instead of P/E) | S | Computed |
| **AFFO payout ratio** (dividend ÷ AFFO) | Dividend sustainability — under ~80% is generally healthy | S | Computed |
| **Net debt / EBITDA (REIT-adjusted)** | Standard REIT leverage metric; IRM targets a specific band | S | Data fetch |
| **Interest coverage ratio** | Ability to service debt; critical given REIT leverage profiles | S | Data fetch |
| **NOI** (Net Operating Income) — total & YoY growth | Property-level profitability before corporate overhead | M | 8-K or supplemental |
| **Occupancy / utilization** (where reported — IRM reports storage utilization, data-center leased MW) | Demand signal | M | Supplemental package (parsed; may need Playwright) |
| **Same-store NOI growth** | Organic growth excluding M&A — preferred quality metric | M | Supplemental package |

**UI guidance:** When `sector === 'REIT'`, render an additional "REIT Metrics" block below Common Metrics. Hide P/E (or de-emphasize) and elevate P/AFFO. Show AFFO payout ratio with a colored band (green <80%, yellow 80–95%, red >95%).

**Note on data acquisition:** FFO/AFFO/NOI are not in `yfinance`. They live in REIT earnings releases (8-K supplemental). For v1, parsing the latest supplemental PDF/HTML via Playwright + a small extraction prompt is the realistic path. Expect this to be the highest-friction data source.

---

### 1.4 Fundamentals Panel — Semiconductor Sector Helpers (applies to MU)

Lighter than the REIT extension. These are not strictly required for v1 but pay off the heterogeneity premise of MU + IRM coexisting.

| Metric | Why for MU | Complexity |
|---|---|---|
| Inventory & inventory days | Memory cycle indicator — rising inventory days precede downturns | S |
| CapEx (TTM) and CapEx / Revenue | Capital intensity of the cycle | S |
| Forward guidance (revenue + EPS midpoint from latest earnings call) | Memory market is guided heavily — current guide is the signal | M (parse from press release) |
| DRAM / NAND segment split (if reported) | Mix shift matters for margin | M (parse) |

**Verdict:** Inventory days + CapEx are S complexity (already in financials) — include in v1. Segment split and guidance parse are M — defer to v1.1.

---

### 1.5 Forecasts Panel

The user explicitly wants **three models side-by-side, not blended**, across three horizons (1–3 mo, 3–12 mo, 1–3 yr).

| Feature | Description | Complexity | Dependencies | Applies To |
|---|---|---|---|---|
| Analyst consensus forecast | Mean / high / low analyst price target + # of analysts (yfinance `info.targetMeanPrice` etc.) | S | yfinance | Both |
| Statistical baseline forecast (Prophet or ARIMA) | Run on N years of daily OHLC; emit point forecast + 80% confidence interval | M | Historical price cache (small parquet) | Both |
| ML forecast | LSTM or gradient-boosted regressor on engineered features (returns lags, RSI, MACD, sentiment, etc.) | XL | Historical price cache, sentiment history (optional), feature pipeline | Both |
| Horizon selector | Toggle 1–3 mo / 3–12 mo / 1–3 yr; each model emits forecast for selected horizon | S | All three models | Both |
| Unified forecast visualization | One chart: historical price (gray) + three model trajectories with their CI bands. Distinct colors per model. | M | All three models | Both |
| Tabular forecast summary | Below chart: 3-column table — Model / Point estimate / 80% CI / Implied % vs current | S | All three models | Both |
| "Model says vs current price" delta | Big number, color-coded by direction and magnitude | S | All three models | Both |

**Minimum useful UI (do not skip):** point estimate + confidence interval per model. A bare number is decision-poor; a fan chart with overlapping bands is the smallest useful presentation.

**Side-by-side without overload:** one chart with three forecast lines + their CI bands, plus a 3-row summary table. Do not generate 9 charts (3 models × 3 horizons) — the horizon is the selector, not a small-multiple. Resist the urge to ensemble; the user wants the disagreement visible.

**Phasing recommendation:** v1 = analyst consensus + Prophet baseline. ML model is XL and depends on having a feature pipeline + historical labels — defer to v1.1 unless explicitly prioritized for v1. The forecasts panel works with 2 of 3 models from day one; the third slot reads "coming soon" until the ML model lands.

---

### 1.6 Sentiment Panel — News

| Feature | Description | Complexity | Dependencies | Applies To |
|---|---|---|---|---|
| Headline ingestion | Pull last N headlines for the ticker (yfinance.news, NewsAPI free tier, or Google News RSS) | S | Ticker list | Both |
| Per-headline sentiment score | Run a finance-tuned classifier (FinBERT via Python sidecar) or VADER as a cheaper baseline. Score in [-1, +1]. | M (FinBERT) / S (VADER) | Headlines | Both |
| Aggregated score (last 7d / 30d) | Volume-weighted mean of headline scores | S | Per-headline scoring | Both |
| Headline list with score + link | Table: date / source / headline / score / open-link | S | Per-headline scoring | Both |
| Sentiment trend mini-chart (last 30 days) | Daily mean sentiment as a sparkline | M | 30-day headline history (small cache) | Both |

**What makes a score actually useful vs noise:**
- **Per-source visible, not just aggregated.** A single tabloid headline should not drown three Reuters pieces. Show per-headline scores so the user can sanity-check.
- **Time series, not a single number.** A "-0.2" today is meaningless without the 30-day baseline. Sparkline or "vs prior 7d" delta is the smallest useful framing.
- **Volume + tone, both.** A +0.5 score on 2 headlines is weaker than +0.2 on 40 headlines. Display both.

---

### 1.7 Sentiment Panel — Reddit

| Feature | Description | Complexity | Dependencies | Applies To |
|---|---|---|---|---|
| Reddit ingestion (WSB + r/stocks + r/investing) | Pull posts/comments containing ticker via Reddit JSON API (free, no auth needed for read) over trailing N days | M | Ticker list | Both |
| Mention count (24h / 7d / 30d) + delta vs prior period | Volume is the primary Reddit signal; sudden spikes matter | S | Reddit ingestion | Both |
| Per-post sentiment (VADER is fine here — WSB-speak ≠ formal finance language) | Score each post body | M | Reddit ingestion | Both |
| Aggregated tone (mean sentiment, weighted by score / upvotes) | Single number for tone | S | Per-post scoring | Both |
| Hot posts surface | Top 5 posts by upvotes in the window, with link + score | S | Reddit ingestion | Both |

**Calibration warning:** Reddit signal-to-noise differs by subreddit. WSB skews bullish and emoji-heavy; r/investing skews skeptical. Show per-subreddit breakdown so the user can read the room. Do not naively merge.

**Verified pattern** (from sentiment platforms in the wild): mention-volume spike is a higher-signal feature than tone alone. Surface volume first, tone second.

---

### 1.8 Momentum / Technicals Panel

The user named four candidates (RSI, MACD, moving-average crosses, volume surges). All four are decision-grade for personal investors and standard across every charting platform. Recommendation: ship all four, plus one more.

| Signal | What it tells you | Complexity | Dependencies |
|---|---|---|---|
| RSI (14-day) with overbought (>70) / oversold (<30) bands | Momentum extremes | S | Daily OHLC |
| MACD (12, 26, 9) with histogram + signal-line crossover flag | Trend direction + momentum shifts | S | Daily OHLC |
| Moving-average state (price vs 50-day and 200-day SMA; golden/death cross detection) | Trend regime | S | Daily OHLC |
| Volume surge flag (today's volume vs 20-day avg, >2x triggers flag) | Conviction signal | S | Daily OHLC |
| 52-week high/low distance (% from each) | Position in range | S | Daily OHLC |

**What to NOT add to v1** (vanity-chart territory): Bollinger Bands, Ichimoku, Williams %R, stochastic oscillator, on-balance volume, Parabolic SAR. These are noise unless the user explicitly works with them. Source consensus: RSI + MACD + MA combination, optionally with volume confirmation, is the durable retail-investor toolkit.

**UI guidance:** a single "Signals" panel that lists each signal as a row with: name / current value / state badge (e.g., "Oversold", "Bullish crossover", "Golden cross active"). Below: one combined price chart with 50/200 SMA overlay, RSI sub-panel, MACD sub-panel. Don't make the user interpret raw numbers — surface the state.

---

### 1.9 Insider Trades Panel

Source: SEC EDGAR Form 4 filings (free, structured XML). Free APIs exist (e.g., `sec-api` free tier) or scrape directly.

#### Transaction table — fields per row

| Field | Why |
|---|---|
| Filing date | Recency |
| Transaction date | Actual trade date (often days before filing) |
| Insider name | Who |
| Insider role (CEO, CFO, Director, 10%-owner) | Significance — CEO/CFO buys are highest signal |
| Transaction code (P = open-market buy, S = open-market sale, A = grant, M = option exercise, etc.) | **Critical — only code P/S carry signal; grants/exercises are noise** |
| Shares | Size |
| Price per share | Value |
| Total $ value | Conviction |
| Shares owned post-transaction | Ownership trajectory |

**Mandatory filter:** default to transaction codes P (purchase) and S (open-market sale). Hide grants and option exercises behind an "All transactions" toggle. Source consensus: only code P represents voluntary capital commitment and is the academic signal.

#### Cluster / pattern aggregations (the higher-signal layer)

| Aggregation | Default threshold | Why this matters |
|---|---|---|
| **Cluster buy flag** | ≥3 unique insiders bought (code P) within 10 days, each transaction ≥$25k | The canonical cluster-buy signal; academically linked to above-average forward returns |
| **Cluster sell flag** | ≥3 unique insiders sold (code S) within 10 days | Mirror signal; weaker (sales have many non-signal motivations) |
| **Net insider $ flow (30/90 day)** | Sum(code P $) − Sum(code S $) over window | Direction and magnitude of net insider conviction |
| **CEO/CFO action highlight** | Surface any code P or S by CEO/CFO regardless of cluster | C-suite trades carry asymmetric signal |
| **Trailing-12-month transaction count by type** | Row count P vs S vs A vs M | Context for "is this normal activity for this name?" |

**UI guidance:** Top of panel = signal badges (cluster buy active, net 90d flow $X). Middle = filterable transaction table. Bottom = simple bar chart of net insider $ flow per month over last 12 months.

**REIT vs common-stock note:** Same panel works for both. No REIT-specific extension needed here.

---

## 2. Differentiators

These are what separate this tool from "what I can get free on Yahoo Finance / Finviz / Seeking Alpha." Ranked by leverage relative to implementation cost.

### 2.1 Sector-Aware Fundamentals (HIGH leverage — already in v1 scope)

| Feature | Why it differentiates | Complexity | Dependencies |
|---|---|---|---|
| Auto-render REIT metrics for REITs, semiconductor helpers for semis, etc. | Yahoo Finance shows the same template for everything. P/E for a REIT is misleading. A dashboard that knows the difference is a real edge. | M (per sector template) | Sector tag on ticker, Fundamentals panel |

**Recommendation:** v1. This is the whole reason MU + IRM were chosen as the initial pair — the dashboard must demonstrate the heterogeneity premise from day one.

### 2.2 Cross-Ticker Comparison View (MEDIUM leverage)

| Feature | Why it differentiates | Complexity | Dependencies |
|---|---|---|---|
| Side-by-side comparison of 2–4 tickers across the common metrics (fundamentals, momentum state, sentiment, insider flow) | Lets the user think relatively — "is MU's debt load normal vs peers" | M | All panels exist |

**Recommendation:** v1.1. Genuinely valuable but the per-ticker view must be solid first; comparison is cheap to layer on once panels are stable.

### 2.3 Forecast Back-Tracking (HIGH leverage, MEDIUM cost)

| Feature | Why it differentiates | Complexity | Dependencies |
|---|---|---|---|
| "Model said $X 90 days ago; actual is $Y. Hit rate: Z% in CI." | No public dashboard shows this. Builds trust (or appropriate distrust) in each forecast source over time. | M | Persist forecasts when generated, compare on later loads |

**Recommendation:** v1.1. Requires a small persistence layer (forecast log), but the per-model honesty is the killer feature for a "comparison, not ensemble" forecast UX. **The forecast panel without back-testing is just guesses; with it, it's a calibrated tool.**

### 2.4 Combined Sentiment × Momentum Quadrant (MEDIUM leverage)

| Feature | Why it differentiates | Complexity | Dependencies |
|---|---|---|---|
| 2D scatter — x = momentum score composite, y = sentiment composite — ticker dot per watchlist member | Visual portfolio-level read of "what's hot, what's hated, what's quietly turning" | M | Momentum + Sentiment panels |

**Recommendation:** v1.1. Cheap if both inputs exist; nice glance value across the watchlist.

### 2.5 Thesis Tracker (MEDIUM leverage, MEDIUM cost)

| Feature | Why it differentiates | Complexity | Dependencies |
|---|---|---|---|
| Per-ticker free-text "why I'm long" field + a tag-set of "things that would invalidate this" (e.g., "DRAM ASP turns negative", "AFFO payout >95%") | Surface news / metric crossings that match invalidation tags | L | Fundamentals + Sentiment panels; light NLP for tag matching |

**Recommendation:** v2. Strategically attractive but invalidation-matching is fuzzy NLP work. The plain-text thesis field alone (no matching) is a v1.1 S-complexity quick win.

### 2.6 "What Changed Since Last Open" Diff (HIGH leverage, MEDIUM cost — see §6 below)

| Feature | Why it differentiates | Complexity | Dependencies |
|---|---|---|---|
| On app open, show a per-ticker summary: new headlines since last seen, new insider filings, large price moves, new analyst rating changes, new earnings dates approaching. | This is *exactly* the value prop ("one place to look") taken to its endpoint — the user doesn't even have to look at each panel to know if anything new happened. | M | A `last_seen_at` cache per (ticker, source) — see §6 |

**Recommendation:** v1.1, possibly v1 if it can be implemented cheaply. This is the differentiator most aligned with "replaces bouncing between sites."

### 2.7 Alerts (LOW leverage given on-demand model)

| Feature | Why | Complexity |
|---|---|---|
| Threshold alerts (RSI < 30, AFFO payout > 90%, etc.) shown on next app open | Cannot push notifications without a background worker; reduce to "warnings shown when you open the ticker" | S |

**Recommendation:** v1.1. Trivial to layer on top of momentum/fundamentals panels as a "warnings" row. Do not build push notifications.

### 2.8 Earnings Calendar Awareness (LOW cost, HIGH usefulness)

| Feature | Why | Complexity |
|---|---|---|
| Banner showing "Next earnings: 2026-06-25 (T+31 days)" + last 4 quarters' EPS surprise % | Earnings dates are universally relevant; surprise history is free signal | S |

**Recommendation:** v1. Trivial to add (yfinance has both), high relevance, surfaces context for any other panel.

---

## 3. Anti-Features (Explicit Out of Scope)

Each entry includes the rationale so this section serves as defense against scope creep.

| Anti-Feature | Why NOT | What to Do Instead |
|---|---|---|
| **Trade execution / brokerage integration** | This is an informational tool. Order placement adds auth, regulatory, and security surface that is wildly out of proportion to personal use. | Show the data; the user trades in their broker's app. |
| **Real-time intraday tick data / streaming** | Requires WebSocket infra, costs at scale, and contradicts the "on-demand, no background workers" constraint. | Daily OHLC + a manual "refresh" button is sufficient for the multi-horizon thesis use case. |
| **Twitter/X sentiment** | X API in 2026 is paid-only above trivial limits and rate-restricted. Re-evaluate when a stable free path exists. | Reddit covers retail sentiment adequately; news covers institutional narrative. |
| **Multi-user / hosted version / auth** | Personal use only. Auth + deployment + multi-tenancy is months of work for zero value to a user of one. | Stays as a local Tauri app. Sharing = screenshot. |
| **Paid data vendors (Bloomberg, Refinitiv, Polygon Pro, FactSet, S&P Capital IQ)** | Cost. Personal-use budget. Vendor lock-in. | Free APIs (yfinance, SEC EDGAR, Reddit JSON, FMP free tier, NewsAPI free tier) + Playwright fallback for what's not API-accessible. |
| **Full charting at TradingView's level** (drawing tools, dozens of indicator overlays, custom Pine scripts) | TradingView is best of breed and free. Cannot beat them. | Lightweight chart with 50/200 SMA + RSI/MACD sub-panels only. For deeper charting, the user opens TradingView in a browser tab — that's fine. |
| **Heavy historical timeseries DB** (years of tick data, alt-data warehousing) | Storage cost and complexity. Providers serve history on demand. | Small local cache (SQLite or parquet) only for ML training windows and the "last seen" diff state. |
| **Portfolio P&L / cost-basis tracking** | Requires importing positions, lots, cost basis, dividends received, tax-lot accounting. That is a brokerage tool's job. | Stays as a research dashboard. |
| **Options chain / Greeks / unusual options flow** | Different problem domain; data is expensive at quality; user did not request. | If interest emerges later, propose as v2 dedicated panel. |
| **Backtesting engine for arbitrary strategies** | Adjacent product. The forecast back-tracking feature (§2.3) is the appropriate scoped version. | Use a real backtester (vectorbt, backtrader) standalone if needed. |
| **Auto-rebalancing recommendations / robo-advisor logic** | Crosses into giving advice; regulatory minefield even for personal use; not the goal. | Show data; user decides. |
| **General-purpose news reader** | Out of scope creep; not a Feedly replacement. | News panel only surfaces ticker-tagged headlines. |
| **Auto-refresh / background polling** | Explicitly violates the "no background workers" constraint. | Manual "refresh" buttons + on-open diff (§2.6). |
| **Generic AI chatbot / "ask anything about this stock"** | High build cost, low marginal value over the structured panels, and high hallucination risk on financial data. | If LLM is used, scope it tightly — e.g., summarize the parsed 10-Q sections, not free-form Q&A. |

---

## 4. Complexity Matrix (Sortable Roll-up)

| Panel / Feature | Complexity | v1 / v1.1 / v2 |
|---|---|---|
| Watchlist + ticker add/remove + per-ticker view | S | v1 |
| Sector tagging on ticker | S | v1 |
| Manual refresh per panel | S | v1 |
| Common fundamentals (revenue, margins, EPS, FCF, cash, debt, ratios, valuation) | S | v1 |
| Trend sparklines (8 quarters) | M | v1 |
| REIT extension (FFO, AFFO, payout, leverage, interest coverage) | S–M | v1 |
| REIT NOI & occupancy (requires supplemental parse) | M | v1.1 |
| Semiconductor helpers (inventory days, CapEx) | S | v1 |
| Earnings calendar + surprise history | S | v1 |
| Forecasts — analyst consensus | S | v1 |
| Forecasts — Prophet/ARIMA baseline | M | v1 |
| Forecasts — ML model | XL | v1.1 |
| Forecast horizon selector + side-by-side chart | M | v1 |
| Forecast back-tracking (model vs actual) | M | v1.1 |
| News sentiment — ingestion + per-headline scoring | S–M | v1 |
| News sentiment — 30-day trend sparkline | M | v1 |
| Reddit sentiment — mention volume + tone | M | v1 |
| Reddit sentiment — per-subreddit breakdown + hot posts | S | v1 |
| Technical signals (RSI, MACD, MA cross, volume surge, 52w distance) | S each, M combined | v1 |
| Combined price chart with SMA + RSI/MACD sub-panels | M | v1 |
| Insider transactions table (code P/S filter) | M | v1 |
| Insider cluster signals (3+ buyers/10d, net flow, CEO/CFO highlights) | M | v1 |
| Cross-ticker comparison view | M | v1.1 |
| Combined sentiment × momentum quadrant | M | v1.1 |
| Thesis tracker (free-text only) | S | v1.1 |
| Thesis tracker with invalidation matching | L | v2 |
| "What changed since last open" diff | M | v1.1 (or v1 if §6 is resolved early) |
| Threshold warnings on next open | S | v1.1 |

**v1 sum:** ~14 features, mix of S/M with one absent XL (ML forecast deferred). Achievable for a personal-use scope in a reasonable number of phases.

---

## 5. Inter-Feature Dependencies

The dependency graph determines phase order in the roadmap.

```
[Foundation]
  Tauri shell + Angular UI + Python sidecar wiring
        |
        v
  Ticker list / watchlist persistence
        |
        v
  Sector tagging
        |
        v
  Per-ticker dashboard route
        |
        v
  Data-fetch abstraction layer (yfinance + EDGAR + Reddit JSON + News)
        |
        +-------------------------+----------------------+-------------------+
        v                         v                      v                   v
  [Fundamentals panel]   [Forecasts panel]      [Sentiment panel]    [Insider panel]
   (Common metrics)       (Analyst consensus)    (News + Reddit)      (Form 4 table)
        |                         |                      |                   |
        v                         v                      v                   v
  REIT extension            Prophet baseline      Per-headline scoring   Cluster signals
  Semi helpers              Horizon selector      30-day trend           Net flow chart
  Trend sparklines          Side-by-side chart    Hot posts
  Earnings calendar
        |                         |                      |                   |
        +-------------------------+----------------------+-------------------+
                                     |
                                     v
                          [Cross-cutting layer (v1.1)]
                          - Cross-ticker comparison
                          - Sentiment × Momentum quadrant
                          - "What changed since last open" diff
                          - Threshold warnings
                          - Forecast back-tracking
                          - Thesis tracker (text)
                                     |
                                     v
                          [ML + advanced (v2)]
                          - ML forecast model
                          - Thesis invalidation matching
                          - REIT NOI / occupancy parser
```

**Hard dependencies (cannot reorder):**
1. Watchlist persistence → everything else (no ticker = no view)
2. Sector tagging → REIT extension (sector decides which template renders)
3. Data-fetch abstraction → all panels (centralized rate limiting + caching)
4. Per-panel data ingestion → that panel's UI
5. Both Sentiment + Momentum panels → quadrant visualization
6. Forecast generation → forecast back-tracking (need stored forecasts to back-test)
7. Historical price cache → Prophet / ARIMA / ML (training input)

**Soft dependencies (preferred order):**
- Build common fundamentals before REIT extension (template emerges from common)
- Build single-ticker views before cross-ticker comparison (don't generalize prematurely)
- Build per-headline sentiment scoring before aggregations and trends

---

## 6. The "Diff Since Last Open" Tension

**Stated constraint:** on-demand fetch, no background workers, no heavy historical DB.
**Stated desire:** surface "what's new since I last looked."

These are not actually in conflict. Resolution:

### A minimal cache is acceptable and necessary

The constraint in `PROJECT.md` reads: *"no server-side DB; small on-disk cache (e.g., SQLite or parquet files) is acceptable where ML training data or per-ticker history is needed."* The "what changed since last open" feature is a legitimate case for that small cache.

### Recommended data model (SQLite, <10 tables, <1 MB for a watchlist of 20 tickers over a year)

| Table | Purpose | Approx row count for 20 tickers / 1 year |
|---|---|---|
| `tickers` | watchlist | 20 |
| `last_seen` | (ticker, source) → last_fetched_at, last_top_item_id | 20 × ~6 sources = 120 |
| `headlines_seen` | headline IDs already shown to the user (dedup the diff) | ~50k |
| `insider_filings_seen` | Form 4 accession numbers already shown | ~1k |
| `forecasts_log` | (ticker, model, horizon, generated_at, point, ci_lo, ci_hi) — enables back-testing | ~hundreds |
| `prices_cache` | daily OHLC cache (only what Prophet/ML needs) | ~5k per ticker × 20 ≈ 100k |

This is small, file-based, and within the constraint as written.

### How the diff renders without persistent intraday tracking

On app open:
1. Fetch each source on demand (the existing model).
2. For each item returned (headline, Form 4, analyst-rating change, large-move flag), compare against `headlines_seen` / `insider_filings_seen`.
3. **Items not previously seen are marked `NEW` with a badge**, displayed in a "What's new since last open" digest at the top of the ticker view.
4. After display, mark them seen.

No background worker is required because the diff is computed at fetch time, not pre-computed. The cache is not "tracking changes" — it's just remembering what was shown to the user, so the next fetch can highlight novelty.

### Trade-offs to call out in REQUIREMENTS

- **Cache file location:** Tauri's `app_data_dir()` (cross-platform). Document this so the user knows where the state lives.
- **Cache wipe:** offer a "reset seen state" action (so the user can re-trigger the firehose if desired).
- **Bound the seen tables:** prune `headlines_seen` rows older than 90 days. Headlines from 6 months ago are not interesting to surface as "new."
- **Forecasts log is never pruned** — the whole point is long-horizon back-testing.

### Verdict for REQUIREMENTS

**Recommend:** "small on-disk SQLite cache for `last_seen` state, `headlines_seen`, `insider_filings_seen`, `forecasts_log`, and `prices_cache`" is in scope and consistent with the existing constraint. The "no DB" boundary referred to *server-side* DB (multi-user, hosted). A local SQLite file is not that.

**Anti-feature reinforced:** no automatic polling, no background sync, no notification daemon. The diff is computed at user-initiated open/refresh time only.

---

## 7. REIT vs Common-Stock Applicability Cheat Sheet

For quick reference when writing REQUIREMENTS:

| Panel | Common Stock (MU) | REIT (IRM) | Difference |
|---|---|---|---|
| Watchlist | Same | Same | None |
| Common fundamentals | Full template | Full template | None |
| Sector extension | Optional: inventory days, CapEx, guidance | **Required**: FFO, AFFO, P/AFFO, AFFO payout, NOI, occupancy | REIT extension is the headline differentiator |
| Forecasts | Same | Same | None |
| News sentiment | Same | Same | None |
| Reddit sentiment | Same | Same (note: REITs have lower WSB mention volume; expect sparser data) | None functional |
| Momentum | Same | Same | None |
| Insider trades | Same | Same | None |
| Earnings calendar | Same | Same | None |

**Implication:** the only sector-aware logic lives in the Fundamentals panel. Everything else is a uniform template across tickers. This is a clean architectural separation.

---

## 8. MVP Recommendation (Crisp)

**Ship in v1 (the minimum that delivers "one place to look"):**

1. Watchlist with MU + IRM, add/remove, sector tagging
2. Per-ticker dashboard with five panels: Fundamentals (common + REIT extension), Forecasts (analyst + Prophet, 3 horizons), News sentiment, Reddit sentiment, Momentum signals, Insider trades + cluster flags, Earnings calendar banner
3. Manual refresh per panel + global "refresh all"
4. Small SQLite cache for prices, forecasts log, and seen-items state

**Defer to v1.1:**
- ML forecast model (XL — needs feature pipeline + labels)
- "What changed since last open" diff (M — needs §6 cache to mature; can be added incrementally)
- Cross-ticker comparison view
- Sentiment × Momentum quadrant
- Thesis tracker (free text only)
- Forecast back-tracking UI
- REIT NOI / occupancy parsing
- Threshold warnings

**Defer to v2:**
- Thesis tracker with NLP invalidation matching
- Any further sector templates beyond REIT + Semi

**Hard exclusions (anti-features):** trade execution, real-time intraday, Twitter, multi-user, paid vendors, full charting, portfolio P&L, options flow, backtesting engine, auto-rebalancing, general news reader, auto-refresh, generic AI chatbot.

---

## Sources

- [How to Analyze Semiconductor Stocks — Investing.com](https://www.investing.com/academy/trading/how-to-analyze-semiconductor-stocks/)
- [Micron Technology (MU) — StockStory Research Report](https://stockstory.org/us/stocks/nasdaq/mu)
- [Iron Mountain (IRM) — REIT Notes](https://www.reitnotes.com/reit/Iron-Mountain-Inc/symbol/IRM)
- [Iron Mountain Q1 2026 Supplemental Reporting Package (SEC 8-K)](https://www.sec.gov/Archives/edgar/data/0001020569/000102056926000036/q12026srp_final.htm)
- [Top Data Center REITs Analysis — TheStreet Pro](https://pro.thestreet.com/investing/3-data-center-reits-with-attractive-dividends-and-growth-16142840)
- [Insider Trading Signals: SEC Form 4 Guide — MarketTriage](https://markettriage.com/insider-trading-signals)
- [SEC Form 4 Insider Trading Alerts — PageCrawl.io](https://pagecrawl.io/blog/sec-form-4-insider-trading-alerts)
- [SEC Form 4 Explained — HedgeTrace](https://hedgetrace.com/learn/sec-form-4-explained)
- [InsiderDashboard — Insider Trading Filings Dashboard](https://www.insiderdashboard.com/)
- [WSB Sentiment Dashboard](https://www.wsbsentiment.com/)
- [SwaggyStocks WSB Social Sentiment](https://swaggystocks.com/dashboard/wallstreetbets/ticker-sentiment)
- [AltIndex WallStreetBets Mentions Methodology](https://altindex.com/wallstreetbets)
- [Predicting GME Stock Price Movement Using Sentiment from Reddit (FinNLP 2021)](https://aclanthology.org/2021.finnlp-1.4.pdf)
- [Performance of Prophet in Stock-Price Forecasting: Comparison with ARIMA and MLP — IEEE](https://ieeexplore.ieee.org/document/10756299/)
- [Comparative Analysis of ARIMA, SARIMA and Prophet in Forecasting (ResearchGate)](https://www.researchgate.net/publication/385157901_Comparative_Analysis_of_ARIMA_SARIMA_and_Prophet_Model_in_Forecasting)
- [Comparing Prophet and Deep Learning to ARIMA in Forecasting (arXiv)](https://arxiv.org/pdf/2107.12770)
- [Building a Stock Forecasting Dashboard with Prophet & ARIMA — Medium](https://medium.com/learning-data/building-a-stock-forecasting-dashboard-with-prophet-arima-modeling-68101307a0bf)
- [RSI and MACD Combined Strategy — Investing.com Academy](https://www.investing.com/academy/analysis/how-to-use-rsi-and-macd-together/)
- [MACD and RSI Strategy Backtest Results — QuantifiedStrategies](https://www.quantifiedstrategies.com/macd-and-rsi-strategy/)
- [Oscillators: MACD, RSI, Stochastics — CME Group](https://www.cmegroup.com/education/courses/technical-analysis/oscillators-macd-rsi-stochastics)
