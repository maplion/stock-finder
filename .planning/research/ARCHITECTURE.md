# Architecture — Stock Finder

**Project:** Stock Finder (personal-use desktop stock-information aggregator)
**Researched:** 2026-05-25
**Stack:** Already decided — see `.planning/research/STACK.md` (Tauri 2 + Angular 20 + Python sidecar via PyInstaller + SQLite/Parquet hybrid cache)
**Confidence:** HIGH (this is a single-user, local desktop app following well-trodden Tauri + sidecar patterns)

---

## 0. Reading Guide

This document is prescriptive. It names exactly what to build in Phase 1 ("MU Fundamentals End-to-End"), what the Tauri commands are called, what the NDJSON contract looks like, what tables exist in SQLite, and how the per-ticker fan-out works. The roadmapper should treat the **Build Order** section (§7) as a phase-decomposition source of truth.

---

## 1. System Topology — Three-Process Architecture

```
+----------------------------------------------------------------------+
|                       USER MACHINE (single user)                     |
|                                                                      |
|   +--------------------+        +--------------------------------+   |
|   |   Angular 20 UI    |        |    Tauri 2 Rust Shell          |   |
|   |   (WebView)        |        |    (Orchestrator + Cache)      |   |
|   |                    |        |                                |   |
|   |  - Signals + svcs  |<------>|  - #[tauri::command] handlers  |   |
|   |  - ngx-echarts     | invoke |  - SQLite via plugin-sql       |   |
|   |  - PrimeNG tables  | events |  - Sidecar lifecycle (spawn,   |   |
|   |  - HttpClient/     |        |    monitor exit, kill on app   |   |
|   |    httpResource    |        |    shutdown)                   |   |
|   +--------------------+        |  - NDJSON stdin/stdout parser  |   |
|                                 |  - Event emitter (per-panel)   |   |
|                                 +-------------------+------------+   |
|                                                     |                |
|                                                     | spawn          |
|                                                     | per request    |
|                                                     | (stdio JSON)   |
|                                                     v                |
|                                 +--------------------------------+   |
|                                 |   Python Sidecar               |   |
|                                 |   (PyInstaller one-file)       |   |
|                                 |                                |   |
|                                 |  - orchestrator.py (entry)     |   |
|                                 |  - asyncio fan-out             |   |
|                                 |  - adapters/* (data sources)   |   |
|                                 |  - forecast/*, sentiment/*,    |   |
|                                 |    technicals/*, fundamentals/*|   |
|                                 |  - Parquet I/O (pyarrow)       |   |
|                                 +-------------------+------------+   |
|                                                     |                |
+-----------------------------------------------------|----------------+
                                                      |
                                          external HTTPS calls
                                                      v
              +----------+ +--------+ +-------+ +---------+ +------------+
              | yfinance | | EDGAR  | | Reddit| | GDELT   | | openinsider|
              | (Yahoo)  | | (SEC)  | | (PRAW)| | / RSS   | | (scrape)   |
              +----------+ +--------+ +-------+ +---------+ +------------+
```

### Responsibility split (strict)

| Layer | Owns | Does NOT |
|-------|------|----------|
| **Angular UI** | All rendering, user interaction, panel state via signals, routing, kicking off `invoke()` calls, subscribing to event-bus channels | Fetch external data. Talk to SQLite directly (goes through Tauri commands). Spawn processes. |
| **Tauri Rust shell** | Sidecar lifecycle, SQLite reads/writes via `plugin-sql`, watchlist CRUD, refresh-log writes, NDJSON parsing, re-emitting panel events to the WebView, error envelope normalization | Fetch external data (no `reqwest` calls to yfinance/EDGAR). Run any Python. Compute derived metrics. |
| **Python sidecar** | All external HTTP, all data parsing (XBRL, RSS, HTML), all numeric/ML work, Parquet I/O for OHLCV + ML training frames | Talk to SQLite (Rust owns it). Persist UI state. Long-lived process (spawn-per-request only). |

This split is the load-bearing decision: **Rust is orchestration and cache, Python is data and compute.** It keeps the Rust surface tiny (handful of commands), keeps Python disposable (crash = re-spawn next request), and avoids two layers fighting for cache ownership.

---

## 2. Angular Frontend — Screens, State, Routing

### 2.1 Routes

```
/                          → Watchlist (default landing)
/ticker/:symbol            → TickerDetail (lazy-loaded standalone component)
/ticker/:symbol/fundamentals  (default child)
/ticker/:symbol/forecast
/ticker/:symbol/sentiment
/ticker/:symbol/technicals
/ticker/:symbol/insider
/settings                  → Single-page settings (refresh TTLs, Reddit creds, FMP key)
```

All routes use **standalone components** (Angular 19+ default) with **lazy `loadComponent()`** so the WebView startup cost is just Watchlist + shell.

### 2.2 Component Tree

```
AppShell (top bar + side nav)
├── WatchlistPage
│   ├── WatchlistTable (PrimeNG p-table)
│   ├── AddTickerDialog
│   └── RefreshAllButton
├── TickerDetailPage (route param :symbol)
│   ├── TickerHeader (price, name, sector badge, last-refreshed, refresh-all)
│   ├── EarningsBanner (next earnings + last 4 surprise %)
│   ├── PanelTabs (PrimeNG p-tabView)
│   │   ├── FundamentalsPanel
│   │   │   ├── CommonMetricsBlock (always)
│   │   │   ├── ReitMetricsBlock (only if sector === 'REIT')
│   │   │   └── SemiHelpersBlock (only if sector === 'Semiconductor')
│   │   ├── ForecastPanel
│   │   │   ├── HorizonSelector (1-3mo / 3-12mo / 1-3yr)
│   │   │   ├── ForecastChart (ngx-echarts overlay: history + 3 model lines + CI bands)
│   │   │   └── ForecastSummaryTable
│   │   ├── SentimentPanel
│   │   │   ├── NewsHeadlinesTable
│   │   │   ├── SentimentTrendSparkline
│   │   │   └── RedditMentionsBlock (per-subreddit breakdown)
│   │   ├── TechnicalsPanel
│   │   │   ├── SignalsList (RSI/MACD/MA-cross/volume-surge/52w-distance state badges)
│   │   │   └── PriceChart (ngx-echarts: candlestick + 50/200 SMA + RSI/MACD sub-panels)
│   │   └── InsiderPanel
│   │       ├── ClusterSignalBadges
│   │       ├── NetFlowChart (bar chart by month)
│   │       └── TransactionsTable (code P/S default filter)
│   └── PanelErrorBoundary (per-panel; one panel error never breaks siblings)
└── SettingsPage
```

### 2.3 State Sharing — Signals + Services (no NgRx)

```
TickerStore (providedIn: 'root')
├── watchlist = signal<Ticker[]>([])
├── selectedTicker = signal<string | null>(null)
├── addTicker(sym), removeTicker(sym), reorder(...)
└── (each mutation calls invoke('add_ticker' | ...) and refreshes from Rust)

TickerDataStore (providedIn: 'root')
├── byTicker = signal<Record<string, TickerPanels>>({})
│   where TickerPanels = {
│     fundamentals: PanelState<FundamentalsData>,
│     forecast:     PanelState<ForecastData>,
│     sentiment:    PanelState<SentimentData>,
│     technicals:   PanelState<TechnicalsData>,
│     insider:      PanelState<InsiderData>,
│   }
│   and PanelState<T> = {
│     status: 'idle' | 'loading' | 'partial' | 'ready' | 'error',
│     data:   T | null,
│     error?: string,
│     refreshedAt?: number,
│   }
├── fetch(ticker, panels[]) → calls invoke('fetch_ticker_data', { ticker, panels })
│                              and subscribes to per-panel event channel
└── refreshPanel(ticker, panel) → invoke('refresh_panel', { ticker, panel })

SettingsStore (providedIn: 'root')
└── settings = signal<Settings>(...) backed by tauri-plugin-store
```

**Why this works:** signals propagate panel-level updates without re-rendering siblings; `PanelErrorBoundary` reads `status === 'error'` and shows a retry button without taking down the tab.

### 2.4 Progressive Rendering Pattern

When `TickerDetailPage` mounts for `MU`:

1. Set every panel's `status` to `'loading'` (skeletons render).
2. `invoke('fetch_ticker_data', { ticker: 'MU', panels: ['fundamentals','forecast','sentiment','technicals','insider'] })`.
3. Subscribe to event channel `ticker-data://MU` (Tauri event bus).
4. As each `{panel, data}` event arrives, set that panel's signal slice → `status: 'ready'` → that subtree re-renders with real data.
5. If `{panel, status: 'error'}` arrives, that panel shows error UI with retry; other panels keep loading.
6. Final `{event: 'done'}` event triggers refresh-timestamp updates.

This is the headline UX promise: **fundamentals can render in 800ms while forecast is still computing 8 seconds later, with no blocking.**

---

## 3. Tauri Rust Shell — Commands and Events

### 3.1 `#[tauri::command]` Surface (small on purpose)

| Command | Args | Returns | Purpose |
|---------|------|---------|---------|
| `list_tickers` | — | `Vec<Ticker>` | Read watchlist from SQLite |
| `add_ticker` | `{ symbol, sector? }` | `Ticker` | Validate via sidecar (`op: "validate_ticker"`), insert |
| `remove_ticker` | `{ symbol }` | `()` | Delete watchlist row + cached parquet files for that symbol |
| `update_ticker_sector` | `{ symbol, sector }` | `Ticker` | Overrides auto-detected sector |
| `fetch_ticker_data` | `{ ticker, panels: [String], force_refresh: bool }` | `request_id: String` | **Spawns sidecar**, returns request id immediately; data arrives via events |
| `refresh_panel` | `{ ticker, panel }` | `request_id: String` | Single-panel refresh (sugar over `fetch_ticker_data` with one panel) |
| `get_cached_panel` | `{ ticker, panel }` | `Option<PanelPayload>` | Read-only — return last cached payload + timestamp without fetching |
| `cancel_request` | `{ request_id }` | `()` | Kill the sidecar PID associated with this request (panel timeouts) |
| `get_refresh_log` | `{ ticker }` | `Vec<RefreshLogRow>` | Powers "last refreshed at" badges |
| `get_settings` / `set_settings` | — / `Settings` | `Settings` | Backed by `tauri-plugin-store` (JSON in app data dir) |
| `wipe_cache` | `{ scope: 'all' \| 'ticker', ticker? }` | `()` | "Reset seen state" action |

That is the **entire** Rust surface. Eleven commands. Everything else is in the sidecar.

### 3.2 Event Bus Channels

Rust emits panel events into a deterministic channel name so Angular can subscribe per ticker:

| Channel | Payload | When |
|---------|---------|------|
| `ticker-data://{symbol}` | `{ request_id, panel, status, data?, error?, ts }` | Every NDJSON line from sidecar matching that ticker |
| `ticker-data-done://{symbol}` | `{ request_id, ts, panels_succeeded: [...], panels_failed: [...] }` | Sidecar emits final `{event:"done"}` |
| `sidecar-fatal://{request_id}` | `{ request_id, exit_code, stderr_tail }` | Non-zero exit before `{event:"done"}` |

### 3.3 Sidecar Lifecycle Management (Rust side)

- One sidecar process **per `fetch_ticker_data` invocation**. Process is short-lived (seconds), not a daemon.
- Each spawn gets a UUID `request_id`, tracked in a `HashMap<RequestId, Child>` behind a `Mutex`.
- On NDJSON line received → parse → re-emit to event channel.
- On EOF/exit:
  - If exit code == 0 and `{event:"done"}` was emitted → success.
  - If exit code != 0 OR EOF without `done` → emit `sidecar-fatal://...` with last ~4KB of stderr.
- On app shutdown → kill all tracked children.
- On `cancel_request` → kill that PID, emit `error: "cancelled"` for any panels not yet completed.
- Per-request hard wall-clock timeout: **45 seconds** (configurable in settings). If exceeded → kill + emit cancelled events for unfinished panels.

---

## 4. Python Sidecar — Module Layout

```
python/
├── main.py                       # PyInstaller entry: read stdin envelope, dispatch, write NDJSON
├── orchestrator.py               # asyncio fan-out across requested panels; emits NDJSON per completion
├── envelope.py                   # request/response schemas (Pydantic)
├── ipc.py                        # NDJSON line writer (flushes immediately; never buffers)
├── adapters/
│   ├── __init__.py
│   ├── yfinance_adapter.py       # primary price/quote/news
│   ├── yahooquery_adapter.py     # fallback for yfinance
│   ├── edgar_adapter.py          # fundamentals (10-K/10-Q), Form 4 insider, SIC lookup
│   ├── fmp_adapter.py            # secondary fundamentals + analyst estimates
│   ├── reddit_adapter.py         # PRAW wrapper
│   ├── news_adapter.py           # feedparser + GDELT
│   └── openinsider_adapter.py    # HTML scrape for cluster pages
├── fundamentals/
│   ├── common.py                 # standardized P&L / BS / CF + ratios
│   ├── reit.py                   # FFO/AFFO/payout/leverage extension
│   └── semi.py                   # inventory days, capex helpers
├── forecast/
│   ├── analyst.py                # straight passthrough from yfinance/FMP
│   ├── statistical.py            # statsforecast AutoARIMA/Theta/AutoETS + Prophet
│   ├── ml.py                     # darts NHiTS (deferred — Phase N)
│   └── interface.py              # Forecast protocol: {horizon, point, lo, hi, model_name, generated_at}
├── sentiment/
│   ├── finvader_scorer.py        # fast lexicon
│   ├── finbert_scorer.py         # top-N rescoring (lazy-load model)
│   └── aggregate.py              # rolling means, per-subreddit splits
├── technicals/
│   └── indicators.py             # pandas-ta-classic wrapper: RSI/MACD/SMA/cross/volume-surge
├── insider/
│   ├── transactions.py           # normalize Form 4 + openinsider rows
│   └── clusters.py               # 3+ insiders / 10d, net flow, CEO/CFO highlights
├── cache/
│   ├── parquet_store.py          # per-ticker OHLCV + ML training frames
│   └── paths.py                  # resolve app_cache_dir() from env (Rust passes it in envelope)
├── http_client.py                # shared httpx.AsyncClient + tenacity + aiolimiter (per-source token buckets)
└── logging_setup.py              # writes to stderr (Rust forwards to tauri-plugin-log)
```

### 4.1 Dispatch table (orchestrator.py)

```python
PANEL_DISPATCH = {
    "fundamentals": fetch_fundamentals,   # sector-dispatches inside (REIT vs default)
    "forecast":     fetch_forecast,
    "sentiment":    fetch_sentiment,
    "technicals":   fetch_technicals,
    "insider":      fetch_insider,
}

# Per-panel timeout (soft) — partial results still flushed on timeout
PANEL_TIMEOUT_S = {
    "fundamentals": 15,
    "forecast":     30,
    "sentiment":    20,
    "technicals":   10,
    "insider":      20,
}
```

`run()` wraps each panel in `asyncio.wait_for(panel_fn(...), timeout=PANEL_TIMEOUT_S[panel])` and ships them concurrently via `asyncio.as_completed()`, writing each result to stdout as it lands.

### 4.2 Sector-Aware Fundamentals Dispatch

```python
async def fetch_fundamentals(ticker: str, sector: str | None, cache_dir: Path) -> dict:
    common = await fundamentals.common.fetch(ticker)  # always
    if sector == "REIT" or await edgar_adapter.is_reit(ticker):   # SIC 6798 etc.
        reit = await fundamentals.reit.fetch(ticker)
        return {"common": common, "reit": reit}
    if sector == "Semiconductor":
        semi = await fundamentals.semi.fetch(ticker)
        return {"common": common, "semi": semi}
    return {"common": common}
```

Sector is passed in by the Tauri command (from the `tickers` table). If absent, sidecar falls back to SIC lookup.

---

## 5. IPC Contract — NDJSON over stdio

### 5.1 Request envelope (Rust → Python stdin, single JSON object then EOF on stdin)

```json
{
  "request_id": "8f3a2c10-...",
  "op": "fetch_ticker_data",
  "ticker": "MU",
  "panels": ["fundamentals", "forecast", "sentiment", "technicals", "insider"],
  "sector": "Semiconductor",
  "force_refresh": false,
  "cache_dir": "/Users/.../Library/Caches/com.onepointtwocapital.stockfinder",
  "settings": {
    "fmp_api_key": "...",
    "reddit": { "client_id": "...", "client_secret": "...", "user_agent": "..." },
    "edgar_user_agent": "StockFinder ryandammrose@onepointtwocapital.com",
    "horizons": ["1-3mo", "3-12mo", "1-3yr"]
  }
}
```

### 5.2 Response stream (Python → Rust stdout, NDJSON — one JSON object per line, flushed immediately)

**Panel success:**
```json
{"request_id":"8f3a2c10-...","panel":"fundamentals","status":"ok","ts":1748131200,"data":{"common":{"revenue_ttm":25000000000,"gross_margin":0.42,"...":"..."}}}
```

**Panel partial (e.g., REIT computed FFO without reported FFO):**
```json
{"request_id":"8f3a2c10-...","panel":"fundamentals","status":"partial","ts":1748131200,"data":{"common":{...},"reit":{"ffo_computed":1200000000,"ffo_reported":null}},"warnings":["No us-gaap:RealEstateInvestmentTrustFundsFromOperations tag in latest 10-Q; computed from components."]}
```

**Panel error (one panel fails, others still run):**
```json
{"request_id":"8f3a2c10-...","panel":"sentiment","status":"error","ts":1748131200,"error":{"code":"RATE_LIMITED","msg":"Reddit returned 429 after 3 retries","retry_after_s":60}}
```

**Fallback used (signal to UI):**
```json
{"request_id":"8f3a2c10-...","panel":"forecast","status":"ok","ts":1748131200,"data":{...},"notes":["yfinance returned 401; retried with yahooquery (success)"]}
```

**Optional progress event (long-running ML forecast):**
```json
{"request_id":"8f3a2c10-...","panel":"forecast","status":"progress","progress":0.4,"msg":"fitting AutoARIMA"}
```

**Terminal event (always last line):**
```json
{"request_id":"8f3a2c10-...","event":"done","ts":1748131201,"summary":{"ok":["fundamentals","technicals","insider","forecast"],"error":["sentiment"]}}
```

### 5.3 Contract invariants

1. **Every line is a valid JSON object terminated by `\n`.** No multi-line JSON.
2. **`request_id` is on every line** so the Rust router can never get confused if futures are added later.
3. **Each `panel` appears at most twice** — optionally once as `progress`, exactly once with a terminal status (`ok` | `partial` | `error`).
4. **`{event:"done"}` is always the final line** before process exit 0.
5. **stderr is for logs only** (forwarded to `tauri-plugin-log`); never structured data.
6. **Exit code 0** = orchestrator completed. Non-zero = unrecoverable crash; Rust emits `sidecar-fatal://`.

### 5.4 Why NDJSON and not gRPC / a single response blob

- Single blob = no progressive rendering (defeats the headline UX).
- gRPC = additional dep + port binding + antivirus alarms; not worth it for two processes on the same machine talking briefly.
- NDJSON = trivial to parse line-by-line in Rust (`tokio::io::BufReader::lines()`); trivial to write in Python (`print(json.dumps(...), flush=True)`); human-readable in logs; matches Tauri sidecar community examples.

---

## 6. Per-Ticker Data Flow — Sequence Diagram (Text Form)

```
User clicks "MU" in WatchlistTable
  │
  ▼
WatchlistPage.router.navigate(['/ticker', 'MU'])
  │
  ▼
TickerDetailPage.ngOnInit():
  - tickerDataStore.fetch('MU', ['fundamentals','forecast','sentiment','technicals','insider'])
      │
      ▼
  invoke('fetch_ticker_data', { ticker:'MU', panels:[...], force_refresh:false })
      │
      ▼  (Rust)
  Rust handler:
    1. Look up sector from tickers table     ──► SQLite read
    2. Resolve cache_dir from app_cache_dir()
    3. Spawn sidecar binary with `tauri-plugin-shell`
    4. Write envelope JSON to stdin; close stdin
    5. Register Child in HashMap<request_id, Child>
    6. Return request_id to UI immediately (≤5 ms)
    7. Spawn tokio task to:
         - Read stdout line-by-line
         - For each line: emit('ticker-data://MU', line_json)
         - On EOF: check exit code; emit done or fatal
      │
      ▼  (Python sidecar)
  main.py reads envelope from stdin
    └► orchestrator.run(envelope)
         └► asyncio.gather(
               with_timeout(fetch_fundamentals(...), 15),
               with_timeout(fetch_forecast(...),     30),
               with_timeout(fetch_sentiment(...),    20),
               with_timeout(fetch_technicals(...),   10),
               with_timeout(fetch_insider(...),      20),
            )
         (each panel writes its NDJSON line as soon as it resolves)
      │
      ▼
  Inside each panel coroutine, e.g. fetch_fundamentals:
    1. http_client.get(EDGAR /submissions/...)  ──► via aiolimiter (8 req/s cap)
    2. http_client.get(EDGAR /XBRL companyconcept/...)
    3. fundamentals.common.normalize(...)
    4. if sector == REIT: fundamentals.reit.compute(...)
    5. ipc.emit({panel:"fundamentals", status:"ok", data:{...}})  ──► stdout flush
      │
      ▼  (Rust task picks up the line)
  emit('ticker-data://MU', { request_id, panel:'fundamentals', status:'ok', data:{...} })
      │
      ▼  (Angular)
  TickerDataStore listener:
    byTicker['MU'].fundamentals.set({ status:'ready', data, refreshedAt: now })
      │
      ▼  (Angular change detection — signals only re-render this subtree)
  FundamentalsPanel renders real metrics. The other four panels keep their skeletons
  until their own events arrive.
      │
      ▼  (Repeat for technicals, insider, sentiment, forecast as they each land)
      │
      ▼  (Python emits final line)
  ipc.emit({event:"done", summary:{ok:[...], error:[...]}})
  sys.exit(0)
      │
      ▼  (Rust)
  Detect exit 0 + done event → emit('ticker-data-done://MU', summary)
  Rust writes refresh_log rows for each ok panel.
      │
      ▼  (Angular)
  TickerHeader updates "Last refreshed" badge.
  PanelErrorBoundary shows retry buttons for any panel in summary.error.
```

**Timing budget (rough, on a warm cache):**
- Tauri command return: ≤5 ms
- Sidecar cold start (PyInstaller bootstrap): 1–3 s
- Fundamentals panel: ~3 s (EDGAR + parse)
- Technicals: ~1 s (uses cached parquet OHLCV)
- Insider: ~5 s (Form 4 fetch + cluster compute)
- Sentiment: ~10 s (Reddit + headlines + FinVADER; FinBERT skipped or async)
- Forecast (statistical only): ~8 s (statsforecast on daily OHLCV)
- Total wall-clock: ~12–15 s, but **fundamentals visible to user in ~5 s**.

---

## 7. Build Order — Roadmapper Input

This is the prescriptive recommendation for phase decomposition. Each phase is a vertical slice that produces a runnable, demoable thing.

### Phase 1 — **MU Fundamentals End-to-End** (the named v1 vertical slice)

**Goal:** Prove the entire pipe with the minimum feature surface — common-stock fundamentals for MU, end-to-end, no REIT and no ML.

**Includes (and only includes):**
- Tauri 2 scaffold + Angular 20 standalone shell + Python sidecar bootstrap
- All required Tauri plugins installed and capabilities configured (`shell`, `sql`, `fs`, `store`, `log`)
- PyInstaller build pipeline for `stockfinder-py-<triple>` (mac arm64 only is acceptable for Phase 1)
- SQLite schema (subset): `tickers`, `refresh_log` only — created on app start via `CREATE TABLE IF NOT EXISTS`
- Tauri commands: `list_tickers`, `add_ticker`, `remove_ticker`, `fetch_ticker_data`, `get_refresh_log`, `cancel_request`
- Sidecar IPC: full NDJSON contract (only `fundamentals` panel supported; other panel names return `error: NOT_IMPLEMENTED`)
- `adapters/yfinance_adapter.py`, `adapters/edgar_adapter.py`, `fundamentals/common.py`
- Angular UI: `WatchlistPage`, `TickerDetailPage` with `FundamentalsPanel` only (other tabs disabled/coming-soon)
- `TickerDataStore` + `PanelErrorBoundary` complete (architecture must support five panels even if four are stubbed)
- Seed watchlist with MU on first launch
- Error path: kill yfinance, verify "yfinance error, fell back to yahooquery" status is visible

**Acceptance:** Launch app → MU is in watchlist → click MU → Fundamentals panel renders revenue/margins/EPS/FCF/cash/debt/ratios within ~10 seconds → "refresh" button works → killing the sidecar mid-fetch shows a clean panel error.

**Why this slice and not "wire up all five panels with mocks":** mocks don't exercise the IPC, the rate limiter, the cache layer, the PyInstaller packaging path, or the cross-OS shell capabilities — the things most likely to bite. A real end-to-end vertical of one panel exercises every layer.

### Phase 2 — REIT Extension via IRM Fundamentals

**Adds:** `fundamentals/reit.py` + `adapters/edgar_adapter.is_reit()` + `ReitMetricsBlock` Angular component + sector dispatch in `fetch_fundamentals` + add IRM to seed watchlist. **Validates** the sector-aware seam.

**Acceptance:** IRM ticker view shows the additional REIT Metrics block (FFO computed + AFFO + payout ratio); MU view does not.

### Phase 3 — Technicals Panel

**Adds:** `technicals/indicators.py` (pandas-ta-classic), `cache/parquet_store.py` for OHLCV cache, `TechnicalsPanel` Angular with SignalsList + PriceChart (ngx-echarts candlestick + RSI/MACD sub-panels). **Validates** the Parquet cache and the chart toolchain.

### Phase 4 — Insider Panel

**Adds:** `adapters/edgar_adapter` Form 4 + `adapters/openinsider_adapter.py` + `insider/clusters.py` + `InsiderPanel` Angular with cluster badges + net-flow chart + transactions table (default filter to P/S codes). **Validates** the dual-source-with-fallback pattern and HTML scraping.

### Phase 5 — Sentiment Panel

**Adds:** `adapters/news_adapter.py` (feedparser + GDELT) + `adapters/reddit_adapter.py` (PRAW) + `sentiment/finvader_scorer.py` + `SentimentPanel` Angular (headline table + sparkline + per-subreddit reddit block). **Defer FinBERT** to a sub-phase once latency is measured — initial ship is FinVADER-only. **Validates** Reddit OAuth setup flow.

### Phase 6 — Forecast Panel (statistical only)

**Adds:** `forecast/analyst.py` + `forecast/statistical.py` (statsforecast + prophet) + `forecast/interface.py` + `ForecastPanel` Angular with horizon selector + side-by-side chart + summary table. **Defers ML.**

### Phase 7 — Forecast Log + Back-Tracking UI

**Adds:** `forecasts_log` SQLite table populated on every forecast run + back-test view ("model said $X 90 days ago; actual $Y; CI hit y/n"). High-leverage differentiator per FEATURES.md §2.3.

### Phase 8 — "What Changed Since Last Open" Diff

**Adds:** `headlines_seen`, `insider_filings_seen` SQLite tables + diff computation at fetch time + digest banner on TickerDetailPage. This is the single most aligned feature with the "one place to look" value prop.

### Phase 9 — ML Forecast Model

**Adds:** `forecast/ml.py` (darts NHiTS) + ML training frame builder in `cache/parquet_store.py` + third forecast line in ForecastChart. **XL complexity per FEATURES.md.** Deliberately last because it depends on everything above (history cache, sentiment features, technicals features as covariates).

### Phase 10+ — v1.1 cross-cutting features

Cross-ticker comparison, sentiment × momentum quadrant, thesis tracker (text), threshold warnings, REIT NOI/occupancy parser.

### Build-order rationale (for the roadmapper)

The order respects three constraints:

1. **Vertical slices, never horizontal.** Each phase ships something usable end-to-end, never "the SQLite layer for everything."
2. **Cheapest-to-validate-the-architecture first.** Phase 1 = one panel, one ticker — exposes every layer (UI, IPC, cache, sidecar packaging) with minimum scope.
3. **Risk-ordered, not feature-ordered.** REIT extension is Phase 2 because it validates the sector seam (the architectural premise of the whole project). ML forecast is last because it's XL and depends on everything else; doing it earlier would block other panels behind ML choices.

---

## 8. Extension Points / Seams (Where Future Work Plugs In)

| Extension | How to add | Touches |
|-----------|------------|---------|
| **New data source** (e.g., a new news provider) | Create `adapters/<source>.py` exposing `async def fetch(ticker) -> dict` + register in the panel module that uses it (e.g., `sentiment/aggregate.py`) | 1 new file + 1 import line; no Rust changes; no UI changes |
| **New ticker** | User clicks "Add ticker" → `invoke('add_ticker', {symbol})` → row inserted | Zero code |
| **New sector template** (e.g., Banks → NIM, efficiency ratio) | Add `fundamentals/banks.py` + add `'Banks'` case to sector dispatch in `fetch_fundamentals` + add `BanksMetricsBlock` Angular standalone component | 1 Python file + 1 dispatch line + 1 Angular component |
| **New forecast model** | Implement `forecast/<model>.py` conforming to `forecast/interface.py` (returns `{horizon, point, lo, hi, model_name, generated_at}`) + add to `forecast.statistical.MODELS` registry | 1 Python file + 1 registry line; UI iterates over models so no Angular change |
| **New panel** (e.g., "Options flow") | Add `<panel>` to `PANEL_DISPATCH` in `orchestrator.py` + add `<Panel>Component` standalone + add tab to `PanelTabs` + add slice to `TickerPanels` type | 3 places, all additive |
| **Swap FinVADER → FinBERT for default scoring** | Change one line in `sentiment/aggregate.py`; FinBERT model already in adapter | 1 line |

**The seam discipline:** every extension point is **add-only** (new file, new registry row), not modify-only. No "open up an existing file and add a branch" required for the common extensions.

---

## 9. Cache Layer — Detailed Layout

### 9.1 SQLite (owned by Rust; opened via `tauri-plugin-sql`)

Lives at `app_data_dir()/stockfinder.db`. All schemas use `CREATE TABLE IF NOT EXISTS` — no migration system (single-user simplification).

```sql
CREATE TABLE IF NOT EXISTS tickers (
    symbol     TEXT PRIMARY KEY,
    sector     TEXT,                -- 'REIT' | 'Semiconductor' | 'Default' | NULL (auto-detect)
    position   INTEGER NOT NULL,
    added_at   INTEGER NOT NULL
);

CREATE TABLE IF NOT EXISTS refresh_log (
    symbol         TEXT NOT NULL,
    panel          TEXT NOT NULL,      -- 'fundamentals' | 'forecast' | ...
    refreshed_at   INTEGER NOT NULL,
    status         TEXT NOT NULL,      -- 'ok' | 'partial' | 'error'
    notes          TEXT,
    PRIMARY KEY (symbol, panel)
);

CREATE TABLE IF NOT EXISTS headlines_seen (
    id             TEXT PRIMARY KEY,   -- source-stable hash of (url + ticker)
    symbol         TEXT NOT NULL,
    first_seen_at  INTEGER NOT NULL
);
CREATE INDEX IF NOT EXISTS idx_headlines_seen_symbol ON headlines_seen(symbol);

CREATE TABLE IF NOT EXISTS insider_filings_seen (
    accession      TEXT PRIMARY KEY,
    symbol         TEXT NOT NULL,
    first_seen_at  INTEGER NOT NULL
);

CREATE TABLE IF NOT EXISTS forecasts_log (
    id             INTEGER PRIMARY KEY AUTOINCREMENT,
    symbol         TEXT NOT NULL,
    model_name     TEXT NOT NULL,      -- 'analyst' | 'autoarima' | 'prophet' | 'nhits'
    horizon        TEXT NOT NULL,      -- '1-3mo' | '3-12mo' | '1-3yr'
    generated_at   INTEGER NOT NULL,
    target_date    INTEGER NOT NULL,   -- when the forecast resolves
    point          REAL NOT NULL,
    ci_lo          REAL,
    ci_hi          REAL
);
CREATE INDEX IF NOT EXISTS idx_forecasts_log_symbol_target ON forecasts_log(symbol, target_date);

CREATE TABLE IF NOT EXISTS cache_index (
    cache_key      TEXT PRIMARY KEY,   -- e.g., 'ohlcv:MU:1d'
    file_path      TEXT NOT NULL,
    expires_at     INTEGER NOT NULL
);
```

### 9.2 Parquet (owned by Python; written via pyarrow)

Lives at `app_cache_dir()/parquet/`. Rust only knows the paths via `cache_index`; it never reads these files.

```
parquet/
├── ohlcv/
│   ├── MU_1d.parquet           # daily OHLCV, full available history
│   ├── MU_1wk.parquet          # weekly resample (for 3-12mo forecast horizon)
│   ├── MU_1mo.parquet          # monthly resample (for 1-3yr forecast horizon)
│   └── IRM_1d.parquet, ...
├── ml_features/
│   └── MU_feat_v1.parquet      # engineered features for ML forecast (Phase 9+)
└── models/
    ├── MU_autoarima.pkl        # joblib-dumped fitted model
    └── MU_prophet.pkl
```

### 9.3 Cache TTLs (from STACK.md, codified)

| Data | TTL | Where enforced |
|------|-----|----------------|
| Intraday quote | 5 min | Sidecar checks `cache_index.expires_at` before refetching |
| Daily OHLCV | until end of US trading day | Same |
| Fundamentals | 24 h or next filing | Same |
| Insider | 6 h | Same |
| News + sentiment | 1 h | Same |
| Reddit | 30 min | Same |
| Analyst targets | 24 h | Same |

`force_refresh: true` in the envelope bypasses TTLs (powers the "Refresh" button).

---

## 10. Single-User Simplifications (What We Don't Build)

These are explicit non-decisions documented so future phases don't re-litigate them:

| Concern | Decision | Why |
|---------|----------|-----|
| Authentication | None | One user, local app. |
| Schema migrations | `CREATE TABLE IF NOT EXISTS` only | If schema changes destructively, ship a one-time `wipe_cache` and re-fetch. No migration framework. |
| Observability | `tauri-plugin-log` to a rotating file in `app_log_dir()` | No metrics, no traces, no dashboards. |
| Background scheduling | None | Refresh = user click. No cron, no daemons. |
| Settings | Single JSON via `tauri-plugin-store` | No settings UI hierarchy; flat key-value. |
| Secrets | Settings JSON for now (FMP key, Reddit creds) | If user objects, swap to OS keychain via `tauri-plugin-store-keyring` later. |
| Multi-window | Single window | Stock Finder is not a multi-monitor trading terminal. |
| Update channel | Defer; user rebuilds locally for now | Signed installers require Apple Developer + Windows code-signing certs. Park until ship. |
| Telemetry | None | Personal use. |
| Tests | Phase 1: smoke tests only (one E2E "MU fundamentals loads"). Per-adapter unit tests added as adapters are built. | Test discipline rises with shipped surface. |

---

## 11. Failure Modes — Concrete Handling

| Failure | Where caught | What user sees |
|---------|--------------|----------------|
| Sidecar crash mid-fetch (segfault, unhandled exception) | Rust: child exit code != 0 before `{event:"done"}` | All in-flight panels show "Sidecar crashed — view logs" error; other panels (if already completed) remain. |
| Slow source — Reddit takes 30s | Python: `asyncio.wait_for(panel_fn, timeout=20)` | Sentiment panel shows "Timed out — partial data" with whatever the news adapter returned; other panels unaffected. |
| yfinance returns 401/429 | `adapters/yfinance_adapter` catches, retries with `tenacity`, then escalates to `yahooquery_adapter` | Panel renders with a yellow `notes: ["yfinance error, fell back to yahooquery"]` badge — data is present, source attribution is honest. |
| EDGAR rate-limited | `http_client.py` global `aiolimiter` caps requests at 8/s before they go out | User never sees this; throttling is preventive. |
| First-launch dependency bootstrap | PyInstaller one-file binary bundles everything | No `pip install` on user's machine; first launch downloads only FinBERT model (~210 MB) to cache dir on first sentiment fetch. |
| FinBERT model not downloaded yet | `sentiment/finbert_scorer.py` checks cache dir; if missing, downloads + shows progress events | Sentiment panel emits `status:"progress"` with `msg:"Downloading FinBERT model (210 MB, one-time)"` then proceeds. |
| User has no internet | Each adapter raises `NetworkError`; orchestrator marks all panels `status:"error"` with `code:"OFFLINE"` | App still opens, watchlist still readable from SQLite; clicking refresh shows "Offline — no cached data" per panel. |
| SQLite locked (rare; rust-driven, no concurrent writers) | `tauri-plugin-sql` returns Err; Rust surfaces as command error | Toast: "Database busy, retry." |
| Disk full (parquet write fails) | Python catches `OSError`, emits panel `status:"error"` with `code:"DISK_FULL"` | Panel shows error with a "Clear cache" action. |
| App shutdown during fetch | Rust drops all `Child` handles → OS terminates sidecar PIDs | No leaked processes; next launch is clean. |
| User cancels (clicks away mid-fetch) | Angular calls `invoke('cancel_request', { request_id })` | Sidecar killed; pending panels marked cancelled (not error). |

**The unifying principle:** errors are **panel-scoped**, not app-scoped. The architecture never lets one slow API kill the dashboard.

---

## 12. Pre-Phase-1 Infrastructure Checklist (what MUST exist before any feature panel)

This is the answer to "what's Phase 0 / what's the foundation under MU Fundamentals." For the roadmapper: bundle these into Phase 1 since they're the cost of doing the first vertical slice; do NOT try to ship them as a separate phase (you'd have nothing to demo).

- [ ] Tauri 2 scaffold with all required plugins installed (`shell`, `sql`, `fs`, `store`, `log`)
- [ ] Angular 20 standalone shell + routing + at least `WatchlistPage` and `TickerDetailPage` skeletons
- [ ] Python sidecar project (`python/`) with `main.py` reading envelope from stdin, writing NDJSON to stdout
- [ ] PyInstaller build script (initially macOS arm64 only)
- [ ] Tauri capabilities config granting sidecar execute + SQLite + fs writes scoped to cache dir
- [ ] Minimum SQLite schema (`tickers`, `refresh_log`)
- [ ] `TickerStore` and `TickerDataStore` services (even if `TickerDataStore` only knows the `fundamentals` panel for now)
- [ ] One end-to-end smoke test: launch app, click MU, see fundamentals data, see refresh-log update

---

## 13. Open Questions Carried Forward to Per-Phase Research

(Mostly inherited from STACK.md §Open Questions; restated here for the roadmapper's convenience.)

1. **FFO disclosure in IRM's actual 10-K** — verify in Phase 2 whether `RealEstateInvestmentTrustFundsFromOperations` is tagged or must be computed from components.
2. **FinBERT latency on Intel Mac** — measure in Phase 5; may need ONNX export.
3. **Reddit OAuth UX** — Phase 5 decides between shared-client-id and per-user-app-credentials.
4. **Cross-platform PyInstaller CI** — defer to a packaging phase after Phase 5 (mac-arm64-only is fine for personal dogfooding).
5. **DuckDB introduction** — only if Phase 7+ back-testing UI grows joins across parquet files.

---

## Sources

Architecture-pattern references (not stack-choice references — those are in STACK.md):

- [Tauri 2 sidecar developer docs](https://v2.tauri.app/develop/sidecar/) — definitive guide to `externalBin`, capabilities, NDJSON-over-stdio
- [Tauri Python sidecar example (dieharders)](https://github.com/dieharders/example-tauri-v2-python-server-sidecar) — community-validated topology
- [Writing a pandas sidecar for Tauri (MClare)](https://mclare.blog/posts/writing-a-pandas-sidecar-for-tauri/) — single-shot subprocess pattern with stdio JSON
- [Angular standalone components blog (Angular team)](https://blog.angular.dev/the-future-is-standalone-475d7edbc706) — confirms standalone-by-default in v19+, no NgModule
- [Angular signals architecture guide](https://angular.dev/guide/signals) — primary reference for the signals+services state pattern
- [Apache ECharts candlestick examples](https://echarts.apache.org/examples/en/index.html#chart-type-candlestick) — chart-component design source
- [SEC EDGAR XBRL companyconcept API](https://www.sec.gov/edgar/sec-api-documentation) — confirms the REIT FFO XBRL tag strategy in §4.2 / §8

---

*Last updated: 2026-05-25*
