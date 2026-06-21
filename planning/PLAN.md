# FinAlly — AI Trading Workstation

## Project Specification

## 1. Vision

FinAlly (Finance Ally) is a visually stunning AI-powered trading workstation that streams live market data, lets users trade a simulated portfolio, and integrates an LLM chat assistant that can analyze positions and execute trades on the user's behalf. It looks and feels like a modern Bloomberg terminal with an AI copilot.

This is the capstone project for an agentic AI coding course. It is built entirely by Coding Agents demonstrating how orchestrated AI agents can produce a production-quality full-stack application. Agents interact through files in `planning/`.

## 2. User Experience

### First Launch

The user runs a single Docker command (or a provided start script). The app is reachable at `http://localhost:8000` — the container itself cannot open the host browser, so `docker run` simply prints the URL; the host-side start scripts (§11) are what optionally open the browser. No login, no signup. They immediately see:

- A watchlist of 10 default tickers with live-updating prices in a grid
- $10,000 in virtual cash
- A dark, data-rich trading terminal aesthetic
- An AI chat panel ready to assist

### What the User Can Do

- **Watch prices stream** — prices flash green (uptick) or red (downtick) with subtle CSS animations that fade
- **View sparkline mini-charts** — price action beside each ticker in the watchlist, accumulated on the frontend from the SSE stream since page load (sparklines fill in progressively)
- **Click a ticker** to see a larger detailed chart in the main chart area
- **Buy and sell shares** — market orders only, instant fill at current price, no fees, no confirmation dialog
- **Monitor their portfolio** — a heatmap (treemap) showing positions sized by weight and colored by P&L, plus a P&L chart tracking total portfolio value over time
- **View a positions table** — ticker, quantity, average cost, current price, unrealized P&L, % change
- **Chat with the AI assistant** — ask about their portfolio, get analysis, and have the AI execute trades and manage the watchlist through natural language
- **Manage the watchlist** — add/remove tickers manually or via the AI chat

### Visual Design

- **Dark theme**: backgrounds around `#0d1117` or `#1a1a2e`, muted gray borders, no pure black
- **Price flash animations**: brief green/red background highlight on price change, fading over ~500ms via CSS transitions
- **Connection status indicator**: a small colored dot (green = connected, yellow = reconnecting, red = disconnected) visible in the header
- **Professional, data-dense layout**: inspired by Bloomberg/trading terminals — every pixel earns its place
- **Responsive but desktop-first**: optimized for wide screens, functional on tablet

### Color Scheme
- Accent Yellow: `#ecad0a`
- Blue Primary: `#209dd7`
- Purple Secondary: `#753991` (submit buttons)

## 3. Architecture Overview

### Single Container, Single Port

```
┌─────────────────────────────────────────────────┐
│  Docker Container (port 8000)                   │
│                                                 │
│  FastAPI (Python/uv)                            │
│  ├── /api/*          REST endpoints             │
│  ├── /api/stream/*   SSE streaming              │
│  └── /*              Static file serving         │
│                      (Next.js export)            │
│                                                 │
│  SQLite database (volume-mounted)               │
│  Background task: market data polling/sim        │
└─────────────────────────────────────────────────┘
```

- **Frontend**: Next.js with TypeScript, built as a static export (`output: 'export'`), served by FastAPI as static files
- **Backend**: FastAPI (Python), managed as a `uv` project
- **Database**: SQLite, single file at `db/finally.db`, volume-mounted for persistence
- **Real-time data**: Server-Sent Events (SSE) — simpler than WebSockets, one-way server→client push, works everywhere
- **AI integration**: LiteLLM → OpenRouter (Cerebras for fast inference), with structured outputs for trade execution
- **Market data**: Environment-variable driven — simulator by default, real data via Massive API if key provided

### Why These Choices

| Decision | Rationale |
|---|---|
| SSE over WebSockets | One-way push is all we need; simpler, no bidirectional complexity, universal browser support |
| Static Next.js export | Single origin, no CORS issues, one port, one container, simple deployment |
| SQLite over Postgres | No auth = no multi-user = no need for a database server; self-contained, zero config |
| Single Docker container | Students run one command; no docker-compose for production, no service orchestration |
| uv for Python | Fast, modern Python project management; reproducible lockfile; what students should learn |
| Market orders only | Eliminates order book, limit order logic, partial fills — dramatically simpler portfolio math |

---

## 4. Directory Structure

```
finally/
├── frontend/                 # Next.js TypeScript project (static export)
├── backend/                  # FastAPI uv project (Python)
│   └── db/                   # Schema definitions, seed data, migration logic
├── planning/                 # Project-wide documentation for agents
│   ├── PLAN.md               # This document
│   └── ...                   # Additional agent reference docs
├── scripts/
│   ├── start_mac.sh          # Launch Docker container (macOS/Linux)
│   ├── stop_mac.sh           # Stop Docker container (macOS/Linux)
│   ├── start_windows.ps1     # Launch Docker container (Windows PowerShell)
│   └── stop_windows.ps1      # Stop Docker container (Windows PowerShell)
├── test/                     # Playwright E2E tests + docker-compose.test.yml
├── db/                       # Volume mount target (SQLite file lives here at runtime)
│   └── .gitkeep              # Directory exists in repo; finally.db is gitignored
├── Dockerfile                # Multi-stage build (Node → Python)
├── docker-compose.yml        # Optional convenience wrapper
├── .env                      # Environment variables (gitignored, .env.example committed)
└── .gitignore
```

### Key Boundaries

- **`frontend/`** is a self-contained Next.js project. It knows nothing about Python. It talks to the backend via `/api/*` endpoints and `/api/stream/*` SSE endpoints. Internal structure is up to the Frontend Engineer agent.
- **`backend/`** is a self-contained uv project with its own `pyproject.toml`. It owns all server logic including database initialization, schema, seed data, API routes, SSE streaming, market data, and LLM integration. Internal structure is up to the Backend/Market Data agents.
- **`backend/db/`** contains schema SQL definitions and seed logic. The backend lazily initializes the database on first request — creating tables and seeding default data if the SQLite file doesn't exist or is empty.
- **`db/`** at the top level is the runtime volume mount point. The SQLite file (`db/finally.db`) is created here by the backend and persists across container restarts via Docker volume.
- **`planning/`** contains project-wide documentation, including this plan. All agents reference files here as the shared contract.
- **`test/`** contains Playwright E2E tests and supporting infrastructure (e.g., `docker-compose.test.yml`). Unit tests live within `frontend/` and `backend/` respectively, following each framework's conventions.
- **`scripts/`** contains start/stop scripts that wrap Docker commands.

---

## 5. Environment Variables

```bash
# Required: OpenRouter API key for LLM chat functionality
OPENROUTER_API_KEY=your-openrouter-api-key-here

# Optional: Massive (Polygon.io) API key for real market data
# If not set, the built-in market simulator is used (recommended for most users)
MASSIVE_API_KEY=

# Optional: Set to "true" for deterministic mock LLM responses (testing)
LLM_MOCK=false
```

### Behavior

- If `MASSIVE_API_KEY` is set and non-empty → backend uses Massive REST API for market data
- If `MASSIVE_API_KEY` is absent or empty → backend uses the built-in market simulator
- If `LLM_MOCK=true` → backend returns deterministic mock LLM responses (for E2E tests)
- The backend reads `.env` from the project root (mounted into the container or read via docker `--env-file`)

---

## 6. Market Data

### Two Implementations, One Interface

Both the simulator and the Massive client implement the same abstract interface. The backend selects which to use based on the environment variable. All downstream code (SSE streaming, price cache, frontend) is agnostic to the source.

### Simulator (Default)

- Generates prices using geometric Brownian motion (GBM) with configurable drift and volatility per ticker
- Updates at ~500ms intervals
- Correlated moves across tickers (e.g., tech stocks move together)
- Occasional random "events" — sudden 2-5% moves on a ticker for drama
- Starts from realistic seed prices (e.g., AAPL ~$190, GOOGL ~$175, etc.)
- Runs as an in-process background task — no external dependencies

### Massive API (Optional)

- REST API polling (not WebSocket) — simpler, works on all tiers
- Polls for the union of all watched tickers on a configurable interval
- Free tier (5 calls/min): poll every 15 seconds
- Paid tiers: poll every 2-15 seconds depending on tier
- Parses REST response into the same format as the simulator

### Shared Price Cache

- A single background task (simulator or Massive poller) writes to an in-memory price cache
- The cache holds the latest price, previous price, and timestamp for each ticker, plus a per-ticker freshness `status` (`live` / `stale`). The simulator's prices are always `live`. On the Massive path, prices go `stale` when a ticker hasn't updated within an expected window (closed market, weekend, or a failed poll) — staleness is a property of the data, distinct from connectivity. The header's connection dot reflects SSE connectivity only (§10); per-ticker freshness is carried in the price payload so the frontend can dim a stale quote without implying the whole stream is down.
- **Priced ticker set** — the source tracks `watchlist ∪ tickers-with-an-open-position`. The API layer keeps this in sync: watchlist add/remove calls `source.add_ticker`/`source.remove_ticker`, and a buy that opens a new position adds its ticker. A watchlisted ticker the user still holds stays priced even after it's removed from the watchlist, until the position is closed — this guarantees every position always has a live price for valuation and P&L.
- SSE streams read from this cache and push updates to connected clients
- This architecture supports future multi-user scenarios without changes to the data layer

### SSE Streaming

- Endpoint: `GET /api/stream/prices`
- Long-lived SSE connection; client uses native `EventSource` API
- The server pushes a ticker's update **only when its price actually changes** — the cache carries a version counter for change detection, so a slow source like Massive's 15s poll never emits redundant events
- A periodic heartbeat (an SSE comment line, every few seconds) keeps the connection and the header's connection-status indicator alive during quiet periods
- Each SSE event contains ticker, price, previous price, timestamp, change direction, and freshness `status` (§ Shared Price Cache)
- Client handles reconnection automatically (EventSource has built-in retry)

---

## 7. Database

### SQLite with Lazy Initialization

The backend checks for the SQLite database on startup (or first request). If the file doesn't exist or tables are missing, it creates the schema and seeds default data. This means:

- No separate migration step
- No manual database setup
- Fresh Docker volumes start with a clean, seeded database automatically

### Schema

All tables include a `user_id` column defaulting to `"default"`. This is hardcoded for now (single-user) but enables future multi-user support without schema migration.

Monetary and quantity columns use SQLite `REAL` (binary float). For this single-user demo that's a deliberate, accepted trade-off: rather than integer-cents/decimal storage, the application layer keeps the artifacts invisible by rounding on every write — cash to 2 decimals (cents) and share quantities to 4 decimals (§8). Comparisons that gate trades (e.g. "enough cash", "enough shares") use these rounded values, so accumulated float drift never blocks or mis-fills a valid trade.

**users_profile** — User state (cash balance)
- `id` TEXT PRIMARY KEY (default: `"default"`)
- `cash_balance` REAL (default: `10000.0`)
- `created_at` TEXT (ISO timestamp)

**watchlist** — Tickers the user is watching
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (default: `"default"`)
- `ticker` TEXT
- `added_at` TEXT (ISO timestamp)
- UNIQUE constraint on `(user_id, ticker)`

**positions** — Current holdings (one row per ticker per user)
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (default: `"default"`)
- `ticker` TEXT
- `quantity` REAL (fractional shares supported)
- `avg_cost` REAL
- `updated_at` TEXT (ISO timestamp)
- UNIQUE constraint on `(user_id, ticker)`
- When a sell reduces quantity to 0, the row is **deleted** — there are no zero-quantity rows, so the positions table and heatmap stay clean.

**trades** — Trade history (append-only log)
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (default: `"default"`)
- `ticker` TEXT
- `side` TEXT (`"buy"` or `"sell"`)
- `quantity` REAL (fractional shares supported)
- `price` REAL
- `executed_at` TEXT (ISO timestamp)

**portfolio_snapshots** — Portfolio value over time (for P&L chart). Recorded every 30 seconds by a background task, immediately after each trade execution, and once at startup so the P&L chart is never empty on first launch.
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (default: `"default"`)
- `total_value` REAL
- `recorded_at` TEXT (ISO timestamp)

**chat_messages** — Conversation history with LLM
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (default: `"default"`)
- `role` TEXT (`"user"` or `"assistant"`)
- `content` TEXT
- `actions` TEXT (JSON — trades executed, **rejected trades with their validation error**, and watchlist changes made; null for user messages)
- `created_at` TEXT (ISO timestamp)

### Default Seed Data

- One user profile: `id="default"`, `cash_balance=10000.0`
- Ten watchlist entries: AAPL, GOOGL, MSFT, AMZN, TSLA, NVDA, META, JPM, V, NFLX

---

## 8. API Endpoints

### Market Data
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/stream/prices` | SSE stream of live price updates |

### Portfolio
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/portfolio` | Current positions, cash balance, total value, unrealized P&L |
| POST | `/api/portfolio/trade` | Execute a trade: `{ticker, quantity, side}` |
| GET | `/api/portfolio/history` | Portfolio value snapshots over time (for P&L chart) |

`quantity` may be fractional. Share quantities are stored to 4 decimal places and cash math is rounded to cents. A sell that brings a position to 0 deletes the position row (§7).

**Trade atomicity & serialization.** A trade touches three tables (`users_profile` cash, `positions`, `trades`) and they must move together. Each trade executes inside a single SQLite transaction: validate against current cash/holdings, then write the cash update, the position upsert/delete, and the append to `trades` — commit all or nothing. Because manual trades, AI-triggered trades (§9), and the snapshot jobs (§7) all run in one process, writes are serialized per user with an `asyncio` lock so two trades can't read the same cash balance and both pass validation. A `portfolio_snapshot` reads cash + positions + cached prices as one consistent view (taken outside any in-flight trade), so `total_value` never captures a half-applied trade.

### Watchlist
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/watchlist` | Current watchlist tickers with latest prices |
| POST | `/api/watchlist` | Add a ticker: `{ticker}` |
| DELETE | `/api/watchlist/{ticker}` | Remove a ticker |

Both endpoints keep the market source's ticker set in sync (§6). Removing a ticker the user still holds succeeds, but the ticker remains priced until the position is closed.

### Chat
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/chat` | Send a message, receive complete JSON response (message + executed actions) |

### Ticker Validation & Normalization

One rule, applied at the API boundary and reused by every path that accepts a ticker (watchlist add, manual trade, AI trade, market source):

- **Normalize**: trim and uppercase the input (`aapl` → `AAPL`).
- **Validate**: 1–5 characters, `A–Z` plus `.` and `-` only (covers symbols like `BRK.B`). Reject anything else with a 422.
- **Supported set**: a ticker the market source can't price (unknown symbol) is rejected with a clear error rather than silently added — this keeps the watchlist, positions, and price cache from ever holding an unpriceable symbol.
- **Duplicates**: adding a ticker already on the watchlist is idempotent (200, no duplicate row — enforced by the `UNIQUE(user_id, ticker)` constraint), not an error.

### Response & Error Shapes

All endpoints return JSON. Validation/business errors use FastAPI's default error envelope `{"detail": "..."}` with the status codes below; success shapes are illustrated by example.

`GET /api/portfolio`:
```json
{
  "cash_balance": 8500.00,
  "total_value": 10120.50,
  "positions": [
    {"ticker": "AAPL", "quantity": 10.0, "avg_cost": 190.00,
     "current_price": 192.05, "market_value": 1920.50,
     "unrealized_pnl": 20.50, "pnl_pct": 1.08, "price_status": "live"}
  ]
}
```

`POST /api/portfolio/trade` — request `{"ticker": "AAPL", "quantity": 10, "side": "buy"}`; success `200`:
```json
{
  "trade": {"ticker": "AAPL", "side": "buy", "quantity": 10.0, "price": 192.05,
            "executed_at": "2026-06-21T15:00:00Z"},
  "cash_balance": 6580.00
}
```
Errors: `422` invalid ticker/side/quantity (non-positive); `400` insufficient cash (buy) or insufficient shares (sell); `400` when no live price is available for the ticker.

`POST /api/watchlist` — `{"ticker": "PYPL"}` → `200` with the added entry (or existing entry if already present); `422` invalid/unsupported ticker. `DELETE /api/watchlist/{ticker}` → `200` (`204`-style success body `{"removed": "PYPL"}`); removing a ticker not on the watchlist returns `404`.

`POST /api/chat` — `{"message": "..."}` → `200`:
```json
{
  "message": "Bought 10 AAPL at $192.05. Your tech weight is now 40%.",
  "actions": {
    "trades": [{"ticker": "AAPL", "side": "buy", "quantity": 10, "price": 192.05, "status": "executed"}],
    "rejected_trades": [{"ticker": "TSLA", "side": "buy", "quantity": 100, "status": "rejected", "error": "insufficient cash"}],
    "watchlist_changes": [{"ticker": "PYPL", "action": "add", "status": "applied"}]
  }
}
```
The `actions` object mirrors what is persisted to `chat_messages.actions` (§7) — executed trades, rejected trades with their validation error, and watchlist changes.

### System
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/health` | Health check (for Docker/deployment) |

---

## 9. LLM Integration

When writing code to make calls to LLMs, use cerebras-inference skill to use LiteLLM via OpenRouter to the `openrouter/openai/gpt-oss-120b` model with Cerebras as the inference provider. Structured Outputs should be used to interpret the results.

There is an OPENROUTER_API_KEY in the .env file in the project root.

### How It Works

When the user sends a chat message, the backend:

1. Loads the user's current portfolio context (cash, positions with P&L, watchlist with live prices, total portfolio value)
2. Loads recent conversation history from the `chat_messages` table — the last 20 messages — to bound context size and per-turn cost
3. Constructs a prompt with a system message, portfolio context, conversation history, and the user's new message
4. Calls the LLM via LiteLLM → OpenRouter, requesting structured output, using the cerebras-inference skill
5. Parses the complete structured JSON response
6. Auto-executes any trades or watchlist changes specified in the response
7. Stores the message and executed actions in `chat_messages`
8. Returns the complete JSON response to the frontend (no token-by-token streaming — Cerebras inference is fast enough that a loading indicator is sufficient)

### Structured Output Schema

The LLM is instructed to respond with JSON matching this schema:

```json
{
  "message": "Your conversational response to the user",
  "trades": [
    {"ticker": "AAPL", "side": "buy", "quantity": 10}
  ],
  "watchlist_changes": [
    {"ticker": "PYPL", "action": "add"}
  ]
}
```

- `message` (required): The conversational text shown to the user
- `trades` (optional): Array of trades to auto-execute. Each trade goes through the same validation as manual trades (sufficient cash for buys, sufficient shares for sells)
- `watchlist_changes` (optional): Array of watchlist modifications

### Auto-Execution

Trades specified by the LLM execute automatically — no confirmation dialog. This is a deliberate design choice:
- It's a simulated environment with fake money, so the stakes are zero
- It creates an impressive, fluid demo experience
- It demonstrates agentic AI capabilities — the core theme of the course

If a trade fails validation (e.g., insufficient cash), the error is included in the chat response so the LLM can inform the user, and the rejected trade is recorded in that message's `actions` JSON (§7) for debugging.

### Auto-Execution Guardrails

Auto-execution is convenient but unbounded model output is a risk even with fake money — a single prompt could liquidate the portfolio or fire dozens of trades. The backend enforces these limits on the actions array *before* executing any of them (they are backend rules, not just prompt instructions):

- **Max 5 actions per response** (trades + watchlist changes combined). Extras are dropped and reported back as rejected so the LLM can explain.
- **Per-trade notional cap**: a single AI trade may not exceed 50% of current total portfolio value. Larger trades are rejected with an explanatory error.
- **Same validation as manual trades**: sufficient cash for buys, sufficient shares for sells. This already means **no short selling** (can't sell shares you don't own) and **no margin/leverage** (can't buy beyond cash).
- **Ticker normalization & validation** per §8 applies to every AI-specified ticker.
- **Explicit-intent only**: the system prompt instructs the model to execute trades only when the user clearly asks for or agrees to a portfolio change — analysis and suggestions alone must not trigger trades.

Every rejected or dropped action is surfaced in the chat `message` and recorded in `actions` (§7).

### System Prompt Guidance

The LLM should be prompted as "FinAlly, an AI trading assistant" with instructions to:
- Analyze portfolio composition, risk concentration, and P&L
- Suggest trades with reasoning
- Execute trades when the user asks or agrees
- Manage the watchlist proactively
- Be concise and data-driven in responses
- Always respond with valid structured JSON

### LLM Mock Mode

When `LLM_MOCK=true`, the backend returns deterministic mock responses instead of calling OpenRouter. This enables:
- Fast, free, reproducible E2E tests
- Development without an API key
- CI/CD pipelines

---

## 10. Frontend Design

### Layout

The frontend is a single-page application with a dense, terminal-inspired layout. The specific component architecture and layout system is up to the Frontend Engineer, but the UI should include these elements:

- **Watchlist panel** — grid/table of watched tickers with: ticker symbol, current price (flashing green/red on change), daily change %, and a sparkline mini-chart (accumulated from SSE since page load)
- **Main chart area** — larger chart for the currently selected ticker, with at minimum price over time. Clicking a ticker in the watchlist selects it here. Like the sparklines, this chart is accumulated client-side from the SSE stream since page load (it resets on refresh) — no historical-price storage or endpoint is needed.
- **Portfolio heatmap** — treemap visualization where each rectangle is a position, sized by portfolio weight, colored by P&L (green = profit, red = loss)
- **P&L chart** — line chart showing total portfolio value over time, using data from `portfolio_snapshots`
- **Positions table** — tabular view of all positions: ticker, quantity, avg cost, current price, unrealized P&L, % change
- **Trade bar** — simple input area: ticker field, quantity field, buy button, sell button. Market orders, instant fill.
- **AI chat panel** — docked/collapsible sidebar. Message input, scrolling conversation history, loading indicator while waiting for LLM response. Trade executions and watchlist changes shown inline as confirmations.
- **Header** — portfolio total value (updating live), connection status indicator, cash balance

### Technical Notes

- Use `EventSource` for SSE connection to `/api/stream/prices`
- Canvas-based charting library preferred (Lightweight Charts or Recharts) for performance
- Price flash effect: on receiving a new price, briefly apply a CSS class with background color transition, then remove it
- All API calls go to the same origin (`/api/*`) — no CORS configuration needed
- Tailwind CSS for styling with a custom dark theme

---

## 11. Docker & Deployment

### Multi-Stage Dockerfile

```
Stage 1: Node 20 slim
  - Copy frontend/
  - npm install && npm run build (produces static export)

Stage 2: Python 3.12 slim
  - Install uv
  - Copy backend/
  - uv sync (install Python dependencies from lockfile)
  - Copy frontend build output into a static/ directory
  - Expose port 8000
  - CMD: uvicorn serving FastAPI app
```

FastAPI serves the static frontend files and all API routes on port 8000.

### Routing Precedence

A single app serves both the API and the static export, so route order is explicit:

1. `/api/*` and `/api/stream/*` are matched first. An unknown path under `/api/` returns a JSON `404` (`{"detail": "Not Found"}`) — it never falls through to HTML.
2. Static assets from the export (`/_next/*`, `*.js`, `*.css`, images) are served directly, with long-lived cache headers on hashed asset filenames.
3. Any other path (including deep links and post-refresh SPA routes) returns `index.html`, letting the client router take over. Because this is a static export it's effectively a single page, but the catch-all keeps a hard refresh on any URL from 404-ing.

### Docker Volume

The SQLite database persists via a named Docker volume:

```bash
docker run -v finally-data:/app/db -p 8000:8000 --env-file .env finally
```

The `db/` directory in the project root maps to `/app/db` in the container. The backend writes `finally.db` to this path.

### Start/Stop Scripts

**`scripts/start_mac.sh`** (macOS/Linux):
- Builds the Docker image if not already built (or if `--build` flag passed)
- Runs the container with the volume mount, port mapping, and `.env` file
- Prints the URL to access the app
- Optionally opens the browser

**`scripts/stop_mac.sh`** (macOS/Linux):
- Stops and removes the running container
- Does NOT remove the volume (data persists)

**`scripts/start_windows.ps1`** / **`scripts/stop_windows.ps1`**: PowerShell equivalents for Windows.

All scripts should be idempotent — safe to run multiple times.

### Optional Cloud Deployment

The container is designed to deploy to AWS App Runner, Render, or any container platform. A Terraform configuration for App Runner may be provided in a `deploy/` directory as a stretch goal, but is not part of the core build.

> **Caution:** The app has no authentication, and `/api/chat` calls a paid LLM. This is fine for a local single-user app, but the container must not be exposed publicly as-is — anyone reaching it could run up API cost. Add auth/rate-limiting before any public deployment.

---

## 12. Testing Strategy

### Unit Tests (within `frontend/` and `backend/`)

**Backend (pytest)**:
- Market data: simulator generates valid prices, GBM math is correct, Massive API response parsing works, both implementations conform to the abstract interface
- Portfolio: trade execution logic, P&L calculations, edge cases (selling more than owned, buying with insufficient cash, selling at a loss)
- LLM: structured output parsing handles all valid schemas, graceful handling of malformed responses, trade validation within chat flow
- API routes: correct status codes, response shapes, error handling

**Frontend (React Testing Library or similar)**:
- Component rendering with mock data
- Price flash animation triggers correctly on price changes
- Watchlist CRUD operations
- Portfolio display calculations
- Chat message rendering and loading state

### E2E Tests (in `test/`)

**Infrastructure**: A separate `docker-compose.test.yml` in `test/` that spins up the app container plus a Playwright container. This keeps browser dependencies out of the production image.

**Environment**: Tests run with `LLM_MOCK=true` by default for speed and determinism. All scenarios run against the simulator (no `MASSIVE_API_KEY`); the Massive path is a tested-but-secondary integration, and new features are developed and demoed against the simulator.

**Key Scenarios**:
- Fresh start: default watchlist appears, $10k balance shown, prices are streaming
- Add and remove a ticker from the watchlist
- Buy shares: cash decreases, position appears, portfolio updates
- Sell shares: cash increases, position updates or disappears
- Portfolio visualization: heatmap renders with correct colors, P&L chart has data points
- AI chat (mocked): send a message, receive a response, trade execution appears inline
- SSE resilience: disconnect and verify reconnection

---

## 13. Document Review — Questions, Clarifications & Simplifications

_Added 2026-06-20 during a documentation review. The market data subsystem (§6) is built; everything below concerns the parts still to be developed. The plan is solid and internally consistent — these are the open questions and decisions worth nailing down before the backend/frontend work starts._

### Open questions that affect implementation

1. **Removing a watchlisted ticker you still hold.** If the user owns AAPL and removes it from the watchlist, the market source stops tracking it (`source.remove_ticker`), so the price cache no longer has a price — breaking portfolio valuation and P&L for that position. What should `DELETE /api/watchlist/{ticker}` do when an open position exists? Options: (a) block removal with an error, (b) allow removal but keep pricing the ticker as long as a position exists, (c) decouple "priced tickers" from "watchlist" entirely. **Recommendation: (b)** — the set of priced tickers = `watchlist ∪ tickers-with-positions`. This needs to be stated explicitly; it's the most likely correctness bug in the portfolio path.
   → **Resolved (b):** folded into §6 (Shared Price Cache) and §8 (Watchlist).

2. **Who wires the watchlist to the market source?** §8 has `POST/DELETE /api/watchlist`, and §6's source exposes `add_ticker`/`remove_ticker`, but nothing states that the watchlist endpoints call them. Confirm the API layer is responsible for keeping the market source's ticker set in sync with the watchlist (plus the position-set from item 1).
   → **Resolved:** API layer owns the sync; stated in §6 (Shared Price Cache) and §8 (Watchlist).

3. **What does the "main chart" plot, and from where?** §2 and §10 describe a larger detail chart of "price over time," but there is no historical price storage and the simulator has no history. Is the main chart also accumulated client-side from the SSE stream since page load (like the sparklines)? If so, say so — it means a page refresh resets the chart, which is fine but should be a deliberate, documented choice. If real historical bars are wanted, that only works on the Massive path and needs a new endpoint + schema.
   → **Resolved:** client-accumulated from SSE since page load (resets on refresh), no new endpoint; stated in §10 (Main chart area).

4. **SSE push semantics under slow sources.** §6 says SSE pushes "~500ms." On the Massive free tier prices only change every 15s. Does the stream push only on a version change (the cache already has a version counter), emit periodic heartbeats to keep `EventSource` alive, or push the full snapshot every 500ms regardless? Clarify so the frontend flash logic and the connection-status dot behave correctly.
   → **Resolved:** push only on a version change, plus periodic heartbeats to keep `EventSource` alive; stated in §6 (SSE Streaming).

5. **How much chat history is loaded?** §9 step 2 says "recent conversation history." Define the cap (last N messages or a token budget). Unbounded growth will eventually blow the context window and inflate cost on every turn.
   → **Resolved:** last 20 messages; stated in §9 (How It Works, step 2).

6. **Failed AI trades — persisted where?** §9 says a failed trade's error goes into the chat response so the LLM can explain it, but the `actions` column (§7) is described as "trades executed / watchlist changes made." Confirm whether rejected trades are recorded in `actions` (useful for debugging) or only surfaced in the message text.
   → **Resolved:** rejected trades are recorded in `actions`; stated in §7 (chat_messages) and §9 (Auto-Execution).

7. **Selling the entire position.** §12 says a position "updates or disappears." Pick one: delete the `positions` row when quantity hits 0 (cleaner — avoids zero-quantity rows in the heatmap/table). State it so trade logic and the UI agree.
   → **Resolved:** delete the `positions` row when quantity hits 0; stated in §7 (positions) and §8 (Portfolio).

### Smaller clarifications

- **Trade quantity type.** Schema supports fractional shares but the §9 example uses integer `10`. Confirm the API accepts and round-trips fractional quantities, and decide on a rounding/precision rule for cash math.
  → **Resolved:** fractional quantities accepted; shares stored to 4 decimals, cash rounded to cents; stated in §8 (Portfolio).
- **Cost exposure.** The open `/api/chat` endpoint calls a paid LLM with no auth. Acceptable for a local single-user app, but worth a one-line note that this must not be exposed publicly as-is (relevant to the §11 "optional cloud deployment" stretch goal).
  → **Resolved:** caution note added to §11 (Optional Cloud Deployment).

### Opportunities to simplify


- **Snapshot cadence.** The 30s background snapshot is justified (the P&L chart must move even when the user isn't trading), so keep it — but note that on first launch the P&L chart is empty until the first snapshot lands. A single snapshot taken at startup avoids an awkward empty-chart first impression.
  → **Resolved:** startup snapshot added to §7 (portfolio_snapshots).
- **Defer the Massive path in docs, not code.** The Massive client is already built and tested, so no code change is needed — but for the remaining work, all new features (charts, portfolio, chat) should be developed and demoed against the simulator only. Treat Massive as a tested-but-secondary path so it doesn't add surface area to every new feature's testing.
  → **Resolved:** noted in §12 (E2E Tests, Environment).

---

## 14. Document Review — Contract Precision

_Added 2026-06-21 during a second documentation review. §13's open questions were already folded into the body; this pass focused on contract precision (schemas, validation, atomicity, and the LLM action policy) for the parts still to be developed. All findings below have been folded into the body sections they cite._

### High

1. **LLM trade execution needs guardrails beyond "same validation as manual trades."** Auto-execution with no per-turn limits could let one prompt liquidate the portfolio or fire many trades. Define max actions per response, a per-trade notional cap, no short/no margin, ticker normalization, and execute-on-explicit-intent-only.
   → **Resolved:** new "Auto-Execution Guardrails" subsection in §9 (max 5 actions/response, 50%-of-portfolio notional cap, no short/margin via existing validation, §8 ticker rules, explicit-intent only).

2. **Concurrent trade and snapshot writes are underspecified for SQLite.** Manual trades, AI trades, startup/30s snapshots, and price reads all run in one process. State that a trade is atomic in one transaction, writes are serialized per user, and snapshots read a consistent view.
   → **Resolved:** "Trade atomicity & serialization" note in §8 (single transaction across the three tables, per-user `asyncio` lock, consistent snapshot read).

### Medium

3. **API response schemas and error shapes are not defined.** Add JSON examples and error codes for portfolio, trade, watchlist, and chat — including validation errors, rejected AI trades, duplicate watchlist, and unavailable-price states.
   → **Resolved:** "Response & Error Shapes" subsection in §8 with success examples and status codes per endpoint.

4. **Static export plus FastAPI fallback behavior needs specifying.** Unknown non-API paths should return `index.html`; `/api/*` should return JSON errors; asset caching and refresh behavior should be explicit.
   → **Resolved:** "Routing Precedence" subsection in §11.

5. **Market session behavior is missing.** Real data has closed hours, stale prices, and API failures. Expose freshness/status metadata, since the connection dot reflects SSE connectivity, not price freshness.
   → **Resolved:** per-ticker freshness `status` (`live`/`stale`) added to the price cache and SSE payload in §6; connection-dot-vs-freshness distinction stated.

6. **Ticker validation and normalization should be centralized.** Define uppercase normalization, allowed characters, length, duplicate behavior, and unsupported-ticker behavior across all paths.
   → **Resolved:** "Ticker Validation & Normalization" subsection in §8, reused by watchlist, manual trades, AI trades, and the market source.

### Low

7. **"Browser opens" on first launch isn't aligned with Docker.** A container can't open the host browser. Clarify that `docker run` prints the URL while host-side scripts may open the browser.
   → **Resolved:** §2 (First Launch) reworded; §11 start scripts already note optional browser open.

8. **Monetary precision should account for `REAL` floating point.** Either accept small float artifacts for this demo or use integer cents/decimal handling.
   → **Resolved:** §7 schema note — `REAL` accepted as a deliberate demo trade-off, with cents/4-decimal rounding on every write so drift never gates a trade.
