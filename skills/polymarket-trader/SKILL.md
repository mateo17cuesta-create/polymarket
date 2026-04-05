# Polymarket Arbitrage Bot

description: Risk-free arbitrage — buy YES on underpriced market and NO on overpriced market simultaneously to lock in profit in ALL scenarios.

## Objective

When two related markets violate mathematical logic, execute both legs:
1. Buy YES on the underpriced market
2. Buy NO on the overpriced market

This locks in profit regardless of outcome.

**Example:**
```
BTC >$80k → YES 45%, NO 55%
BTC >$70k → YES 38%, NO 62%

Violation: P(>80k) > P(>70k) — impossible

Leg 1: Buy YES on BTC >$70k @ 0.38 → pays $13.15 if yes
Leg 2: Buy NO  on BTC >$80k @ 0.55 → pays $9.09 if no

BTC < $70k  → +$4.09  ✓
$70k-$80k   → +$22.24 ✓
BTC > $80k  → +$8.15  ✓
```

## Step 1: Check Balance
Call get_balance. Each position = 2 x $5 = $10 total. If balance < $10, abort. Max 1 position with ~$10 balance.

## Step 2: Fetch Events
Fetch: https://gamma-api.polymarket.com/events?limit=50&active=true&closed=false&order=volume&ascending=false
Keep events with 2+ markets and liquidity > 20000.

## Step 3: Extract Prices
For each event's markets fetch: https://gamma-api.polymarket.com/markets?slug=SLUG
Extract: outcomePrices[0] (YES), outcomePrices[1] (NO), clobTokenIds[0] (YES token), clobTokenIds[1] (NO token), orderMinSize, acceptingOrders, groupItemTitle.
Batch via COMPOSIO_MULTI_EXECUTE_TOOL.

## Step 4: Detect Violations

Type A — Price Ladder: higher threshold must have lower probability. P(>$80k) ≤ P(>$70k) always.
Type B — Date Ladder: earlier deadline must have lower or equal probability.
Type C — Hierarchy: P(wins) ≤ P(reaches final).

For each violation calculate minimum guaranteed profit:
```
cost = $10 (two legs of $5 each)
payout_leg1 = 5 / yes_price_underpriced
payout_leg2 = 5 / no_price_overpriced
worst_case = min(payout_leg2, payout_leg1 + payout_leg2)
min_profit = worst_case - 10
```
Only trade if min_profit > $0.50.

## Step 5: Execute Both Legs

Leg 1 — Buy YES on underpriced market:
- token_id: YES token of underpriced market
- side: BUY, size: 5, price: yes_price + 0.01 (max 0.97)

Leg 2 — Buy NO on overpriced market:
- token_id: NO token of overpriced market
- side: BUY, size: 5, price: no_price + 0.01 (max 0.97)

If either leg fails, do not place the other.

## Step 6: Report
```
## Polymarket Arbitrage Bot — [DATE]
### Balance: $X.XX | Positions: N | Spent: $N.NN

### EXECUTED
| Event | Leg 1 BUY YES | Price | Leg 2 BUY NO | Price | Min profit | Max profit |

### NOT TRADED
| Event | Gap | Min profit | Reason |

### SUMMARY: scanned / violations / profitable / executed
```

## Rules
1. Always execute BOTH legs — one leg alone is a directional bet, not arbitrage
2. Min guaranteed profit > $0.50 after fees
3. Both markets must be in the same event
4. Skip if either market: acceptingOrders=false, price >0.95 or <0.05, orderMinSize>5
5. Skip if resolution criteria differ between markets
6. If one leg fails, do not place the other
