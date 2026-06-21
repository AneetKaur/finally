# Review of PLAN.md

## Findings

1. **High: LLM trade execution needs explicit guardrails beyond "same validation as manual trades."**
   The plan says AI trades auto-execute with no confirmation because this is simulated money, but it does not define execution limits, ticker allow/deny behavior, or how to handle ambiguous user intent. Even in a demo, an LLM could liquidate the portfolio, buy an unsupported ticker, or make many sequential trades from one prompt. Add a contract such as: maximum actions per response, maximum notional per AI trade or per turn, valid ticker normalization, no short selling, no margin, and execution only when the user explicitly asks for portfolio changes.

2. **High: Concurrent trade and snapshot writes are underspecified for SQLite.**
   The backend will have manual trades, AI-triggered trades, startup snapshots, 30-second snapshot jobs, and SSE/price-cache reads happening in one process. The plan should state that trade execution is atomic in a single database transaction, that cash and position updates are serialized per user, and that portfolio snapshots read a consistent price/cash/position view.

3. **Medium: API response schemas and error shapes are not defined.**
   Add Pydantic-style schemas or JSON examples for `/api/portfolio`, `/api/portfolio/trade`, `/api/watchlist`, and `/api/chat`, including validation errors, rejected AI trades, duplicate watchlist handling, and unavailable-price states.

4. **Medium: Static export plus FastAPI fallback behavior needs to be specified.**
   Unknown non-API paths should return `index.html`, while `/api/*` should return JSON errors. Route precedence, asset caching, and SPA refresh behavior should be explicit.

5. **Medium: Market session behavior is missing.**
   Real market data has closed hours, stale prices, weekends, and API failures. The interface should expose freshness/status metadata, because the frontend connection dot reflects SSE connectivity, not whether prices are current.

6. **Medium: Ticker validation and normalization should be centralized.**
   Define uppercase normalization, allowed characters, length limits, duplicate behavior, and unsupported ticker behavior across watchlist, manual trades, AI trades, market data, and positions.

7. **Low: The "browser opens" first-launch requirement is not fully aligned with Docker.**
   A container cannot reliably open the host browser. Clarify that `docker run` prints the URL, while host-side scripts may open the browser.

8. **Low: Monetary precision should avoid floating point in persisted calculations.**
   The schema uses `REAL`. Either explicitly accept small floating-point artifacts for this demo or require integer cents/decimal handling for cash and trade values.

## Overall

The plan is coherent and scoped well for a capstone build. The highest-risk remaining gap is contract precision: backend/frontend schemas, ticker validation, trade atomicity, and LLM action policy.
