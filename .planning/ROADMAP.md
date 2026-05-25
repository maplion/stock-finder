# Roadmap: Stock Finder

## Overview

A personal-use desktop stock-information aggregator (Tauri 2 + Angular 20 + Python sidecar) built as **six vertical-slice phases**, each shipping a demoable end-to-end capability for tracked tickers (starting with MU + IRM). The journey starts with one panel for one ticker (Phase 1 proves the entire IPC + packaging pipeline), expands to the high-signal panel tier in Phase 2 (sector-aware fundamentals + insider + earnings + technicals for both MU and IRM), adds the medium-signal forecast/sentiment/analyst panels in Phase 3 (with honesty UX baked in — `forecasts_log` from day one, naive baseline, backtest headline), lands the analysis-driven panel tier in Phase 4a (Disconfirmation + Macro context + Earnings-Call LLM Summary — the panels the project analysis added as primary/medium-signal), surfaces the cross-cutting differentiators in Phase 4b (what-changed digest, forecast back-tracking, cross-ticker comparison, thesis tracker, IRM NOI/occupancy), then hardens to a ML forecast + cross-platform signed installer in Phase 5. Phase ordering is risk-ordered and respects the signal hierarchy from PROJECT.md and PROJECT-ANALYSIS.md.

## Phases

**Phase Numbering:**
- Integer phases (1, 2, 3, 5): Planned milestone work
- Decimal phases (4a, 4b): Phase 4 split into two because the analysis-driven panel additions (DISC, MACRO, SUM, expanded CAL) plus the original cross-cutting differentiators (DIFF) exceed the coarse-granularity 3-plan cap if combined

- [ ] **Phase 1: MU Fundamentals End-to-End** — Tauri + Angular + Python sidecar + watchlist + common-stock fundamentals for MU, end-to-end
- [ ] **Phase 2: Sector-Aware Primary Panels (MU + IRM)** — REIT extension + semi helpers + earnings cycle + insider clusters + technicals for both tickers
- [ ] **Phase 3: Analyst + Sentiment + Statistical Forecast** — medium/low-signal panels with mandatory honesty UX (`forecasts_log`, naive baseline, backtest headline)
- [ ] **Phase 4a: Disconfirmation + Macro + Earnings-Call Summary** — the analysis-driven panel additions (anti-confirmation-bias, sector context, LLM filing summary)
- [ ] **Phase 4b: Cross-Cutting Differentiators** — what-changed digest, forecast back-tracking, cross-ticker comparison, thesis tracker, IRM NOI/occupancy, threshold warnings, sentiment×momentum quadrant
- [ ] **Phase 5: ML Forecast + Packaging Hardening** — darts NHiTS as fourth forecast line + cross-platform signed/notarized installers

## Phase Details

### Phase 1: MU Fundamentals End-to-End
**Goal:** User can launch the desktop app, see MU in their watchlist, click into MU, and view a complete common-stock Fundamentals panel rendered end-to-end from the SEC + Yahoo sources within ~10 seconds — with the panel-priority dashboard layout (Fundamentals → Insider → Earnings → Disconfirmation → Macro → Analyst → Technicals → Summary → Sentiment → Forecast) already structurally in place even though most tabs are stubbed.
**Mode:** mvp
**Depends on:** Nothing (first phase)
**Requirements:** INFRA-01, INFRA-02, INFRA-03, INFRA-04, INFRA-05, INFRA-06, INFRA-07, INFRA-08, INFRA-09, INFRA-10, INFRA-11, INFRA-12, INFRA-13, WATCH-01, WATCH-02, WATCH-03, WATCH-04, WATCH-05, WATCH-06, WATCH-07, WATCH-08, FUND-01, FUND-02, FUND-03, FUND-04, FUND-05, FUND-06, FUND-07, FUND-08, FUND-09, FUND-10
**Success Criteria** (what must be TRUE):
  1. User launches the bundled `.app` on a clean macOS arm64 user account (not just the dev machine), sees MU and IRM seeded as data rows in their watchlist, clicks MU, and a Fundamentals panel renders revenue / margins / EPS / FCF / cash / debt / valuation ratios with 8-quarter sparklines within ~10 seconds.
  2. The TickerDetail page shows ten tabs in the signal-hierarchy order specified in WATCH-06 (Fundamentals → Insider → Earnings → Disconfirmation → Macro → Analyst → Technicals → Summary → Sentiment → Forecast); only Fundamentals is functional, the other nine render "Coming in Phase 2/3/4" placeholders without breaking the layout.
  3. When yfinance is forced to fail (test toggle or unplug network mid-request), the panel renders successfully via yahooquery and shows a visible "yfinance error, fell back to yahooquery" badge per FUND-10; the user always knows which source they're seeing per INFRA-06.
  4. The user can add a new ticker by symbol (validated against yfinance per WATCH-02), remove a ticker, cancel an in-flight refresh, and see a per-panel last-refreshed timestamp with TTL countdown; killing the app leaves zero orphan `stockfinder-py` processes in `ps`.
  5. SEC EDGAR calls use the project User-Agent `"StockFinder ryandammrose@onepointtwocapital.com"` and the global rate limiter caps at ≤8 req/s (verified by request log); the SQLite cache TTL config (INFRA-12) is loaded at startup and visible in a settings/diagnostics view; Claude API key field exists in settings (INFRA-13) but is unused by any panel yet.
**Plans:** TBD
**UI hint:** yes

### Phase 2: Sector-Aware Primary Panels (MU + IRM)
**Goal:** User can view all four high-signal primary-tier panels (Fundamentals with REIT/Semi extensions, Insider with cluster signals, Earnings cycle with beat/miss + guide-vs-actual, Technicals with the "context not signal" framing) for both MU and IRM — exercising the sector-aware architectural seam end-to-end and proving the REIT-vs-Semi heterogeneity premise.
**Mode:** mvp
**Depends on:** Phase 1
**Requirements:** REIT-01, REIT-02, REIT-03, REIT-04, REIT-05, REIT-06, REIT-07, REIT-08, SEMI-01, SEMI-02, INS-01, INS-02, INS-03, INS-04, INS-05, INS-06, INS-07, INS-08, INS-09, INS-10, CAL-01, CAL-02, CAL-03, CAL-04, CAL-05, CAL-06, TECH-01, TECH-02, TECH-03, TECH-04, TECH-05, TECH-06, TECH-07, TECH-08
**Success Criteria** (what must be TRUE):
  1. Opening IRM shows a REIT Metrics block (FFO TTM + Q, AFFO TTM + Q, FFO/share, AFFO/share, P/FFO + P/AFFO in the primary valuation slot, AFFO payout ratio with green/yellow/red traffic light, REIT-adjusted net-debt/EBITDA, interest coverage); P/E is suppressed with the "use FFO/AFFO for REITs" tooltip; if IRM XBRL tags `RealEstateInvestmentTrustFundsFromOperations` directly, both "Reported FFO" and "Computed FFO" appear side-by-side. Opening MU shows the Semiconductor helpers block (inventory + inventory days, CapEx TTM + CapEx/Revenue) and no REIT Metrics block.
  2. The Insider panel for MU (and IRM) shows a transactions table sorted by `transaction_date DESC` with a filing-lag caption, defaulted to code P/S only with a "Show all" toggle exposing M/A/G/etc.; M+S same-day pairs are tagged `exercise-sale` and excluded from sale aggregates; 10b5-1 flagged trades are excluded from cluster signals by default; a green prominent "Cluster Buy" badge fires only when ≥3 unique insiders made code-P purchases ≥$25k each within ≤10 days (matching the academic standard); cluster-sell badges are labeled "investigate" with conservative styling per the asymmetric weighting requirement.
  3. The Earnings Cycle panel shows a "Next earnings: YYYY-MM-DD (T+N days)" banner plus a last-4-quarters table with traffic-light cells per quarter on (revenue surprise %, EPS surprise %, guidance hit/miss) and a separate guide-vs-actual track record table; this panel sits in the primary-tier visual position above Sentiment and Forecast.
  4. The Technicals panel renders a single combined chart (candlestick + 50/200 SMA overlay + RSI(14) sub-panel + MACD(12,26,9) sub-panel via ngx-echarts) sourced from a Parquet-cached OHLCV file, plus a signals list showing RSI overbought/oversold state, MACD crossover flag, golden/death cross detection, volume-surge flag (>2× 20-day avg), and distance from 52-week high/low; the panel header carries the "Context — not a trading signal" subtitle.
  5. Adjusted vs raw close convention is established and documented (`auto_adjust=False` in yfinance); both Adj Close and Close are persisted in the OHLCV parquet cache; technicals/forecasts consume adjusted, the "current price" badge uses raw — and this is asserted by a smoke test.
**Plans:** TBD
**UI hint:** yes

### Phase 3: Analyst + Sentiment + Statistical Forecast
**Goal:** User can view the medium-signal Analyst Consensus panel and the explicitly-demoted low-signal Sentiment (news-only) and Statistical Forecast panels with their mandatory honesty UX in place — backtest accuracy headline, naive baseline, `forecasts_log` populated from day one, walk-forward CV, FinBERT lazy-downloaded with latency kill-switch.
**Mode:** mvp
**Depends on:** Phase 2
**Requirements:** ANL-01, ANL-02, ANL-03, SENT-01, SENT-02, SENT-03, SENT-04, SENT-05, SENT-06, SENT-07, SENT-08, SENT-09, SENT-10, FCST-01, FCST-02, FCST-03, FCST-04, FCST-05, FCST-06, FCST-07, FCST-08, FCST-09, FCST-10, FCST-11, FCST-12, FCST-13
**Success Criteria** (what must be TRUE):
  1. The Forecast panel **leads with a backtest-accuracy headline** ("Prophet directional hit rate on this ticker last 24 months: NN%" + "in-CI rate: NN%") rendered above any forecast chart; the panel sits below primary panels with smaller real estate and carries the "Reference only — not decision input" subtitle; every forecast generation writes a row to the `forecasts_log` SQLite table (point, ci_lo, ci_hi, generated_at, target_date, model_name, horizon, symbol) — no forecast is rendered without being logged first.
  2. The Forecast panel renders three labeled lines side-by-side (never blended): analyst consensus, Prophet, and a naive last-value-persistence baseline; user can switch the secondary statistical model (AutoARIMA / ETS / Theta) via dropdown per FCST-05; horizons 1-3mo (daily), 3-12mo (weekly), 1-3yr (monthly) are selectable, with CI bands visibly widening at longer horizons and a long-horizon disclaimer; any model that fails to beat the naive baseline over a rolling 90-day window shows a "below naive" badge; all model evaluation uses walk-forward CV (no `train_test_split`, no shuffled CV) — asserted by a unit test on the evaluation harness.
  3. The Analyst Consensus block (mean target, high, low, # analysts, rating distribution) is visually separated from the Forecast panel — distinct section header and styling — and labeled "targets, not forecasts" so analyst PTs and model outputs are never blended into a single number.
  4. The Sentiment panel ingests headlines from ≥2 of (yfinance news, RSS feeds, GDELT DOC 2.0), scores every headline with FinVADER, re-scores top-N with FinBERT, renders 7d and 30d aggregated tone plus a 30-day sparkline plus a per-source breakdown where any single source's contribution is capped at ~30%; FinBERT is lazy-downloaded to app cache on first sentiment-panel use with a progress dialog (not bundled); if FinBERT first-use latency exceeds 500 ms per top-N rescore, the panel transparently falls back to FinVADER-only and surfaces this in a status badge; panel header carries the "Lower-signal — headlines often coincident with price" subtitle.
  5. Reddit panel is **not present** in this phase (deferred to v2 per the analysis re-evaluation); the slot where it would have lived in the tab order shows no tab at all, confirming the v1 signal-hierarchy commitment.
**Plans:** TBD
**UI hint:** yes

### Phase 4a: Disconfirmation + Macro + Earnings-Call Summary
**Goal:** User can view the three analysis-driven panels added per PROJECT-ANALYSIS.md — the Disconfirmation bear-signal surface (anti-confirmation-bias, primary tier), the Macro/sector context panel (medium tier), and the Earnings-Call LLM Summary panel (medium tier, graceful degradation without API key) — for both MU and IRM, with sector-aware bear signals (REIT debt maturity wall for IRM, semiconductor inventory days rising for MU).
**Mode:** mvp
**Depends on:** Phase 3
**Requirements:** DISC-01, DISC-02, DISC-03, DISC-04, DISC-05, DISC-06, DISC-07, DISC-08, DISC-09, MACRO-01, MACRO-02, MACRO-03, MACRO-04, MACRO-05, MACRO-06, MACRO-07, MACRO-08, SUM-01, SUM-02, SUM-03, SUM-04, SUM-05, SUM-06, SUM-07, SUM-08, SUM-09
**Success Criteria** (what must be TRUE):
  1. The Disconfirmation panel sits in the primary visual tier (alongside Fundamentals and Insider, framed as "what could go wrong") and surfaces sector-appropriate bear signals: recent analyst downgrades with prior/new rating + prior/new PT, gross/operating margin compression flags (≥150 bps YoY trailing-4Q drop), insider selling cluster flag (reusing INS-* logic at the asymmetric threshold), plus — for IRM — a debt maturity wall visualization (refi-risk flag if any single year >25% of total debt) and — for MU — an inventory-days-rising flag (DIO trending up ≥10% YoY); every bear signal carries a "why this matters" tooltip with academic/market rationale; bear-signal count badge appears on the watchlist row (e.g., "MU: 2 bear signals") so the user sees it without drilling in.
  2. The Macro panel renders sector-appropriate context driven by the ticker's `sector` tag: for MU it shows the SOX index (^SOXX) chart + 30/90-day delta, a DRAM-price proxy from TrendForce blog headlines (flagged "directional indicator only"), and a hyperscaler-capex commentary feed scraped for AWS/Azure/GCP capex mentions in last 30 days; for IRM it shows 10Y Treasury (DGS10) from FRED API + 30/90-day delta and a data-center context news scrape; all macro indicators are correlation-tagged ("typically inverse to price" / "typically directional") with the standard "correlations break" disclaimer; macro adapters honor the 6h cache TTL from INFRA-12.
  3. The Earnings-Call Summary panel fetches the latest 10-Q and 10-K via EDGAR and the latest earnings-call transcript per ticker; when a Claude API key is configured (settings field from INFRA-13), filings and transcript are summarized to exactly 5 bullets each — headline financial result, management tone shift, guidance change, one risk surfaced, one opportunity — with each bullet citing its source paragraph via a clickable scrollable excerpt; summaries are cached per-filing forever (filings don't change post-publication); user can manually re-run a summarization; per-summary token cost is logged and visible.
  4. When no Claude API key is configured, the Summary panel **degrades gracefully** — it shows the raw filing links + transcript link with a "Configure API key in Settings to enable summary" status; no error, no broken state, the rest of the dashboard works identically.
  5. Summarization defaults to Claude Haiku (low-cost, sufficient for summarization) with a settings override to Sonnet or Opus; switching models doesn't break cached summaries (cache key includes model name).
**Plans:** TBD
**UI hint:** yes

### Phase 4b: Cross-Cutting Differentiators
**Goal:** User can experience the features that turn ten standalone panels into one consolidated tool — "what changed since last open" digest banner on every TickerDetail page, forecast back-tracking UI ("model said $X N days ago; actual $Y; hit rate Z%"), cross-ticker comparison view (sector-aware), free-text thesis tracker, IRM-specific NOI/occupancy from 8-K supplemental, threshold warnings on the digest, and sentiment×momentum quadrant — all built on top of the panels delivered in Phases 1-4a.
**Mode:** mvp
**Depends on:** Phase 4a
**Requirements:** DIFF-01, DIFF-02, DIFF-03, DIFF-04, DIFF-05, DIFF-06, DIFF-07, DIFF-08
**Success Criteria** (what must be TRUE):
  1. On every TickerDetail page, a "What Changed Since Last Open" digest banner shows the items the user has not previously seen: new headlines since last visit, new Form 4 filings, new analyst target/rating changes, plus any user-configured threshold breaches (e.g., "MU RSI > 70"); items are deduped against `headlines_seen` and `insider_filings_seen` SQLite tables which persist last-seen state per ticker; `headlines_seen` rows are pruned at 90 days; computed at fetch time only (no background workers).
  2. A Forecast Back-Tracking view (per-model, read-side over the `forecasts_log` populated from Phase 3 day one) shows: for each historical forecast, "model said $X N days ago for target Y; actual $Z; in-CI: yes/no; rolling hit rate W%"; this surface is the antidote to forecast magical thinking — the user can see at a glance whether to trust each model's track record on each ticker.
  3. A Cross-Ticker Comparison view renders fundamentals side-by-side for 2-4 tracked tickers in a single table — sector-aware so REIT metrics appear only in REIT-ticker columns and Semi helpers only in semi columns; the IRM column additionally shows NOI, same-store NOI growth, and occupancy/utilization sourced from IRM's 8-K supplemental package (DIFF-08); each ticker has a free-text thesis-tracker note field that persists across sessions (no NLP, no auto-invalidation in v1).
  4. A Sentiment × Momentum quadrant scatter plot renders watchlist tickers by (Δ sentiment over trailing baseline, momentum score) — the sentiment axis is **delta, not level** (per the anti-confirmation-bias / anti-lag pitfall mitigation); each ticker appears as a labeled dot so the user can see at a glance which names are quietly turning.
  5. User can configure threshold warnings per ticker (e.g., "alert when MU RSI > 70" or "AFFO payout > 95%") in settings; breached thresholds appear in the "What Changed" digest banner on next open — never as push notifications, never as background polling.
**Plans:** TBD
**UI hint:** yes

### Phase 5: ML Forecast + Packaging Hardening
**Goal:** User receives a code-signed and notarized installer (macOS arm64) and a code-signed Windows x64 installer that they can hand to themselves on a clean machine and have working in one click — and the Forecast panel now renders a fourth selectable line (darts NHiTS or equivalent ML baseline) with the same honesty contract (walk-forward CV, "below naive" badge when applicable, `ForecastCard` contract preserved).
**Mode:** mvp
**Depends on:** Phase 4b
**Requirements:** ML-01, ML-02, ML-03, ML-04, ML-05, PKG-01, PKG-02, PKG-03, PKG-04, PKG-05, PKG-06, PKG-07
**Success Criteria** (what must be TRUE):
  1. User installs the macOS arm64 build on a clean macOS user account by double-clicking a notarized `.dmg` — Gatekeeper accepts it without "cannot be verified", the app launches, the sidecar runs, MU + IRM panels render exactly as in Phase 4b; same install flow works for Windows x64 with a code-signed installer (no SmartScreen block for a signed binary with reputation in progress); no Linux build ships in v1 (Linux deferred per PROJECT.md "Out of Scope").
  2. The CI matrix runs PyInstaller builds for macOS arm64 + Windows x64 only (per the v1 platform decision); each build runs a post-build smoke test that invokes every supported sidecar `op` against a synthetic ticker on a fresh VM; any ImportError or missing-hidden-import fails CI; the `hooks/` directory captures known hidden imports (sklearn.utils._typedefs, finvader data, prophet data).
  3. Installer size budget is enforced at <150 MB (FinBERT is lazy-downloaded on first sentiment-panel use, not bundled — verified on a clean user account); macOS bundle uses hardened runtime with minimal-but-sufficient entitlements (`allow-jit`, `allow-unsigned-executable-memory`, `disable-library-validation`, `network.client`).
  4. The Forecast panel now offers a fourth model line: darts NHiTS (or equivalent neural baseline) is computed and rendered with the same `ForecastCard` contract as the statistical models — point estimate, 80% CI, model name, training window, last-updated timestamp, naive-baseline comparison, backtest hit rate, "Not financial advice" footer; ML training data is cached as Parquet frames per ticker; feature engineering is documented in code.
  5. The ML model is evaluated with walk-forward CV (same harness as Phase 3); if the ML model cannot beat the naive baseline on the 90-day rolling validation for a given ticker, its `ForecastCard` renders with a prominent "below naive" badge plus a tooltip explaining what that means — preserving the project's commitment that "model couldn't beat naive" is a feature working as designed, not a bug.
**Plans:** TBD
**UI hint:** yes

## Progress

**Execution Order:**
Phases execute in numeric order: 1 → 2 → 3 → 4a → 4b → 5

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. MU Fundamentals End-to-End | 0/TBD | Not started | - |
| 2. Sector-Aware Primary Panels (MU + IRM) | 0/TBD | Not started | - |
| 3. Analyst + Sentiment + Statistical Forecast | 0/TBD | Not started | - |
| 4a. Disconfirmation + Macro + Earnings-Call Summary | 0/TBD | Not started | - |
| 4b. Cross-Cutting Differentiators | 0/TBD | Not started | - |
| 5. ML Forecast + Packaging Hardening | 0/TBD | Not started | - |

---

*Roadmap created: 2026-05-25*
*Granularity: coarse (6 phases — Phase 4 split per scope guidance in roadmapper instructions)*
*Project mode: vertical MVP — every phase ships a demoable end-to-end vertical slice*
*Signal hierarchy honored: primary panels (Fundamentals + Insider + Earnings + Disconfirmation) ship before medium/low; Forecast & Sentiment ship with mandatory honesty UX*
