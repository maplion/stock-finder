# Domain Pitfalls — Stock Finder

**Project:** Stock Finder (personal-use desktop stock-information aggregator)
**Researched:** 2026-05-25
**Stack context:** Tauri 2.x + Angular 20 (zoneless, signals) + Python sidecar (PyInstaller one-file, JSON/NDJSON over stdio)
**Domain context:** fundamentals + multi-horizon forecasts + sentiment + insider trades for MU (semi cyclical) + IRM (data-center REIT)

This document organises pitfalls by category. Each pitfall has: a one-line description, warning signs, prevention (referencing concrete libraries from STACK.md), and a phase mapping for the roadmap.

Phase labels used:
- **P0 — Foundation** (Tauri shell + Angular skeleton + Python sidecar wiring)
- **P1 — Data Fetch Abstraction** (rate-limiting, caching, source adapters)
- **P2 — Fundamentals** (common + REIT extension)
- **P3 — Insider Trades** (Form 4 + cluster detection)
- **P4 — Sentiment** (news + Reddit + FinVADER/FinBERT)
- **P5 — Forecasts** (analyst + statistical + ML, plus forecast log)
- **P6 — Momentum / Technicals**
- **P7 — Cross-cutting v1.1** (diff-since-last-open, comparison, quadrant)
- **P8 — Packaging & Distribution** (PyInstaller per-target, code-sign, notarize, updater)
- **Ongoing** — applies across multiple phases or for the life of the app

---

## 1. Data Quality Pitfalls

### 1.1 yfinance silently returns stale or wrong data
**Pitfall:** `yfinance` is unofficial scraping over Yahoo's undocumented endpoints. Yahoo restructures HTML/JSON roughly quarterly; `yfinance` then returns empty frames, partial frames, or — worst — *stale data with no exception thrown.*

**Warning signs:**
- `Ticker.info` returns a small dict with mostly `None` values where you expected ratios
- Historical OHLCV ends two days ago when market closed yesterday
- `Ticker.financials` has the same period repeated for "latest" across multiple weekly runs
- A `YFRateLimitError` exception in logs (this is the *good* failure mode — visible)

**Prevention:**
- Treat yfinance as **primary but never sole**. Wire the layered fallback already in STACK.md: yfinance → yahooquery → FMP (free tier) → cached previous value with `stale=true` flag exposed to UI.
- Implement a freshness check in the data-fetch adapter: every numeric panel field carries `(value, as_of_ts, source)`. UI shows the source badge and `as_of` age.
- Schema-check the response: if expected fields are `None` or arrays are empty, raise an internal `StaleDataError` rather than passing the empty dict upstream.
- Pin yfinance version in `pyproject.toml` (`yfinance>=0.2.50,<0.3`) — minor releases can break parsers.
- Run a tiny daily smoke test in CI (or on-demand on user machine) that fetches a known-stable ticker (e.g., AAPL) and asserts a handful of fields are non-null and recent.

**Phase mapping:** **P1** (build the fallback adapter and freshness contract). **Ongoing** (smoke test, version monitoring).

---

### 1.2 REIT GAAP metrics are misleading for IRM (P/E, net income, EPS)
**Pitfall:** REITs carry massive real-estate depreciation that crushes GAAP earnings. Showing IRM's P/E ratio or net income next to MU's *as if comparable* gives the user actively misleading information. The REIT-correct metrics are FFO, AFFO, P/FFO, P/AFFO, and AFFO payout ratio.

**Warning signs:**
- IRM dashboard shows a P/E above 100 (or negative) and the user goes "that's broken"
- "Net income margin" for IRM looks dramatically worse than other data-centre infrastructure plays
- No FFO/AFFO row visible on the IRM ticker view
- `yfinance.Ticker('IRM').info['trailingPE']` is being rendered without a REIT-aware override

**Prevention:**
- Implement the **sector tag** (per FEATURES.md §1.1) early — detect SIC 6798 (REITs) via EDGAR submissions endpoint at ticker-add time. IRM SIC = 6798 is the canary.
- For `sector === 'REIT'`: **suppress or de-emphasise P/E, EPS, net-income-based ratios**. Replace primary valuation slot with P/AFFO.
- Use the FFO/AFFO computation strategy from STACK.md §3b: pull XBRL `RealEstateInvestmentTrustFundsFromOperations` when filer tags it (compute when not), then derive AFFO from FFO − recurring capex − straight-line rent adjustments.
- For IRM specifically, the 8-K supplemental reporting package (e.g., `q12026srp_final.htm`, cited in FEATURES.md sources) is the authoritative source for AFFO, NOI, data-centre leased MW, occupancy. v1: pull reported FFO from XBRL via edgartools; v1.1: parse the 8-K supplemental.
- Render BOTH "Reported FFO" (when disclosed) and "Computed FFO" — the gap itself is a signal of one-time items / management adjustments.

**Phase mapping:** **P2** (REIT extension is the headline differentiator). Sector-tagging plumbing belongs in **P1** so P2 can branch on it cleanly.

---

### 1.3 SEC EDGAR XBRL concept inconsistency across filers
**Pitfall:** XBRL is standardised in theory; in practice filers tag the same economic concept with different tag names, units, or context refs. A given `us-gaap:` concept may be missing for one filer and present for another. Hand-rolled "just pull `us-gaap:Revenues`" code returns blanks for ~10–20% of tickers.

**Warning signs:**
- A field is populated for MU but null for IRM (or vice versa) with no clear reason
- Same concept comes back with different units (`USD` vs `USD/share`) across filers
- Periods don't line up (some filers use fiscal-year-end Dec, others use weird offsets)
- Hand-pulled XBRL values disagree with what the company prints in its press release

**Prevention:**
- Use `edgartools` (already in STACK.md) — its 5.x concept-mapping layer normalises across 32k+ filings. Do NOT roll your own XBRL parser as a default.
- For concepts edgartools doesn't surface, fall through to `httpx` against `https://data.sec.gov/api/xbrl/companyconcept/CIK{cik}/us-gaap/{concept}.json` with an explicit allow-list of acceptable tag aliases per concept (e.g., revenue: `Revenues`, `RevenueFromContractWithCustomerExcludingAssessedTax`, `SalesRevenueNet`).
- Always emit `(value, concept_tag_used, period_end_date, units)` so debugging "why is this number weird" is one query away.
- Sanity-check totals: revenue from XBRL should equal the sum of segment revenues from the same filing within rounding; if not, log a warning.

**Phase mapping:** **P2** (Fundamentals — both common and REIT extension).

---

### 1.4 Split-adjusted vs raw price confusion
**Pitfall:** Historical price arrays from yfinance default to *split-and-dividend-adjusted* close prices. If the forecast model trains on adjusted prices but the UI displays raw prices (or vice versa), the model's outputs and the chart's "current price" can disagree by 10–30% on tickers with a recent split or large dividend.

**Warning signs:**
- Forecast point estimate is way off from the latest displayed price, with no model error
- Historical chart shows a smooth line where everyone knows a split happened
- Dividend yield calculation gives weird numbers (using adjusted close in the denominator)
- Returns calculated from adjusted prices differ materially from broker statements

**Prevention:**
- Adopt a single project convention: **store adjusted close in the price cache, store raw close in a separate column, and label every API/UI surface with which it is using.**
- For forecast models: train on *adjusted* close (continuous return series, no artificial split gaps). For "current price" badges and P/E denominators: use *raw* close.
- yfinance: pass `auto_adjust=False` and keep `Close` + `Adj Close` both. Document this in the data-fetch adapter docstring.
- Display price-axis chart with adjusted close but ANNOTATE split events as vertical lines so the user understands what they're looking at.

**Phase mapping:** **P1** (data-fetch contract — the schema decision lives here). **P5** (forecasts) and **P6** (momentum) must consume the contract correctly.

---

### 1.5 SEC EDGAR 10 req/s ban + missing User-Agent
**Pitfall:** SEC EDGAR enforces a hard 10 req/s cap and *requires* a descriptive User-Agent (format: `"Company contact@example.com"`). Exceeding the cap, or sending a generic UA, gets your IP rate-limited or temporarily banned. This is easy to trigger when fanning out a panel fetch across many concepts × filings.

**Warning signs:**
- HTTP 403 with body "Request blocked" or "User-Agent required"
- Sudden cascade of timeouts to `data.sec.gov` after a flurry of requests
- IP-level blocks that persist for ~10 minutes
- edgartools throwing rate-limit exceptions during a multi-filing pull

**Prevention:**
- Set `httpx` headers globally in the EDGAR adapter:
  `User-Agent: "StockFinder ryandammrose@onepointtwocapital.com"` (matches STACK.md §3a — this is the user's email from environment).
- Use `aiolimiter` (already in deps) with a **token-bucket capped at 8 req/s** (below the 10/s ceiling for safety margin).
- Wrap calls in `tenacity` retry with exponential backoff + jitter; cap at 3 retries; back off >60s if a 429/403 fires.
- Serialise EDGAR work into a single async queue (don't let separate panel adapters each spawn their own rate-limiter).
- Log every EDGAR call with status code so we can spot the slide into block territory before it gets bad.

**Phase mapping:** **P1** (this is foundational data-fetch infrastructure; everything else assumes it).

---

## 2. Forecast Honesty Pitfalls

### 2.1 ML stock forecasts overfit and beat naive baselines rarely
**Pitfall:** Almost any ML model trained on price-only features will look great in-sample, then underperform a "tomorrow = today" naive baseline out-of-sample. Showing a single forecast number implies false predictive power and can drive bad decisions.

**Warning signs:**
- Backtest accuracy report missing or never surfaced to the UI
- Forecast presented as a single bold number with no confidence interval
- ML model "always seems to know" the direction in the training window
- No persistent forecast log to compare past predictions to actual outcomes

**Prevention:**
- **Never** show a point forecast without a confidence interval. Statsforecast and prophet emit 80% CI by default — pass through to UI.
- Implement `forecasts_log` table (per FEATURES.md §6) on day one of the forecast panel. Every generated forecast is persisted with `(ticker, model, horizon, generated_at, point, ci_lo, ci_hi)`.
- Surface **forecast-vs-actual log** prominently in the UI (FEATURES.md §2.3 — "Model said $X 90 days ago; actual is $Y. Hit rate: Z% in CI."). This is the antidote to magical thinking.
- Compute a **naive baseline** (last-value persistence model) and show it alongside the three "real" models. If the ML model can't beat naive over rolling 90-day windows on the user's actual tickers, flag the model row with a "below naive" warning badge.
- Each forecast card includes: model name, training window (e.g., "5y daily"), last-updated timestamp, 80% CI, naive-baseline comparison, **and a "Not financial advice" footer.**

**Phase mapping:** **P5** (forecasts) — forecast log table and naive baseline must ship with the first forecast panel, not be tacked on later.

---

### 2.2 Look-ahead bias when training
**Pitfall:** Training features must only contain information that was *actually knowable* at time T. The most common bias: using **revised** fundamentals (10-K/Q after later restatements) as if they were known at the original report date. Models look brilliant in backtest, fail live.

**Warning signs:**
- Backtest accuracy is suspiciously high (>60% directional on stock prices is a red flag)
- Model uses fundamentals or analyst estimates as features without an "as-of" timestamp
- Training set includes the same row's future-target alongside features without lag
- Walk-forward CV not used; flat train/test split or shuffled CV used instead

**Prevention:**
- Always use **walk-forward (expanding-window) cross-validation** for time-series. Statsforecast and darts both support this natively — use `cross_validation()` not `train_test_split()`.
- For fundamentals features: store *first-reported* values with their filing date. When building a training row for time T, only include fundamentals with `filing_date <= T`.
- For analyst targets and sentiment: store with `as_of_ts`; never join on "latest" — always join with `<= T`.
- Lag every feature by at least one trading day before joining onto the target. A feature representing today's MACD value can only predict tomorrow's price, not today's.
- Document the feature-time invariant in code comments AND in the FEATURE.md for the ML phase.

**Phase mapping:** **P5** (forecasts — applies to statistical and ML models alike; also affects forecast back-tracking accuracy).

---

### 2.3 Conflating analyst price targets with statistical/ML forecasts
**Pitfall:** A "12-month analyst consensus price target" is partly a sell-side advocacy artifact (banker relationships, IPO underwriting, coverage initiation conventions) — it is NOT commensurable with a statistical or ML forecast which is just a model output. Showing them in the same row/format implies false equivalence.

**Warning signs:**
- UI puts analyst target, Prophet forecast, and ML forecast in a single uniform table row with the same styling
- Implied % vs current price displayed identically for all three sources
- No tooltip distinguishing "what the street says" from "what the model predicts"
- Analyst-rating distribution (5 buys / 10 holds / 2 sells) not shown alongside the target — losing the dispersion signal

**Prevention:**
- Visually separate analyst targets from model forecasts: distinct section header ("**Street Consensus**" vs "**Model Forecasts**"), distinct styling/color band.
- For analyst consensus: show **mean target, high target, low target, # of analysts, rating distribution** (per FEATURES.md §1.5). The dispersion is more honest than the mean.
- Each forecast card has an explicit "what this is" subtitle:
  - Analyst: "Aggregated price targets from sell-side analysts. May reflect institutional relationships."
  - Statistical (statsforecast/prophet): "Trend + seasonality projection from historical price only."
  - ML (darts NHiTS): "Neural network forecast from price + engineered features. Treat with skepticism."
- Never combine the three into a single "blended" number. Per PROJECT.md key decisions: "show 3 models side-by-side, do not blend."

**Phase mapping:** **P5** (forecasts panel UI).

---

### 2.4 1-3 year horizon forecasts are essentially noise
**Pitfall:** Daily-price forecasts at 1-3 year horizons have confidence intervals so wide they cover ~any plausible outcome. Showing a tight-looking 3-year forecast line creates false precision.

**Warning signs:**
- 1-3 year forecast lines look as smooth and decisive as the 1-3 month lines
- CI band visually identical at all three horizons (CI should widen with horizon)
- User asks "the model thinks IRM will be $X in three years?" with conviction
- No explicit disclaimer on long-horizon forecasts

**Prevention:**
- Resample to **weekly for 3-12 month** and **monthly for 1-3 year** horizons (already in STACK.md §3e). Re-fit per horizon rather than rolling forward a daily forecast.
- The longer-horizon forecasts MUST visually show widening CI — never clip the CI band for visual cleanliness.
- Add a "long-horizon disclaimer" badge on the 1-3yr forecast card: "Multi-year price forecasts are statistically poor on individual equities. Use as a sanity check, not a target."
- Consider hiding the ML model entirely from 1-3 year horizons in v1 — it's the most likely to project false confidence at long horizons.

**Phase mapping:** **P5** (forecasts).

---

## 3. Sentiment Pitfalls

### 3.1 Plain VADER on financial text is unreliable
**Pitfall:** VADER's lexicon was tuned on social-media text. Finance-specific vocabulary — "beat", "miss", "guidance raised", "in-line", "below consensus", "downgrade", "outperform" — is not handled correctly. Published evaluations cite ~44% F1 on financial text.

**Warning signs:**
- "MU misses Q2 EPS estimate" scored as neutral or weakly negative
- "AAPL beats and raises" scored mildly positive instead of strongly positive
- Aggregated sentiment doesn't track earnings reactions
- Top headlines list shows obvious mis-classifications

**Prevention:**
- Use the two-tier scoring already in STACK.md §3d: **FinVADER** (fast lexicon, finance-tuned) on every headline + **FinBERT** (ProsusAI/finbert) on top-N most-recent/most-upvoted per ticker per refresh.
- Spot-check: build a small fixture of 50 hand-labelled financial headlines, assert FinVADER+FinBERT both clear ~70%+ accuracy. Re-run after dep upgrades.
- Don't fall back to plain VADER even temporarily ("just to get started") — the wrong-sentiment data will pollute the user's mental model of the panel quality.

**Phase mapping:** **P4** (sentiment).

---

### 3.2 Reddit mention-volume spikes LAG price moves, not lead them
**Pitfall:** Retail buzz is mostly a *reaction* to price action, not a predictor. A WSB mention spike on a ticker that already ran +30% is the meme catching up, not a forecast signal. Naively interpreting "volume spike → bullish" gets causality backwards.

**Warning signs:**
- Reddit panel shows huge mention spike right after a known price catalyst
- "Mention volume vs price" chart shows volume tracking price by 0-2 days lag
- User starts saying "WSB likes MU, so I should buy" without checking if MU already ran
- Mention spikes correlate near-zero with forward 30-day returns in backtest

**Prevention:**
- Always display Reddit **mention volume alongside the trailing price chart** so the user can SEE the lag.
- Compute and display "**mention spike vs prior-30d-baseline**" — but caption it: *"Reddit buzz typically follows price moves; treat as confirmation, not prediction."*
- In the cluster/quadrant feature (FEATURES.md §2.4), make the sentiment axis a *change* in sentiment (delta vs trailing baseline), not a level. Changes are slightly more leading than levels.
- Surface per-subreddit breakdown (FEATURES.md §1.7): r/SecurityAnalysis or r/investing posts have higher signal than WSB hype.

**Phase mapping:** **P4** (sentiment — Reddit specifically) and **P7** (sentiment × momentum quadrant).

---

### 3.3 Reddit promo/bot accounts skew sentiment for popular tickers
**Pitfall:** Popular tickers attract pump-and-dump bot accounts, low-karma astroturfers, and crypto-style promo accounts. Their volume can dominate the raw sentiment signal.

**Warning signs:**
- Same talking points repeated across multiple posts from low-karma accounts
- Sentiment spikes positive while professional analysis (r/SecurityAnalysis, r/investing) stays neutral
- Account age distribution skews very young (<30 days) for ticker-specific posts
- Mention volume on a small-cap ticker dwarfs comparable mid-caps

**Prevention:**
- PRAW filter: drop posts where `redditor.comment_karma + redditor.link_karma < 100` or `redditor.created_utc` within 30 days. Document the threshold (these are knobs).
- Weight posts by `score` (upvotes) when computing aggregated tone — Reddit's own voting partially counter-weights spam.
- Surface "Filter excluded N low-karma posts in this window" in the UI so the user knows the cleanup is happening.
- Per-subreddit breakdown (already in FEATURES.md §1.7) — r/SecurityAnalysis has stronger moderation and lower bot density than WSB.

**Phase mapping:** **P4** (sentiment — Reddit ingestion).

---

### 3.4 News headline tone is publisher-biased
**Pitfall:** A bullish-leaning publisher will phrase the same news event positively; a bearish-leaning publisher negatively. A sentiment panel weighted heavily toward one publisher reflects that publisher's house style, not the market.

**Warning signs:**
- All top headlines come from 1-2 sources (e.g., Seeking Alpha contributor articles)
- Sentiment drifts whenever a particular publisher releases a piece
- "Top sources" chart shows a flat skew toward one or two outlets

**Prevention:**
- Multi-source ingestion is already in STACK.md §3d: yfinance news + Finviz + RSS (Reuters, Bloomberg public, WSJ Markets) + GDELT DOC.
- Compute **per-source aggregate** as well as overall — surface a small "by source" table so the user can spot a publisher's house tone.
- Cap any single source's contribution to overall aggregated sentiment at e.g. 30% of weighted volume.
- Prefer GDELT DOC's tone score (objective, computed by GDELT) as one of multiple inputs — not as the sole one (every source has biases, but they don't share them).

**Phase mapping:** **P4** (sentiment — news).

---

### 3.5 Headline-only sentiment misses body tone (but body-fetch is expensive)
**Pitfall:** Headlines are written for engagement; bodies often carry the nuanced tone. A "MU misses Q2 estimates" headline is negative; the body might read "...but raised forward guidance and beat on margins" (net positive). However, fetching every body multiplies request volume and Playwright cost.

**Warning signs:**
- Specific events (earnings, guidance, M&A) score directionally wrong vs market reaction
- The user reads an article linked in the panel and disagrees with its score
- Big news days have sentiment scores that don't match next-day price action

**Prevention:**
- v1: headline-only is acceptable. Document the limitation in the panel tooltip.
- For high-signal events (detected via keywords: "earnings", "guidance", "merger", "acquires", "downgrade", "upgrade"): **fetch and score the body** as a v1.1 enhancement. Cap to top-3 per day per ticker to bound cost.
- When body sentiment is fetched, show BOTH the headline and body score in the panel — divergence is interesting data ("headline-vs-body delta").
- Cache body sentiment per URL forever (article content doesn't change) — `headlines` SQLite table can store both scores.

**Phase mapping:** **P4** for headline-only baseline. **P7** for body-fetch on high-signal events.

---

## 4. Insider Trades Pitfalls (Form 4 signal/noise)

### 4.1 Option exercises + immediate sale (M then S) are mechanical, not informative
**Pitfall:** Transaction code **M** (option exercise) often appears the same day as code **S** (sale). This is mechanical hedge / liquidity behaviour — executive exercises a vested option and sells to cover taxes. It tells you nothing about conviction but appears as "insider selling $X million" in naive aggregates.

**Warning signs:**
- Large insider "sale" appears in the table on the same date as an M-coded transaction by the same insider
- Net insider flow trends bearish around vesting dates (often quarterly cliffs)
- The same executive appears with paired M/S transactions every quarter like clockwork

**Prevention:**
- In the Form 4 parsing layer: when an insider has both an M and an S transaction on the same date (±1 trading day), **tag the S transaction as "exercise-sale"** and exclude from the "open-market sale" signal aggregate.
- Surface in a separate "mechanical transactions" sub-table behind an "Include exercise/cover sales" toggle.
- Default UI hides these from cluster sell counts entirely.
- Document the heuristic in the insider adapter docstring (with a link to SEC Form 4 instructions for transaction codes).

**Phase mapping:** **P3** (insider trades).

---

### 4.2 10b5-1 plan trades are pre-scheduled — not signal
**Pitfall:** Trades made under a 10b5-1 plan are pre-scheduled (adopted weeks or months in advance to provide an affirmative defence against insider-trading allegations). They are NOT a real-time conviction signal. Form 4 has an explicit "10b5-1" checkbox.

**Warning signs:**
- Insider transactions on suspiciously regular cadence (every 15th of the month, etc.)
- Form 4 XML has `<rule10b5-1Indicator>1</rule10b5-1Indicator>` or footnote indicating a plan
- Cluster sell signal triggers on a date that turns out to be a known 10b5-1 cadence date

**Prevention:**
- edgartools `Form 4` typed objects expose the 10b5-1 flag. Filter at ingestion: tag each transaction with `is_10b5_1: bool`.
- Default: **exclude 10b5-1 transactions** from cluster signals and net-flow aggregates.
- Show them in the full transactions table with a dedicated "10b5-1" badge column.
- Offer an "Include planned trades" toggle for users who want the raw view.
- This is non-negotiable for the panel to have any predictive value — cluster signals on 10b5-1 sells are pure false positives.

**Phase mapping:** **P3** (insider trades).

---

### 4.3 Form 4 filing delay (up to 2 business days)
**Pitfall:** Form 4 must be filed within 2 business days of the transaction. Data shown is therefore up to 2 days lagged at best (often more — companies frequently file at the deadline).

**Warning signs:**
- Insider panel shows "no recent activity" right after a known earnings move
- Trades suddenly appear days after a price catalyst with no warning in the panel
- The "filing date" and "transaction date" columns often differ by 1-2 days

**Prevention:**
- Always display BOTH `transaction_date` and `filing_date` (per FEATURES.md §1.9 schema).
- Order the table by `transaction_date DESC` not `filing_date DESC` — the user cares about when the trade happened.
- Add a panel-level caption: "Form 4 filings are required within 2 business days; expect up to 2-day lag from actual transaction."
- For cluster detection windows: define the window on `transaction_date`, not `filing_date`, so late-filed trades fall into the correct cluster window when they arrive.

**Phase mapping:** **P3** (insider trades).

---

### 4.4 Insider SELLING signal is weak/asymmetric (BUYING is the real signal)
**Pitfall:** Academic literature is unambiguous: insider **buying** carries positive predictive signal (when an exec spends personal cash on shares, they presumably know something). Insider **selling** is mostly noise — diversification, taxes, kid's college, lifestyle, divorce, charitable giving. Showing aggregate "insider sentiment" as `buys − sells` falsely implies symmetry.

**Warning signs:**
- Dashboard shows "net insider flow negative, bearish signal" based purely on dollar-value-weighted sales
- UI weights buy and sell signals identically (same color saturation, same font weight)
- User makes a bearish trade decision based on a single executive's diversification sale

**Prevention:**
- In the UI: **elevate cluster BUY signals** with prominent positive styling (green badge, top of panel). Display cluster SELL signals more conservatively with a "directional context, not actionable signal" caption.
- Compute and display two separate metrics, not one net flow:
  - "Open-market buys (P): N insiders, $X total, last 90d"
  - "Open-market sales (S, ex-mechanical): N insiders, $X total, last 90d"
- The cluster-buy default (per FEATURES.md §1.9: ≥3 unique insiders, ≥$25k each, within 10 days, all code P) is the academically-grounded canonical signal. Match that bar exactly.
- Reserve the cluster-SELL flag for unambiguous cases: ≥4 insiders, very large $ values, no associated 10b5-1 or M-paired explanation — and even then, label it "investigate", not "bearish".

**Phase mapping:** **P3** (insider trades — cluster detection logic + UI emphasis).

---

### 4.5 Small-cap noise — single trades are not signal
**Pitfall:** A single insider transaction on a small/mid-cap can look dramatic but is just one person's portfolio decision. The cluster threshold exists precisely to suppress this noise.

**Warning signs:**
- Insider panel "flags" a signal on every refresh because the threshold is too loose
- A single executive's transaction dominates the visual layout
- Cluster signal fires on tickers with thin Form 4 history (no baseline of normal activity)

**Prevention:**
- Hold to the FEATURES.md §1.9 default: cluster buy = **≥3 unique insiders** in **≤10 days** with **each transaction ≥$25k**, all code **P** (after the M/S and 10b5-1 filters).
- For tickers with very few historical Form 4s (`count < 10 in last 12 months`), suppress cluster flags entirely and show only the table — there's not enough baseline for "cluster" to mean anything.
- openinsider.com (per STACK.md §3c) already publishes precomputed cluster screens; cross-check our cluster detection against their `/screener?...&cluster=1` results on the same window. Disagreement = bug in our detection.

**Phase mapping:** **P3** (insider trades — cluster detection thresholds).

---

### 4.6 Grants (code A) vs purchases (code P) — never conflate
**Pitfall:** Code **A** is a stock grant (compensation, no cash from insider). Code **P** is an open-market purchase (insider spent personal money). Conflating these is the most common rookie mistake in insider-trade dashboards. A "buy" that's actually a grant tells you nothing.

**Warning signs:**
- "Insider buying $X million" headline number that turns out to be mostly A-coded grants
- Cluster buy signal triggers around grant-vesting dates
- "Insider conviction" metric correlates with quarterly grant cadence rather than market events
- Code column missing or hidden in transaction table

**Prevention:**
- **Mandatory column** in the transaction table: `Transaction Code` with full code legend tooltip (P, S, A, M, F, G, etc. — at minimum the seven most common).
- Default filter: cluster-buy detection looks **only at code P**. Cluster-sell looks only at code S (after mechanical and 10b5-1 filters).
- The full transactions table defaults to P+S only; toggle to show all codes.
- Aggregated "net insider $ flow" sums **only P and S** (not A, M, F, G).
- Surface a small "transaction-type breakdown" sub-panel: P / S / A / M counts in trailing 12 months, so the user can see how much activity is grants/mechanical vs voluntary.

**Phase mapping:** **P3** (insider trades — table + cluster detection).

---

## 5. Tauri + Angular + Python Sidecar Pitfalls

### 5.1 Bundling Python with Tauri is painful cross-platform
**Pitfall:** PyInstaller one-file binaries are slow to cold-start (extract to temp, then exec), can be flagged by antivirus heuristics ("self-extracting binary, large, hits network"), and on macOS require code-signing + notarization for Gatekeeper to permit execution. On Windows, large unsigned exes trigger SmartScreen.

**Warning signs:**
- Antivirus quarantines `stockfinder-py-x86_64-pc-windows-msvc.exe` post-install
- macOS Gatekeeper refuses to run the sidecar with "cannot be verified"
- First-run sidecar invocation takes 5-10s while the user sees a frozen UI
- CI bundle works on the dev machine but fails on a clean test machine

**Prevention:**
- Per STACK.md §5: build PyInstaller binaries **per target triple** (`aarch64-apple-darwin`, `x86_64-apple-darwin`, `x86_64-pc-windows-msvc`, `x86_64-unknown-linux-gnu`) in a CI matrix (GH Actions: `macos-13`, `macos-14`, `windows-latest`, `ubuntu-latest`).
- macOS: **codesign** with Developer ID Application cert, **notarize** via `notarytool`, **staple** the ticket onto the bundle. Apply `--options runtime` (hardened runtime) entitlements.
- Windows: code-sign the sidecar exe (and the Tauri installer) with an EV or OV certificate. SmartScreen reputation builds over time post-signing.
- Linux: AppImage or .deb; less ceremony.
- Test the bundled binary on a fresh VM / fresh user account, not just the dev machine where Python is already installed. PyInstaller "works on my machine" is the canonical failure mode.

**Phase mapping:** **P8** (packaging & distribution) — but a "thin sidecar that exits cleanly" smoke test should ship in **P0** to verify the bundling story works end-to-end before building real features.

---

### 5.2 PyInstaller dependency hell (hidden imports for FinBERT/torch/yfinance)
**Pitfall:** PyInstaller analyses imports statically. Libraries that dynamically import (sklearn's typed dispatch, transformers' model loading, torch's backend selection) often have missing modules in the bundle. Error appears only at runtime, often deep in a callstack from an end-user log file, hard to reproduce.

**Warning signs:**
- ImportError or ModuleNotFoundError in production logs (never in dev)
- Sidecar runs fine in `python python/main.py` but fails when invoked via the PyInstaller binary
- "No module named 'sklearn.utils._typedefs'" or similar in user-reported errors
- Some endpoints work, others crash with import errors

**Prevention:**
- Maintain a `hooks/` directory of PyInstaller `--hidden-import` and `--collect-data` flags. Per STACK.md §5: at minimum `--hidden-import sklearn.utils._typedefs`, `--collect-data finvader`, `--collect-data prophet`.
- For transformers/FinBERT: include `--collect-all transformers` and `--collect-all torch` in the build, then prune via PyInstaller's `excludes` for known unneeded subpackages to keep size down.
- Build the bundle in CI, then run a **post-build smoke test** that invokes every supported `op` in the JSON protocol against a synthetic ticker. Fail CI on any ImportError.
- Ship `tauri-plugin-log` (already in STACK.md §1) and pipe sidecar stderr through it — end-user error reports will then include the actual ImportError, not just "sidecar exited with code 1".

**Phase mapping:** **P0** (initial smoke test) + **P8** (full bundle harness as new libraries are added in P2-P5).

---

### 5.3 Sidecar binary size — FinBERT + torch + statsmodels + darts bloat
**Pitfall:** A naive bundle ships at 400-600 MB (per STACK.md §5 budget). FinBERT model weights alone are ~210 MB. This makes installers slow to download, painful to code-sign (large notarization upload), and antivirus-suspicious.

**Warning signs:**
- Installer over 500 MB
- Notarization takes 10+ minutes
- Users complain about download size before they even use the app
- Disk usage post-install crosses 1 GB

**Prevention:**
- **Lazy-download FinBERT on first sentiment-panel use** (STACK.md §5 explicitly suggests this option). Cache to `app_cache_dir()`. Show a one-time download progress dialog.
- Lazy-import heavy modules inside the Python sidecar: `import torch` only when the FinBERT op is invoked, not at module load. Cold start drops dramatically when most operations don't touch the ML stack.
- Use the CPU-only `torch` wheel — much smaller than the CUDA-enabled default.
- Consider ONNX export for FinBERT (open question #3 in STACK.md) — onnxruntime + ONNX model is significantly smaller than torch + checkpoint.
- Prune unused libraries: if `darts` NHiTS is v1.1 and statsforecast is the v1 model, exclude darts from the v1 bundle entirely.

**Phase mapping:** **P4** (sentiment — FinBERT lazy load) and **P8** (packaging — size budget enforcement).

---

### 5.4 Tauri IPC payload size — large arrays over IPC are slow
**Pitfall:** Tauri IPC serialises payloads as JSON (or via the binary IPC channel). Passing a full daily-OHLCV array (5 years × 252 days × 6 columns = ~7500 numbers per ticker per resolution) over IPC every refresh is slow and pegs the main thread.

**Warning signs:**
- UI hangs/jank when switching tickers
- Chart re-render takes >500ms after data arrives
- Memory profiler shows large transient JSON allocations in the webview
- Profiling shows IPC serialisation dominating refresh time

**Prevention:**
- **Downsample server-side** for chart display: chart only needs ~500-1000 points to render cleanly. Send weekly resampled OHLCV for the 5y chart; daily only for the trailing 90 days.
- For tables: paginate at the Python sidecar boundary (request `{op: "transactions", limit: 100, offset: 0}` not "give me everything").
- For the price cache: store full daily in parquet (Python-side), but the sidecar only ships the requested view across IPC.
- Use NDJSON streaming (STACK.md §4) for ops that return large lists — UI can start rendering before the full payload arrives.

**Phase mapping:** **P1** (data-fetch contract — define the "view" vs "store" boundary). **P5/P6** (charts) consume it.

---

### 5.5 Angular change detection with zoneless + signals — stale UI from non-signal updates
**Pitfall:** Zoneless Angular relies entirely on signals/computed/effect for reactivity. Forgetting to wrap a mutable piece of state in a signal — or mutating an object in place — leaves the view stale with no warning.

**Warning signs:**
- A button click triggers data but the UI doesn't update until another interaction
- `console.log` inside a component shows new values but template doesn't re-render
- Calling `array.push(...)` instead of `signal.set([...array, newItem])`
- Service methods mutate `this.state` directly without notifying

**Prevention:**
- Project-wide convention: **all reactive state in services is `signal()` or `computed()`**, period. No bare mutable properties on `@Injectable` services exposed to components.
- Mutation pattern: `mySignal.update(prev => [...prev, newItem])` — never `mySignal().push(newItem)`.
- Enable `provideExperimentalZonelessChangeDetection()` from day one (P0) so any stale-UI bug surfaces immediately rather than being masked by Zone.js's "re-render on every async".
- Lint rule: forbid `Promise.then` / `setTimeout` callbacks that mutate signals without `.set()` / `.update()` — easy to write a custom ESLint rule, or rely on code review.
- Use `httpResource()` (Angular 20) for HTTP-driven data — it's signals-native and removes a class of footguns.

**Phase mapping:** **P0** (Angular conventions established at scaffolding). **Ongoing** (linting + code review).

---

### 5.6 Sidecar process leaks — children outliving the app
**Pitfall:** If Tauri doesn't kill spawned Python children on app quit, orphan processes accumulate. Particularly bad on macOS (process shows up in Activity Monitor) and during dev iteration (`cargo tauri dev` restarts every save).

**Warning signs:**
- `ps aux | grep stockfinder-py` shows multiple processes after closing the app
- Activity Monitor shows growing Python process count over a dev session
- CPU/memory creep over time during dev
- "Address already in use" errors if you ever moved to a long-lived sidecar pattern

**Prevention:**
- The chosen pattern (per-request subprocess via `tauri-plugin-shell`) is inherently leak-resistant — each process exits after responding. Don't drift to a long-lived sidecar without solving lifecycle first.
- Register a Tauri `on_window_event(CloseRequested)` handler that explicitly kills any tracked sidecar PIDs before allowing the window to close.
- During dev: add a `killall stockfinder-py` to the npm `predev` script as belt-and-braces.
- If you ever move to a persistent sidecar (warm-start optimisation), use Rust's `Child` with explicit `.kill()` on Drop; never rely on the OS to reap.

**Phase mapping:** **P0** (foundation — process lifecycle is part of the wiring). **Ongoing**.

---

### 5.7 PyInstaller cold start (1-3s per invocation)
**Pitfall:** PyInstaller one-file binaries extract to a temp directory on each launch — measurable 1-3 second cold start before any user code runs. Multiplied across panel-refresh fan-out, this is visible UI latency.

**Warning signs:**
- "Refresh all panels" takes >10s wall time even when data is cached
- First-request-after-app-launch is much slower than subsequent ones (it isn't — every request is a cold start in onefile mode)
- Profiling shows most time spent outside the actual Python code
- User reports "the app feels sluggish"

**Prevention:**
- v1: accept it (per STACK.md §4 — "right default" for on-demand personal use).
- **Batch panel work into a single sidecar call per refresh.** Instead of 6 separate calls (`fetch_fundamentals`, `fetch_forecast`, `fetch_sentiment`, etc.), accept `{op: "refresh_ticker", ticker: "MU"}` that runs all of them in one process. Single cold start, parallel I/O inside.
- If cold start measurably hurts UX after v1: explore PyInstaller `--onedir` mode (extracts at install time, much faster subsequent launches; larger install footprint).
- Last resort (STACK.md §4): switch to a long-lived FastAPI sidecar — but only if profiling shows cold start is the dominant cost.

**Phase mapping:** **P1** (data-fetch boundary — batch-vs-split decision lives here). **P8** (consider onedir if performance demands).

---

### 5.8 macOS Gatekeeper / hardened runtime — shipped Python binary needs entitlements
**Pitfall:** With hardened runtime enabled (required for notarization), the Python sidecar may need explicit entitlements for behaviours it relies on: JIT (some torch ops), unsigned-executable-memory, network access, file access outside sandbox.

**Warning signs:**
- App passes notarization but the sidecar fails to launch on a clean Mac with "killed: 9"
- Console.app shows `kernel: EXC_BAD_ACCESS` or `code-signing` violations for the sidecar
- FinBERT inference crashes immediately on user machine; works fine in dev

**Prevention:**
- Use a minimal-but-sufficient entitlements file for the sidecar:
  - `com.apple.security.cs.allow-jit` (if any JIT code paths)
  - `com.apple.security.cs.allow-unsigned-executable-memory` (torch sometimes needs this)
  - `com.apple.security.cs.disable-library-validation` (PyInstaller dynamically loads .dylibs)
  - `com.apple.security.network.client` (yes, the sidecar makes outbound HTTP)
- Sign the sidecar separately first, THEN sign the Tauri app bundle that embeds it (nested signing).
- Test on a clean macOS user account or VM before each release — the dev machine has all signing certs trusted, hiding failures users will see.

**Phase mapping:** **P8** (packaging — macOS-specific code-sign/notarize/entitlements).

---

## 6. API / Scraping Pitfalls

### 6.1 yfinance breaks roughly quarterly when Yahoo changes
**Pitfall:** yfinance is unofficial — every Yahoo Finance HTML/JSON redesign breaks it. Historically this happens every 2-4 months. Without fallback, panels go dark.

**Warning signs:**
- `Ticker.info` returns mostly empty fields where they previously had values
- New `YFRateLimitError` or generic parser exceptions appearing in logs
- GitHub `ranaroussi/yfinance` issue tracker showing a fresh wave of "stopped working" reports
- Same call works against AAPL but fails against newer tickers

**Prevention:**
- The layered fallback in STACK.md §3a is non-negotiable: **yfinance → yahooquery → FMP**. Implement the cascade as a single `PriceQuoteAdapter` interface so adding a new source later is a one-method change.
- Pin `yfinance` version; subscribe to its release notes (or watch GitHub releases). Run `pip-audit` weekly.
- Each adapter response carries a `source: "yfinance" | "yahooquery" | "fmp" | "cache"` field — surface it in the UI as a small badge so when yfinance breaks the user can SEE the app is on a fallback.
- Cache aggressively (TTLs in STACK.md §6). When all live sources fail, return cached data with `stale=true` rather than dropping the panel.

**Phase mapping:** **P1** (data-fetch adapter pattern). **Ongoing** (monitoring + version pinning).

---

### 6.2 FMP free tier — 250 req/day is easy to blow through
**Pitfall:** FMP's free tier caps at 250 requests/day total across all endpoints. A naive full-panel fetch across 5 tickers × 6-8 fundamentals endpoints = 30-40 req per refresh. Refresh 7-8 times in a day → daily cap blown.

**Warning signs:**
- FMP responses suddenly return 429 / "Limit Reach" mid-day
- Refresh works fine in the morning, fails in the afternoon
- Error logs show "Free plan limit exceeded"
- No cache-hit metric to confirm caching is actually working

**Prevention:**
- Cache TTLs from STACK.md §6 must be enforced strictly: fundamentals cached until next 10-Q/K or 24h; analyst targets cached 24h.
- Track FMP usage server-side (Python): maintain a `fmp_usage` row in SQLite keyed by date, increment on each call. When count > 200, **stop calling FMP** for the day and rely on cached + yfinance.
- Surface FMP usage in a small "Diagnostics" view so the user can see "FMP: 187 / 250 used today".
- Use FMP only as fallback / sanity-check, never primary (already in STACK.md §3a).

**Phase mapping:** **P1** (data-fetch — rate-limit accounting). **Ongoing**.

---

### 6.3 Reddit — anonymous JSON is rate-limited; use PRAW with OAuth
**Pitfall:** Reddit increasingly blocks anonymous `.json` endpoint scraping. The "free" path is PRAW with OAuth and a registered Reddit dev app — still free for personal use, but requires the user to register an app on reddit.com/prefs/apps and provide client_id + client_secret + user_agent.

**Warning signs:**
- 429 / 403 responses from `https://reddit.com/r/.../search.json`
- Posts return for 5 minutes then stop
- IP-level blocks after a few minutes of querying
- "old.reddit.com" workarounds breaking

**Prevention:**
- Use PRAW 7.7+ (STACK.md §3d) — handles OAuth, respects `X-Ratelimit-*` headers automatically, 60 req/min quota.
- First-run UX: prompt the user to register a Reddit personal-use app, paste `client_id`, `client_secret`, `user_agent`. Store securely (Tauri `plugin-store` or OS keychain).
- Open question #4 in STACK.md is unresolved: "Reddit OAuth account setup UX — first-run flow to register the user's own Reddit app credentials, or ship a shared client_id?" — recommend per-user app registration to avoid shared-quota issues and Reddit ToS friction.
- Cache Reddit responses for 30 min (STACK.md §6 TTLs) to stay well within the 60 req/min budget across all watchlist tickers.

**Phase mapping:** **P4** (sentiment — Reddit ingestion). First-run UX touches **P0/P7**.

---

### 6.4 Playwright scraping is fragile — openinsider, Yahoo, news sites redesign
**Pitfall:** Any scraper based on CSS selectors or DOM structure breaks when the source redesigns. openinsider, Finviz, Yahoo Finance, Seeking Alpha all redesign occasionally. Multiplied across many scrapers, expect monthly breakage somewhere.

**Warning signs:**
- A panel goes empty with no API error (the request succeeded; the parse failed)
- Selectors that worked last week return zero elements
- Playwright timing out waiting for elements that no longer exist
- Subtle data corruption: column shifted by one because of a new TH

**Prevention:**
- **Minimize scraping surface.** Prefer APIs, then RSS, then scraping as last resort (per PROJECT.md).
- For each scraper: write a fixture HTML file + a unit test asserting expected fields parse correctly. When the live site changes, the unit test still passes (you're testing the parser); a live-site smoke test catches the divergence.
- Wrap every scraper in a `try/except` with a structured "scrape_failed" telemetry event; the UI shows a "data temporarily unavailable" placeholder rather than crashing.
- Use `selectolax` instead of BeautifulSoup (STACK.md §3c) — faster, but more importantly forces explicit CSS selectors which are easier to fix than fragile XPath.
- Prefer openinsider's cluster-screen page (precomputed) over raw transaction pages — fewer parsers to maintain.

**Phase mapping:** **P3** (insider scrape) and **P4** (news scrapes). **Ongoing** maintenance.

---

### 6.5 User-Agent and IP rate-limiting (especially Cloudflare-fronted sites)
**Pitfall:** openinsider sits behind Cloudflare; a default Python User-Agent (`python-requests/2.x`) gets immediately challenge-flagged. SEC requires descriptive UA. yfinance shares a UA pool that Yahoo has occasionally blocked.

**Warning signs:**
- HTTP 403 with Cloudflare challenge HTML in the body
- Sudden cascade of failures after working fine for a session
- IP-level slowdowns affecting all requests, not just one source
- "Please complete the security check to access" responses

**Prevention:**
- **Project-wide UA convention**: `"StockFinder/<version> (ryandammrose@onepointtwocapital.com)"` for all outbound HTTP. SEC's required format matches this shape exactly.
- Pace openinsider requests at ≤1 req/s (STACK.md §7). Use `aiolimiter` with a per-host bucket.
- yfinance: pass `session=httpx.Client(headers={"User-Agent": ...})` to override the default UA.
- If Cloudflare blocks persist, accept that openinsider may not work from some IPs and degrade gracefully — surface insider data from edgartools (primary) and skip the openinsider cluster cross-check.
- Cache aggressively to minimize live request volume.

**Phase mapping:** **P1** (HTTP client setup with global UA). **P3** (openinsider specifically).

---

## 7. Personal-Software Over-Engineering Pitfalls

### 7.1 NgRx for a single-user app
**Pitfall:** Adding NgRx, NGXS, or Akita to a single-user app multiplies boilerplate (actions, reducers, effects, selectors) by 10× with zero benefit. Angular signals + plain services cover all state needs for personal-use scale.

**Warning signs:**
- Anyone proposes "we'll need this for scale later"
- A reducer file appears for storing watchlist tickers
- Effects are wrapped around HTTP calls that already use `httpResource()`
- More state-management LOC than feature LOC

**Prevention:**
- Hold the line. STACK.md §2 is explicit: "Signals + services". Reject NgRx PRs without further discussion.
- Document the decision in the project's `CLAUDE.md` / contributing guide.
- If genuinely complex orchestration ever appears, prefer signals + computed + effect composition before reaching for a state library.

**Phase mapping:** **Ongoing** (architectural discipline).

---

### 7.2 Schema migrations system for a 5-table SQLite
**Pitfall:** Sqlx migrations, Flyway-style versioned SQL files, or a custom migration runner for a desktop-app SQLite with <10 tables is over-engineered. `CREATE TABLE IF NOT EXISTS` plus `ALTER TABLE` on schema bumps is sufficient.

**Warning signs:**
- A `migrations/` directory with timestamps in filenames
- A version table tracking applied migrations for a personal app
- A "rollback migration" PR
- More code in the migrations runner than in the table definitions

**Prevention:**
- Use Tauri `plugin-sql`'s built-in migration helper if available — it's exactly the right amount of ceremony (a single ordered list of SQL statements that run on app start).
- For new columns: `ALTER TABLE ... ADD COLUMN ... DEFAULT ...` is idempotent if guarded by a `PRAGMA table_info` check.
- Do not write down-migrations. If the app moves on, the migration is one-way.
- For breaking schema changes (rare): increment the cache directory name (`/cache/v2/`), let the old one age out.

**Phase mapping:** **P1** (cache initialisation). **Ongoing**.

---

### 7.3 Plugin/extension architecture before any user asks for one
**Pitfall:** Building a "plugin SDK" so other panels can be added by third parties to a single-user app. There are no third parties. There is one user.

**Warning signs:**
- An abstract `PanelPlugin` interface with no concrete need
- A "plugin loader" service in the bootstrap
- Documentation about "how to write a custom panel"
- Talk of a plugin marketplace

**Prevention:**
- Concrete panel modules, organised by feature in the Angular app. No plugin abstraction.
- If/when a 2nd user emerges (unlikely per scope), consider it then.
- Refactoring later (when the use case is concrete) is cheaper than building speculative abstraction now.

**Phase mapping:** **Ongoing** (architectural discipline).

---

### 7.4 Observability stack (Prometheus, OTel, Datadog)
**Pitfall:** Wiring Prometheus exporters, OpenTelemetry collectors, or APM agents into a personal desktop app. There is no SRE rotation. A rotating local log file solves the actual problem (debugging when a panel breaks).

**Warning signs:**
- A `/metrics` endpoint in the Python sidecar
- OTel SDK imports anywhere
- Discussion of a "metrics backend"
- More observability code than feature code

**Prevention:**
- Use `tauri-plugin-log` (already in STACK.md §1) on the Rust side. Pipe Python sidecar stderr through it too.
- Configure log rotation: max 10 MB per file, keep last 5 files, drop on rollover.
- "Open log directory" menu item for the user to grab logs when reporting a bug to themselves.
- Done. No metrics. No traces. No dashboards.

**Phase mapping:** **P0** (logging) — and that's it.

---

### 7.5 Multi-window / multi-monitor sophistication
**Pitfall:** Building tear-out panels, multi-window layouts, monitor-aware positioning for a personal app. One window, multiple tabs (or ticker switcher) is enough.

**Warning signs:**
- Discussions of "pop out the forecast chart to a 2nd monitor"
- Multi-window state synchronisation logic
- Tauri `WebviewWindow` proliferation in the Rust code

**Prevention:**
- Single window. Per-ticker route or tab switcher inside the same webview (FEATURES.md §1.1 already specifies "per-ticker route / dashboard view").
- If the user wants a 2nd view, they can open the app twice (Tauri supports this).

**Phase mapping:** **Ongoing**.

---

### 7.6 Hard-coding MU/IRM rather than using a ticker config table
**Pitfall:** Embedding `"MU"` and `"IRM"` as string literals throughout the codebase, then needing to add a third ticker becomes a refactor instead of a UI action. PROJECT.md "Active" requirement #8 explicitly says ticker add/remove without code changes.

**Warning signs:**
- `if (ticker === "MU") ...` anywhere in component logic
- Sector branches hard-coded by ticker (not by sector tag)
- "Add a ticker" requires editing a TypeScript file
- Tests reference specific tickers as fixtures rather than synthetic ones

**Prevention:**
- The `watchlist` SQLite table (STACK.md §6 schema) is the single source of truth. MU and IRM are seeded *as data*, not as constants.
- Sector-aware logic branches on the `sector` column (`'REIT' | 'Semiconductor' | 'Default'`), not on ticker string.
- Tests use synthetic tickers (`TEST_REIT`, `TEST_SEMI`) where possible; integration tests against live MU/IRM are flagged separately.

**Phase mapping:** **P0** (watchlist table seeded at first-run). **P1** (sector tagging plumbing).

---

## 8. Forecasting Honesty UI Pattern (Cross-Cutting)

### 8.1 Forecast cards missing one of: model name, training window, last-updated, backtest accuracy, CI, disclaimer
**Pitfall:** Showing a forecast as a single number with no provenance encourages magical thinking. The honest minimum surface is six pieces of metadata.

**Warning signs:**
- Forecast card just shows "MU: $145" with no other text
- No "as-of" timestamp visible
- Backtest metrics live on a separate page no one visits
- No "Not financial advice" footer

**Prevention:**
- Define a `ForecastCard` Angular component that REQUIRES (via input contract) the full set: `{model_name, training_window, last_updated_ts, point, ci_lo, ci_hi, backtest_hit_rate_in_ci, horizon, disclaimer}`. Types enforce — missing fields fail at compile.
- Disclaimer text: "Statistical/ML stock-price forecasts are unreliable, especially at long horizons. This is not financial advice."
- Backtest hit rate is sourced from the `forecasts_log` table — count of past forecasts where actual fell inside the CI / total count of forecasts past their horizon. If <10 forecasts have matured yet, show "insufficient backtest data" instead of a fake metric.

**Phase mapping:** **P5** (forecasts).

---

### 8.2 Forecast-vs-actual log not surfaced
**Pitfall:** The `forecasts_log` table exists but the user never sees the back-test view. The forecast panel stays "magic" instead of "calibrated".

**Warning signs:**
- "Forecast history" is a hidden / unlinked route
- The forecast cards show point estimates with no link to past performance
- Users continue to trust forecast models that have been historically wrong

**Prevention:**
- Per FEATURES.md §2.3: forecast back-tracking is HIGH leverage and lands by v1.1.
- Each `ForecastCard` includes an inline mini-component: "Past 90 days: this model was inside its 80% CI 7/10 times (70%)." This surfaces calibration without requiring a click.
- Dedicated "Forecast history" tab in v1.1: rows of past forecasts, columns = forecast, actual, in-CI?, error %.
- When a model's hit-rate drops below 60% over its trailing 20 forecasts, surface a "**model degraded**" warning badge on the forecast card.

**Phase mapping:** **P5** (logging from day one). **P7** (full back-test UI in v1.1).

---

## 9. Phase-Specific Warnings (Roll-up)

Quick reference table: which phase should be wary of which pitfalls.

| Phase | Critical pitfalls to address |
|---|---|
| **P0 — Foundation** | 5.1 (bundling smoke test), 5.2 (PyInstaller hidden imports — verify early), 5.5 (zoneless signals discipline), 5.6 (sidecar lifecycle), 7.6 (watchlist as data) |
| **P1 — Data Fetch** | 1.1 (yfinance fallback layer), 1.4 (adjusted vs raw price contract), 1.5 (SEC UA + 8 req/s limiter), 5.4 (IPC payload size — view vs store), 5.7 (batch panel fetches), 6.1 (adapter cascade), 6.2 (FMP usage tracking), 6.5 (UA convention) |
| **P2 — Fundamentals** | 1.2 (REIT GAAP — IRM-specific), 1.3 (XBRL inconsistency — use edgartools) |
| **P3 — Insider Trades** | 4.1 (M+S mechanical filter), 4.2 (10b5-1 flag), 4.3 (transaction-date ordering), 4.4 (buy-vs-sell asymmetry), 4.5 (cluster thresholds), 4.6 (code P only for clusters), 6.4 (openinsider scrape fragility) |
| **P4 — Sentiment** | 3.1 (no plain VADER), 3.2 (Reddit lag caption), 3.3 (low-karma filter), 3.4 (per-source breakdown), 3.5 (headline-only v1), 5.3 (FinBERT lazy download), 6.3 (PRAW OAuth) |
| **P5 — Forecasts** | 2.1 (CI + naive baseline + log), 2.2 (walk-forward CV, no look-ahead), 2.3 (separate analyst from model), 2.4 (long-horizon disclaimer), 8.1 (ForecastCard contract), 8.2 (forecast log surfaced) |
| **P6 — Momentum** | 1.4 (use adjusted close consistently), 5.4 (downsample for chart) |
| **P7 — v1.1 Cross-cutting** | 2.3 (back-test UI), 3.2 (sentiment × momentum quadrant uses delta not level), 3.5 (body-fetch on high-signal events), 6.3 (Reddit OAuth UX) |
| **P8 — Packaging** | 5.1 (codesign + notarize + per-target builds), 5.2 (post-build smoke test), 5.3 (bundle size budget), 5.7 (consider onedir), 5.8 (macOS entitlements) |
| **Ongoing** | 1.1 (yfinance version monitoring), 6.1 (fallback adapter), 6.4 (scraper fixture tests), 7.1-7.5 (resist over-engineering), 5.5 (signal discipline) |

---

## Sources

- [Tauri 2 sidecar docs](https://v2.tauri.app/develop/sidecar/)
- [Tauri Python sidecar example (dieharders)](https://github.com/dieharders/example-tauri-v2-python-server-sidecar)
- [SEC EDGAR rate-limit announcement](https://www.sec.gov/filergroup/announcements-old/new-rate-control-limits)
- [SEC EDGAR API guide (tldrfiling)](https://tldrfiling.com/blog/sec-edgar-api-rate-limits-best-practices)
- [Why yfinance keeps getting blocked (Medium)](https://medium.com/@trading.dude/why-yfinance-keeps-getting-blocked-and-what-to-use-instead-92d84bb2cc01)
- [yfinance rate-limit issue thread](https://github.com/ranaroussi/yfinance/issues/2289)
- [yfinance vs yahooquery discussion](https://github.com/ranaroussi/yfinance/discussions/1420)
- [FMP pricing](https://site.financialmodelingprep.com/pricing-plans)
- [PRAW rate limits docs](https://praw.readthedocs.io/en/stable/getting_started/ratelimits.html)
- [VADER vs FinBERT evaluation (ACM)](https://dl.acm.org/doi/fullHtml/10.1145/3677052.3698675)
- [FinBERT (ProsusAI) on Hugging Face](https://huggingface.co/ProsusAI/finbert)
- [SEC Form 4 Insider Trading Alerts — PageCrawl.io](https://pagecrawl.io/blog/sec-form-4-insider-trading-alerts)
- [Insider Trading Signals: SEC Form 4 Guide — MarketTriage](https://markettriage.com/insider-trading-signals)
- [SEC Form 4 Explained — HedgeTrace](https://hedgetrace.com/learn/sec-form-4-explained)
- [OpenInsider scraper repo](https://github.com/sd3v/openinsiderData)
- [Iron Mountain Q1 2026 Supplemental Reporting Package (SEC 8-K)](https://www.sec.gov/Archives/edgar/data/0001020569/000102056926000036/q12026srp_final.htm)
- [edgartools GitHub](https://github.com/dgunning/edgartools)
- [Angular zoneless change detection blog (Angular.dev)](https://angular.dev/guide/zoneless)
- [PyInstaller documentation — hooks](https://pyinstaller.org/en/stable/hooks.html)
- [Apple notarization workflow (Developer)](https://developer.apple.com/documentation/security/notarizing-macos-software-before-distribution)
