# Requirements: Stock Finder

**Defined:** 2026-05-25
**Core Value:** One place to look that combines high-signal data (fundamentals + insider + earnings cycle + macro context) with explicitly-demoted low-signal context (sentiment + forecast) for each tracked ticker — including disconfirmation of my own thesis — so I can form better-informed personal investment decisions without manual aggregation.

**Re-prioritized per `.planning/PROJECT-ANALYSIS.md`:** Panels are ranked by signal quality; UI prominence follows that ranking. Forecast panel is visually demoted with backtest-accuracy headline. Disconfirmation, macro context, earnings-cycle expansion, and earnings-call LLM summary added as primary/medium-signal panels.

**Decisions locked at requirements time:**

- Platforms: macOS arm64 (primary) + Windows (Phase E); Linux deferred
- Reddit sentiment: **deferred to v2** — v1 sentiment is news-only
- REIT NOI / occupancy / same-store: **Phase D** (basic REIT extension in Phase B)
- Default forecast model alongside analyst consensus + naive baseline: **Prophet** (visually demoted)
- New panels (Disconfirmation, Macro context, Earnings-call summary, Expanded earnings cycle): **Phase D**
- LLM summary uses Claude API with user-provided key; absence triggers graceful degradation

## v1 Requirements

### Infrastructure / Foundation (Phase A)

- [ ] **INFRA-01**: App launches as a Tauri 2.x desktop binary on macOS arm64
- [ ] **INFRA-02**: Python sidecar (PyInstaller one-file) is bundled with the app and spawned per-request via `tauri-plugin-shell`
- [ ] **INFRA-03**: Sidecar IPC uses NDJSON over stdin/stdout with a documented contract (request envelope + per-panel response/error/progress/done events)
- [ ] **INFRA-04**: SQLite cache initialized on first launch via `tauri-plugin-sql` with `tickers`, `refresh_log` tables (no migration framework; idempotent `CREATE TABLE IF NOT EXISTS`)
- [ ] **INFRA-05**: HTTP client in sidecar enforces SEC EDGAR rate limit of ≤8 req/s with descriptive User-Agent header `"StockFinder ryandammrose@onepointtwocapital.com"`
- [ ] **INFRA-06**: Adapter responses carry `(value, as_of_ts, source)` so the UI displays a source/freshness badge per data point
- [ ] **INFRA-07**: Sidecar child processes are killed cleanly on app shutdown (no orphan PIDs)
- [ ] **INFRA-08**: Panel-scoped errors when a sidecar request fails — other panels continue to render
- [ ] **INFRA-09**: User can cancel an in-flight refresh from the UI
- [ ] **INFRA-10**: Logs rotate via `tauri-plugin-log` to a known app data directory
- [ ] **INFRA-11**: PyInstaller bundle launches successfully on a clean macOS user account (not just the dev machine)
- [ ] **INFRA-12**: **Explicit cache TTLs codified in a single config:** fundamentals 24h, insider 6h, news sentiment 1h, prices on-demand (no cache), forecasts_log forever, macro context 6h, earnings-call summary forever (per filing). TTLs drive freshness badges and auto-refresh decisions.
- [ ] **INFRA-13**: Optional Claude API key configuration via settings (used only by earnings-call summary panel); panel degrades gracefully when key absent

### Watchlist / Ticker Management (Phase A)

- [ ] **WATCH-01**: User sees a watchlist of tracked tickers on app launch
- [ ] **WATCH-02**: User can add a ticker by symbol; ticker is validated against yfinance before being persisted
- [ ] **WATCH-03**: User can remove a ticker from the watchlist
- [ ] **WATCH-04**: Watchlist is seeded on first launch with **MU** and **IRM** (as data rows, not hard-coded constants)
- [ ] **WATCH-05**: Each ticker row carries a `sector` tag, auto-detected (SIC 6798 → `REIT`, SIC 3674 → `Semiconductor`); sector tag drives sector-aware rendering and macro panel selection
- [ ] **WATCH-06**: User can navigate from a watchlist row to a `TickerDetail` page with tabbed panels in priority order: Fundamentals → Insider → Earnings → Disconfirmation → Macro → Analyst → Technicals → Summary (LLM) → Sentiment → Forecast (in that order, top-to-bottom or left-to-right)
- [ ] **WATCH-07**: Manual per-panel refresh and "refresh all" button
- [ ] **WATCH-08**: Last-refreshed timestamp visible per panel; TTL countdown indicator

### Fundamentals — Common-Stock (Phase A, expanded Phase B)

- [ ] **FUND-01**: Fundamentals panel shows revenue (TTM, current quarter, YoY %), gross / operating margin, net income
- [ ] **FUND-02**: GAAP EPS and non-GAAP EPS where available
- [ ] **FUND-03**: Free cash flow (TTM)
- [ ] **FUND-04**: Cash & equivalents, total debt, net-debt / EBITDA, current ratio
- [ ] **FUND-05**: Shares outstanding (basic and diluted)
- [ ] **FUND-06**: Dividend per share, yield, payout ratio
- [ ] **FUND-07**: Valuation ratios: trailing P/E, forward P/E, P/S, EV/EBITDA (with REIT suppression — see REIT-04)
- [ ] **FUND-08**: 8-quarter trend sparklines per key metric
- [ ] **FUND-09**: Data source cascade: yfinance → yahooquery → FMP free tier → cached-with-stale-flag; current source visible to user
- [ ] **FUND-10**: Forcing yfinance to error produces visible "fell back to yahooquery" status badge; panel still renders

### Fundamentals — REIT Extension (Phase B, required for IRM)

- [ ] **REIT-01**: When `sector === 'REIT'`, fundamentals panel renders an additional REIT Metrics block
- [ ] **REIT-02**: FFO (TTM and current quarter), AFFO (TTM and current quarter), FFO/share, AFFO/share
- [ ] **REIT-03**: P/FFO and P/AFFO in the primary valuation slot (replacing P/E for REITs)
- [ ] **REIT-04**: P/E is suppressed or visibly de-emphasized for REITs with tooltip "use FFO/AFFO for REITs"
- [ ] **REIT-05**: AFFO payout ratio with traffic-light coloring (green <80%, yellow 80–95%, red >95%)
- [ ] **REIT-06**: Net-debt / EBITDA (REIT-adjusted) and interest coverage
- [ ] **REIT-07**: When IRM tags FFO directly in XBRL, "Reported FFO" is shown; a "Computed FFO" (NetIncome + RE D&A − gains on RE sales + impairments) shown alongside for transparency
- [ ] **REIT-08**: For non-REIT tickers (e.g., MU), the REIT Metrics block is not rendered

### Fundamentals — Semiconductor Helpers (Phase B)

- [ ] **SEMI-01**: For semiconductor-sector tickers (MU and similar), fundamentals panel shows inventory and inventory days
- [ ] **SEMI-02**: CapEx (TTM) and CapEx / Revenue ratio

### Insider Trades (Phase B — HIGH PRIORITY per analysis: "probably your best panel")

- [ ] **INS-01**: Transactions table: filing date, transaction date, insider name, role, transaction code (with tooltip legend), shares, price, $ value, post-transaction ownership
- [ ] **INS-02**: Default filter: code P (purchase) and S (sale) only; toggle exposes M (option exercise), A (grant), G (gift), etc.
- [ ] **INS-03**: M (option exercise) + S (sale) same-day pairs tagged `exercise-sale` and excluded from sale aggregates
- [ ] **INS-04**: 10b5-1 pre-planned trades (flagged on Form 4) excluded from cluster-signal aggregates by default
- [ ] **INS-05**: Cluster-buy detection: ≥3 unique insiders / ≤10 days / each transaction ≥$25k / code P only — surfaced as a prominent badge
- [ ] **INS-06**: Cluster-sell signal uses a tighter threshold and is labeled "investigate"; UI elevates buys, de-emphasizes sells (asymmetric weighting)
- [ ] **INS-07**: Net flow chart: Purchase $ minus Sale $ aggregated by month over last 12 months
- [ ] **INS-08**: Transactions table sorted by `transaction_date DESC` (not filing date); caption notes typical 2-business-day Form 4 filing lag
- [ ] **INS-09**: Data sourced from edgartools (Form 4 primary); openinsider scrape used to cross-check pre-computed clusters
- [ ] **INS-10**: Insider panel sits in the visual prominence tier of Fundamentals (not below momentum/sentiment)

### Earnings Cycle (Phase B — expanded per analysis)

- [ ] **CAL-01**: TickerDetail page shows a banner: "Next earnings: YYYY-MM-DD (T+N days)" from yfinance
- [ ] **CAL-02**: Last-4-quarters EPS surprise % (actual vs consensus)
- [ ] **CAL-03**: Last-4-quarters revenue surprise % (actual vs consensus)
- [ ] **CAL-04**: Guidance-vs-actual track record table: prior-quarter guidance (when company gave it) vs subsequent actual, with hit/miss flag and magnitude
- [ ] **CAL-05**: Beat/miss visual: traffic-light per quarter on (revenue, EPS, guidance)
- [ ] **CAL-06**: Earnings-cycle panel sits in the primary tier (above momentum/sentiment)

### Momentum / Technicals (Phase B — medium-signal, "context not signal" framing)

- [ ] **TECH-01**: RSI(14) with overbought (>70) and oversold (<30) bands
- [ ] **TECH-02**: MACD(12, 26, 9) with current crossover flag
- [ ] **TECH-03**: 50-day and 200-day SMA state with golden / death cross detection
- [ ] **TECH-04**: Volume surge flag when current volume >2× 20-day average
- [ ] **TECH-05**: Distance from 52-week high and 52-week low
- [ ] **TECH-06**: Single combined chart (candlestick + SMA overlays + RSI sub-panel + MACD sub-panel) via ngx-echarts
- [ ] **TECH-07**: OHLCV history cached as Parquet in app cache directory; cache freshness shown
- [ ] **TECH-08**: Panel header carries "Context — not a trading signal" subtitle

### Disconfirmation / Bear-Signal Panel (Phase D — NEW from analysis, primary-tier panel)

- [ ] **DISC-01**: Disconfirmation panel surfaces bear signals against the implicit/explicit long thesis; framed as "what could go wrong"
- [ ] **DISC-02**: Recent analyst downgrades (last 90 days) with date, firm, prior/new rating, prior/new price target
- [ ] **DISC-03**: Margin compression flags: gross margin trailing 4Q trend (down ≥150 bps YoY = flag), operating margin same
- [ ] **DISC-04**: Insider SELLING cluster flag (uses INS-* logic with asymmetric threshold; labeled "investigate" per INS-06)
- [ ] **DISC-05**: For REIT sector (IRM): debt maturity wall visualization — schedule of debt principal due over next 5 years with refi-risk flag if any single year >25% of total debt
- [ ] **DISC-06**: For Semiconductor sector (MU): inventory days rising flag (DIO trending up ≥10% YoY = flag; cyclical risk indicator)
- [ ] **DISC-07**: Bear-signal count badge on the watchlist row (so user sees "MU: 2 bear signals" without drilling in)
- [ ] **DISC-08**: Each bear signal carries a "why this matters" tooltip with the academic / market rationale
- [ ] **DISC-09**: Panel positioned in primary visual tier alongside Fundamentals and Insider

### Macro / Sector Context (Phase D — NEW from analysis, medium-signal)

- [ ] **MACRO-01**: Macro panel shows sector-appropriate context based on ticker `sector` tag
- [ ] **MACRO-02**: For Semiconductor sector (MU): SOX index (^SOXX) chart + delta % over 30/90 days
- [ ] **MACRO-03**: For Semiconductor sector (MU): DRAM-price proxy — TrendForce blog headlines scraped + summarized (best-available free proxy; flagged as "directional indicator only")
- [ ] **MACRO-04**: For Semiconductor sector (MU): hyperscaler capex commentary — news scrape filtered for AWS / Azure / GCP capex mentions in last 30 days
- [ ] **MACRO-05**: For REIT sector (IRM): 10Y Treasury (DGS10) from FRED API + delta over 30/90 days
- [ ] **MACRO-06**: For REIT sector (IRM): data-center context — news scrape for "data center" + "absorption" / "vacancy" in last 30 days (best-available free proxy)
- [ ] **MACRO-07**: Macro indicators are correlation-tagged ("typically inverse to price," "typically directional," etc.) with disclaimer that correlations break
- [ ] **MACRO-08**: Macro adapters use the standard cache TTL (6h per INFRA-12)

### Earnings-Call LLM Summary (Phase D — NEW from analysis)

- [ ] **SUM-01**: Summary panel fetches the latest 10-Q and 10-K via EDGAR per ticker
- [ ] **SUM-02**: Summary panel fetches the latest earnings-call transcript (source: yfinance / earnings-call sites; fall back to "transcript not available" if absent)
- [ ] **SUM-03**: When Claude API key is configured (per INFRA-13), filings + transcript are summarized to 5 bullets each: (1) headline financial result, (2) management tone shift, (3) guidance change, (4) one risk surfaced, (5) one opportunity
- [ ] **SUM-04**: Each bullet cites the source paragraph (clickable link / scrollable excerpt)
- [ ] **SUM-05**: Summary is cached per filing forever (per INFRA-12) since filings don't change after publication
- [ ] **SUM-06**: When no Claude API key configured, panel shows raw filing links + transcript link with no summary; status: "Configure API key in Settings to enable summary"
- [ ] **SUM-07**: User can manually re-run summarization on a filing (in case of prompt iteration)
- [ ] **SUM-08**: Summarization uses Claude Haiku (lower cost, sufficient for summarization) by default; setting allows Sonnet/Opus override
- [ ] **SUM-09**: Per-summary token cost is logged so user can see cumulative API spend

### Analyst Consensus (Phase C — medium-signal)

- [ ] **ANL-01**: Analyst-consensus block shows: mean target, high, low, # analysts, rating distribution (Strong Buy / Buy / Hold / Sell / Strong Sell counts)
- [ ] **ANL-02**: Analyst targets are visually labeled as "targets, not forecasts" with own section header
- [ ] **ANL-03**: Analyst section is separate from the Forecast panel to prevent false-equivalence with model output

### Sentiment — News (Phase C — DEMOTED, low-signal; Reddit deferred to v2)

- [ ] **SENT-01**: News headlines ingested from ≥2 of: yfinance news, RSS feeds, GDELT DOC 2.0
- [ ] **SENT-02**: Every headline scored with FinVADER (fast, all headlines)
- [ ] **SENT-03**: Top-N headlines (by recency × relevance) re-scored with FinBERT for higher-fidelity tone
- [ ] **SENT-04**: Aggregated tone for last 7 days and last 30 days
- [ ] **SENT-05**: Headlines listed with score, source, age, clickable link
- [ ] **SENT-06**: 30-day sentiment trend sparkline
- [ ] **SENT-07**: Per-source breakdown; any single source's contribution to aggregated tone capped at ~30%
- [ ] **SENT-08**: FinBERT model lazy-downloaded to app cache on first sentiment-panel use, with progress dialog (not bundled)
- [ ] **SENT-09**: FinBERT latency measured on first use; fall back to FinVADER-only if exceeds 500ms per top-N rescore
- [ ] **SENT-10**: Panel header carries "Lower-signal — headlines often coincident with price" subtitle (per analysis demotion)

### Forecasts — Statistical + Analyst (Phase C — DEMOTED per analysis)

- [ ] **FCST-01**: Forecast panel **headline metric is backtest accuracy**, not the next forecast point: "Prophet directional hit rate on this ticker last 24 months: NN%" + "in-CI rate: NN%" — displayed at panel top before any forecast chart
- [ ] **FCST-02**: Forecast panel sits visually below primary panels with smaller real estate; panel subtitle: "Reference only — not decision input"
- [ ] **FCST-03**: Forecast panel shows a naive (last-value persistence) baseline forecast alongside model forecasts
- [ ] **FCST-04**: Forecast panel shows Prophet-based forecast as the default visible third line (alongside analyst consensus and naive baseline)
- [ ] **FCST-05**: A statsforecast (AutoARIMA / ETS / Theta) forecast computed in the backend; selectable via dropdown (not visible by default)
- [ ] **FCST-06**: Forecast horizon user-selectable: 1–3 months (daily), 3–12 months (weekly), 1–3 years (monthly)
- [ ] **FCST-07**: All forecasts display point estimate and 80% confidence interval; CI visibly widens for longer horizons
- [ ] **FCST-08**: `ForecastCard` displays: model name, training window, last-updated timestamp, point estimate, 80% CI, naive-baseline comparison, horizon, "Not financial advice" footer
- [ ] **FCST-09**: `forecasts_log` SQLite table populated on every forecast generation (point, ci_lo, ci_hi, generated_at, target_date, model_name, horizon, symbol) — no forecast is shown without being logged
- [ ] **FCST-10**: Model that fails to beat naive baseline over rolling 90-day windows is flagged with "below naive" badge
- [ ] **FCST-11**: All model evaluation uses walk-forward CV (no `train_test_split`, no shuffled CV)
- [ ] **FCST-12**: 1–3 year forecasts carry long-horizon disclaimer noting reduced reliability
- [ ] **FCST-13**: Unified chart renders: historical price (grey) + model forecast lines + CI bands; summary table shows model / point / CI / implied % vs current

### Cross-Cutting Differentiators (Phase D)

- [ ] **DIFF-01**: "What Changed Since Last Open" digest banner on TickerDetail: new headlines, new Form 4 filings, new analyst target changes, threshold breaches
- [ ] **DIFF-02**: `headlines_seen` and `insider_filings_seen` SQLite tables persist last-seen state per ticker; headlines pruned at 90 days
- [ ] **DIFF-03**: Forecast back-tracking UI: per historical forecast, "model said $X N days ago for target Y; actual $Z; in-CI: yes/no; rolling hit rate W%" — read-side over `forecasts_log`
- [ ] **DIFF-04**: Cross-ticker comparison view: fundamentals side-by-side for 2–4 tracked tickers (sector-aware: REIT metrics shown only for REIT tickers in their column)
- [ ] **DIFF-05**: Sentiment × momentum quadrant: tracked tickers plotted by (Δ sentiment, momentum score); sentiment axis is delta-not-level
- [ ] **DIFF-06**: Thesis tracker: user records free-text thesis notes per ticker (no NLP, no auto-invalidation in v1)
- [ ] **DIFF-07**: User-configured threshold warnings (e.g., "alert when MU RSI > 70") shown in the digest banner
- [ ] **DIFF-08**: IRM-specific NOI, same-store NOI growth, occupancy / utilization added to REIT Metrics block — sourced from IRM's 8-K supplemental package

### ML Forecast (Phase E)

- [ ] **ML-01**: A fourth forecast model (darts NHiTS or equivalent neural baseline) computed and rendered as a fourth selectable line in ForecastChart
- [ ] **ML-02**: Walk-forward CV; backtest results stored and shown to user (same headline treatment as FCST-01)
- [ ] **ML-03**: ML model carries same `ForecastCard` contract as statistical models (CI, naive comparison, "below naive" badge when applicable)
- [ ] **ML-04**: ML training data cached as Parquet frames per ticker; feature engineering documented in code
- [ ] **ML-05**: ML model that cannot beat naive on 90-day rolling validation rendered with "below naive" badge + tooltip explaining the result

### Packaging Hardening (Phase E)

- [ ] **PKG-01**: PyInstaller CI matrix builds for macOS arm64 + Windows x64
- [ ] **PKG-02**: macOS build code-signed (Developer ID) and notarized with hardened runtime + minimal-but-sufficient entitlements
- [ ] **PKG-03**: Windows build code-signed
- [ ] **PKG-04**: Post-build smoke test invokes every supported sidecar `op` on a clean VM (mac + win)
- [ ] **PKG-05**: Installer size budget enforced: <150 MB (FinBERT lazy-downloaded, not bundled)
- [ ] **PKG-06**: FinBERT lazy-download verified working on a clean user account
- [ ] **PKG-07**: PyInstaller `hooks/` directory captures known hidden imports (sklearn, finvader data, prophet data)

## v2 Requirements

Deferred to future releases.

### Reddit Sentiment (deferred from v1 after analysis re-evaluation)

- **REDDIT-01**: Reddit ingestion via PRAW (OAuth) of WSB / investing / stocks / SecurityAnalysis subreddits per ticker
- **REDDIT-02**: First-run wizard for per-user Reddit dev-app credential registration
- **REDDIT-03**: Per-post FinVADER scoring; aggregated tone weighted by upvotes
- **REDDIT-04**: Mention count 24h / 7d / 30d with delta
- **REDDIT-05**: Top-5 hot posts per ticker with link-through; per-subreddit breakdown
- **REDDIT-06**: Bot/promo filtering on low-karma / young accounts
- **REDDIT-07**: Reddit panel caption: "buzz follows price; treat as confirmation not prediction"

### Linux Support

- **LINUX-01**: PyInstaller Linux build added to CI matrix; AppImage or DEB distribution

### Advanced Differentiators

- **THESIS-NLP-01**: Thesis tracker with NLP-based invalidation matching against incoming news
- **BODY-FETCH-01**: News-body-fetch on high-signal events for higher-fidelity sentiment (vs headline-only)
- **OPTIONS-01**: Options chain + Greeks + unusual options flow (revisit if free quality source emerges)
- **SUM-LOCAL-01**: Local LLM option for earnings-call summary (Ollama / llama.cpp) to remove Claude API dependency

## Out of Scope

Explicitly excluded.

| Feature | Reason |
|---------|--------|
| Trade execution / brokerage integration | Informational tool only |
| Real-time intraday tick data / streaming | Violates "on-demand, no background workers" |
| Twitter/X sentiment | X API in 2026 is paid-only above trivial limits |
| Multi-user / hosted version / auth | Personal use only |
| Paid data vendors (Bloomberg, Refinitiv, Polygon Pro, FactSet) | Cost + lock-in |
| Full charting at TradingView's level | TradingView is best-of-breed and free; lightweight charts only |
| Heavy historical timeseries DB | Providers serve history on demand; small local Parquet cache only |
| Portfolio P&L / cost-basis tracking | Brokerage tool's job |
| Backtesting engine for arbitrary strategies | Use vectorbt/backtrader standalone; forecast back-tracking (DIFF-03) is the scoped version |
| Auto-rebalancing / robo-advisor logic | Regulatory minefield |
| General-purpose news reader | Feedly's job; only ticker-tagged headlines surface |
| Auto-refresh / background polling | Violates "no background workers"; manual refresh + on-open diff only |
| Generic AI chatbot ("ask anything") | LLM is scoped to earnings-call summary (SUM-*); no free-form chat in v1 (hallucination risk on financials) |
| NgRx / NGXS / Akita | Over-engineering for single-user app |
| Schema migration framework | `CREATE TABLE IF NOT EXISTS`; bump cache dir for breaking changes |
| Plugin / extension architecture | One user, no third parties |
| Observability stack (Prometheus, OTel, Datadog) | `tauri-plugin-log` with rotation suffices |
| Multi-window / multi-monitor sophistication | Single window |
| Hard-coded MU / IRM ticker constants | Tickers are data (SQLite), not constants |
| Linux build for v1 | Personal use is macOS+Windows |
| Reddit sentiment in v1 | Deferred to v2 per analysis (low signal) |
| Forecast as decision-grade output | Panel framed as reference / context; FCST-01 backtest headline + FCST-02 demotion ensure user is not misled |
| Trade alerts / notifications outside the app | "What changed" digest at open only; no email/push/SMS |

## Traceability

Populated during roadmap creation 2026-05-25.

**Phase legend:**
- Phase 1 = MU Fundamentals End-to-End
- Phase 2 = Sector-Aware Primary Panels (MU + IRM)
- Phase 3 = Analyst + Sentiment + Statistical Forecast
- Phase 4a = Disconfirmation + Macro + Earnings-Call Summary
- Phase 4b = Cross-Cutting Differentiators
- Phase 5 = ML Forecast + Packaging Hardening

| Requirement | Phase | Status |
|-------------|-------|--------|
| INFRA-01 | Phase 1 | Pending |
| INFRA-02 | Phase 1 | Pending |
| INFRA-03 | Phase 1 | Pending |
| INFRA-04 | Phase 1 | Pending |
| INFRA-05 | Phase 1 | Pending |
| INFRA-06 | Phase 1 | Pending |
| INFRA-07 | Phase 1 | Pending |
| INFRA-08 | Phase 1 | Pending |
| INFRA-09 | Phase 1 | Pending |
| INFRA-10 | Phase 1 | Pending |
| INFRA-11 | Phase 1 | Pending |
| INFRA-12 | Phase 1 | Pending |
| INFRA-13 | Phase 1 | Pending |
| WATCH-01 | Phase 1 | Pending |
| WATCH-02 | Phase 1 | Pending |
| WATCH-03 | Phase 1 | Pending |
| WATCH-04 | Phase 1 | Pending |
| WATCH-05 | Phase 1 | Pending |
| WATCH-06 | Phase 1 | Pending |
| WATCH-07 | Phase 1 | Pending |
| WATCH-08 | Phase 1 | Pending |
| FUND-01 | Phase 1 | Pending |
| FUND-02 | Phase 1 | Pending |
| FUND-03 | Phase 1 | Pending |
| FUND-04 | Phase 1 | Pending |
| FUND-05 | Phase 1 | Pending |
| FUND-06 | Phase 1 | Pending |
| FUND-07 | Phase 1 | Pending |
| FUND-08 | Phase 1 | Pending |
| FUND-09 | Phase 1 | Pending |
| FUND-10 | Phase 1 | Pending |
| REIT-01 | Phase 2 | Pending |
| REIT-02 | Phase 2 | Pending |
| REIT-03 | Phase 2 | Pending |
| REIT-04 | Phase 2 | Pending |
| REIT-05 | Phase 2 | Pending |
| REIT-06 | Phase 2 | Pending |
| REIT-07 | Phase 2 | Pending |
| REIT-08 | Phase 2 | Pending |
| SEMI-01 | Phase 2 | Pending |
| SEMI-02 | Phase 2 | Pending |
| INS-01 | Phase 2 | Pending |
| INS-02 | Phase 2 | Pending |
| INS-03 | Phase 2 | Pending |
| INS-04 | Phase 2 | Pending |
| INS-05 | Phase 2 | Pending |
| INS-06 | Phase 2 | Pending |
| INS-07 | Phase 2 | Pending |
| INS-08 | Phase 2 | Pending |
| INS-09 | Phase 2 | Pending |
| INS-10 | Phase 2 | Pending |
| CAL-01 | Phase 2 | Pending |
| CAL-02 | Phase 2 | Pending |
| CAL-03 | Phase 2 | Pending |
| CAL-04 | Phase 2 | Pending |
| CAL-05 | Phase 2 | Pending |
| CAL-06 | Phase 2 | Pending |
| TECH-01 | Phase 2 | Pending |
| TECH-02 | Phase 2 | Pending |
| TECH-03 | Phase 2 | Pending |
| TECH-04 | Phase 2 | Pending |
| TECH-05 | Phase 2 | Pending |
| TECH-06 | Phase 2 | Pending |
| TECH-07 | Phase 2 | Pending |
| TECH-08 | Phase 2 | Pending |
| ANL-01 | Phase 3 | Pending |
| ANL-02 | Phase 3 | Pending |
| ANL-03 | Phase 3 | Pending |
| SENT-01 | Phase 3 | Pending |
| SENT-02 | Phase 3 | Pending |
| SENT-03 | Phase 3 | Pending |
| SENT-04 | Phase 3 | Pending |
| SENT-05 | Phase 3 | Pending |
| SENT-06 | Phase 3 | Pending |
| SENT-07 | Phase 3 | Pending |
| SENT-08 | Phase 3 | Pending |
| SENT-09 | Phase 3 | Pending |
| SENT-10 | Phase 3 | Pending |
| FCST-01 | Phase 3 | Pending |
| FCST-02 | Phase 3 | Pending |
| FCST-03 | Phase 3 | Pending |
| FCST-04 | Phase 3 | Pending |
| FCST-05 | Phase 3 | Pending |
| FCST-06 | Phase 3 | Pending |
| FCST-07 | Phase 3 | Pending |
| FCST-08 | Phase 3 | Pending |
| FCST-09 | Phase 3 | Pending |
| FCST-10 | Phase 3 | Pending |
| FCST-11 | Phase 3 | Pending |
| FCST-12 | Phase 3 | Pending |
| FCST-13 | Phase 3 | Pending |
| DISC-01 | Phase 4a | Pending |
| DISC-02 | Phase 4a | Pending |
| DISC-03 | Phase 4a | Pending |
| DISC-04 | Phase 4a | Pending |
| DISC-05 | Phase 4a | Pending |
| DISC-06 | Phase 4a | Pending |
| DISC-07 | Phase 4a | Pending |
| DISC-08 | Phase 4a | Pending |
| DISC-09 | Phase 4a | Pending |
| MACRO-01 | Phase 4a | Pending |
| MACRO-02 | Phase 4a | Pending |
| MACRO-03 | Phase 4a | Pending |
| MACRO-04 | Phase 4a | Pending |
| MACRO-05 | Phase 4a | Pending |
| MACRO-06 | Phase 4a | Pending |
| MACRO-07 | Phase 4a | Pending |
| MACRO-08 | Phase 4a | Pending |
| SUM-01 | Phase 4a | Pending |
| SUM-02 | Phase 4a | Pending |
| SUM-03 | Phase 4a | Pending |
| SUM-04 | Phase 4a | Pending |
| SUM-05 | Phase 4a | Pending |
| SUM-06 | Phase 4a | Pending |
| SUM-07 | Phase 4a | Pending |
| SUM-08 | Phase 4a | Pending |
| SUM-09 | Phase 4a | Pending |
| DIFF-01 | Phase 4b | Pending |
| DIFF-02 | Phase 4b | Pending |
| DIFF-03 | Phase 4b | Pending |
| DIFF-04 | Phase 4b | Pending |
| DIFF-05 | Phase 4b | Pending |
| DIFF-06 | Phase 4b | Pending |
| DIFF-07 | Phase 4b | Pending |
| DIFF-08 | Phase 4b | Pending |
| ML-01 | Phase 5 | Pending |
| ML-02 | Phase 5 | Pending |
| ML-03 | Phase 5 | Pending |
| ML-04 | Phase 5 | Pending |
| ML-05 | Phase 5 | Pending |
| PKG-01 | Phase 5 | Pending |
| PKG-02 | Phase 5 | Pending |
| PKG-03 | Phase 5 | Pending |
| PKG-04 | Phase 5 | Pending |
| PKG-05 | Phase 5 | Pending |
| PKG-06 | Phase 5 | Pending |
| PKG-07 | Phase 5 | Pending |

**Coverage:**
- v1 requirements: 137 total
- Mapped to phases: 137 (Phase 1: 31, Phase 2: 34, Phase 3: 26, Phase 4a: 26, Phase 4b: 8, Phase 5: 12)
- Unmapped: 0

**Per-phase requirement counts:**

| Phase | Count | Categories |
|-------|-------|------------|
| Phase 1 — MU Fundamentals End-to-End | 31 | INFRA (13), WATCH (8), FUND (10) |
| Phase 2 — Sector-Aware Primary Panels (MU + IRM) | 34 | REIT (8), SEMI (2), INS (10), CAL (6), TECH (8) |
| Phase 3 — Analyst + Sentiment + Statistical Forecast | 26 | ANL (3), SENT (10), FCST (13) |
| Phase 4a — Disconfirmation + Macro + Earnings-Call Summary | 26 | DISC (9), MACRO (8), SUM (9) |
| Phase 4b — Cross-Cutting Differentiators | 8 | DIFF (8) |
| Phase 5 — ML Forecast + Packaging Hardening | 12 | ML (5), PKG (7) |
| **Total v1** | **137** | |

---
*Requirements defined: 2026-05-25*
*Last updated: 2026-05-25 — traceability populated by roadmapper, 137/137 v1 requirements mapped to 6 phases*
