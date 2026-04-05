# Polymarket Arbitrage Bot

description: Risk-free arbitrage — find mathematical pricing violations between related markets, execute both legs (buy YES underpriced + buy NO overpriced) to lock in guaranteed profit in ALL scenarios. Resolution date is irrelevant — profit is locked at execution.

---

## Objective

When two related markets violate mathematical logic, execute both legs simultaneously:
1. **Buy YES** on the underpriced market
2. **Buy NO** on the overpriced market

Profit is locked in at the moment of execution. It does not matter when the market resolves.

**Example:**
```
BTC >$80k → YES 45%, NO 55%
BTC >$70k → YES 38%, NO 62%

Violation: P(>80k) > P(>70k) — mathematically impossible

Leg 1: Buy YES on BTC >$70k @ 0.38 → pays $13.15 if YES
Leg 2: Buy NO  on BTC >$80k @ 0.55 → pays $9.09 if NO

BTC < $70k  → -$5 + $9.09 = +$4.09  ✓
$70k-$80k   → +$13.15 + $9.09 = +$22.24 ✓
BTC > $80k  → +$13.15 - $5 = +$8.15  ✓
```

---

## Step 1: Check Balance

Call `mcp__polymarket-mcp__get_balance`.
- Each position = 2 x $5 = $10
- If balance < $10, abort
- Max 1 position with ~$10 balance

---

## Step 2: Fetch Events with Multiple Markets

Fetch: `https://gamma-api.polymarket.com/events?limit=50&active=true&closed=false&order=volume&ascending=false`

Keep events where:
- `markets` array has 2+ entries
- `liquidity` > 20000
- All markets `active: true`, `closed: false`

---

## Step 3: Extract Prices

For each qualifying event, fetch per market: `https://gamma-api.polymarket.com/markets?slug=SLUG`

Extract:
- `outcomePrices[0]` = YES price
- `outcomePrices[1]` = NO price
- `clobTokenIds[0]` = YES token ID
- `clobTokenIds[1]` = NO token ID
- `orderMinSize` (must be ≤ 5)
- `acceptingOrders` (must be true)
- `groupItemTitle`

Batch via `COMPOSIO_MULTI_EXECUTE_TOOL`.

---

## Step 4: Detect Violations and Rank

### Violation types:

**Type A — Price Ladder**: higher numeric threshold must have lower YES price.
Violation: P(>$80k) > P(>$70k)

**Type B — Date Ladder**: earlier deadline must have lower or equal YES price.
Violation: P(by March) > P(by June)

**Type C — Hierarchy**: P(wins) ≤ P(reaches final).

### Calculate guaranteed profit for each violation:
```
cost = $10
payout_leg1 = 5 / yes_price_underpriced
payout_leg2 = 5 / (1 - yes_price_overpriced)
worst_case = min(payout_leg2, payout_leg1 + payout_leg2)
min_profit = worst_case - 10
```

### Filter and rank:
- Discard if min_profit ≤ $0.50
- **Pick the violation with the highest min_profit** — resolution date is irrelevant, profit locks in at execution
- Tiebreak: largest violation gap wins

---

## Step 5: Execute Both Legs

For the single best opportunity:

**Leg 1 — Buy YES on underpriced market:**
- `token_id`: YES token of underpriced market
- `side`: BUY, `size`: 5
- `price`: yes_price + 0.01, capped at 0.97

**Leg 2 — Buy NO on overpriced market:**
- `token_id`: NO token of overpriced market
- `side`: BUY, `size`: 5
- `price`: no_price + 0.01, capped at 0.97

If Leg 1 fails, do NOT place Leg 2. If Leg 2 fails after Leg 1 fills, report it immediately.

---

## Step 6: Session Report

```
## Polymarket Arbitrage Bot — [DATE]

### Balance: $X.XX | Position executed: Y/N | Spent: $N.NN

### EXECUTED
| Event | Leg 1 BUY YES | Price | Leg 2 BUY NO | Price | Min profit | Max profit |
|-------|--------------|-------|--------------|-------|------------|------------|

### ALL VIOLATIONS FOUND
| Event | Type | Gap | Min profit | Status |
|-------|------|-----|------------|--------|

### SUMMARY
Events scanned: N | With 2+ markets: N | Violations: N | Profitable: N | Executed: 1
```

---

## Rules
1. Always execute BOTH legs — one leg alone is a directional bet, not arbitrage
2. Min guaranteed profit > $0.50 after fees
3. No restriction on resolution date — profit is locked in at execution
4. Pick the single violation with highest guaranteed profit
5. Skip if: acceptingOrders=false, price >0.95 or <0.05, orderMinSize>5
6. Skip if resolution criteria differ between the two markets
7. If one leg fails, do not place the other
