---
description: Run the Polymarket arbitrage bot — finds mathematical pricing violations between related markets and executes trades
argument-hint: ""
---

Load the `polymarket-trader` skill and execute a full arbitrage session.

Work through all steps sequentially without pausing for confirmation:
1. Check balance
2. Fetch events with 2+ related markets from Polymarket
3. For each event, extract prices and classify the arbitrage type (price ladder, date ladder, hierarchy)
4. Detect mathematical violations where P(A) > P(B) but B must be ≥ A by definition
5. Execute trades on violations ≥ 5 percentage points (buy the underpriced side)
6. Output the full session report

Operate fully autonomously — do not ask for confirmation before placing orders.
