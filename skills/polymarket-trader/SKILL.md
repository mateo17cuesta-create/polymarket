# Polymarket Arbitrage Bot

description: Find and exploit mathematical pricing inconsistencies between related markets in the same Polymarket event. When P(A) > P(B) but B should be more probable than A by definition, buy the underpriced side. Triggers on "polymarket trade", "run the bot", "trade polymarket", or any invocation of the polymarket-bot:trade command.

---

## Objective

Pure arbitrage: find events where Polymarket has multiple related markets whose prices violate mathematical logic. These violations are not opinions — they are provable mispricing.

**Examples of violations:**
- "BTC above $80k by June" (45%) > "BTC above $70k by June" (38%) → IMPOSSIBLE. P(>80k) can never exceed P(>70k)
- "X wins championship" (30%) > "X reaches the final" (25%) → IMPOSSIBLE. You can't win without reaching the final
- "Event happens by March" (40%) > "Event happens by June" (35%) → IMPOSSIBLE. A longer window can't have lower probability

---

## Step 1: Check Balance

Call `mcp__polymarket-mcp__get_balance`.

- Balance has 6 decimal places (e.g. `10082693` = $10.08 USDC)
- If balance < $5, abort: "Insufficient balance."
- Max trades = floor(balance_in_usdc / 5), capped at 2

---

## Step 2: Fetch Events with Multiple Markets

Call `COMPOSIO_SEARCH_FETCH_URL_CONTENT` with:
```
https://gamma-api.polymarket.com/events?limit=50&active=true&closed=false&order=volume&ascending=false
```

Keep only events where:
- `markets` array has **2 or more** entries
- All markets have `active: true` and `closed: false` and `acceptingOrders: true`
- Total event `liquidity` > 20000

---

## Step 3: Identify Arbitrage Type and Extract Prices

For each multi-market event, fetch full market data via `COMPOSIO_SEARCH_FETCH_URL_CONTENT`:
```
https://gamma-api.polymarket.com/markets?slug=SLUG
```

Extract per market:
- `outcomePrices[0]` = YES price
- `clobTokenIds[0]` = YES token ID
- `clobTokenIds[1]` = NO token ID
- `orderMinSize`
- `groupItemTitle` = distinguishing factor
- `slug`

Classify into:

### Type A — Price Ladder
Markets differ by numeric threshold. Rule: higher threshold = lower probability.
Violation: P(above $80k) > P(above $70k)

### Type B — Date Ladder
Markets differ by deadline. Rule: earlier deadline = lower or equal probability.
Violation: P(happens by March) > P(happens by June)

### Type C — Hierarchy
Logical containment. Rule: P(subset) ≤ P(superset).
Violation: P(wins title) > P(reaches final)

---

## Step 4: Detect Violations

For each event apply the rule and find violations.

```
violation_magnitude = |price_A - price_B|
```

Minimum to trade: **0.05 (5 percentage points)**

Always BUY the **underpriced** market (the one whose YES price is too low).

---

## Step 5: Rank and Execute

Sort violations by magnitude descending. For each violation ≥ 0.05:

Call `mcp__polymarket-mcp__place_order`:
- `token_id`: YES token ID of underpriced market
- `side`: "BUY"
- `size`: "5"
- `price`: current YES price + 0.01, capped at 0.97, rounded to 2 decimals

Stop after 2 trades.

---

## Step 6: Session Report

```
## Polymarket Arbitrage Bot — [DATE]

### Balance: $X.XX USDC | Trades: N | Spent: $N.NN

### VIOLATIONS FOUND
| Event | Underpriced Market | Price | Overpriced Market | Price | Gap | Action |
|-------|-------------------|-------|-------------------|-------|-----|--------|

### BELOW THRESHOLD
| Event | Gap | Reason skipped |

### SUMMARY
- Events scanned: N
- Events with 2+ markets: N
- Violations found: N
- Trades executed: N
```

---

## Rules

1. **Only trade mathematical violations** — no news, no opinions
2. **Minimum 5pp violation** — below this, fees eat the profit
3. **Always buy the underpriced side** — never short
4. **Max 2 trades per session**
5. **Skip if `acceptingOrders: false`**
6. **Skip if price > 0.95 or < 0.05** — likely already resolved
7. **Read both market descriptions** before trading — skip if resolution criteria differ

---

## Edge Cases

- **Different resolution sources between markets**: skip
- **One market near resolution (>0.95 or <0.05)**: skip
- **outcomePrices missing**: fetch individual slug for fresh data
- **orderMinSize > 5**: skip, can't afford minimum
