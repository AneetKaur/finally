# FinAlly — AI Trading Workstation

An AI-powered trading workstation that streams live market data, lets you trade a
simulated portfolio, and ships with an LLM copilot that can analyze positions and
execute trades on your behalf. Think Bloomberg terminal with an AI assistant.

This is a capstone project for an agentic AI coding course — built entirely by
coding agents coordinating through files in `planning/`.

## Quick Start

```bash
docker run -v finally-data:/app/db -p 8000:8000 --env-file .env finally
```

Then open **http://localhost:8000**. No login, no signup. You start with
$10,000 in virtual cash and a 10-ticker watchlist streaming live prices.

Or use the host-side scripts (they build, run, and optionally open the browser):

```bash
scripts/start_mac.sh        # macOS/Linux   (scripts/start_windows.ps1 on Windows)
scripts/stop_mac.sh         # stop (data persists)
```

## What You Can Do

- Watch prices stream with green/red flash animations and progressive sparklines
- Click a ticker for a larger detail chart
- Buy/sell shares — market orders, instant fill, no fees
- Monitor a portfolio heatmap, P&L chart, and positions table
- Chat with the AI assistant to get analysis and have it trade or manage the watchlist

## Architecture

Single Docker container, single port (8000):

- **Frontend** — Next.js + TypeScript, static export (`output: 'export'`), served by FastAPI
- **Backend** — FastAPI (Python, managed with `uv`)
- **Database** — SQLite at `db/finally.db`, volume-mounted, lazily initialized + seeded
- **Real-time** — Server-Sent Events (`/api/stream/prices`), native `EventSource`
- **AI** — LiteLLM → OpenRouter (`openai/gpt-oss-120b` via Cerebras), structured outputs
- **Market data** — built-in GBM simulator by default; real data via Massive API if a key is set

## Configuration

Set in `.env` at the project root (see `.env.example`):

| Variable | Required | Purpose |
|---|---|---|
| `OPENROUTER_API_KEY` | Yes | LLM chat |
| `MASSIVE_API_KEY` | No | Real market data; simulator used if absent |
| `LLM_MOCK` | No | `true` for deterministic mock LLM responses (tests) |

## API Surface

- **Market** — `GET /api/stream/prices` (SSE)
- **Portfolio** — `GET /api/portfolio`, `POST /api/portfolio/trade`, `GET /api/portfolio/history`
- **Watchlist** — `GET/POST /api/watchlist`, `DELETE /api/watchlist/{ticker}`
- **Chat** — `POST /api/chat`
- **Health** — `GET /api/health`

## Testing

- **Backend** — pytest (market data, portfolio math, LLM parsing, API routes)
- **Frontend** — React Testing Library (components, price flash, CRUD, chat)
- **E2E** — Playwright via `test/docker-compose.test.yml`, run with `LLM_MOCK=true`
  against the simulator

## Status

The market data subsystem is complete (see `planning/MARKET_DATA_SUMMARY.md`).
The rest of the platform — portfolio, charts, chat, and frontend — is still to be built.

## Documentation

Full specification: [`planning/PLAN.md`](./PLAN.md).

> ⚠️ No authentication, and `/api/chat` calls a paid LLM. Safe for local single-user
> use only — do not expose publicly without adding auth and rate-limiting.
