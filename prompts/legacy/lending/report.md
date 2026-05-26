Generate a comprehensive report for this LENDING vault. Follow every section below in order.

---

## Protocol Reference

**Morpho Blue**: isolated markets — each is a unique (loan asset, collateral, LLTV, oracle) pair. Markets are permissionlessly created; no governance approval required.
- **LLTV**: the LTV at which a borrower is liquidatable. Lower = more collateral buffer for suppliers.
- **Utilization**: borrowed ÷ supplied. High utilization = thin withdrawal liquidity.

**Aave V3 / Compound V3**: pooled markets. Risk parameters set by governance per asset, not per market.

---

## Section 1 — Data Collection (mandatory tool calls)

You MUST call all of the following tools before writing the report:

1. **`factor_vault_analytics`** — get current positions, balances, total USD value, and per-position APY data.
2. **`factor_get_lending_tokens`** — call once per protocol in `config.whitelistedMarkets` to get all available markets and their current supply APY. This gives you the comparison table data.
3. **`defi_llama_yields`** — cross-reference APYs from an independent source. Use this to validate on-chain rates and to retrieve historical APY trend data (7d mean, 30d mean) where available.

Do NOT skip any of these calls. Do NOT write the report from memory or assumptions.

---

## Section 2 — Current Positions

For each supplied position, report:

| Token | Protocol | Market | Amount | USD Value | Current APY |
|-------|----------|--------|--------|-----------|-------------|
| ...   | ...      | ...    | ...    | ...       | ...         |

If the vault has positions across multiple protocols, list each row separately.

**Total vault value**: $X.XX USD

---

## Section 3 — Idle Token Check

List any tokens sitting in the vault wallet that are NOT supplied to any lending protocol.

- If idle tokens exist: report the token symbol, amount, approximate USD value, and flag as **IDLE — earning 0% APY**.
- If no idle tokens: report "All tokens are deployed."

---

## Section 4 — APY Comparison Table

Build a table comparing the current position APY against ALL available markets from `config.whitelistedMarkets`:

| Protocol | Market | Supply APY | DeFi Llama 7d Mean | DeFi Llama 30d Mean | vs Current (bps) |
|----------|--------|------------|---------------------|----------------------|-------------------|
| ...      | ...    | ...        | ...                 | ...                  | +/- XXX bps       |

Sort by Supply APY descending. Highlight the current position's row. Highlight the best available row.

---

## Section 5 — Morpho Market Health (if applicable)

If ANY current position is in a Morpho market, report for each:

- **LLTV**: the market's Liquidation Loan-To-Value ratio.
- **Utilization rate**: current utilization. Flag if very high (withdrawal liquidity may be thin).
- **Collateral type**: name the collateral asset.
- **Config alignment**: is this market consistent with the vault's `config.morphoRiskTier` (`"none"` / `"low"` / `"all"`)?

If no Morpho positions exist, write "No Morpho positions — skipping market health check."

---

## Section 6 — Optimization Assessment

For each supplied token, determine whether the current position is optimal:

1. Identify the best available APY from the comparison table (Section 4).
2. Calculate the difference: `(bestAPY - currentAPY) * 10000` bps.
3. Compare against `config.minApyDiffBps` (default 50 bps = 0.5%).

**Verdict per token:**
- If difference < `minApyDiffBps`: **OPTIMAL** — current position is within threshold. No action needed.
- If difference >= `minApyDiffBps`: **SUBOPTIMAL** — a rebalance to [protocol/market] would gain +XXX bps. Recommend triggering a rebalance job.
- If token is idle: **NOT DEPLOYED** — immediate supply recommended.

---

## Section 7 — Risk Assessment

Evaluate overall vault risk:

- **Protocol concentration**: is 100% of capital in a single protocol? If yes, flag as "Single-protocol concentration risk." If spread across 2+, note the distribution.
- **Protocol maturity**: for each protocol in use, note its approximate TVL and time since deployment. Do not assign risk tier labels — these are observable facts for the user to interpret.
- **Liquidity risk**: can the vault withdraw its full position without significant slippage? Flag if utilization >90% on any market.

---

## Section 8 — Historical APY Trend

Using DeFi Llama data, report the trend for the current position's market:

- 7-day average APY
- 30-day average APY
- Current APY
- Trend direction: RISING / STABLE / FALLING (compare current vs 30d mean)

If historical data is unavailable, note "Historical APY data not available for this market."

---

## Section 9 — Summary

Write a 2-3 sentence executive summary:
- Is the vault healthy and optimally positioned?
- Are there any action items (idle tokens to deploy, better rates to capture, risk flags)?
- What is the overall yield performance?