# Stock Finder

## What This Is

A personal-use desktop stock-information aggregator. It pulls fundamentals, insider-trade activity, market context, and a deliberately-demoted forecast view for a small set of tracked tickers — starting with **Micron (MU)** and **Iron Mountain (IRM)**, with low-friction expansion to others. The aim is one consolidated view that replaces bouncing between Yahoo Finance, SEC EDGAR, Reddit, news sites, and analyst-rating pages, **weighted toward the panels that carry real signal** (fundamentals + insider clusters) rather than the panels that carry noise (statistical/ML price forecasts).

## Core Value

**Give me one place to look** that combines high-signal data (fundamentals + insider clusters + earnings-cycle + macro context) with explicitly-demoted lower-signal context (sentiment + statistical forecast) for each tracked ticker — so I can form a better-informed personal investment thesis, **including disconfirmation of that thesis**, without manual aggregation.

## Signal Hierarchy

Per `.planning/PROJECT-ANALYSIS.md`, panels are explicitly ranked by signal quality. The dashboard's design must reflect this — high-signal panels get prominence; low-signal panels are present but visually demoted with honesty controls.

| Tier | Panels |
|---|---|
| **High-signal (primary)** | Fundamentals (sector-aware), Insider clusters, Earnings cycle (beat/miss + guide-vs-actual), Disconfirmation (bear-signal surface) |
| **Medium-signal (contextual)** | Macro / sector context, Analyst consensus, Momentum / technicals, Earnings-call LLM summary |
| **Low-signal (demoted)** | News sentiment, Statistical price forecast (Prophet/AutoARIMA), ML price forecast (Phase E) |

Forecast and sentiment panels ship with mandatory honesty UX (backtest accuracy headline, naive baseline, "reference only — not decision input" labeling, walk-forward CV, `forecasts_log`).

## Requirements

### Validated

<!-- Shipped and confirmed valuable. -->

(None yet — ship to validate)

### Active

<!-- v1 hypotheses. Building toward these. -->

**Primary (high-signal):**
- [ ] Fundamentals view with sector-aware extension: P&L summary, cash, debt, key ratios for common stocks; FFO/AFFO/payout/leverage for REITs (IRM)
- [ ] Insider-trade panel: recent transactions table + cluster signals (code-P only, 10b5-1 filtered, M+S filtered, asymmetric buy-elevated)
- [ ] Earnings cycle: next-earnings banner + last-4-quarter beat/miss vs consensus + guide-vs-actual track record
- [ ] **Disconfirmation panel** (anti-confirmation-bias): analyst downgrades, margin compression, insider selling clusters, debt maturity walls (IRM), inventory days rising (MU)
- [ ] Tracked-tickers list starts with MU + IRM; user can add/remove tickers without code changes

**Contextual (medium-signal):**
- [ ] Macro / sector context per ticker: SOX index + DRAM proxy for MU; 10Y treasury + data-center context for IRM
- [ ] Analyst consensus (mean / high / low / # analysts / rating distribution) — labeled distinctly from model forecasts
- [ ] Momentum panel: RSI, MACD, MA crosses, volume surges (context-not-signal framing)
- [ ] Earnings-call LLM summary: latest 10-Q/10-K + transcript summarized to 5 bullets per filing (Claude API, user-provided key, graceful degradation when absent)

**Demoted (low-signal, present with honesty controls):**
- [ ] News sentiment panel from financial headlines (FinVADER + FinBERT top-N rescoring)
- [ ] Statistical price forecast panel (analyst consensus + naive baseline + Prophet) — visually demoted, backtest accuracy headline up top, "reference only" framing

**Differentiators that turn panels into a tool:**
- [ ] "What changed since last open" digest (new filings, headlines, threshold breaches)
- [ ] Forecast back-tracking UI ("model said X N days ago; actual Y; hit rate Z%") — read-side over `forecasts_log`
- [ ] Cross-ticker comparison view (sector-aware)
- [ ] Free-text thesis tracker per ticker (no NLP in v1)

**Infrastructure:**
- [ ] On-demand fetch from free APIs (yfinance → yahooquery → FMP cascade); Playwright + web search only where no API exists
- [ ] Explicit cache TTLs per data type (fundamentals 24h, insider 6h, sentiment 1h, prices on-demand, forecasts_log forever)
- [ ] Tauri 2.x desktop app (Rust shell, Angular 20 frontend, Python PyInstaller sidecar)

### Out of Scope

<!-- Explicit boundaries with reasoning. -->

- **Multi-user / hosted version** — personal use only; no auth, no deployment surface
- **Trade execution / brokerage integration** — informational tool only; no order placement, no portfolio P&L
- **Real-time intraday tick data** — on-demand refresh is sufficient
- **Twitter/X sentiment** — current API restrictions make it unreliable
- **Reddit sentiment in v1** — deferred to v2 after re-evaluation (low signal per analysis; news covers most of the need)
- **Heavy historical timeseries DB** — small local Parquet cache only where ML training / diff state needs it
- **Paid data vendors (Bloomberg, Refinitiv, Polygon Pro, FactSet)** — free/freemium tier only
- **Forecast as decision input** — forecast panel is explicitly framed as reference / context; the UX prevents treating model output as decision-grade
- **Linux build for v1** — macOS arm64 + Windows only

## Context

- **Domain background:** User makes personal investment decisions and currently aggregates manually across multiple sites. Frustrated enough to want a single tool.
- **Tickers chosen are heterogeneous on purpose:** MU is a cyclical semi (capex cycle, inventory days, gross margin trend matter); IRM is a data-center / records-storage REIT (FFO, AFFO, dividend coverage, leverage matter). Fundamentals view must accommodate both — a common-metrics layer plus sector-specific extensions.
- **Bias-aware design:** Per the analysis, personal investing fails on confirmation bias more than on data scarcity. The dashboard explicitly surfaces bear signals as a primary panel (DISCONFIRMATION), and downgrades the forecast panel rather than letting elaborate quant machinery quietly bias thesis formation.
- **Signal hierarchy is load-bearing:** The panel-priority ranking (high / medium / low signal) is not cosmetic — it drives visual prominence, panel sizing, honesty controls, and which panels' failure modes are acceptable. A failing fundamentals panel is a bug; a failing forecast panel is a feature working as designed (model couldn't beat naive).

## Constraints

- **Tech stack — frontend:** Angular 20 (standalone, signals, zoneless) + TypeScript — Tauri webview (user preference)
- **Tech stack — shell:** Tauri 2.x (Rust) — smaller binary, lower RAM than Electron
- **Tech stack — data/ML:** Python sidecar (PyInstaller one-file) invoked from Rust via JSON-over-stdio (NDJSON)
- **Data sources:** free / freemium APIs only; Playwright + web search are acceptable fallbacks
- **Persistence:** no server-side DB; small on-disk SQLite (metadata) + Parquet (OHLCV / ML frames) cache
- **Deployment:** local-only desktop app; no networked services, no auth
- **Platforms (v1):** macOS arm64 (primary) + Windows x64; Linux deferred to v2
- **LLM dependency:** Earnings-call summary requires user-provided Claude API key; absence = panel shows raw filing link with no summary (graceful degradation)
- **Cache TTLs (explicit, infrastructure-level):** fundamentals 24h, insider 6h, news sentiment 1h, prices on-demand, `forecasts_log` never pruned
- **Scope discipline:** start with MU + IRM only; ticker-add is a v1 capability, but onboarding more tickers is a user task, not a product surface

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Tauri over Electron | Smaller footprint, native feel | — Pending |
| Angular over React for frontend | User preference; familiar stack | — Pending |
| Python sidecar pattern | ML/quant libs live in Python; keep UI in TS | — Pending |
| On-demand fetch, no background worker | Personal-use scale; no need for cron infra | — Pending |
| **Panels ranked by signal quality, not feature parity** | Per PROJECT-ANALYSIS.md — confirmation bias is the failure mode to design against | — Pending |
| **Forecast panel visually demoted in v1** | Statistical forecasting on stocks is near-martingale; elaborate UI would imply false predictive power | — Pending |
| **Disconfirmation panel added as primary tier** | Designs against confirmation bias; surfaces bear signals the user would naturally avoid | — Pending |
| **Earnings-call LLM summary added** | Cheap to build via EDGAR + Claude API; very high time-saved | — Pending |
| **Macro/sector context added** | MU + IRM aren't decontextualized assets; SOX + 10Y matter | — Pending |
| Defer Reddit sentiment to v2 | Low signal per analysis; news + insider cover the need | — Pending |
| Forecast: keep all three lines but lead with backtest accuracy | Honesty UX makes the noise visible; user retains the comparison view | — Pending |
| Insider focus on clusters/patterns + asymmetric weighting | Single trades are noisy; clusters are higher signal; buying ≫ selling as signal | — Pending |
| Free APIs first; Playwright/web search as fallback | Personal-use budget | — Pending |
| Platforms: macOS arm64 + Windows; Linux deferred | User uses macOS + Windows primarily | — Pending |
| Explicit cache TTLs codified upfront | Rate limits never bite; "stale" is defined for every number | — Pending |

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
3. Signal Hierarchy check — has any panel changed tier based on observed value?
4. Audit Out of Scope — reasons still valid?
5. Update Context with current state

---
*Last updated: 2026-05-25 after PROJECT-ANALYSIS.md re-prioritization*
