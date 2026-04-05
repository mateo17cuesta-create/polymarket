---
description: Run the autonomous Polymarket trading bot — scans markets, finds unpriced news, executes trades
argument-hint: ""
---

Load the `polymarket-trader` skill and execute a full autonomous trading session.

Work through all steps in the skill sequentially without pausing for confirmation:
1. Check balance
2. Fetch liquid active markets
3. Get current prices
4. Search Composio for news published in the last 6 hours
5. Analyze each market for pricing edge
6. Execute trades on high-conviction opportunities ($1 USDC each, max 5 trades)
7. Output the full session report

Operate fully autonomously — do not ask for confirmation before placing orders.
