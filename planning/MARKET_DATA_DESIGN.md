# Market Data Backend — Implementation Design

Reference document for the market data subsystem. The code lives in `backend/app/market/` and is **already fully implemented and tested** (73 tests, 84% coverage). This document serves as the authoritative technical reference for:

- Engineers wiring the market data layer into the portfolio, watchlist, and chat features
- Anyone needing to understand the architecture, extension points, or integration contracts

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Data Model — `models.py`](#2-data-model)
3. [Price Cache — `cache.py`](#3-price-cache)
4. [Abstract Interface — `interface.py`](#4-abstract-interface)
5. [GBM Simulator — `simulator.py`](#5-gbm-simulator)
6. [Massive API Client — `massive_client.py`](#6-massive-api-client)
7. [Factory — `factory.py`](#7-factory)
8. [SSE Streaming — `stream.py`](#8-sse-streaming)
9. [FastAPI Lifecycle Integration](#9-fastapi-lifecycle-integration)
10. [Watchlist Coordination](#10-watchlist-coordination)
11. [Ticker Set: Watchlist ∪ Positions](#11-ticker-set-watchlist--positions)
12. [Per-Ticker Freshness & Staleness](#12-per-ticker-freshness--staleness)
13. [SSE Heartbeats](#13-sse-heartbeats)
14. [Frontend Integration Contract](#14-frontend-integration-contract)
15. [Error Handling & Edge Cases](#15-error-handling--edge-cases)
16. [Configuration Summary](#16-configuration-summary)

---

## 1. Architecture Overview

```
MASSIVE_API_KEY set?
       │
   yes │          no
       ▼          ▼
MassiveDataSource  SimulatorDataSource
       │                │
       └──────┬─────────┘
              ▼
         PriceCache  (thread-safe, in-memory, version-stamped)
              │
    ┌─────────┼──────────────┐
    ▼         ▼              ▼
SSE stream  Portfolio     Trade execution
/api/stream/prices  valuation  (price lookup)
```

**Strategy pattern.** Both sources implement `MarketDataSource` (ABC). All downstream code — SSE, portfolio, trades — reads from `PriceCache`. No downstream code imports `SimulatorDataSource` or `MassiveDataSource` directly.

**Single background task.** One asyncio task (the simulator loop or the Massive poller) writes to the cache. One FastAPI app reads from it. No concurrency between writers.

**Public API** (import from `app.market`):

```python
from app.market import (
    PriceUpdate,               # Immutable price snapshot
    PriceCache,                # Thread-safe in-memory store
    MarketDataSource,          # ABC for type hints
    create_market_data_source, # Factory: picks simulator or Massive
    create_stream_router,      # FastAPI SSE router factory
)
```

---

## 2. Data Model

**File:** `backend/app/market/models.py`

```python
@dataclass(frozen=True, slots=True)
class PriceUpdate:
    ticker: str
    price: float            # Rounded to 2 decimal places
    previous_price: float   # Price at the previous update
    timestamp: float        # Unix seconds (float)

    @property
    def change(self) -> float: ...         # price - previous_price, rounded to 4dp
    @property
    def change_percent(self) -> float: ... # %, rounded to 4dp
    @property
    def direction(self) -> str: ...        # "up" | "down" | "flat"

    def to_dict(self) -> dict: ...         # For JSON / SSE serialization
```

**Serialized shape** (used in SSE events and any API that exposes prices):

```json
{
  "ticker": "AAPL",
  "price": 192.05,
  "previous_price": 191.88,
  "timestamp": 1750000000.123,
  "change": 0.17,
  "change_percent": 0.0887,
  "direction": "up"
}
```

**Note:** `PriceUpdate` does not yet carry a `status` field (`"live"` / `"stale"`). See [§12](#12-per-ticker-freshness--staleness) for the extension plan.

---

## 3. Price Cache

**File:** `backend/app/market/cache.py`

Thread-safe with a `threading.Lock`. Safe to call from both asyncio tasks (`asyncio.to_thread`) and the main event loop.

```python
cache = PriceCache()

# Write (called by the data source background task)
update: PriceUpdate = cache.update("AAPL", 192.05)
# → first call: previous_price == price, direction == "flat"
# → subsequent calls: previous_price == last price, direction computed

# Read
update: PriceUpdate | None = cache.get("AAPL")
price:  float        | None = cache.get_price("AAPL")   # convenience
all:    dict[str, PriceUpdate] = cache.get_all()         # shallow copy

# Delete (watchlist removal)
cache.remove("AAPL")

# Change detection (used by SSE)
version: int = cache.version   # monotonically increasing, bumped on every update
```

**Version counter** — the cache bumps `_version` on every `update()` call. The SSE generator compares `current_version != last_version` to decide whether to push an event. This means:

- Simulator (500ms updates): events fire every ~500ms per changed ticker
- Massive free tier (15s poll): events fire only when new data arrives, not on every SSE interval check
- No spurious empty events are ever sent

---

## 4. Abstract Interface

**File:** `backend/app/market/interface.py`

```python
class MarketDataSource(ABC):
    async def start(self, tickers: list[str]) -> None: ...
    async def stop(self) -> None: ...
    async def add_ticker(self, ticker: str) -> None: ...
    async def remove_ticker(self, ticker: str) -> None: ...
    def get_tickers(self) -> list[str]: ...
```

**Lifecycle contract:**

```python
# At app startup (FastAPI lifespan)
source = create_market_data_source(cache)
await source.start(initial_tickers)   # starts background task, seeds cache

# During operation (watchlist/trade endpoints)
await source.add_ticker("PYPL")       # added to next update cycle
await source.remove_ticker("NFLX")   # removed from cycle, removed from cache

# At app shutdown
await source.stop()                   # cancels background task, safe to call twice
```

`start()` must be called exactly once. `stop()` is idempotent.

---

## 5. GBM Simulator

**Files:** `backend/app/market/simulator.py`, `backend/app/market/seed_prices.py`

### Math

Geometric Brownian Motion:

```
S(t+dt) = S(t) × exp((μ - σ²/2)·dt  +  σ·√dt·Z)
```

Where:
- `S(t)` = current price
- `μ` = annualized drift (per-ticker, e.g. 0.05 for 5%/year)
- `σ` = annualized volatility (per-ticker, e.g. 0.22 for AAPL)
- `dt` = `0.5 / (252 × 6.5 × 3600)` ≈ 8.48×10⁻⁸ (500ms as fraction of trading year)
- `Z` = correlated standard normal (via Cholesky)

At this `dt`, each 500ms tick moves prices by ~`σ × √dt` ≈ 0.006% for AAPL — imperceptible per tick, but visually realistic over minutes.

### Correlated Moves

```python
# Sector groups defined in seed_prices.py
CORRELATION_GROUPS = {
    "tech":    {"AAPL", "GOOGL", "MSFT", "AMZN", "META", "NVDA", "NFLX"},
    "finance": {"JPM", "V"},
}

# Correlation coefficients
INTRA_TECH_CORR    = 0.6   # Tech stocks move together
INTRA_FINANCE_CORR = 0.5   # Finance stocks move together
TSLA_CORR          = 0.3   # TSLA does its own thing
CROSS_GROUP_CORR   = 0.3   # Between sectors, or unknown tickers
```

The `GBMSimulator` builds an `n×n` correlation matrix and takes its Cholesky decomposition `L` such that `L @ z_independent` gives correlated draws. Rebuilt via `_rebuild_cholesky()` whenever tickers are added/removed. O(n²) but `n < 50` in practice.

### Random Shock Events

```python
# ~0.1% chance per tick per ticker of a sudden 2-5% move
if random.random() < 0.001:
    shock = random.uniform(0.02, 0.05) * random.choice([-1, 1])
    price *= (1 + shock)
```

With 10 tickers at 2 ticks/sec, expect ~1 shock event every 50 seconds. Logged at DEBUG level.

### Seed Prices & Parameters

```python
# seed_prices.py — starting prices and per-ticker GBM params
SEED_PRICES = {
    "AAPL": 190.00, "GOOGL": 175.00, "MSFT": 420.00,
    "AMZN": 185.00, "TSLA": 250.00,  "NVDA": 800.00,
    "META": 500.00, "JPM":  195.00,  "V":    280.00,
    "NFLX": 600.00,
}

TICKER_PARAMS = {
    "AAPL": {"sigma": 0.22, "mu": 0.05},
    "TSLA": {"sigma": 0.50, "mu": 0.03},  # High vol
    "NVDA": {"sigma": 0.40, "mu": 0.08},  # High vol, strong drift
    "JPM":  {"sigma": 0.18, "mu": 0.04},  # Low vol (bank)
    # ...
}

DEFAULT_PARAMS = {"sigma": 0.25, "mu": 0.05}  # For dynamically added tickers
```

Unknown tickers (not in `SEED_PRICES`) get a random starting price in `[50, 300]` and default params. Add real seed prices as needed.

### `SimulatorDataSource` Implementation Pattern

```python
class SimulatorDataSource(MarketDataSource):
    async def start(self, tickers: list[str]) -> None:
        self._sim = GBMSimulator(tickers=tickers)
        # Seed cache immediately — SSE has data from the first event
        for ticker in tickers:
            price = self._sim.get_price(ticker)
            if price is not None:
                self._cache.update(ticker=ticker, price=price)
        self._task = asyncio.create_task(self._run_loop())

    async def _run_loop(self) -> None:
        while True:
            try:
                prices = self._sim.step()
                for ticker, price in prices.items():
                    self._cache.update(ticker=ticker, price=price)
            except Exception:
                logger.exception("Simulator step failed")
            await asyncio.sleep(0.5)

    async def add_ticker(self, ticker: str) -> None:
        self._sim.add_ticker(ticker)           # rebuilds Cholesky
        price = self._sim.get_price(ticker)    # seeds immediately
        if price is not None:
            self._cache.update(ticker=ticker, price=price)

    async def remove_ticker(self, ticker: str) -> None:
        self._sim.remove_ticker(ticker)        # rebuilds Cholesky
        self._cache.remove(ticker)             # purge from cache
```

---

## 6. Massive API Client

**File:** `backend/app/market/massive_client.py`

Wraps the `massive` Python package (Polygon.io REST API). The `RESTClient` is synchronous, so every call runs in `asyncio.to_thread()`.

### Poll Cycle

```python
async def _poll_once(self) -> None:
    snapshots = await asyncio.to_thread(self._fetch_snapshots)
    for snap in snapshots:
        price = snap.last_trade.price
        timestamp = snap.last_trade.timestamp / 1000.0  # ms → seconds
        self._cache.update(ticker=snap.ticker, price=price, timestamp=timestamp)

def _fetch_snapshots(self) -> list:
    # Runs in a thread pool — not on the event loop
    return self._client.get_snapshot_all(
        market_type=SnapshotMarketType.STOCKS,
        tickers=self._tickers,
    )
```

### Rate Limit Guidance

| Tier | Requests/min | Recommended poll interval |
|------|-------------|--------------------------|
| Free | 5 | 15s (default) |
| Starter | 30 | 5s |
| Developer | 300 | 2s |

The `poll_interval` constructor parameter defaults to `15.0`. Override via:

```python
MassiveDataSource(api_key=key, price_cache=cache, poll_interval=5.0)
```

### Failure Behavior

Poll failures are logged at `ERROR` level and swallowed — the loop continues. The cache retains the last-known prices. Common failure modes:

- `401` — bad API key
- `429` — rate limit exceeded (increase `poll_interval`)
- Network error — transient, retries on next cycle

---

## 7. Factory

**File:** `backend/app/market/factory.py`

```python
def create_market_data_source(price_cache: PriceCache) -> MarketDataSource:
    api_key = os.environ.get("MASSIVE_API_KEY", "").strip()
    if api_key:
        return MassiveDataSource(api_key=api_key, price_cache=price_cache)
    else:
        return SimulatorDataSource(price_cache=price_cache)
```

Reads `MASSIVE_API_KEY` from the environment. Returns an **unstarted** source. The caller must `await source.start(tickers)`.

---

## 8. SSE Streaming

**File:** `backend/app/market/stream.py`

### Creating the Router

```python
from app.market import PriceCache, create_stream_router

cache = PriceCache()
stream_router = create_stream_router(cache)

app = FastAPI()
app.include_router(stream_router)
```

### Endpoint: `GET /api/stream/prices`

Long-lived SSE connection. Each event is a JSON object keyed by ticker:

```
retry: 1000

data: {"AAPL": {"ticker": "AAPL", "price": 192.05, "previous_price": 191.88, "timestamp": 1750000000.1, "change": 0.17, "change_percent": 0.0887, "direction": "up"}, "GOOGL": {...}, ...}

data: {"TSLA": {"ticker": "TSLA", "price": 251.32, ...}}

: heartbeat

data: {"AAPL": {...}, "MSFT": {...}}
```

**Event semantics:**
- Only emitted when `cache.version` has changed since the last event — no spurious empty events
- Full snapshot of all currently-cached tickers on each event (not a diff)
- `retry: 1000` tells the browser `EventSource` to reconnect after 1s if the connection drops

**Heartbeat** (not yet implemented — see [§13](#13-sse-heartbeats)):
- SSE comment lines (`: heartbeat`) keep the connection alive between price updates
- Required for the Massive path where prices may not change for 15+ seconds

### Internal Generator

```python
async def _generate_events(
    price_cache: PriceCache,
    request: Request,
    interval: float = 0.5,
) -> AsyncGenerator[str, None]:
    yield "retry: 1000\n\n"
    last_version = -1

    while True:
        if await request.is_disconnected():
            break

        current_version = price_cache.version
        if current_version != last_version:
            last_version = current_version
            prices = price_cache.get_all()
            if prices:
                data = {ticker: update.to_dict() for ticker, update in prices.items()}
                yield f"data: {json.dumps(data)}\n\n"

        await asyncio.sleep(interval)
```

---

## 9. FastAPI Lifecycle Integration

Wire the market data source into FastAPI's `lifespan` context manager so it starts cleanly on boot and shuts down on exit.

```python
# backend/app/main.py
from contextlib import asynccontextmanager
from fastapi import FastAPI
from app.market import PriceCache, create_market_data_source, create_stream_router
from app.db import get_initial_tickers   # reads watchlist + open positions from DB

@asynccontextmanager
async def lifespan(app: FastAPI):
    # --- Startup ---
    cache = PriceCache()
    source = create_market_data_source(cache)

    # Initial ticker set = watchlist ∪ tickers with open positions
    initial_tickers = await get_initial_tickers()
    await source.start(initial_tickers)

    # Attach to app.state for injection into route handlers
    app.state.price_cache = cache
    app.state.market_source = source

    yield   # App is now running

    # --- Shutdown ---
    await source.stop()


app = FastAPI(lifespan=lifespan)

# Mount the SSE stream router (needs cache, injected at startup)
# Because create_stream_router is called after startup, use a dependency instead:
```

### Dependency Injection Pattern

```python
# backend/app/dependencies.py
from fastapi import Request
from app.market import PriceCache, MarketDataSource

def get_price_cache(request: Request) -> PriceCache:
    return request.app.state.price_cache

def get_market_source(request: Request) -> MarketDataSource:
    return request.app.state.market_source
```

```python
# In a route handler:
from fastapi import Depends
from app.dependencies import get_price_cache, get_market_source

@router.get("/api/portfolio")
async def get_portfolio(
    cache: PriceCache = Depends(get_price_cache),
):
    # Look up current prices for all positions
    aapl_price = cache.get_price("AAPL")   # float | None
    all_prices = cache.get_all()            # dict[str, PriceUpdate]
    ...
```

### Alternative: Module-Level Singletons

If FastAPI's `app.state` feels cumbersome, module-level singletons are equally valid for a single-process app:

```python
# backend/app/state.py
from app.market import PriceCache, create_market_data_source

price_cache: PriceCache = PriceCache()
market_source = create_market_data_source(price_cache)
```

Then import `price_cache` and `market_source` wherever needed. The lifespan handler calls `await market_source.start(tickers)` / `await market_source.stop()`.

---

## 10. Watchlist Coordination

The API layer is responsible for keeping the market source's ticker set in sync with database state. This happens in the watchlist and trade endpoints.

### Watchlist Add

```python
@router.post("/api/watchlist")
async def add_to_watchlist(
    body: AddTickerRequest,
    source: MarketDataSource = Depends(get_market_source),
    db: AsyncSession = Depends(get_db),
):
    ticker = normalize_ticker(body.ticker)   # uppercase, validate format
    validate_ticker(ticker)                  # reject invalid symbols

    # Idempotent DB insert (UNIQUE constraint → ignore duplicate)
    await db.add_to_watchlist(ticker)

    # Tell the market source to start pricing this ticker
    await source.add_ticker(ticker)

    return {"ticker": ticker, "added_at": ...}
```

### Watchlist Remove

```python
@router.delete("/api/watchlist/{ticker}")
async def remove_from_watchlist(
    ticker: str,
    source: MarketDataSource = Depends(get_market_source),
    db: AsyncSession = Depends(get_db),
):
    ticker = normalize_ticker(ticker)
    await db.remove_from_watchlist(ticker)   # 404 if not present

    # Only stop pricing if the user has no open position in this ticker
    has_position = await db.has_open_position(ticker)
    if not has_position:
        await source.remove_ticker(ticker)   # removes from cache too

    return {"removed": ticker}
```

**Key invariant:** `source.remove_ticker()` is only called when the ticker is absent from both the watchlist AND open positions. This ensures positions always have a live price for valuation.

---

## 11. Ticker Set: Watchlist ∪ Positions

The set of tickers the market source prices is always:

```
priced = watchlist_tickers ∪ open_position_tickers
```

This must be enforced at three points:

**1. Startup** — query both tables:

```python
async def get_initial_tickers(db) -> list[str]:
    watchlist = await db.get_watchlist_tickers()
    positions = await db.get_open_position_tickers()
    return list(set(watchlist) | set(positions))
```

**2. Trade buy that opens a new position** — if the bought ticker is not already in the watchlist, the position-set grows:

```python
@router.post("/api/portfolio/trade")
async def execute_trade(body: TradeRequest, source, db, cache):
    ticker = normalize_ticker(body.ticker)
    price = cache.get_price(ticker)
    if price is None:
        raise HTTPException(400, f"No live price for {ticker}")

    # Execute trade in DB (atomic transaction)
    ...

    # If this is a new position and not already priced, start pricing it
    if body.side == "buy":
        if ticker not in source.get_tickers():
            await source.add_ticker(ticker)
```

**3. Trade sell that closes a position** — if the ticker is no longer on the watchlist, stop pricing it:

```python
    if body.side == "sell" and quantity_remaining == 0:
        is_watchlisted = await db.is_in_watchlist(ticker)
        if not is_watchlisted:
            await source.remove_ticker(ticker)
```

---

## 12. Per-Ticker Freshness & Staleness

**Current state:** `PriceUpdate` has no `status` field. The simulator always produces live prices; the Massive client's prices go stale between polls or when the market is closed.

**Planned extension** — add `status: Literal["live", "stale"]` to `PriceUpdate`:

```python
@dataclass(frozen=True, slots=True)
class PriceUpdate:
    ticker: str
    price: float
    previous_price: float
    timestamp: float
    status: str = "live"   # "live" | "stale"
```

**Staleness logic in `MassiveDataSource._poll_once()`:**

```python
import time

STALE_THRESHOLD = 60.0  # seconds — a Massive price older than this is stale

async def _poll_once(self) -> None:
    snapshots = await asyncio.to_thread(self._fetch_snapshots)
    now = time.time()
    for snap in snapshots:
        ts = snap.last_trade.timestamp / 1000.0
        status = "live" if (now - ts) < STALE_THRESHOLD else "stale"
        self._cache.update(ticker=snap.ticker, price=snap.last_trade.price,
                           timestamp=ts, status=status)
```

**Frontend behavior:**
- `status: "live"` → display normally
- `status: "stale"` → dim the price, show a ⚠ icon or gray color
- The connection-status dot in the header reflects **SSE connectivity**, not price freshness — these are independent

**Simulator:** always emits `status: "live"`. No staleness concept.

---

## 13. SSE Heartbeats

**Current state:** The SSE generator does not emit heartbeat lines. On the Massive free tier, if no prices change for 15+ seconds, `EventSource` may time out in some proxies or load balancers (60s default).

**Fix** — emit an SSE comment line every N seconds:

```python
async def _generate_events(
    price_cache: PriceCache,
    request: Request,
    interval: float = 0.5,
    heartbeat_interval: float = 5.0,
) -> AsyncGenerator[str, None]:
    yield "retry: 1000\n\n"
    last_version = -1
    last_heartbeat = time.monotonic()

    while True:
        if await request.is_disconnected():
            break

        current_version = price_cache.version
        if current_version != last_version:
            last_version = current_version
            prices = price_cache.get_all()
            if prices:
                data = {t: u.to_dict() for t, u in prices.items()}
                yield f"data: {json.dumps(data)}\n\n"

        # Heartbeat: SSE comment, ignored by EventSource, keeps connection alive
        now = time.monotonic()
        if now - last_heartbeat >= heartbeat_interval:
            yield ": heartbeat\n\n"
            last_heartbeat = now

        await asyncio.sleep(interval)
```

SSE comment lines (`": ..."`) are defined in the spec and ignored by all `EventSource` implementations — they carry no data but prevent proxy timeouts.

---

## 14. Frontend Integration Contract

### Connecting

```typescript
const es = new EventSource("/api/stream/prices");

es.onmessage = (event) => {
  const updates: Record<string, PriceUpdate> = JSON.parse(event.data);
  for (const [ticker, update] of Object.entries(updates)) {
    handlePriceUpdate(ticker, update);
  }
};

es.onerror = () => {
  // EventSource auto-reconnects (respects retry: 1000 from server)
  // Update connection status indicator to "reconnecting"
};

es.onopen = () => {
  // Update connection status indicator to "connected"
};
```

### TypeScript Type

```typescript
interface PriceUpdate {
  ticker: string;
  price: number;
  previous_price: number;
  timestamp: number;        // Unix seconds
  change: number;
  change_percent: number;
  direction: "up" | "down" | "flat";
  status?: "live" | "stale"; // Will be added when §12 is implemented
}
```

### Price Flash Animation

On receiving a new price for a ticker, briefly apply a CSS class to the ticker's row/cell:

```typescript
function handlePriceUpdate(ticker: string, update: PriceUpdate) {
  const el = document.getElementById(`price-${ticker}`);
  if (!el) return;

  el.textContent = update.price.toFixed(2);

  // Flash green for up, red for down
  const flashClass = update.direction === "up" ? "flash-up" : "flash-down";
  el.classList.remove("flash-up", "flash-down");
  // Force reflow so re-adding works if the direction didn't change
  void el.offsetWidth;
  el.classList.add(flashClass);
}
```

```css
.flash-up {
  background-color: rgba(34, 197, 94, 0.3);  /* green-500 at 30% */
  transition: background-color 500ms ease-out;
}
.flash-down {
  background-color: rgba(239, 68, 68, 0.3);   /* red-500 at 30% */
  transition: background-color 500ms ease-out;
}
```

### Sparkline Accumulation

Sparkline data is accumulated client-side from the SSE stream since page load — there is no historical price storage or endpoint:

```typescript
const sparklineData: Record<string, number[]> = {};

function handlePriceUpdate(ticker: string, update: PriceUpdate) {
  if (!sparklineData[ticker]) sparklineData[ticker] = [];
  sparklineData[ticker].push(update.price);
  // Keep last N points for sparkline rendering
  if (sparklineData[ticker].length > 100) sparklineData[ticker].shift();
  renderSparkline(ticker, sparklineData[ticker]);
}
```

The main detail chart works the same way — it resets on page refresh, which is a deliberate documented choice.

---

## 15. Error Handling & Edge Cases

| Scenario | Behavior |
|---|---|
| SSE client disconnects | Generator detects `request.is_disconnected()` and exits cleanly |
| Simulator step throws | Exception logged, loop continues on next interval |
| Massive poll returns 401 | Error logged, loop continues; cache retains last-known prices |
| Massive poll returns 429 | Error logged, loop continues; increase `poll_interval` |
| `get_price()` returns `None` | Ticker not yet in cache (race at startup) or removed — trade endpoint must reject with HTTP 400 |
| `add_ticker` called while source is stopped | `_sim` is `None`; `SimulatorDataSource.add_ticker()` is a no-op — safe |
| Unknown ticker (not in `SEED_PRICES`) | Gets random `[50, 300]` seed price and default GBM params — visually consistent |
| Cholesky decomposition fails | Would throw if the correlation matrix is not positive-definite — only possible with a custom (non-default) correlation matrix. Default params are safe. |

### Trade Price Guard

**Every trade endpoint must call `cache.get_price(ticker)` and reject if `None`:**

```python
price = cache.get_price(ticker)
if price is None:
    raise HTTPException(
        status_code=400,
        detail=f"No live price available for {ticker}. Cannot execute trade."
    )
```

This should not happen for watchlisted tickers after the initial seed, but can briefly occur for a freshly added ticker (before the first simulator step writes to the cache). `add_ticker()` seeds the cache immediately in `SimulatorDataSource`, so the window is effectively zero for the simulator — but the guard is still required for correctness on the Massive path.

---

## 16. Configuration Summary

| Variable | Default | Effect |
|---|---|---|
| `MASSIVE_API_KEY` | _(unset)_ | If set → `MassiveDataSource`; if unset → `SimulatorDataSource` |
| `LLM_MOCK` | `false` | No effect on market data; affects `POST /api/chat` |

### `SimulatorDataSource` Constructor Parameters

| Parameter | Default | Description |
|---|---|---|
| `update_interval` | `0.5` | Seconds between GBM steps |
| `event_probability` | `0.001` | Per-ticker probability of a shock event per tick |

### `MassiveDataSource` Constructor Parameters

| Parameter | Default | Description |
|---|---|---|
| `poll_interval` | `15.0` | Seconds between REST API polls |

### `GBMSimulator` Constructor Parameters

| Parameter | Default | Description |
|---|---|---|
| `dt` | `0.5 / TRADING_SECONDS_PER_YEAR` | Time step (fraction of trading year) |
| `event_probability` | `0.001` | Per-ticker shock probability per step |

---

## Quick-Start Checklist for Downstream Engineers

When wiring the market data layer into a new feature:

1. **Import from `app.market`**, not from submodules:
   ```python
   from app.market import PriceCache, MarketDataSource
   ```

2. **Get the cache via `app.state` or a dependency** — never instantiate a second `PriceCache`.

3. **Guard every price read**: `price = cache.get_price(ticker)` → check for `None`.

4. **Watchlist mutations must call `source.add_ticker` / `source.remove_ticker`** — see §10.

5. **Respect the Watchlist ∪ Positions invariant** — see §11.

6. **SSE freshness vs. connectivity** — the header connection dot = SSE socket state; per-ticker `status` = data freshness. Don't conflate them.

7. **Heartbeats** (§13) are not yet in the stream — add them before Massive path testing.
