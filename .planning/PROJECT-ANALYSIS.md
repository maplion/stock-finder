# Analysis of Stock Finder

## Value of the project

**Where the thesis holds up.** The "one place to look" frustration is real — aggregating fundamentals, sentiment, and insider data across 5+ sites is genuinely tedious, and for tickers you hold long-term you re-do that work every time you reconsider the position. A consolidated read-only view that you can open in 10 seconds is meaningfully more valuable than the sum of its tabs, because the friction reduction changes how often you actually check the data.

**Where I'd push back.** The dashboard's value is uneven across panels:

| Panel | Signal quality | Notes |
|---|---|---|
| Fundamentals | High | These are facts. Aggregation is the value. |
| Insider clusters | High | Academic literature supports cluster signals; this is probably your best panel. |
| Momentum (RSI/MACD/etc.) | Medium | Useful as context, not signal. |
| News sentiment | Low-medium | Noisy, often coincident with price. |
| Reddit sentiment | Low | Often contrarian/coincident; WSB is sentiment about sentiment. |
| Statistical forecast (Prophet/ARIMA) | ~Zero | Stock prices are near-martingale. These will produce numbers, not insight. |
| ML forecast | ~Zero unless feature-engineered hard | Same caveat. |
| Analyst consensus | Medium | Useful as a "what is the street pricing in" anchor. |

The risk is that **showing all forecasts side-by-side gives false equivalence** to three models, when realistically only the analyst consensus has interpretable meaning. A dashboard that displays a Prophet 12-month price point next to an analyst PT visually implies they're peers; they're not.

## Value of *this* dashboard specifically

For your stated goal — informing personal investment theses on a small set of long-hold tickers — the dashboard delivers ~70% of its value through three panels: **fundamentals (sector-aware), insider clusters, and a thesis-vs-data view** (which isn't currently in the spec). The forecast and sentiment panels are nice-to-have but at high risk of becoming attention sinks that feel quantitative without being predictive.

The **MU + IRM heterogeneity** is a double-edged choice: pedagogically smart (forces you to build sector-aware fundamentals from day one), but expensive — REIT metrics (FFO/AFFO/dividend coverage/debt ladder/WALE) and semi metrics (days inventory, gross margin, capex intensity, FCF conversion) share almost nothing. For 2 tickers, you're effectively building two dashboards.

## Recommendations

**1. Reconsider Tauri + Angular + Python sidecar vs. a simpler frame.** For personal use with 2–5 tickers, a Streamlit/Gradio Python app gets you 80% of this in 10% of the code, no IPC, no Rust. Tauri/Angular is worth it *if you also want to learn that stack*; if the goal is data-on-screen, the three-layer architecture is real maintenance overhead. Worth being explicit about which goal dominates.

**2. Add an earnings-cycle panel.** Next report date, last quarter's beat/miss vs consensus on revenue/EPS/guidance, the guide-vs-actual track record. This is high-signal, freely available (EDGAR + Yahoo), and underrated.

**3. Add a thesis-and-disconfirmation feature.** A free-text "my thesis" note per ticker, plus a panel that explicitly surfaces *bear* signals: recent downgrades, gross-margin compression, insider *selling* clusters, debt maturity walls (IRM), inventory days rising (MU). Personal investing fails on confirmation bias more than on data scarcity; design against your own bias.

**4. Add an "earnings-call summary" panel.** Pull the latest 10-Q/10-K and earnings transcript via EDGAR + LLM-summarize to 5 bullets per ticker. Cheap to build, very high time-saved.

**5. Add macro/sector context per ticker.** MU doesn't move on its own — DRAM spot, SOX, hyperscaler capex commentary matter. IRM moves on rates and data-center net absorption. Without this layer, the ticker view is decontextualized.

**6. Filter insider trades by discretionary vs 10b5-1 plan trades.** Rule 10b5-1 trades are pre-scheduled and carry near-zero signal; lumping them with discretionary buys/sells dilutes the cluster signal that's the actual reason the panel matters.

**7. Demote the statistical/ML forecast panel — or add backtest honesty to it.** If you keep it, show "this model's directional accuracy on this ticker over the last 24 months: 47%." Make the noisiness visible. Otherwise the elaborate machinery will quietly bias your thesis-formation.

**8. Add a "what changed since last visit" view.** Not a background worker — just a launch-time diff against last cached snapshot: new insider filings, sentiment shifts, price moves >X%, upcoming earnings within Y days. This compounds the dashboard's core value (reduced friction → checking more often → catching things sooner).

**9. Be explicit about cache TTLs upfront.** Fundamentals: daily. Insider Form 4: 6h. Sentiment: 1h. Price: on-demand. Codify it so rate limits never bite, and so you know what "stale" means when looking at any number.

## Bottom line

The project is worth building if you weight it as **a personal data-aggregator with a fundamentals + insider core**, with sentiment/forecast as context rather than signal. If you weight it as a **forecasting tool**, the architecture is solid but the underlying signal isn't there and you'll end up with a beautiful UI for noise. The single biggest design lever is whether you treat the forecast panel as decoration (low risk) or as decision input (high risk of self-deception).

---
*Generated 2026-05-25*
