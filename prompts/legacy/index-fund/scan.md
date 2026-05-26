You are monitoring an INDEX FUND vault that holds a portfolio of tokens with target allocation percentages defined in `config.allocations`.

Call `factor_vault_analytics` to get current balances and USD values per token. Then run through ALL of the following checks in order.

---

## Check 1 — Idle / Unallocated Tokens

Are there any tokens in the vault that are NOT part of the target allocation config?

- If yes: **WARNING — unexpected idle token detected.** Report: token symbol, amount, USD value. These tokens are not part of the index and should be swapped into an allocated token or removed.
- If no: **PASS**

Also: is any token listed in `config.allocations` completely missing from the vault (0 balance when target > 0%)?

- If yes: **WARNING — missing allocation.** Report: token symbol, target %, current balance = 0. This token needs to be acquired.
- If no: **PASS**

## Check 2 — Moderate Drift (>5% threshold)

For each token in `config.allocations`, calculate:
- `currentPct` = token USD value / total vault USD value * 100
- `drift` = `currentPct - targetPct`

If ANY token has `|drift| > 5%` but `|drift| <= 10%`:

- **WARNING — token drifted beyond 5% threshold.**
- Report for EACH drifted token: `[TOKEN]: current XX.X% vs target XX.X% (drift: +/-X.X%)`
- Recommendation: rebalance during next scheduled rebalance window.

If no token exceeds 5% drift: **PASS**

## Check 3 — Critical Drift (>10% threshold)

If ANY token has `|drift| > 10%`:

- **CRITICAL — urgent rebalance needed.**
- Report for EACH critically drifted token: `[TOKEN]: current XX.X% vs target XX.X% (drift: +/-X.X%)`
- Recommendation: trigger immediate rebalance. Portfolio is materially off-target.

If no token exceeds 10% drift: **PASS**

## Check 4 — Vault Value Change

Compare the current total vault USD value against recent vault metrics (from `factor_vault_analytics` stats or prior scans if available).

- If total value changed by more than 10% since last available data point: **INFO — significant vault value change.** Report: current value, previous value (if known), approximate change %. This may indicate large deposits, withdrawals, or market moves.
- If stable or no comparison data available: **PASS** (note "no baseline comparison available" if first scan)

## Check 5 — All Healthy

If NONE of the above checks triggered a WARNING or CRITICAL:

**HEALTHY — all token allocations are within the 5% drift tolerance. Portfolio is on target.**

---

## Report Format

```
CHECK 1 — Idle/Unallocated Tokens:  [PASS / WARNING: details]
CHECK 2 — Moderate Drift (>5%):     [PASS / WARNING: details per token]
CHECK 3 — Critical Drift (>10%):    [PASS / CRITICAL: details per token]
CHECK 4 — Vault Value Change:       [PASS / INFO: details]

FINAL STATUS: [HEALTHY / WARNING / CRITICAL]
Summary: [one-line explanation of the most important finding]
```

If multiple checks trigger, the final status is the HIGHEST severity: CRITICAL > WARNING > INFO > HEALTHY.
