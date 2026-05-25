# Technology Stack

**Project:** Stock Finder (personal-use desktop stock-information aggregator)
**Researched:** 2026-05-25
**Overall confidence:** HIGH for shell/frontend/data-source choices; MEDIUM for forecasting and sentiment (model selection has personal-taste / experimentation component)

---

## TL;DR — Headline Choices

| Layer | Pick | One-line "why over alternatives" |
|------|------|----------------------------------|
| Shell | **Tauri 2.x** (latest 2.11.x as of May 2026) | Already committed; smaller bundle + lower RAM than Electron, native menus, mature v2 plugin ecosystem |
| Frontend framework | **Angular 20 LTS** (or 21 stable) standalone components, zoneless | Latest LTS in May 2026; standalone is default since v19, zoneless stable since v20.2 — cleaner reactivity story |
| State | **Angular Signals + plain services** (no NgRx) | Personal-use app; NgRx is overkill, signals are native and reactive enough |
| Charts | **ngx-echarts** wrapping Apache ECharts 5.x | Free, MIT/Apache, built-in candlestick + volume + technical-indicator overlays; ECharts is the de-facto OSS finance chart |
| Tables | **PrimeNG 19/20 DataTable** | Sort/filter/pivot/freeze-column out of the box; MIT-licensed; ag-grid Community is feature-thin and Enterprise is paid |
| Styling | **Tailwind CSS v4 + PrimeNG unstyled mode** | Utility-first for layout, PrimeNG for behaviour-heavy components; Angular Material's opinionated look fights a dashboard aesthetic |
| HTTP | **Angular `HttpClient` + `httpResource` (v20+)** | Native, no need for TanStack Query; signals-aware data fetching landed in v20 |
| Sidecar transport | **Tauri `plugin-shell` + PyInstaller one-file binary, JSON-over-stdio request/response** | Simplest packaging story, no FastAPI port-binding, no PyO3 GIL/linking pain |
| Python sidecar packager | **PyInstaller 6.x one-file** | Mature, fast builds, easy `-$TARGET_TRIPLE` naming for Tauri bundler; Nuitka's 2-4× runtime speedup not worth the build-time pain for I/O-bound work |
| Price/quote data | **yfinance 0.2.x (primary)** + **yahooquery (fallback)** + **FMP free tier (secondary)** | yfinance still actively maintained (1.4.0 May 2026) but fragile; layered fallbacks |
| Fundamentals (common + REIT) | **edgartools 5.x** (XBRL standardisation across 32k+ filings) + FMP as redundancy | Only library with serious cross-company concept mapping; handles 10-K/10-Q income statement, balance sheet, cash flow |
| REIT-specific (FFO/AFFO) | **edgartools XBRL pull + custom FFO/AFFO calculator** (Net Income + RE D&A ± gains/losses on property + impairments) | XBRL has `RealEstateInvestmentTrustFundsFromOperations` and component tags; manual calc gives both reported and computed FFO |
| Insider trades | **edgartools Form 3/4/5** (primary) + **openinsider.com scrape** (cluster signals) | edgartools is typed & MIT, openinsider already aggregates patterns |
| News / RSS | **feedparser** + **GDELT DOC API** | Free, no key for RSS; GDELT has structured global news with tone scores |
| Reddit | **PRAW 7.7+** (60 req/min, free non-commercial) | Official wrapper, handles backoff, still works in 2026 |
| Sentiment | **FinVADER (fast lexicon)** + **FinBERT (ProsusAI/finbert) via Hugging Face Transformers** | FinVADER for cheap per-headline scoring, FinBERT (~210 MB) for higher-quality scoring on top-N headlines; both run CPU-only |
| Forecasting | **statsforecast (AutoARIMA, AutoETS, Theta)** + **prophet** + **darts** for 1 NN model | statsforecast is 20× faster than pmdarima and a clean baseline; prophet still the readable "business" model; darts hosts NHiTS/NBEATS if we want ML horizon |
| Technicals | **pandas-ta-classic** (community fork of pandas-ta) | Pure-Python, no C-extension install pain; original pandas-ta has discontinuation risk |
| Cache | **SQLite via Tauri `plugin-sql` (sqlx)** for UI metadata + **Parquet via pyarrow** for ML training frames | Rust owns user/app state; Python owns large numeric frames where columnar speed matters |
| HTTP throttling (Python) | **httpx + tenacity + a 10 req/s limiter** | EDGAR caps at 10/s and requires `User-Agent`; built-in async support for parallel ticker pulls |
| ML store | **joblib pickles** for fitted statsforecast/prophet models | Standard; small footprint per model |

---

## Per-Layer Detail

### 1. Shell — Tauri 2

**Pick:** Tauri **2.x** (the latest 2.11.2 is current as of Feb 2026; pull the latest 2.x at scaffolding time)
**Min Rust:** 1.77.2 (required by `tauri-plugin-sql`)
**Confidence:** HIGH

**Required plugins:**

| Plugin | Purpose | Crate / npm |
|--------|---------|-------------|
| `tauri-plugin-shell` | Spawn the Python sidecar | `tauri-plugin-shell` / `@tauri-apps/plugin-shell` |
| `tauri-plugin-sql` (sqlite feature) | App cache, ticker watchlist, fetch metadata | `tauri-plugin-sql` 2.3.x / `@tauri-apps/plugin-sql` |
| `tauri-plugin-fs` | Read/write parquet cache directories | `tauri-plugin-fs` / `@tauri-apps/plugin-fs` |
| `tauri-plugin-http` | Optional: any direct calls (RSS, openinsider HTML) we let Rust make to bypass CORS | `tauri-plugin-http` / `@tauri-apps/plugin-http` |
| `tauri-plugin-store` | Lightweight key/value for user prefs (selected ticker, last refresh times) | `tauri-plugin-store` / `@tauri-apps/plugin-store` |
| `tauri-plugin-dialog` | "Add ticker" prompts, error toasts | `tauri-plugin-dialog` / `@tauri-apps/plugin-dialog` |
| `tauri-plugin-log` | Centralised logging from Rust + sidecar | `tauri-plugin-log` / `@tauri-apps/plugin-log` |
| `tauri-plugin-updater` | (Optional, later) self-update for this single-user app | — |

**Capabilities (`src-tauri/capabilities/default.json`):** must grant `shell:allow-execute` (or `shell:allow-spawn`) scoped to the sidecar binary name; grant `sql:allow-execute`, `sql:allow-load`; grant `fs:allow-read-dir`, `fs:allow-write-file` scoped to the cache dir.

---

### 2. Frontend — Angular

**Pick:** Angular **20 LTS** (entered LTS in late 2025; in LTS until ~Nov 2026) — or Angular 21 stable if comfortable upgrading at Angular 22 release (~May 2026)
**Confidence:** HIGH

**Stack inside Angular:**

| Concern | Library | Version | Why |
|---------|---------|---------|-----|
| Components | Standalone (default since v19) | n/a | No NgModule boilerplate |
| Change detection | Zoneless (stable in v20.2) | n/a | Better perf for chart re-renders, lines up with signal-based reactivity |
| State | `signal()`, `computed()`, `effect()` + plain `@Injectable({providedIn:'root'})` services | core | Personal-use app, NgRx is overkill |
| HTTP | `HttpClient` + `httpResource()` (introduced v20) | core | Native signals-aware data resource; replaces need for TanStack Query |
| Forms | Reactive forms + Signal Forms when stable (Angular 22) | core | Watchlist add/edit |
| Tables | **PrimeNG DataTable** (v19/20 matching Angular major) | ^20 | Sort/filter/freeze/lazy out of the box |
| Charts | **ngx-echarts** wrapping Apache ECharts 5.x | `ngx-echarts ^20` + `echarts ^5` | Candlestick, volume, RSI/MACD subplots, MIT/Apache |
| Styling | **Tailwind CSS v4** + PrimeNG **unstyled mode** | `tailwindcss ^4` | Utility-first; PrimeNG provides behaviour, Tailwind owns visual design |
| Icons | `lucide-angular` or PrimeIcons | latest | Lightweight icon set |
| Date | `date-fns` | ^4 | Smaller than moment, immutable |
| Test | Vitest (via `@analogjs/vitest-angular`) or Karma+Jasmine default | latest | Vitest if greenfield; default if you'd rather not configure |

**Notably NOT picked:**
- **NgRx / Akita / NGXS** — overkill for a single-user dashboard with effectively local-only state.
- **Angular Material** — opinionated Material aesthetic doesn't fit a "Bloomberg-lite" dense data view; PrimeNG + Tailwind gives more design freedom.
- **ag-grid Community** — works, but most useful grid features (pivoting, multi-row grouping) are Enterprise ($999+/dev/yr).
- **TanStack Query for Angular** — `httpResource()` covers our use case natively in v20.

---

### 3. Python Sidecar — Libraries

#### 3a. Price / Quote / Historical OHLCV

| Library | Version (May 2026) | Role | Notes |
|---------|---------------------|------|-------|
| **yfinance** | 0.2.x / 1.4.x | Primary historical OHLCV + quote | Actively maintained (release May 23 2026) but **scraping-based — rate-limited per-IP, fragile**. Cache aggressively. Use `session=` with custom UA. Confidence: MEDIUM (works today, may break tomorrow). |
| **yahooquery** | latest | Hot fallback for yfinance | Uses Yahoo's official-ish JSON endpoints, async-friendly, more stable for bulk pulls |
| **financialmodelingprep** (REST, no official Py SDK needed — use `httpx`) | n/a | Secondary fundamentals + ratio sanity check | **Free tier: 250 req/day, 500 MB/30 days, US tickers only, 5 yrs price + 5 quarters statements** — fine for 2-10 tickers with caching |
| **alpha_vantage** | latest | Optional tertiary | Free tier 25 req/day — too tight, only use for cross-check |

**Rate-limit guards (Python side):**
- `httpx.AsyncClient` + `tenacity` retry with exponential backoff + jitter
- per-source token-bucket limiter (`aiolimiter` or a hand-rolled sliding-window)
- SEC: cap at **8 req/s** (below the 10/s SEC limit), required `User-Agent: "StockFinder ryandammrose@onepointtwocapital.com"`

#### 3b. Fundamentals (Common-stock + REIT)

| Library | Version | Role | Confidence |
|---------|---------|------|------------|
| **edgartools** | 5.22+ | Primary: 10-K / 10-Q / 8-K parsing, XBRL standardisation across 32k+ filings, Form 3/4/5 insider | HIGH |
| **httpx** | latest | Direct EDGAR JSON fallback (`https://data.sec.gov/api/xbrl/companyconcept/...`) for specific tags edgartools doesn't surface | HIGH |

**REIT-specific (FFO / AFFO) strategy:**
1. Pull the income statement + cash-flow statement via `edgartools` standardised concepts.
2. Pull these specific XBRL concepts directly when present:
   - `us-gaap:NetIncomeLoss`
   - `us-gaap:DepreciationAndAmortization` and `us-gaap:DepreciationDepletionAndAmortization`
   - `us-gaap:GainsLossesOnSalesOfInvestmentRealEstate`
   - `us-gaap:AssetImpairmentCharges`
   - Some REITs also tag `RealEstateInvestmentTrustFundsFromOperations` directly (use as the reported FFO if present).
3. Compute **FFO = NetIncome + RE D&A − Gains on RE sales + Impairments** and **AFFO = FFO − recurring capex − straight-line rent adjustments** when components are available.
4. Display "**Reported FFO**" (when company disclosed) **next to** "**Computed FFO**" so the user sees both — the gap itself is a signal.

**Schema sketch (per ticker):**
```
fundamentals = {
  "common": {revenue, gross_profit, op_income, net_income, fcf, cash, total_debt, ratios},
  "reit": {ffo_reported?, ffo_computed, affo_computed?, dividend, payout_ratio_ffo, leverage}  # only populated if SIC is REIT (60xx) or user flags
}
```

Detect "is REIT" via SIC code from EDGAR submissions endpoint (Iron Mountain SIC = 6798 Real Estate Investment Trusts).

#### 3c. Insider Trades

| Source | Library | Role |
|--------|---------|------|
| SEC Form 4 (authoritative) | **edgartools** `Form` 3/4/5 typed objects | Primary; gives transactions with proper field types |
| **openinsider.com** scrape | `httpx` + `selectolax` (faster than BeautifulSoup) | Secondary; gives precomputed cluster pages (e.g. `/screener?...&cluster=1`) — saves us reimplementing pattern detection in v1 |

**Cluster/pattern detection (do in pandas):**
- Group Form 4s per ticker per 30/60/90-day window
- Flag when **N distinct insiders** with **net buying USD > threshold** in the window
- Flag sustained net-buying / net-selling trend across rolling 90-day windows

#### 3d. Sentiment

**Headlines / news (free):**

| Source | Method |
|--------|--------|
| Yahoo Finance ticker news | via `yfinance.Ticker.news` (lightweight) |
| Finviz news block | scrape with `httpx` + `selectolax` (rate-limit kindly) |
| RSS feeds (Seeking Alpha free, Reuters business, Bloomberg public RSS, WSJ markets) | **feedparser** |
| **GDELT DOC 2.0 API** | Direct HTTP — global news with tone scores, 250 results/query, no key needed |

**Reddit:**

| Library | Notes |
|---------|-------|
| **PRAW 7.7+** | 60 req/min OAuth quota; free for non-commercial; subreddits: `wallstreetbets`, `investing`, `stocks`, `SecurityAnalysis`, `dividends`, plus ticker-specific (`MU_Stock` if it exists, `REIT`, `IronMountain`) |
| Fallback: raw `.json` endpoints | No auth, but Reddit may block IPs; **don't use as primary** |

**Tone scoring (two-tier):**

| Tier | Library | When |
|------|---------|------|
| Fast lexicon | **FinVADER** (pip `finvader`) | Score every headline / post — cheap |
| Quality | **FinBERT** via Hugging Face `transformers` + `ProsusAI/finbert` (~210 MB FP16, ~52 MB int4) | Score top-N (e.g. top 20 most-recent or most-upvoted) per ticker per refresh, CPU-only fine |

**Do NOT use:** Plain VADER alone (44% accuracy on financial text per published evaluations); Twitter/X API (already out of scope per PROJECT.md).

#### 3e. Forecasting (3 horizons × 3 models, "show side-by-side, don't blend")

| Tier | Library | Model | Use for |
|------|---------|-------|---------|
| Analyst consensus | yfinance `Ticker.analyst_price_targets` + FMP `/api/v3/analyst-estimates/{ticker}` | n/a | "What does the street say" — straight pass-through |
| Statistical baseline | **statsforecast** | AutoARIMA, AutoETS, Theta | "Dumb baseline" — 20× faster than pmdarima, Numba-accelerated, runs in <1s per ticker |
| Readable model | **prophet** | Prophet | Easy interpretability (trend + seasonality components shown to user) |
| ML | **darts** | NHiTS or NBEATS | Single deep-learning model; only fit if user opts in (slow) |

**What NOT to use (v1):**
- **neuralforecast** — overkill, slow training, model-zoo complexity not needed for 2 tickers
- **xgboost / lightgbm** for direct price prediction — easy to overfit; tree models are better as feature-driven classifiers (e.g. "is up vs down next 30 days") if we later add that surface
- **TensorFlow / PyTorch from scratch** — use darts/neuralforecast wrappers, do not hand-roll

**Horizons:** 1-3 month (daily), 3-12 month (weekly resampled), 1-3 year (monthly resampled) — refit per horizon to avoid pollution.

#### 3f. Technical Indicators

| Library | Version | Pick? | Why |
|---------|---------|-------|-----|
| **pandas-ta-classic** | 0.4.x | **YES — primary** | Pure-Python, MIT, community-maintained fork of pandas-ta with active commits + tests; pip install just works |
| pandas-ta (original) | 0.3.14b | NO | Maintainer flagged sustainability risk; classic fork is the active one |
| TA-Lib | 0.4.x | NO (default) | Faster (C), but install pain on macOS/Linux/Windows; brew/apt dance; not worth it for single-user app |
| finta | latest | NO | Smaller indicator list; pandas-ta-classic covers everything we need |

Indicators in v1: SMA(20/50/200), EMA(12/26), MACD(12,26,9), RSI(14), Bollinger(20,2), volume SMA, OBV, ATR(14).

---

### 4. Sidecar Invocation Pattern (Tauri ↔ Python)

**Pick:** **PyInstaller one-file binary, spawned per-request via `tauri-plugin-shell`, JSON request on argv or stdin, JSON response on stdout, exit on completion.**
**Confidence:** HIGH

**Why this over the alternatives:**

| Pattern | Pros | Cons | Verdict |
|---------|------|------|---------|
| **Subprocess (PyInstaller bundle)** ← pick | Trivial packaging; clean per-call isolation; crash in Python ≠ crash in app; works in dev with `cargo tauri dev`; matches Tauri sidecar docs exactly | ~1-3 s cold start per call (PyInstaller bootstrap); pays interpreter startup per request | **Right default** for personal-use on-demand fetch |
| Long-lived FastAPI/Uvicorn sidecar on `127.0.0.1:0` (random port) | Warm interpreter, sub-100 ms requests, HTTP semantics natural | Port-binding pain, antivirus alarms, lifecycle mgmt complexity, need health-check + restart, harder to bundle | Use **only if** profiling shows cold start is the bottleneck |
| **PyO3 embedded interpreter** in Rust | Sub-ms IPC, shared address space, no subprocess | GIL serialises Python work; static linking is "many complications" per PyO3 docs; ships single binary but build is fiddly; bundling numpy/pandas via PyO3 is rough | Wrong for a sidecar that uses pandas/scikit/transformers |
| Bundled `python3` + script via shell | Easiest dev iteration | Won't run on user machines without Python preinstalled; defeats "desktop app" promise | NO |

**Concrete pattern:**

```
src-tauri/
  binaries/
    stockfinder-py-aarch64-apple-darwin   # PyInstaller one-file for Apple Silicon
    stockfinder-py-x86_64-apple-darwin    # Intel Mac
    stockfinder-py-x86_64-pc-windows-msvc.exe
    stockfinder-py-x86_64-unknown-linux-gnu
  tauri.conf.json:
    "bundle": { "externalBin": ["binaries/stockfinder-py"] }
  capabilities/default.json:
    permissions: ["shell:allow-execute", { "identifier": "shell:scope", "allow": [{ "name": "binaries/stockfinder-py", "sidecar": true }] }]
```

**Request envelope (Rust → Python on stdin):**
```json
{"op": "fetch_fundamentals", "ticker": "MU", "args": {"horizon_years": 5}, "request_id": "uuid"}
```

**Response envelope (Python → Rust on stdout, newline-delimited JSON):**
```json
{"request_id": "uuid", "ok": true, "data": {...}}
{"request_id": "uuid", "ok": false, "error": {"code": "RATE_LIMITED", "msg": "..."}}
```

Use **NDJSON** so the sidecar can stream progress events for slow operations (`{"request_id":"...","ok":true,"progress":0.4}`) before the final result.

---

### 5. Packaging

**Pick:** PyInstaller 6.x one-file, built **per target triple** in CI, output renamed to `<bin>-<triple>` and dropped into `src-tauri/binaries/`.
**Confidence:** HIGH (for macOS); MEDIUM (cross-platform — needs CI matrix you don't have yet)

**macOS build (immediate):**
```bash
pyinstaller --onefile --name stockfinder-py --noconfirm \
  --collect-data finvader \
  --collect-data prophet \
  --hidden-import sklearn.utils._typedefs \
  python/main.py
# Native target on M-series Mac:
mv dist/stockfinder-py src-tauri/binaries/stockfinder-py-aarch64-apple-darwin
```

For Intel Mac you must build on Intel (or use `arch -x86_64` + rosetta'd Python). For cross-platform later, use GitHub Actions matrix `macos-13` (x86), `macos-14` (arm), `windows-latest`, `ubuntu-latest`.

**Alternatives considered:**

| Tool | Why not (for v1) |
|------|------------------|
| Nuitka | 2-4× faster runtime, but builds are 10× slower, more fragile transformer-dep handling, and our workload is I/O-bound (network calls dominate) — no payoff |
| BeeWare Briefcase | Targets full app distribution, not "binary that Tauri bundles" — wrong abstraction |
| PyOxidizer | Cool, but minimal recent maintenance and weaker numpy/pandas story |
| `python -m venv` shipped alongside | Yes if user is dev; no for a polished desktop app |

**Bundle size budget:** PyInstaller bundle with yfinance + edgartools + statsforecast + prophet + transformers + finbert weights ≈ **400-600 MB**. That's the FinBERT model's fault. Option to keep FinBERT model out of the bundle and **download-on-first-use** to the user cache dir to keep the installer under ~150 MB.

---

### 6. Caching Layer

**Pick:** **Hybrid.** SQLite (Tauri plugin-sql, sqlx) for UI-side metadata; Parquet files on disk for the heavy numeric frames.
**Confidence:** HIGH

| Data | Storage | Why |
|------|---------|-----|
| Watchlist (tickers, display order) | SQLite via `plugin-sql` | Read on app open; written by Angular |
| Last-refresh timestamps per ticker per source | SQLite | UI badges "fundamentals refreshed 2h ago" |
| Cache index (which parquet files exist, expiry) | SQLite | Lookup before reading parquet |
| Historical OHLCV (per ticker, per resolution) | Parquet via `pyarrow` | Columnar, compresses ~5×, pandas reads in one call |
| ML training frames (engineered features) | Parquet | Same reasons |
| Fitted forecast models | `joblib.dump()` `.pkl` files | Standard for sklearn/statsforecast/prophet |
| News headlines + sentiment scores | SQLite (one table, indexed by ticker+ts) | Tiny rows, frequent queries, good for join with fundamentals view |
| Insider transactions | SQLite | Same — tabular, queryable |
| FinBERT model weights | Plain file in cache dir (Hugging Face default) | Downloaded once |

**Cache dir (cross-platform via Tauri `plugin-path::app_cache_dir()`):**
- macOS: `~/Library/Caches/com.onepointtwocapital.stockfinder/`
- Windows: `%LOCALAPPDATA%\com.onepointtwocapital.stockfinder\Cache\`
- Linux: `~/.cache/com.onepointtwocapital.stockfinder/`

**Schema sketch (SQLite):**
```sql
CREATE TABLE watchlist (ticker TEXT PRIMARY KEY, position INTEGER, added_at INTEGER);
CREATE TABLE refresh_log (ticker TEXT, source TEXT, refreshed_at INTEGER, ok INTEGER, PRIMARY KEY (ticker, source));
CREATE TABLE cache_index (key TEXT PRIMARY KEY, path TEXT, expires_at INTEGER);
CREATE TABLE headlines (id INTEGER PRIMARY KEY, ticker TEXT, source TEXT, ts INTEGER, title TEXT, url TEXT, finvader_score REAL, finbert_label TEXT, finbert_score REAL);
CREATE TABLE insider_tx (id INTEGER PRIMARY KEY, ticker TEXT, filer TEXT, role TEXT, ts INTEGER, kind TEXT, shares REAL, price REAL, value REAL);
CREATE INDEX idx_headlines_ticker_ts ON headlines(ticker, ts DESC);
CREATE INDEX idx_insider_ticker_ts ON insider_tx(ticker, ts DESC);
```

**TTLs (sane defaults — let user override):**
- Intraday quote: 5 min
- Daily OHLCV: until end of US trading day
- Fundamentals (10-K/10-Q): until next filing date or 24 h, whichever sooner
- Insider transactions: 6 h
- News + sentiment: 1 h
- Reddit: 30 min
- Analyst targets: 24 h

---

### 7. Free-tier rate-limit cheat sheet (for the rate-limiter config)

| Source | Limit | Auth | Notes |
|--------|-------|------|-------|
| yfinance (Yahoo unofficial) | "scraping-based, undocumented", expect blocks above ~1 req/s sustained | none | Use cache aggressively; rotate user-agent; backoff on `YFRateLimitError` |
| yahooquery | Higher tolerance than yfinance; still unofficial | none | Same caveats |
| SEC EDGAR | **10 req/s hard cap**, IP banned briefly if exceeded | none, but `User-Agent` required (format: `"Company contact@example.com"`) | We'll cap at 8/s |
| FMP free tier | **250 req/day, 500 MB/30 days, US only, 5y price + 5q statements** | API key (free) | Strict — keep as secondary |
| Alpha Vantage free | 25 req/day | API key | Too tight — only for sanity check |
| Reddit (PRAW OAuth) | **60 req/min**, 10-min rolling window | OAuth (free non-commercial) | PRAW respects `X-Ratelimit-*` automatically |
| GDELT DOC 2.0 | ~250 results/query, generous overall | none | Solid |
| openinsider.com | Unofficial; scrape gently (≤1 req/s) | none | Cache 6h |
| Hugging Face model download | 1× per model | none for public | Cache locally |

---

### 8. Installation (concrete commands)

**Tauri scaffold:**
```bash
cargo install create-tauri-app --locked
npm create tauri-app@latest stock-finder -- --template angular --manager pnpm
cd stock-finder
pnpm tauri add shell
pnpm tauri add sql
pnpm tauri add fs
pnpm tauri add store
pnpm tauri add dialog
pnpm tauri add log
pnpm tauri add http
```

**Angular extras:**
```bash
pnpm add ngx-echarts echarts primeng primeicons tailwindcss@^4 @tailwindcss/postcss lucide-angular date-fns
```

**Python sidecar (`python/pyproject.toml` deps):**
```
yfinance>=0.2.50
yahooquery
edgartools>=5.22
httpx[http2]>=0.27
tenacity>=8
aiolimiter>=1.1
selectolax
feedparser
praw>=7.7
finvader
transformers>=4.40
torch>=2.3       # CPU wheel; or onnxruntime as a lighter alternative
pandas>=2.2
pyarrow>=15
statsforecast>=2
prophet>=1.1.5
darts>=0.31      # optional, for NHiTS
pandas-ta-classic
scikit-learn>=1.5
joblib
pyinstaller>=6.6
```

---

## Alternatives Considered (summary table)

| Decision | Picked | Considered | Why rejected |
|----------|--------|------------|--------------|
| Shell | Tauri 2 | Electron | Bigger bundle, higher RAM, Node runtime overhead |
| Frontend | Angular 20 LTS | React/Solid | Already a user preference + project constraint |
| State | Signals + services | NgRx, Akita, NGXS | Boilerplate-heavy, overkill for single-user app |
| Charts | ngx-echarts | Highcharts, ApexCharts, ng2-charts, Plotly.js | Highcharts non-commercial-only free license is a footgun; ApexCharts good but ECharts has stronger finance ecosystem; Chart.js lacks polished candlesticks |
| Tables | PrimeNG DataTable | ag-grid Community, Angular Material table | ag-grid Community is feature-thin; ag-grid Enterprise is paid; Material table needs too much hand-built UI |
| Styling | Tailwind v4 + PrimeNG unstyled | Angular Material | Material's opinionated look fights a dense data dashboard |
| Sidecar transport | subprocess + NDJSON over stdio | FastAPI sidecar, PyO3 embed, gRPC | FastAPI: port-binding + lifecycle mgmt; PyO3: GIL + linking pain; gRPC: way too much for 2 processes |
| Python packager | PyInstaller | Nuitka, Briefcase, PyOxidizer | Nuitka: 10× slower builds for no benefit on I/O work; Briefcase wrong abstraction; PyOxidizer underactive |
| Fundamentals | edgartools | sec-edgar-downloader, raw EDGAR JSON, sec-api.io | edgartools has industry-aware concept mapping; raw JSON needs hand-rolled normalisation; sec-api.io is paid |
| Insider trades | edgartools + openinsider | sec-api.io insider endpoints | sec-api.io paid; openinsider gives precomputed clusters free |
| Forecast statistical | statsforecast | pmdarima | statsforecast is 20× faster, Numba-accelerated, actively maintained |
| Sentiment | FinVADER + FinBERT | Plain VADER, TextBlob, GPT API | VADER alone 44% accuracy on finance text; TextBlob worse; GPT API costs money + adds latency |
| Technicals | pandas-ta-classic | TA-Lib, pandas-ta original, finta | TA-Lib install pain; pandas-ta original maintenance-risk; finta less complete |
| Cache | SQLite + Parquet hybrid | SQLite only, DuckDB, plain JSON | SQLite alone bloats with large frames; DuckDB nice but adds a dep; JSON is slow + uncompressed |

---

## Anti-Recommendations (do NOT use)

| ❌ Don't use | Why |
|--------------|-----|
| **Next.js, Remix, Nuxt, SvelteKit** | SSR frameworks targeting a server; useless inside a Tauri webview |
| **Electron** | Already excluded by stack constraint |
| **NgRx / NGXS / Akita** | Heavy ceremony; Angular signals + services cover personal-app state |
| **Angular Material** for primary UI | Opinionated Material look fights dense finance dashboard |
| **ag-grid Enterprise** | Paid; Community tier is too thin |
| **Highcharts (commercial)** | License is non-commercial-only free; locks future flexibility |
| **TA-Lib** (default) | C-extension install pain; pandas-ta-classic covers the indicators we want |
| **pandas-ta** (original) | Maintainer-flagged sustainability risk; use the `pandas-ta-classic` fork |
| **Plain VADER** for finance sentiment | 44% F1 on financial text; FinVADER or FinBERT instead |
| **Twitter/X API** | Already out of scope per PROJECT.md |
| **Alpha Vantage** as primary | 25 req/day free tier is unusable |
| **IEX Cloud** | Shut down free tier; deprecated |
| **Polygon.io paid** | Out of scope (free only) |
| **Bloomberg / Refinitiv** | Out of scope |
| **PyO3 embedding** for the sidecar | GIL + static-link pain for pandas/transformers; subprocess is simpler |
| **FastAPI sidecar** as default | Port-binding, lifecycle, antivirus alarms — use only if cold-start measured to be a real problem |
| **Nuitka** (default) | 10× slower builds for negligible runtime gain on I/O-bound work |
| **TensorFlow / PyTorch hand-rolled forecasting** | Use darts/neuralforecast wrappers if you want NN; don't hand-roll |
| **neuralforecast** (v1) | Overkill for 2 tickers; revisit later |
| **xgboost/lightgbm for direct price regression** | Easy to overfit; use as classifier on engineered features if ever needed |
| **Server-side DB (Postgres, Mongo, etc.)** | Already excluded by "no server DB" constraint |
| **Background workers / cron / systemd timers** | Excluded by "on-demand fetch only" constraint |
| **Raw Reddit JSON as primary** | Reddit increasingly blocks unauthenticated scraping; PRAW with OAuth is the legitimate path |
| **Selenium / Playwright as primary fetcher** | Heavy, slow, fragile; use only as last-resort fallback when no API/RSS exists |
| **Twitter sentiment libraries (tweepy etc.)** | API too restrictive; project explicitly defers Twitter/X |

---

## Confidence Levels (per recommendation cluster)

| Area | Confidence | Reason |
|------|------------|--------|
| Tauri 2 core + plugin choices | HIGH | Verified against current v2 docs and plugin versions (May 2026) |
| Angular 20 LTS + signals + standalone | HIGH | Matches official release/LTS timeline (HeroDevs, Angular.dev) |
| Chart library (ngx-echarts) | HIGH | OSS, mature, finance-feature-complete, licence-safe |
| Table library (PrimeNG) | HIGH | Industry default for analytics dashboards, MIT |
| Sidecar = PyInstaller + stdio JSON | HIGH | Matches Tauri docs + community-validated pattern (MClare, dieharders/example-tauri-v2-python-server-sidecar) |
| yfinance as primary price source | MEDIUM | Works May 2026 but **fragile** by design (scraping); fallbacks necessary |
| edgartools for fundamentals + insider | HIGH | Actively maintained, MIT, XBRL standardisation across 32k+ filings |
| FFO/AFFO computation strategy | MEDIUM | Direct XBRL tag for FFO exists for some filers; for the rest we compute — works but worth validating against IRM's actual 10-K |
| Sentiment two-tier (FinVADER + FinBERT) | MEDIUM | Standard pattern but personal taste / quality-vs-latency tradeoff |
| Forecasting trio (statsforecast + prophet + darts) | MEDIUM | Solid library choices but "which model" and "what horizon" are inherently experimental for stock prices |
| pandas-ta-classic | MEDIUM-HIGH | Fork is active but smaller ecosystem than original; could swap to TA-Lib later if perf needed |
| SQLite + Parquet cache hybrid | HIGH | Standard desktop-app pattern; both libraries well understood |
| Free-tier rate-limit numbers | HIGH | Verified from primary sources (FMP pricing page, SEC announcement, PRAW docs, yfinance issue tracker) |

---

## Open Questions (defer to per-phase research)

1. **FFO disclosure tagging in IRM's actual 10-K** — does Iron Mountain tag `RealEstateInvestmentTrustFundsFromOperations` directly, or do we always have to compute it? (Phase: Fundamentals.)
2. **MU forecast model quality** — semiconductor cyclicality means simple ARIMA may misfire badly around inventory turns; may need volume + capex covariates. (Phase: Forecasting.)
3. **FinBERT inference speed on user laptop** — benchmark needed; if too slow on Intel Macs, swap to ONNX export or distilled variant. (Phase: Sentiment.)
4. **Reddit OAuth account setup UX** — first-run flow to register the user's own Reddit app credentials, or ship a shared client_id? (Phase: Sentiment.)
5. **yfinance circuit-breaker pattern** — when Yahoo blocks, how aggressively do we degrade to yahooquery + FMP? (Phase: Data fetch.)
6. **Update / distribution channel** — Tauri updater + signed installers requires Apple Developer cert + Windows code-signing cert. (Phase: Packaging.)
7. **Whether to add DuckDB** later if cross-source joins on parquet get common. (Phase: Cache.)

---

## Sources

- [Tauri 2 sidecar docs](https://v2.tauri.app/develop/sidecar/)
- [Tauri plugin-sql docs](https://v2.tauri.app/plugin/sql/)
- [Tauri plugin-shell docs](https://v2.tauri.app/plugin/shell/)
- [Tauri Python sidecar example (dieharders)](https://github.com/dieharders/example-tauri-v2-python-server-sidecar)
- [Writing a pandas sidecar for Tauri (MClare)](https://mclare.blog/posts/writing-a-pandas-sidecar-for-tauri/)
- [Tauri release notes](https://github.com/tauri-apps/tauri/releases)
- [Angular versioning and releases](https://angular.dev/reference/releases)
- [Angular 20 release notes (Angular.love)](https://angular.love/angular-20-whats-new)
- [Angular 19 standalone default (Angular blog)](https://blog.angular.dev/the-future-is-standalone-475d7edbc706)
- [HeroDevs Angular EOL timeline](https://www.herodevs.com/blog-posts/angular-version-history-every-release-date-support-window-and-end-of-life-date-from-angularjs-to-angular-22)
- [Angular signals vs NgRx 2026 guide](https://prodevaihub.com/angular-signals-vs-ngrx-store/)
- [Angular chart libraries comparison (Weavelinx)](https://weavelinx.com/best-chart-libraries-for-angular-projects-in-2026/)
- [PrimeNG vs Angular Material 2026 (Syncfusion)](https://www.syncfusion.com/blogs/post/angular-material-vs-primeng)
- [ag-grid alternatives 2026](https://www.simple-table.com/blog/ag-grid-alternatives-free-angular-data-grids-2026)
- [TailAdmin Angular 20 + Tailwind v4 dashboard template](https://tailadmin.com/angular)
- [yfinance PyPI](https://pypi.org/project/yfinance/)
- [Why yfinance keeps getting blocked (Medium)](https://medium.com/@trading.dude/why-yfinance-keeps-getting-blocked-and-what-to-use-instead-92d84bb2cc01)
- [yfinance rate-limit issue thread](https://github.com/ranaroussi/yfinance/issues/2289)
- [yfinance vs yahooquery discussion](https://github.com/ranaroussi/yfinance/discussions/1420)
- [edgartools GitHub](https://github.com/dgunning/edgartools)
- [edgartools docs](https://edgartools.readthedocs.io/)
- [SEC EDGAR rate-limit announcement](https://www.sec.gov/filergroup/announcements-old/new-rate-control-limits)
- [SEC EDGAR API guide (tldrfiling)](https://tldrfiling.com/blog/sec-edgar-api-rate-limits-best-practices)
- [FMP pricing](https://site.financialmodelingprep.com/pricing-plans)
- [FMP FAQs (rate limits)](https://site.financialmodelingprep.com/faqs)
- [PRAW rate limits docs](https://praw.readthedocs.io/en/stable/getting_started/ratelimits.html)
- [Reddit API pricing 2026](https://www.redditcommentscraper.com/article-reddit-api-pricing-alternative.html)
- [OpenInsider scraper](https://github.com/sd3v/openinsiderData)
- [FinVADER on PyPI](https://pypi.org/project/finvader/)
- [FinBERT (ProsusAI) on Hugging Face](https://huggingface.co/ProsusAI/finbert)
- [FinBERT model memory requirements](https://huggingface.co/ProsusAI/finbert/discussions/20)
- [VADER vs FinBERT evaluation (ACM)](https://dl.acm.org/doi/fullHtml/10.1145/3677052.3698675)
- [pandas-ta-classic](https://github.com/xgboosted/pandas-ta-classic)
- [pandas-ta-classic on PyPI](https://pypi.org/project/pandas-ta-classic/)
- [TA-Lib vs pandas-ta comparison (Sling Academy)](https://www.slingacademy.com/article/comparing-ta-lib-to-pandas-ta-which-one-to-choose/)
- [statsforecast (Nixtla)](https://github.com/Nixtla/statsforecast)
- [statsforecast AutoARIMA docs](https://nixtlaverse.nixtla.io/statsforecast/docs/models/autoarima.html)
- [Time series 2026 overview (FutureAGI)](https://futureagi.com/blog/time-series-data-analysis-2025/)
- [PyInstaller vs Nuitka vs cx_Freeze 2026 (AhmedSyntax)](https://ahmedsyntax.com/2026-comparison-pyinstaller-vs-cx-freeze-vs-nui/)
- [Nuitka vs PyInstaller deep dive (KRRT7)](https://krrt7.dev/en/blog/nuitka-vs-pyinstaller)
- [Building production Tauri + FastAPI + PyInstaller (aiechoes)](https://aiechoes.substack.com/p/building-production-ready-desktop)
- [PyO3 embedding discussion](https://github.com/PyO3/pyo3/discussions/4102)
- [PyO3 user guide — building and distribution](https://pyo3.rs/main/building-and-distribution)
