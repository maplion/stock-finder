# Stock Finder

## What This Is

A personal-use desktop stock-information aggregator. It pulls fundamentals, multi-horizon price forecasts, market sentiment, and insider-trade activity for a small set of tracked tickers — starting with **Micron (MU)** and **Iron Mountain (IRM)**, with low-friction expansion to others. The aim is one consolidated view that replaces bouncing between Yahoo Finance, SEC EDGAR, Reddit, news sites, and analyst-rating pages.

## Core Value

**Give me one place to look** that combines fundamentals + sentiment + insider activity + forecast for each ticker I follow, so I can form a better-informed personal investment thesis without manual aggregation.

## Requirements

### Validated

<!-- Shipped and confirmed valuable. -->

(None yet — ship to validate)

### Active

<!-- v1 hypotheses. Building toward these. -->

- [ ] Fundamentals view: P&L summary, cash on hand, debt load, key ratios (per ticker)
- [ ] Multi-horizon price forecasts shown side-by-side: analyst consensus, statistical baseline (e.g., Prophet/ARIMA), ML model
- [ ] Forecast horizons selectable: 1–3 month, 3–12 month, 1–3 year
- [ ] Market-sentiment panel from financial news headlines (scored per ticker)
- [ ] Market-sentiment panel from Reddit (WSB / investing / stocks) — mention volume + tone
- [ ] Price-based momentum signals (RSI, MACD, moving-average crosses, volume surges)
- [ ] Insider-trade panel: recent transactions table + cluster/pattern signals (multi-insider buying, net trend)
- [ ] Tracked-tickers list starts with MU + IRM; user can add/remove tickers without code changes
- [ ] All data fetched on-demand from free APIs / Playwright / web search; no background workers
- [ ] Runs as a Tauri desktop app (Rust shell, Angular/TS frontend, Python sidecar for ML/quant)

### Out of Scope

<!-- Explicit boundaries with reasoning. -->

- **Multi-user / hosted version** — personal use only; no auth, no deployment surface
- **Trade execution / brokerage integration** — informational tool only; no order placement, no portfolio P&L
- **Real-time intraday tick data** — on-demand refresh is sufficient; avoids streaming infra + cost
- **Twitter/X sentiment** — current API restrictions make it unreliable; defer until a stable free path exists
- **Heavy historical timeseries DB** — rely on data providers for history; only a small local cache where ML training requires it
- **Paid data vendors (Bloomberg, Refinitiv, Polygon Pro, etc.)** — free/freemium tier only

## Context

- **Domain expertise:** User makes personal investment decisions and currently aggregates data manually across multiple sites. Frustrated enough to want a single tool.
- **Tickers chosen are heterogeneous on purpose:** MU is a cyclical semiconductor name (capex cycle, inventory days, gross margin trend matter); IRM is a data-center / records-storage REIT (FFO, AFFO, dividend coverage, leverage matter). The fundamentals view must accommodate both — likely a "common metrics" layer plus sector-specific extensions.
- **Multiple forecasts shown side-by-side, not blended:** the goal is informational comparison ("what does the street say vs what does a dumb baseline say vs what does an ML model say?"), not a single recommended number.
- **Insider-trade emphasis on clusters/patterns:** isolated insider trades are noisy; multi-insider clusters and sustained net buying/selling are the higher-signal patterns the dashboard should surface.

## Constraints

- **Tech stack — frontend:** Angular + TypeScript (user preference; default for this project) — Tauri webview
- **Tech stack — shell:** Tauri (Rust) — chosen over Electron for smaller binary, lower memory footprint, native menus
- **Tech stack — data/ML:** Python sidecar invoked from Rust — Python's quant/finance ecosystem (pandas, yfinance, prophet, scikit-learn, statsmodels) is where the relevant libraries live
- **Data sources:** free / freemium APIs only (yfinance, SEC EDGAR, Reddit JSON, FMP free tier, etc.); Playwright + web search are acceptable fallbacks when no API exists
- **Persistence:** no server-side DB; small on-disk cache (e.g., SQLite or parquet files) is acceptable where ML training data or per-ticker history is needed
- **Deployment:** local-only desktop app; no networked services, no shared state, no authentication
- **Scope discipline:** start with MU + IRM only; ticker-add-ability is a v1 requirement, but onboarding more tickers is a user task, not a separate product surface

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Tauri over Electron | Smaller footprint, native feel, ships fast | — Pending |
| Angular over React for frontend | User preference; familiar stack | — Pending |
| Python sidecar pattern | ML/quant libs live in Python; keep UI in TS | — Pending |
| On-demand fetch, no background worker | Personal-use scale; no need for cron infra | — Pending |
| Forecast: show 3 models side-by-side, do not blend | User wants comparison, not a single number | — Pending |
| Insider focus on clusters/patterns | Single trades are noisy; clusters are higher signal | — Pending |
| Defer Twitter/X sentiment | API too restrictive in 2026 | — Pending |
| Free APIs first; Playwright/web search as fallback | Personal-use budget; avoid paid vendor lock-in | — Pending |

## Evolution

This document evolves at phase transitions and milestone boundaries.

**After each phase transition** (via `/gsd-transition`):
1. Requirements invalidated? → Move to Out of Scope with reason
2. Requirements validated? → Move to Validated with phase reference
3. New requirements emerged? → Add to Active
4. Decisions to log? → Add to Key Decisions
5. "What This Is" still accurate? → Update if drifted

**After each milestone** (via `/gsd:complete-milestone`):
1. Full review of all sections
2. Core Value check — still the right priority?
3. Audit Out of Scope — reasons still valid?
4. Update Context with current state

---
*Last updated: 2026-05-25 after initialization*
