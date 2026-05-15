# PLAN.md Review

**Reviewer:** Claude (claude-sonnet-4-6)
**Date:** 2026-05-14
**Document reviewed:** `planning/PLAN.md`
**Current project status:** Market data subsystem complete; backend shell exists; frontend, database, portfolio, chat, and Docker layers not yet built.

---

## Overall Assessment

The plan is well-structured, coherent, and appropriate for its stated purpose: a capstone project for an agentic AI coding course. The scope is well-bounded, the architectural choices are justified, and the document is detailed enough for independent agents to work from without constant clarification. The completed market data component confirms the plan is executable.

The issues below are ranked by severity: **blocking** issues must be resolved before the relevant component is built; **significant** issues will cause integration friction if unaddressed; **minor** issues are polish and clarity improvements.

---

## Blocking Issues

### 1. Watchlist / SSE Coupling Is Underspecified

The plan states the SSE stream pushes updates for "all tickers known to the system." The market data layer is initialized with the 10 seed watchlist tickers. But the watchlist is dynamic — the user (and the AI) can add and remove tickers at runtime.

The plan does not specify:

- When a ticker is added to the watchlist via `POST /api/watchlist`, who calls `source.add_ticker(ticker)` on the running market data source, and how?
- When a ticker is removed via `DELETE /api/watchlist/{ticker}`, who calls `source.remove_ticker(ticker)`?
- If the user removes all instances of a ticker from the watchlist, does the simulator keep generating prices for it in the cache? If yes, the cache grows unboundedly. If no, what is the cleanup trigger?

The market data summary shows `add_ticker` and `remove_ticker` exist on the `MarketDataSource` interface. The plan needs to describe the integration path — specifically that the watchlist API handlers must call through to the running source instance. The backend agent will otherwise implement the REST endpoints and the market data source as two isolated subsystems with no connection.

### 2. Portfolio Snapshot Background Task Is Not Defined

Section 7 states portfolio snapshots are "recorded every 30 seconds by a background task." Section 8 lists no endpoint or mechanism for this task. Section 11 (Docker/Deployment) does not mention it.

The plan does not specify:

- Where this task lives (same FastAPI lifespan as the market data poller? a separate asyncio task?).
- What happens to the snapshot interval when no positions are held (should it still record cash-only value?).
- Whether the snapshot at trade execution is atomic with the trade (same DB transaction) or fire-and-forget.

Without this, the P&L chart will either have no data or be implemented inconsistently by different agents.

### 3. LLM Error Handling Path Is Incomplete

Section 9 says "if a trade fails validation, the error is included in the chat response." It does not specify:

- What the backend returns to the frontend when the LLM call itself fails (network timeout, OpenRouter error, malformed JSON that does not match the structured output schema).
- Whether partial success is possible — e.g., the LLM requests three trades and one fails validation. Do the other two execute? Does the response message reflect only the failed one?
- What the frontend should display in these cases.

This is blocking because the frontend agent needs to know the error response shape to render it, and the backend agent needs a specified behavior rather than inventing one.

---

## Significant Issues

### 4. `POST /api/portfolio/trade` Request Body Is Incomplete

The endpoint table in Section 8 shows the request body as `{ticker, quantity, side}` but does not define:

- The allowed values for `side` (the schema says `"buy"` or `"sell"` in the trades table, but this is not stated in the API table).
- Whether `quantity` must be a positive integer or whether fractional quantities are accepted at this endpoint (the schema supports fractional shares, but the trade bar UI appears to use integer quantities).
- The response body shape on success or failure.

The backend and frontend agents will make different assumptions about these unless they are specified.

### 5. Chat History Truncation Strategy Is Missing

Section 9 says the backend "loads recent conversation history." It does not define what "recent" means — how many messages, how many tokens, or what strategy to use when the history grows large. Without a concrete rule, different implementations will produce different behavior, and long sessions may silently exceed the model's context window.

A concrete rule is needed: for example, "include the last N user/assistant message pairs" or "include messages from the last 24 hours, capped at N pairs."

### 6. Daily Change Percent in the Watchlist Panel Has No Data Source

Section 10 specifies the watchlist panel must show "daily change %." The SSE stream and price cache provide current price, previous price, and a `change` field — but `change` is the delta from the last tick, not from the day's open price.

The plan provides no mechanism to obtain or calculate a true daily percentage change. The simulator has no concept of a "day open" price. This feature as specified cannot be implemented with the current market data design. The plan should either:

- Clarify that "daily change %" means change since the simulator started (i.e., change from seed price), or
- Remove the feature from the watchlist spec, or
- Add a "day open" price field to the simulator seed and price cache.

### 7. The `positions` Table Has No Index on `(user_id, ticker)`

Section 7 defines a UNIQUE constraint on `(user_id, ticker)` in the positions table, but does not specify an index. SQLite creates an index automatically for UNIQUE constraints, so this is not a bug — but the watchlist table has the same pattern and the same implicit behavior. The schema section should note this explicitly so the backend agent does not add redundant explicit indexes or miss the implicit ones.

This is minor for SQLite performance at single-user scale but matters for correctness of the schema definition document.

### 8. `GET /api/watchlist` Response Shape Is Unspecified

The endpoint table says this returns "current watchlist tickers with latest prices," but the exact JSON shape is not defined. The frontend agent needs to know:

- Is it an array of `{ticker, price, previous_price, direction, change}` objects?
- Does it include the sparkline history accumulated server-side, or only the current price (with sparklines accumulated client-side from SSE)?

Section 2 says sparklines are "accumulated on the frontend from the SSE stream since page load," which implies server-side sparkline history is not needed in the watchlist response. This should be stated explicitly in Section 8 to prevent the backend agent from over-building.

---

## Minor Issues

### 9. Model Name in Section 9 Does Not Match the cerebras Skill

Section 9 instructs agents to use the model `openrouter/openai/gpt-oss-120b`. The cerebras skill in the project uses LiteLLM via OpenRouter with the Cerebras inference provider. It is worth confirming this model ID is current and matches what the cerebras skill will actually invoke, since model IDs on OpenRouter change as providers update their offerings.

### 10. `backend/db/` vs Top-Level `db/` Naming Collision Risk

The directory structure in Section 4 shows both `backend/db/` (schema definitions, seed data) and `db/` at the root (runtime SQLite file). These are different directories with different purposes. The naming is potentially confusing for agents that may conflate the two. The Key Boundaries section does distinguish them, but it would be clearer to rename `backend/db/` to `backend/schema/` or `backend/migrations/` to eliminate ambiguity.

### 11. `docker-compose.yml` Is Listed as "Optional" but the Test Infrastructure Requires `docker-compose.test.yml`

Section 4 lists `docker-compose.yml` as an "Optional convenience wrapper." Section 12 describes a `docker-compose.test.yml` in `test/` as the E2E test infrastructure. The Playwright E2E tests are not optional — they are part of the defined testing strategy. The distinction between the optional production compose file and the required test compose file should be made more explicit in Section 4 to prevent an agent from treating compose as entirely optional.

### 12. No Defined Behavior for Selling Down to Zero

Section 12 (unit tests) lists "selling more than owned" as an edge case to test. It does not address what happens when a user sells exactly their full position: should the positions row be deleted, or set to `quantity=0`? The plan supports fractional shares, so a sell that results in `quantity < 0.0001` due to floating-point arithmetic is also possible. A rule for position cleanup should be stated to ensure backend and frontend handle the empty-position case consistently (the positions table and heatmap both need to agree on whether zero-quantity positions exist).

### 13. The `.env` File Is Listed in the Directory Structure but Is Also Gitignored

Section 4 lists `.env` in the directory structure and Section 5 says it is gitignored (with `.env.example` committed). This is standard practice but the directory tree should show `.env.example` instead of `.env` to reflect what is actually in the repository. A new contributor following the directory structure would not find `.env` in the repo.

---

## Suggestions (Not Issues)

**Suggestion A — Add a `GET /api/prices/{ticker}` endpoint for the main chart area.** The main chart panel shows "price over time" for a selected ticker, but the only price source is the SSE stream (which starts accumulating from page load). On first load or after navigation, there will be no historical data for the chart. An endpoint that returns the recent price history from the simulator's internal buffer (or from `portfolio_snapshots`-style records) would give the chart immediate data. This is not critical if the UX accepts a chart that fills in gradually, but that behavior should be stated explicitly.

**Suggestion B — Specify the SSE event format concretely.** Section 6 says each event contains "ticker, price, previous price, timestamp, and change direction." A concrete JSON example (matching what `PriceUpdate` already produces in the market data implementation) would prevent the frontend agent from guessing field names and casing.

**Suggestion C — Clarify what the AI chat panel shows for executed trades.** Section 10 says "trade executions and watchlist changes shown inline as confirmations" in the chat panel, but the `actions` field in `chat_messages` stores executed actions as JSON. The frontend agent needs to know whether these confirmations come from the API response or are reconstructed from the stored `actions` field. Since the plan says the backend returns the "complete JSON response," this likely means the response includes the executed actions directly — but this should be stated explicitly in Section 9.

---

## Summary Table

| # | Category | Severity | Section |
|---|----------|----------|---------|
| 1 | Watchlist/SSE runtime coupling | Blocking | 6, 8 |
| 2 | Portfolio snapshot task definition | Blocking | 7, 8 |
| 3 | LLM error handling path | Blocking | 9 |
| 4 | Trade endpoint request/response shape | Significant | 8 |
| 5 | Chat history truncation strategy | Significant | 9 |
| 6 | Daily change % data source | Significant | 6, 10 |
| 7 | Schema index documentation | Significant | 7 |
| 8 | Watchlist GET response shape | Significant | 8 |
| 9 | Model name currency | Minor | 9 |
| 10 | backend/db naming collision | Minor | 4 |
| 11 | Optional vs required compose files | Minor | 4, 12 |
| 12 | Zero-quantity position behavior | Minor | 7, 12 |
| 13 | .env in directory tree | Minor | 4, 5 |
