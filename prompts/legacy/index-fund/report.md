Generate a comprehensive report for this INDEX FUND vault. Follow every section below in order.

The strategy `config` JSON defines `allocations`: an array of `{ tokenAddress, targetPct }` entries that represent the desired portfolio composition.

---

## Section 1 — Data Collection (mandatory tool call)

You MUST call the following tool before writing the report:

1. **`factor_vault_analytics`** — get current balances, USD values per token, and total vault value.

Do NOT write the report from memory or assumptions.

---

## Section 2 — Portfolio Overview

**Total vault value**: $X.XX USD

List every token in the vault:

| Token | Amount | USD Value | Current % | Target % | Drift | Status |
|-------|--------|-----------|-----------|----------|-------|--------|
| ...   | ...    | ...       | XX.X%     | XX.X%    | +/-X.X% | OK / DRIFTED / CRITICAL |

**Status definitions:**
- **OK**: drift <= 5% (within acceptable range)
- **DRIFTED**: drift > 5% and <= 10% (rebalance recommended)
- **CRITICAL**: drift > 10% (urgent rebalance needed)

Drift is calculated as: `currentPct - targetPct` (positive = overweight, negative = underweight).

---

## Section 3 — Idle Token Check

Are there any tokens in the vault wallet that are NOT part of the target allocation (unexpected tokens)?

- If yes: report token, amount, USD value, and flag as **UNEXPECTED TOKEN — not in allocation config**.
- If no unexpected tokens: "All held tokens match the allocation config."

Also check: is any allocated token at 0% when it should have a non-zero target? If so, flag as **MISSING ALLOCATION — [token] has 0% actual vs [X]% target**.

---

## Section 4 — Drift Analysis

For each token, provide a detailed drift breakdown:

- **Overweight tokens** (current > target): list with exact drift %, sorted by largest drift first
- **Underweight tokens** (current < target): list with exact drift %, sorted by largest drift first
- **On-target tokens** (within 1% drift): list as "well-balanced"

**Maximum drift**: X.X% on [token] (overweight/underweight)
**Average absolute drift**: X.X% across all tokens

---

## Section 5 — Rebalance Assessment

Determine whether a rebalance is needed:

**If any token has drift > 5%:**
- **REBALANCE RECOMMENDED**
- List the specific swaps needed to restore target allocation:
  - Swap $X.XX of [overweight token] to [underweight token]
  - (repeat for each needed swap)
- **Number of swaps needed**: N
- **Estimated gas cost consideration**: N swaps via OpenOcean. Gas is sponsored, but each swap incurs DEX fees (~0.1-0.3%) and slippage.
- **Total rebalance cost estimate**: approximate the DEX fees as a percentage of the swap amounts.

**If all tokens have drift <= 5%:**
- **NO REBALANCE NEEDED** — all allocations are within the 5% tolerance band.
- Note the largest drift and which token it belongs to (for monitoring).

---

## Section 6 — Performance Since Last Rebalance

Using data from `factor_get_executions` (if available), report:

- Date/time of last rebalance execution
- Per-token price change since last rebalance (if derivable from vault metrics history)
- Whether drift has been growing or shrinking since the last rebalance

If no execution history is available, note "No prior rebalance history found."

---

## Section 7 — Summary

Write a 2-3 sentence executive summary:
- Is the portfolio within target allocation or does it need rebalancing?
- What is the largest drift and on which token?
- Total vault value and overall health status (HEALTHY / NEEDS REBALANCE / CRITICAL DRIFT).