# Polymarket Autonomous Trader

description: Scan active Polymarket prediction markets for pricing inefficiencies using real-time news. Identify information published in the last 2-6 hours that is not yet reflected in market prices, then execute trades autonomously. Triggers on "polymarket trade", "run the bot", "trade polymarket", or any invocation of the polymarket-bot:trade command.

---

## Objective

Find and exploit **information asymmetry**: recent news (last 2-6 hours) that changes the true probability of a market outcome but has not yet been priced in by market participants. Execute minimum-size trades (5 USDC) on the highest-conviction opportunities.

---

## Step 1: Check Balance

Call `mcp__polymarket-mcp__get_balance` to confirm available USDC.

- The raw balance has 6 decimal places (e.g., `10082693` = $10.08 USDC).
- If balance is below $5, abort with a message: "Insufficient balance to trade."
- Calculate max trades possible (floor(balance / 5), capped at 2 trades per session).

---

## Step 2: Fetch Active Markets

Call `mcp__polymarket-mcp__get_markets` with these exact parameters:
- `active: true`
- `closed: false`
- `archived: false`
- `order: "volume"`
- `ascending: false`
- `limit: 50`
- `liquidity_min: 10000`

From the results, filter for markets that:
1. Are **binary** (Yes/No outcomes only)
2. Have a **clear, resolvable question** about a specific event
3. Are **NOT sports/esports match outcomes** (too random, no news edge)
4. Prefer markets resolving within **the next 1-90 days** (news has more impact)

Select the **top 15 markets** by volume after filtering.

---

## Step 3: Get CLOB Token IDs and Current Prices

The `get_markets` tool returns internal IDs. For trading and price lookups, you need the **CLOB token IDs** from the Polymarket Gamma API.

For each selected market, fetch via Composio `COMPOSIO_SEARCH_FETCH_URL_CONTENT`:
```
https://gamma-api.polymarket.com/markets?slug=MARKET-SLUG-HERE
```

From the JSON response, extract:
- `clobTokenIds`: array of [YES_token_id, NO_token_id]
- `outcomePrices`: array of [YES_price, NO_price] (current probability as decimal)
- `orderMinSize`: minimum order size in USDC (typically 5)
- `acceptingOrders`: must be `true` to trade

Batch multiple markets in one `COMPOSIO_MULTI_EXECUTE_TOOL` call for speed.

**Skip markets** where the YES price is between 0.45 and 0.55 (too close to 50/50). Focus on markets with more extreme prices where news can cause meaningful movement.

Note: `get_market_prices` from the MCP tool requires the CLOB token_id, not the internal market ID.

---

## Step 4: Search for Recent News

For each remaining market (aim for 10-15), search for breaking news using Composio.

Use `COMPOSIO_SEARCH_NEWS` for each market:
- Build a 2-3 keyword query from the market title
- Use `when: "d"` (last 24 hours)
- Also try `COMPOSIO_SEARCH_WEB` for sparse results

Run searches in **parallel** via `COMPOSIO_MULTI_EXECUTE_TOOL`.

---

## Step 5: Analyze Each Market for Edge

### 5a. Assess News Recency
- Filter news to those published in the **last 6 hours**
- If no news in the last 6 hours → **no edge** → skip
- If news is 6-24 hours old → lower confidence

### 5b. Estimate True Probability
1. What does this news imply for the outcome?
2. How significant/credible is the source?
3. My estimated probability: X%

### 5c. Calculate Edge
```
edge = |my_estimated_probability - current_market_price|
direction = BUY YES (if my_estimate > market_price) or BUY NO (if lower)
```

### 5d. Confidence Thresholds
| Edge | Action |
|------|--------|
| < 10% | Skip |
| 10-15% | Skip unless multiple strong sources |
| 15-25% | Trade $5 |
| > 25% | Trade $5 |

**Maximum 2 trades per session** (balance ~$10 USDC, min order $5)

---

## Step 6: Execute Trades

Call `mcp__polymarket-mcp__place_order`:
- `token_id`: CLOB token ID for YES or NO outcome
- `side`: "BUY"
- `size`: "5" (minimum confirmed via API `orderMinSize`)
- `price`: current_price + 0.02, capped at 0.97, rounded to 2 decimals

---

## Step 7: Session Report

```
## Polymarket Bot Session — [DATE]

### Balance: $X.XX USDC | Trades: N | Spent: $N.NN

### TRADED
| Market | Direction | Paid | My Estimate | Edge | Signal |
|--------|-----------|------|-------------|------|--------|

### SKIPPED (top candidates)
| Market | Price | My Estimate | Edge | Reason |
|--------|-------|-------------|------|--------|

### Key News
[Most significant developments found]
```

---

## Rules

1. **Never trade without news from the last 6 hours**
2. **Minimum 5 USDC per trade** — max 2 trades per session with current balance
3. **Never trade sports match outcomes**
4. **Never trade if conflicting signals**
5. **Markets at extreme prices (>0.92 or <0.08)** require very strong news
6. **Fully autonomous** — place all qualifying orders without confirmation

---

## Edge Cases

- **Market resolves today**: only trade if news is within last 2 hours
- **Composio returns no results**: try broader query once, then skip
- **Price between 0.45-0.55**: skip unless edge is very clear
