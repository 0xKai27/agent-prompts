Generate a comprehensive report for this YIELD OPTIMIZER vault. Follow every section below in order.

The strategy `config` JSON defines:
- `strategies`: enabled strategy types (e.g. `["lending", "stablecoin-carry", "staking-derivatives", "pendle-pt", "pendle-yt"]`)
- `whitelistedMarkets`: lending protocols to consider
- `morphoRiskTier`: Morpho risk tolerance (`"none"` / `"low"` / `"all"`)
- `minApyDiffBps`: minimum improvement to justify rotation (default 50 bps)

---

## Section 1 — Data Collection (mandatory tool calls)

You MUST call all of the following tools before writing the report:

1. **`factor_vault_analytics`** — get current positions, balances, total USD value, and yield data.
2. **`factor_get_lending_tokens`** — call once per protocol in `config.whitelistedMarkets` to get lending market APYs.
3. **`defi_llama_yields`** — cross-reference APYs and get historical trend data.

Do NOT skip any of these calls. Do NOT write the report from memory or assumptions.

---

## Section 2 — Current Strategy & Positions

**Active strategy type**: identify which of the enabled strategies is currently deployed (lending / stablecoin-carry / staking-derivatives / pendle-pt / pendle-yt / none if idle).

For each position, report:

| Token | Strategy | Protocol/Venue | Amount | USD Value | Current Yield |
|-------|----------|----------------|--------|-----------|---------------|
| ...   | ...      | ...            | ...    | ...       | ...           |

**Total vault value**: $X.XX USD

---

## Section 3 — Idle Token Check

List any tokens in the vault wallet NOT deployed to any yield strategy.

- If idle tokens exist: report token, amount, USD value, flag as **IDLE — earning 0%**.
- If no idle tokens: "All tokens are deployed."

---

## Section 4 — Per-Strategy Yield Assessment

Evaluate each strategy type listed in `config.strategies`. Only assess strategies that are enabled.

### 4a — Lending
- Best available supply APY across all `config.whitelistedMarkets`
- Protocol, market, and APY for the top 3 options
- Morpho markets: include LLTV, utilization, collateral type, risk tier rating
- DeFi Llama 7d/30d mean for the top market

### 4b — Stablecoin Carry (if enabled and vault holds stablecoins)
- sUSDe (Ethena) current native yield
- sUSDS (Sky/Maker) current native yield
- Compare against best lending APY for the same stablecoin
- Note: carry involves swapping into yield-bearing stablecoin — depeg risk applies

### 4c — Staking Derivatives (if enabled and vault holds ETH/wstETH/stETH)
- Base staking APR (Lido wstETH)
- Additional lending APY available on wstETH (if supplied to Aave/Morpho/Compound)
- Combined yield: staking APR + lending APY
- Compare against simply lending raw ETH

### 4d — Pendle PT (if enabled)
- Available PT markets and their implied fixed APY
- For any HELD PT position: maturity date, days to expiry, locked rate
  - If maturity < 7 days: flag as **EXPIRY WARNING — position should be redeemed or rolled**
  - If maturity < 30 days: flag as **APPROACHING MATURITY — plan exit strategy**
- Compare locked PT rate against current lending/carry alternatives

### 4e — Pendle YT (if enabled)
- Available YT markets and leveraged variable APY
- For any HELD YT position: maturity date, days to expiry, current leveraged yield
  - If maturity < 14 days: flag as **CRITICAL — YT decays to zero at maturity, exit immediately**
- Is the YT yield at least 2x the best lending rate? If not, flag as **RISK/REWARD UNFAVORABLE**

---

## Section 5 — Strategy Comparison Table

| Strategy | Expected APY | Risk Level | Notes |
|----------|-------------|------------|-------|
| Lending (best market) | X.XX% | LOW/MEDIUM | protocol, market name |
| Stablecoin Carry | X.XX% | MEDIUM | depeg risk on sUSDe/sUSDS |
| Staking Derivatives | X.XX% | LOW | staking APR + lending combo |
| Pendle PT | X.XX% | LOW-MEDIUM | fixed rate, locked until maturity |
| Pendle YT | X.XX% | HIGH | leveraged, decays to zero |

Sort by Expected APY descending. Mark the currently active strategy. Mark the best available strategy.

**Current strategy yield**: X.XX%
**Best available yield**: X.XX% (strategy name)
**Difference**: +/- XXX bps

---

## Section 6 — Rotation Recommendation

Compare the current strategy yield against the best available:

- If difference < `config.minApyDiffBps`: **NO ROTATION NEEDED** — current strategy is within threshold.
- If difference >= `config.minApyDiffBps`: **ROTATION RECOMMENDED** — switching from [current] to [best] would gain +XXX bps. Recommend triggering a rebalance job.
- If tokens are idle: **IMMEDIATE DEPLOYMENT RECOMMENDED** — idle capital earning nothing.

If a rotation is recommended, note any execution considerations:
- Would it require exiting a Pendle PT before maturity (potential loss)?
- Would it require swapping staking derivatives (slippage on large amounts)?
- How many transactions would be needed?

---

## Section 7 — Risk Assessment

- **Protocol diversification**: is all capital in one protocol/strategy? Flag concentration risk.
- **Depeg risk**: if holding staking derivatives (wstETH, rETH, cbETH) or yield-bearing stablecoins (sUSDe, sUSDS), check exchange rate vs underlying. Flag if deviation > 1%.
- **Pendle expiry risk**: any PT within 7 days of maturity? Any YT within 14 days? Flag with urgency level.
- **Morpho market health**: if any Morpho positions, report utilization and LLTV. Flag if utilization > 90%.
- **Smart contract risk**: rate each protocol in use (LOW / MEDIUM / HIGH based on TVL, audit history, maturity).

---

## Section 8 — Summary

Write a 3-4 sentence executive summary:
- What strategy is active and what yield is it generating?
- Is it the optimal strategy or should a rotation happen?
- Are there any risk flags or time-sensitive issues (Pendle expiry, depeg, idle tokens)?
- What is the overall vault health?