# Research Summary — Stock Finder

**Project:** Stock Finder (personal-use desktop stock-information aggregator)
**Synthesized:** 2026-05-25
**Sources:** STACK.md (HIGH confidence), FEATURES.md (HIGH), ARCHITECTURE.md (HIGH), PITFALLS.md (HIGH)
**Initial tickers:** MU (semiconductor cyclical) + IRM (data-center / records-storage REIT)

---

## 1. TL;DR (for the roadmapper)

- **Phase 1 vertical slice is named and prescriptive:** "MU Fundamentals End-to-End" — Tauri scaffold + sidecar IPC + watchlist + one panel (common-stock fundamentals) for one ticker (MU). This exercises every architectural layer with minimum feature surface. Do NOT try to ship horizontal-slice "infrastructure phases" — every phase must be demoable.
- **Coarse build order is already derived (Phases 1–9 in ARCHITECTURE.md §7), compressible to 4–5 coarse roadmap phases.** Risk-ordered, vertical, and respects the sector-aware architectural seam (REIT extension as Phase 2 is deliberate — it validates the whole premise of MU + IRM heterogeneity).
- **REIT-specific handling for IRM is a first-class concern, not a footnote.** Showing IRM's GAAP P/E next to MU's is actively misleading; FFO / AFFO / P-AFFO / payout-ratio must replace P/E for `sector === 'REIT'`. This drives the sector tag plumbing in Phase 1.
- **Forecast honesty is non-negotiable** and shapes the forecast phase's success criteria: every forecast carries CI, a naive baseline comparison, model provenance, and is persisted to a `forecasts_log` table from day one. No `forecasts_log` = no shipping forecasts.
- **Five pitfalls dominate the design:** (1) yfinance fragility (must ship fallback cascade), (2) REIT GAAP misrepresentation (sector tag), (3) forecast magical thinking (`forecasts_log` + CI + naive baseline), (4) insider signal/noise (10b5-1 + M+S + code-P-only filters), (5) Tauri + PyInstaller bundling (per-target builds + entitlements + lazy FinBERT download). Each must show up as success criteria on the phase that owns it.

---

## 2. Recommended Stack (one-liner per layer)

Full detail: `.planning/research/STACK.md`. Headline choices:

| Layer | Pick | Why |
|---|---|---|
| Shell | Tauri 2.x (≥2.11) on Rust ≥1.77.2 | Smaller binary + lower RAM than Electron; mature plugin ecosystem |
| Frontend | Angular 20 LTS, standalone components, zoneless | Latest LTS; cleaner reactivity via signals |
| State | Angular signals + plain services (NO NgRx) | Single-user app; NgRx is over-engineering |
| Tables | PrimeNG DataTable | Sort/filter/freeze out of the box, MIT |
| Charts | ngx-echarts + ECharts 5.x | Candlestick + RSI/MACD sub-panels, finance-grade, MIT/Apache |
| Styling | Tailwind v4 + PrimeNG unstyled | Utility-first; behaviour from PrimeNG, design from Tailwind |
| Sidecar transport | PyInstaller one-file + NDJSON over stdin/stdout | Trivial packaging, panel-by-panel streaming, no port binding |
| Sidecar packager | PyInstaller 6.x one-file per target triple | Mature; Nuitka not worth 10× build time for I/O work |
| Price/quote data | yfinance (primary) → yahooquery (fallback) → FMP free (secondary) | Layered cascade is mandatory — yfinance is fragile by design |
| Fundamentals | edgartools 5.x (XBRL standardisation) + httpx for raw EDGAR | Only library with serious cross-filer concept mapping |
| REIT FFO/AFFO | edgartools XBRL pull + custom FFO/AFFO calculator | Show "Reported FFO" alongside "Computed FFO"; gap is signal |
| Insider trades | edgartools Form 3/4/5 + openinsider scrape (cluster cross-check) | edgartools is typed + MIT; openinsider precomputes clusters |
| News | feedparser + GDELT DOC 2.0 + yfinance.Ticker.news | Multi-source to dilute publisher bias |
| Reddit | PRAW 7.7+ (OAuth) | Official wrapper, respects rate limits, still works in 2026 |
| Sentiment | FinVADER (all) + FinBERT (top-N rescoring, lazy load) | NEVER plain VADER (44% F1 on finance text) |
| Forecasting | statsforecast (AutoARIMA/Theta/ETS) + prophet + darts NHiTS (deferred) | Statistical baseline first; ML last because XL and dependency-heavy |
| Technicals | pandas-ta-classic | Pure-Python, no TA-Lib C-extension install pain |
| Cache | SQLite (Tauri plugin-sql) for metadata + Parquet (pyarrow) for OHLCV / ML frames | Rust owns UI state; Python owns numeric frames |
| Rate limiting | httpx + tenacity + aiolimiter; EDGAR cap 8 req/s; UA `"StockFinder ryandammrose@onepointtwocapital.com"` | SEC requires identifiable UA + ≤10 req/s |

---

## 3. Phase 1 Vertical Slice — "MU Fundamentals End-to-End"

Named and scoped by ARCHITECTURE.md §7. Roadmapper should adopt verbatim.

**Goal:** Prove the entire pipe with the minimum feature surface — common-stock fundamentals for MU, end-to-end, no REIT, no ML, no other panels.

**Includes (and ONLY includes):**
- Tauri 2 scaffold + Angular 20 standalone shell + Python sidecar bootstrap
- Required Tauri plugins installed + capabilities configured: `shell`, `sql`, `fs`, `store`, `log`
- PyInstaller build pipeline for `stockfinder-py-<triple>` (mac arm64 only is acceptable for Phase 1; cross-platform deferred to Phase E)
- Minimum SQLite schema: `tickers`, `refresh_log` (`CREATE TABLE IF NOT EXISTS`; no migration framework)
- Tauri commands: `list_tickers`, `add_ticker`, `remove_ticker`, `fetch_ticker_data`, `get_refresh_log`, `cancel_request`
- Full NDJSON IPC contract implemented (only `fundamentals` panel supported; others return `error: NOT_IMPLEMENTED`)
- Python modules: `adapters/yfinance_adapter.py`, `adapters/yahooquery_adapter.py` (fallback proven), `adapters/edgar_adapter.py`, `fundamentals/common.py`, `http_client.py` (with EDGAR 8 req/s limiter + project UA)
- Angular UI: `WatchlistPage`, `TickerDetailPage` with `FundamentalsPanel` only (other tabs stubbed "coming soon")
- `TickerDataStore` + `PanelErrorBoundary` complete — architecture supports five panels even though four are stubbed
- Seed watchlist with MU on first launch (data, NOT a hard-coded constant — anti-pitfall 7.6)
- Zoneless signals discipline established (anti-pitfall 5.5)
- Sidecar lifecycle: kill children on app shutdown (anti-pitfall 5.6)

**Acceptance criteria (roadmapper: bake these into Phase 1 success):**
- [ ] Launch app → MU appears in watchlist
- [ ] Click MU → Fundamentals panel renders revenue / margins / EPS / FCF / cash / debt / ratios within ~10 seconds
- [ ] "Refresh" button works and updates the last-refreshed timestamp
- [ ] Killing the sidecar mid-fetch produces a clean panel error (PanelErrorBoundary catches; other tabs unaffected)
- [ ] Forcing yfinance to fail produces visible "yfinance error, fell back to yahooquery" status (proves the fallback cascade) — **anti-pitfall 1.1 and 6.1**
- [ ] EDGAR calls use the project User-Agent and stay under 8 req/s — **anti-pitfall 1.5 and 6.5**
- [ ] App opens with sidecar running and exits with no orphan PIDs in `ps` — **anti-pitfall 5.6**
- [ ] PyInstaller bundle runs on a clean macOS user account (not just dev machine) — **anti-pitfall 5.1**

**Why this slice and not "wire up all five panels with mocks":** Mocks don't exercise the IPC, the rate limiter, the cache layer, the PyInstaller packaging path, or the cross-OS shell capabilities — the things most likely to bite. A real end-to-end vertical of one panel exercises every layer.

---

## 4. Recommended Build Order (Coarse: 4–5 phases)

ARCHITECTURE.md §7 lists 9 fine-grained phases. For YOLO + Coarse granularity (3–5 phases, 1–3 plans each), they compress as follows. The roadmapper SHOULD use this grouping unless it has a strong reason otherwise.

### Coarse Phase A — "Foundation + MU Fundamentals" (= ARCHITECTURE Phase 1)
End-to-end vertical slice with one panel, one ticker. Validates IPC, packaging, cache, sidecar lifecycle. **See §3 above for full scope and acceptance.**

### Coarse Phase B — "Sector-Aware Fundamentals + Technicals + Insider" (= ARCHITECTURE Phases 2 + 3 + 4)
The three structurally-similar panels added together, in risk-ordered sub-sequence:
1. **REIT extension via IRM** (Phase 2): `fundamentals/reit.py`, `is_reit()`, `ReitMetricsBlock`, sector dispatch, add IRM to seed watchlist. Validates the sector-aware seam — the architectural premise of the entire project.
2. **Technicals** (Phase 3): `technicals/indicators.py` (pandas-ta-classic), `cache/parquet_store.py` for OHLCV, `TechnicalsPanel` (SignalsList + candlestick + RSI/MACD sub-panels). Validates the Parquet cache and chart toolchain.
3. **Insider** (Phase 4): edgartools Form 4 + openinsider adapter + `insider/clusters.py` + `InsiderPanel` (cluster badges + net-flow chart + transactions table). Validates the dual-source-with-fallback pattern and HTML scraping.

**Must include as success criteria** (from PITFALLS §1.2, §1.3, §1.4, §4.1–4.6):
- IRM ticker view shows REIT Metrics block; MU view does NOT; P/E is suppressed or de-emphasised for REIT
- "Reported FFO" and "Computed FFO" both shown when components are available
- Adjusted vs raw close convention is established and documented (`auto_adjust=False` in yfinance)
- Insider table defaults to code P/S only, with explicit "All transactions" toggle
- Cluster-buy detection: ≥3 unique insiders, ≥$25k each, within 10 days, code P only, AFTER M+S and 10b5-1 filters
- Cluster-SELL signal is labelled "investigate", not "bearish"; buy is elevated, sell de-emphasised (asymmetry)
- Form 4 panel orders by `transaction_date DESC` (not filing date) and shows the lag caption

### Coarse Phase C — "Sentiment + Forecasts (statistical only)" (= ARCHITECTURE Phases 5 + 6)
The two computationally-heavier panels, paired together:
1. **Sentiment** (Phase 5): news adapters (feedparser + GDELT) + reddit adapter (PRAW with OAuth onboarding) + `sentiment/finvader_scorer.py` + `SentimentPanel` (headline table + sparkline + per-subreddit Reddit block). **FinBERT lazy-downloaded on first use** (anti-pitfall 5.3). First-run UX prompts user for Reddit dev-app credentials.
2. **Forecasts — statistical only** (Phase 6): `forecast/analyst.py`, `forecast/statistical.py` (statsforecast + prophet), `forecast/interface.py`, `ForecastPanel` with horizon selector + side-by-side chart + summary table. **ML deferred to Phase E.**

**Must include as success criteria** (from PITFALLS §2.1–2.4, §3.1–3.5, §6.3, §8.1–8.2):
- `forecasts_log` SQLite table populated on EVERY forecast generation from day one (point, ci_lo, ci_hi, generated_at, target_date, model_name, horizon, symbol)
- Every forecast card displays: model name, training window, last-updated timestamp, point estimate, 80% CI, naive-baseline comparison, horizon, "Not financial advice" footer
- Naive (last-value persistence) baseline is computed and rendered alongside the three "real" models; if a model can't beat naive over 90-day rolling windows, flag with "below naive" badge
- Analyst targets visually separated from model forecasts (distinct section header + styling) — never blended into a single number
- 1-3 year forecasts widen CI visibly and carry a long-horizon disclaimer; resampled to monthly
- News sentiment ships per-source breakdown with single-source contribution capped at ~30%
- Reddit panel captions "buzz follows price, treat as confirmation not prediction" and filters low-karma/young accounts
- Reddit OAuth first-run flow prompts user to register their own reddit.com/prefs/apps credentials (recommend per-user app over shared client_id)
- Headline-only sentiment in v1 is documented in panel tooltip; body-fetch on high-signal events deferred to Phase D

### Coarse Phase D — "Cross-Cutting Differentiators" (= ARCHITECTURE Phases 7 + 8 + selected v1.1)
The features that turn the dashboard from "panels rendering data" into "one place to look":
1. **Forecast Back-Tracking UI** (Phase 7): "Model said $X 90 days ago; actual $Y; in-CI y/n; hit rate Z%" — the antidote to forecast magical thinking. `forecasts_log` already populated from Phase C, so this is read-side only.
2. **"What Changed Since Last Open" Diff** (Phase 8): `headlines_seen` + `insider_filings_seen` SQLite tables + diff computation at fetch time + digest banner on TickerDetailPage. **The single most aligned feature with "one place to look".**
3. **v1.1 quick wins** (from FEATURES.md §2): cross-ticker comparison view, sentiment × momentum quadrant (delta-not-level on sentiment axis), thesis tracker (free-text only), threshold warnings, earnings calendar banner if not yet shipped, "what changed" body-fetch on high-signal events.

### Coarse Phase E — "ML Forecast + Packaging Hardening" (= ARCHITECTURE Phase 9 + 8 packaging tasks)
1. **ML forecast** (`forecast/ml.py` darts NHiTS) + ML training-frame builder + third forecast line in ForecastChart. **XL complexity per FEATURES.md; explicitly last because it depends on everything above.** Must use walk-forward CV (no look-ahead bias) and fail visibly if it can't beat naive.
2. **Packaging hardening**: cross-platform PyInstaller CI matrix (mac arm64 + mac x86 + windows + linux), code-sign + notarize on macOS (Developer ID + hardened runtime + entitlements), code-sign on Windows, post-build smoke test that invokes every `op` in the JSON protocol on a clean VM, bundle-size budget enforcement, FinBERT-lazy-download confirmed working.

### Build-order rationale (for the roadmapper)
1. **Vertical slices, never horizontal.** Each phase ships something usable end-to-end, never "the SQLite layer for everything".
2. **Cheapest-to-validate-the-architecture first.** Phase A = one panel, one ticker — exposes every layer with minimum scope.
3. **Risk-ordered, not feature-ordered.** REIT extension is in Phase B because it validates the sector seam (the architectural premise of MU + IRM coexistence). ML forecast is in Phase E because it's XL and depends on everything else.
4. **Honesty features ride with the panel they belong to, not as separate phases.** `forecasts_log` ships in Phase C with the forecast panel, not as Phase D — because a forecast panel without the log is irresponsible. The back-tracking UI is Phase D because it's a read-side surface over data Phase C already wrote.

---

## 5. Table Stakes Features (must-have for v1, grouped by panel)

From FEATURES.md §1.

### Watchlist + Per-Ticker Shell (Phase A)
- Ticker list (add/remove), persisted in SQLite, seeded with MU + IRM
- Ticker validation on add (probe yfinance before persisting)
- Per-ticker route `/ticker/:symbol` with panel tabs
- Sector tag on ticker (auto-detect via SIC 6798 = REIT) — drives sector-aware fundamentals dispatch
- Manual refresh button per panel + global "refresh all"

### Fundamentals — Common (Phase A)
Revenue (TTM + Q + YoY%), gross margin, operating margin, net income, GAAP + non-GAAP EPS, FCF, cash, total debt, net-debt/EBITDA, current ratio, shares out (basic + diluted), dividend + yield + payout, P/E (trailing + forward), P/S, EV/EBITDA, 8-quarter trend sparklines.

### Fundamentals — REIT Extension (Phase B — required for IRM)
**FFO + AFFO** (TTM + Q), FFO/share + AFFO/share, **P/FFO + P/AFFO** (replace P/E in primary slot), **AFFO payout ratio** (green <80%, yellow 80–95%, red >95%), net-debt/EBITDA REIT-adjusted, interest coverage. NOI + same-store NOI growth + occupancy/utilization deferred to Phase D (require 8-K supplemental parse).

### Fundamentals — Semiconductor Helpers (Phase B)
Inventory + inventory days, CapEx (TTM) + CapEx/Revenue. Forward guidance parse + segment split deferred to Phase D.

### Earnings Calendar Banner (Phase A or B)
"Next earnings: 2026-MM-DD (T+N days)" + last 4 quarters EPS surprise %. Trivial via yfinance; high relevance.

### Forecasts Panel (Phase C — statistical only; ML in Phase E)
Analyst consensus (mean + high + low + # analysts + rating distribution), Prophet/ARIMA baseline (point + 80% CI), horizon selector (1-3mo daily / 3-12mo weekly / 1-3yr monthly), unified chart (history grey + model lines + CI bands), 3-column summary table (model / point / CI / implied % vs current), "model says vs current" big number color-coded.

### Sentiment — News (Phase C)
Headline ingestion (yfinance + RSS + GDELT), per-headline FinVADER scoring + top-N FinBERT rescoring, 7d/30d aggregated, headline list with score + link, 30-day trend sparkline, per-source breakdown.

### Sentiment — Reddit (Phase C)
PRAW ingestion (WSB + investing + stocks + SecurityAnalysis), mention count (24h / 7d / 30d + delta), per-post VADER scoring, aggregated tone weighted by upvotes, top-5 hot posts, per-subreddit breakdown.

### Momentum / Technicals (Phase B)
RSI(14) with overbought/oversold bands, MACD(12,26,9) with crossover flag, MA state (50/200 SMA + golden/death cross), volume surge flag (>2× 20-day avg), 52-week high/low distance. Single combined chart (candlestick + SMA overlay + RSI sub-panel + MACD sub-panel).

### Insider Trades (Phase B)
Transactions table (filing date, transaction date, insider name, role, **transaction code with legend**, shares, price, $ value, post-tx ownership) — default-filtered to code P/S; cluster signals (≥3 unique insiders / 10d / ≥$25k each / code P only / after M+S and 10b5-1 filters); net flow chart (P − S in $ by month, last 12 months); CEO/CFO action highlights.

### Cache / Diff State (Phase D — supports "what changed")
`headlines_seen`, `insider_filings_seen`, `forecasts_log`, `prices_cache` SQLite tables; computed at fetch time (no background workers); pruned 90d for headlines; forecasts never pruned.

---

## 6. Differentiators (v1-if-cheap, v2-otherwise)

Ranked by leverage / cost ratio (FEATURES.md §2):

### Ship in v1 (cheap and high-leverage)
- **Sector-aware fundamentals** (M complexity, HIGH leverage) — already in Phase B. The headline differentiator vs Yahoo Finance.
- **Earnings calendar awareness** (S, HIGH usefulness) — trivial via yfinance, Phase A or B.

### Ship in v1.1 if cheap (Phase D)
- **Forecast back-tracking UI** (M, HIGH leverage) — Phase D. Killer feature for "comparison not ensemble" forecast UX; without this, the forecast panel is just guesses.
- **"What Changed Since Last Open" Diff** (M, HIGH leverage) — Phase D. THE feature most aligned with the "one place to look" value prop. The on-demand-fetch + small-cache approach is acceptable per FEATURES.md §6.
- **Cross-ticker comparison view** (M, MEDIUM) — Phase D once per-ticker panels stable.
- **Sentiment × Momentum quadrant** (M, MEDIUM) — Phase D; cheap once both inputs exist. Sentiment axis MUST be delta-not-level (anti-pitfall 3.2).
- **Threshold warnings on next open** (S, LOW leverage given on-demand model) — Phase D add-on.
- **Thesis tracker, free-text only** (S, MEDIUM) — Phase D; quick win.

### Defer to v2
- **Thesis tracker with NLP invalidation matching** (L) — fuzzy NLP work; not v1.
- **REIT NOI / occupancy parse from 8-K supplemental** (M) — high-friction Playwright work; v1.1+ once foundational REIT panel is stable.
- **ML forecast** (XL) — Phase E. Listed as "differentiator" in spirit but operationally treated as deferred capability.

---

## 7. Anti-Features / Out of Scope (explicit, with rationale)

These are hard exclusions. Roadmapper must NOT slot any of these into a phase. From FEATURES.md §3 and PROJECT.md "Out of Scope":

| Anti-Feature | Why excluded |
|---|---|
| **Trade execution / brokerage integration** | Informational tool only; adds auth + regulatory surface wildly out of proportion to personal use |
| **Real-time intraday tick data / streaming** | Contradicts "on-demand, no background workers" constraint; daily OHLC sufficient for multi-horizon thesis use case |
| **Twitter/X sentiment** | X API in 2026 is paid-only above trivial limits; Reddit + news cover the need |
| **Multi-user / hosted version / auth** | Personal use only; zero value to a user of one |
| **Paid data vendors (Bloomberg, Refinitiv, Polygon Pro, FactSet)** | Cost + lock-in; free APIs + Playwright fallback cover personal use |
| **Full charting at TradingView's level** | TradingView is best-of-breed and free; cannot beat them. Lightweight chart only (50/200 SMA + RSI/MACD sub-panels). User opens TradingView for deeper work |
| **Heavy historical timeseries DB** | Providers serve history on demand; small local Parquet cache only for ML training and diff state |
| **Portfolio P&L / cost-basis tracking** | Brokerage tool's job; tax-lot accounting is its own product |
| **Options chain / Greeks / unusual options flow** | Different domain; data is expensive at quality; user did not request |
| **Backtesting engine for arbitrary strategies** | Use vectorbt/backtrader standalone if needed; forecast back-tracking (Phase D) is the scoped version |
| **Auto-rebalancing recommendations / robo-advisor logic** | Regulatory minefield; not the goal |
| **General-purpose news reader** | Feedly's job; news panel only surfaces ticker-tagged headlines |
| **Auto-refresh / background polling** | Explicitly violates "no background workers" constraint; manual refresh + on-open diff only |
| **Generic AI chatbot / "ask anything about this stock"** | High build cost, low marginal value over structured panels, high hallucination risk on financial data |
| **NgRx / NGXS / Akita** | Over-engineering for a single-user app; signals + services |
| **Schema migration framework** | `CREATE TABLE IF NOT EXISTS` + idempotent `ALTER TABLE`; bump cache dir for breaking changes |
| **Plugin/extension architecture** | One user, no third parties; refactor when concrete need emerges |
| **Observability stack (Prometheus, OTel, Datadog)** | `tauri-plugin-log` with rotation; no SRE rotation exists |
| **Multi-window / multi-monitor sophistication** | Single window; user can open twice if they want a second view |
| **Hard-coded MU / IRM in code** | Tickers are data (SQLite), not constants; sector branches on sector tag |

---

## 8. Critical Pitfalls That Shape The Roadmap (5 must-mitigate)

These five pitfalls have direct impact on phase scope and success criteria. The roadmapper must include each mitigation as an explicit acceptance criterion on the noted phase.

### 8.1 yfinance fragility (PITFALLS §1.1, §6.1)
**What:** yfinance silently returns stale or empty data when Yahoo restructures (roughly quarterly). Without a fallback cascade, panels go dark with no error.

**Where this should land:** **Phase A** acceptance criterion: "Forcing yfinance to fail produces visible `fell back to yahooquery` status badge; panel still renders." The fallback cascade (yfinance → yahooquery → FMP → cached-with-stale-flag) is implemented in the Phase A data-fetch abstraction layer. Every adapter response carries `(value, as_of_ts, source)` and the UI shows the source badge.

### 8.2 REIT GAAP misrepresentation for IRM (PITFALLS §1.2)
**What:** Showing IRM's P/E and net income as if comparable to MU's gives the user actively misleading information. REITs need FFO / AFFO / P-AFFO / payout-ratio.

**Where this should land:** Sector tag plumbing in **Phase A** (`tickers.sector` column + SIC 6798 auto-detection). REIT extension in **Phase B** as a discrete success criterion: "IRM view shows REIT Metrics block (FFO + AFFO + P/AFFO + payout ratio); MU view does not; P/E is suppressed for sector === 'REIT'; both 'Reported FFO' and 'Computed FFO' shown when components are available."

### 8.3 Forecast magical thinking (PITFALLS §2.1, §2.2, §2.3, §8.1, §8.2)
**What:** A forecast shown as a single number with no CI, no provenance, no backtest creates false confidence. Look-ahead bias makes backtests look great and live performance terrible.

**Where this should land:** **Phase C** acceptance criteria — non-negotiable:
- `forecasts_log` SQLite table populated on every forecast generation from day one
- Every `ForecastCard` component requires (compile-enforced) `{model_name, training_window, last_updated_ts, point, ci_lo, ci_hi, backtest_hit_rate_in_ci, horizon, disclaimer}`
- Naive (last-value persistence) baseline rendered alongside the three "real" models; "below naive" badge if a model can't beat it
- Analyst targets visually separated from model forecasts (header + styling)
- 1-3 year forecasts use widening CI and a long-horizon disclaimer
- Walk-forward CV used for all model evaluation (no train_test_split, no shuffled CV)

**Phase D** acceptance criterion: full back-test UI surfaced (read-side over `forecasts_log`).

### 8.4 Insider signal/noise (PITFALLS §4.1, §4.2, §4.4, §4.6)
**What:** Naive insider panels mix mechanical option-exercise sales (code M+S), pre-scheduled 10b5-1 trades, and stock grants (code A) into a single "insider sentiment" number, producing pure false positives. Buying is signal; selling is mostly noise.

**Where this should land:** **Phase B** insider sub-phase acceptance criteria:
- Transaction code column mandatory with legend tooltip
- Default filter: code P (purchase) and S (sale) only; "All transactions" toggle exposes the rest
- M+S same-day pairs tagged `exercise-sale` and excluded from sale aggregates
- edgartools 10b5-1 flag respected; planned trades excluded from cluster signals by default
- Cluster-buy threshold: ≥3 unique insiders / ≤10 days / each tx ≥$25k / code P only (matches academic standard)
- Cluster-sell flag uses tighter bar and labels "investigate" not "bearish"; UI elevates buy, de-emphasises sell (asymmetry)
- Form 4 table orders by `transaction_date DESC` not filing date; panel caption notes the 2-day filing lag

### 8.5 Tauri + PyInstaller bundling (PITFALLS §5.1, §5.2, §5.3, §5.8)
**What:** PyInstaller bundles are slow to cold-start, can be antivirus-flagged, need macOS code-sign + notarize + entitlements, and explode in size (FinBERT = 210 MB alone) without lazy-loading.

**Where this should land:**
- **Phase A** acceptance: PyInstaller build runs on a clean macOS user account (not just dev machine); `hooks/` directory with at minimum `--hidden-import sklearn.utils._typedefs`, `--collect-data finvader`, `--collect-data prophet`; post-build smoke test invokes every supported `op`.
- **Phase C** acceptance: FinBERT is lazy-downloaded on first sentiment-panel use (cache to `app_cache_dir()` with progress dialog); `torch` is lazy-imported only inside FinBERT path; CPU-only torch wheel used.
- **Phase E** acceptance: cross-platform CI matrix (mac arm64 + mac x86 + windows + linux); macOS code-sign + notarize + staple with hardened runtime + minimal-but-sufficient entitlements (`allow-jit`, `allow-unsigned-executable-memory`, `disable-library-validation`, `network.client`); Windows code-sign; bundle size budget enforced (<150 MB installer if FinBERT is lazy; <500 MB if bundled).

---

## 9. Open Questions / Tensions (flag for the user)

Items the roadmapper should surface as decisions-to-make rather than silently assume:

1. **Cross-platform support timing.** Architecture supports it (PyInstaller per-target triple); STACK.md §5 says mac arm64 only is acceptable for Phase 1. **Question for user:** is this a mac-only personal tool, or should Phase E include Windows / Linux from the outset? The dual-build CI cost is non-trivial.

2. **Reddit OAuth first-run UX.** STACK.md open question #4. Per-user app registration (user goes to reddit.com/prefs/apps, pastes credentials) is the cleaner Reddit-ToS path; a shared client_id is simpler UX but shares the 60 req/min quota. **Recommendation in this synthesis:** per-user app, with a first-run wizard. **Question for user:** confirm or override.

3. **FFO disclosure pattern in IRM's actual 10-K.** STACK.md open question #1. Does Iron Mountain tag `RealEstateInvestmentTrustFundsFromOperations` directly in XBRL, or must we always compute it? Phase B sub-task should verify against IRM's most recent 10-K (`q12026srp_final.htm` cited in FEATURES.md sources) before declaring the REIT extension done.

4. **FinBERT latency on Intel Macs.** STACK.md open question #3. If too slow on older hardware, ONNX export is the prescribed fallback. **Phase C must include a latency measurement** with kill-switch to fall back to FinVADER-only if FinBERT exceeds a budget (suggest: 500ms per top-N rescore on user's hardware).

5. **8-K supplemental parsing for IRM NOI / occupancy.** FEATURES.md §1.3 flags this as the "highest-friction data source" (M complexity, requires Playwright + extraction). **Deferred to Phase D in this synthesis**, but the user may want to elevate it to Phase B if NOI / occupancy is a primary thesis driver for IRM. Currently treated as v1.1.

6. **Forecast model breadth vs honesty.** PROJECT.md says "three models side-by-side". STACK.md proposes statsforecast (AutoARIMA / ETS / Theta) + prophet + darts NHiTS. That's actually FOUR slots if naive-baseline is also rendered (and per PITFALLS §2.1 it MUST be). **Question for user:** is the third visible model in Phase C "prophet" (more readable) or "statsforecast AutoARIMA" (more accurate baseline)? Both should exist in code; only one shown by default in v1.

7. **Tests in Phase A.** ARCHITECTURE.md §10 says "Phase 1: smoke tests only (one E2E 'MU fundamentals loads')." This is appropriate for YOLO mode and personal-use scope but **the roadmapper should NOT add comprehensive test plan items to Phase A** — that would over-engineer the slice. Per-adapter unit tests rise with shipped surface (Phase B onward).

8. **Tension: "no DB" vs "what changed since last open".** FEATURES.md §6 resolves this convincingly — the PROJECT.md constraint was *server-side* DB; a local SQLite for `last_seen` / `forecasts_log` / `prices_cache` is in scope and necessary for the highest-leverage differentiator. The roadmapper should treat this tension as **resolved** but flag the resolution in the Phase D plan so it's visible.

---

## Sources

Aggregated from the four research files. Full source lists in each.

**Architecture & stack patterns:**
- [Tauri 2 sidecar docs](https://v2.tauri.app/develop/sidecar/)
- [Tauri Python sidecar example (dieharders)](https://github.com/dieharders/example-tauri-v2-python-server-sidecar)
- [Writing a pandas sidecar for Tauri (MClare)](https://mclare.blog/posts/writing-a-pandas-sidecar-for-tauri/)
- [Angular signals architecture guide](https://angular.dev/guide/signals)
- [Angular zoneless change detection](https://angular.dev/guide/zoneless)
- [Apache ECharts candlestick examples](https://echarts.apache.org/examples/en/index.html#chart-type-candlestick)

**Data sources & rate limits:**
- [SEC EDGAR rate-limit announcement](https://www.sec.gov/filergroup/announcements-old/new-rate-control-limits)
- [SEC EDGAR XBRL companyconcept API](https://www.sec.gov/edgar/sec-api-documentation)
- [edgartools GitHub](https://github.com/dgunning/edgartools)
- [yfinance rate-limit issue thread](https://github.com/ranaroussi/yfinance/issues/2289)
- [Why yfinance keeps getting blocked (Medium)](https://medium.com/@trading.dude/why-yfinance-keeps-getting-blocked-and-what-to-use-instead-92d84bb2cc01)
- [FMP pricing](https://site.financialmodelingprep.com/pricing-plans)
- [PRAW rate limits docs](https://praw.readthedocs.io/en/stable/getting_started/ratelimits.html)
- [Iron Mountain Q1 2026 Supplemental Reporting Package (SEC 8-K)](https://www.sec.gov/Archives/edgar/data/0001020569/000102056926000036/q12026srp_final.htm)

**Domain-specific guidance:**
- [SEC Form 4 Insider Trading Alerts — PageCrawl.io](https://pagecrawl.io/blog/sec-form-4-insider-trading-alerts)
- [SEC Form 4 Explained — HedgeTrace](https://hedgetrace.com/learn/sec-form-4-explained)
- [Insider Trading Signals: SEC Form 4 Guide — MarketTriage](https://markettriage.com/insider-trading-signals)
- [OpenInsider scraper repo](https://github.com/sd3v/openinsiderData)
- [VADER vs FinBERT evaluation (ACM)](https://dl.acm.org/doi/fullHtml/10.1145/3677052.3698675)
- [FinBERT (ProsusAI) on Hugging Face](https://huggingface.co/ProsusAI/finbert)
- [Performance of Prophet in Stock-Price Forecasting (IEEE)](https://ieeexplore.ieee.org/document/10756299/)
- [How to Analyze Semiconductor Stocks — Investing.com](https://www.investing.com/academy/trading/how-to-analyze-semiconductor-stocks/)
- [Iron Mountain (IRM) — REIT Notes](https://www.reitnotes.com/reit/Iron-Mountain-Inc/symbol/IRM)

**Packaging & distribution:**
- [PyInstaller documentation — hooks](https://pyinstaller.org/en/stable/hooks.html)
- [Apple notarization workflow](https://developer.apple.com/documentation/security/notarizing-macos-software-before-distribution)

---

*Confidence: HIGH across all four research dimensions. The Phase 1 vertical slice and coarse build order are prescriptive and ready for the roadmapper to consume. The five critical pitfalls each have concrete phase-mapped mitigations. Open questions are flagged for user input rather than silently assumed.*

*Last updated: 2026-05-25*
