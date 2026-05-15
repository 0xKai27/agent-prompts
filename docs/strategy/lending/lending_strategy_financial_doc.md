# Lending Strategy — Financial Strategy Documentation

**Version:** 1.0  
**Scope:** Steady and Balanced tiers  
**Chain:** Base  
**Denomination:** USDC  
**Last validated:** May 2026

---

## 1. Overview

The Lending Strategy is an automated yield optimisation strategy that lends USDC across decentralised lending protocols on Base. An AI agent manages the position on a 6-hour cycle: it scans available markets, evaluates yield and risk conditions, and moves capital when a sufficiently better opportunity exists and all safety conditions are met. When no market passes the safety gates, the agent either exits to idle (Steady) or holds its current position for one cycle (Balanced).

The strategy never borrows, trades, or takes leverage. It holds a single USDC position at all times, deployed to exactly one lending market per cycle.

Two risk tiers are available:

| Tier | Profile | Estimated APY |
|---|---|---|
| **Steady** | Capital preservation. Conservative gates. Moves infrequently and only for meaningful improvements. Exits to idle when no safe market is available. | 2–4% |
| **Balanced** | Yield optimisation within managed risk bounds. Broader protocol and market access. Holds through short-term disruptions before exiting. | 4–8% |

---

## 2. Protocols in Scope

The agent evaluates three lending protocols on Base each cycle. All protocols are evaluated dynamically — eligibility is assessed at runtime, not fixed at vault creation.

### 2.1 Aave V3

Aave V3 is a pooled, multi-asset lending market. Suppliers deposit USDC into a shared pool from which borrowers draw, posting various assets as collateral. Because the collateral pool is diversified across many borrowers and collateral types, a single collateral failure creates partial bad debt rather than total pool loss. Aave V3 on Base holds over $175M in USDC supply, making it the deepest and most liquid USDC market available on the chain. It is the baseline market for both tiers.

**Protocol risk characterisation:** Lowest. Largest pool, most diversified borrower risk, deepest withdrawal liquidity.

### 2.2 Compound V3

Compound V3 operates a single-base-asset model: USDC is the sole borrowable asset in the Base market, and borrowers post a curated set of collateral assets (WETH, cbETH, cbBTC, LINK, UNI) against it. Unlike Aave's diversified pool, Compound's solvency on Base is shared across all borrowers in a single market — a collateral failure affects the entire USDC pool rather than a subset. The Compound V3 USDC market on Base is currently approximately $9M in total supply, making it materially smaller than Aave.

**Protocol risk characterisation:** Medium. Pooled model with shared solvency risk; smaller Base market size means a single vault represents a larger fraction of total supply and has less withdrawal liquidity under stress.

**Tier eligibility:** Steady excludes Compound Base at current market size (floor: $25M). Balanced includes it (floor: $10M), accepting the smaller pool as a higher-risk-adjusted market comparable to a mid-size Morpho market.

### 2.3 Morpho Blue

Morpho Blue operates isolated lending markets. Each market is a fixed, independent pairing of a loan asset (USDC) and a collateral asset (e.g. WETH, cbBTC), with immutable parameters set at deployment: LLTV, oracle, and interest rate model cannot be changed by governance after creation. Isolation means that a problem in one market — an oracle failure, a collateral depeg — cannot contaminate any other market the vault is not in.

On Base, several USDC markets have reached $10M–$150M in supply, anchored by Coinbase-backed products (cbBTC lending is the backbone of Coinbase's on-chain lending product, which runs directly on Morpho). Only 5 markets currently clear the $10M minimum threshold.

**Protocol risk characterisation:** Varies by market. Isolation contains contagion risk but concentrates it within each market. Withdrawal liquidity is per-market and managed by the utilization gate rather than pool depth. Market parameters are permanent — the risk profile of a market the vault enters cannot worsen through governance.

---

## 3. Market Eligibility Gates

Before each rebalance cycle, the agent evaluates every candidate market against a set of risk gates. A market must pass all applicable gates to be eligible. Markets that fail any gate are excluded for that cycle regardless of their APY. Gates are re-evaluated every cycle — a market that fails today may pass next cycle if conditions improve, and vice versa.

### 3.1 TVL Floor

The TVL floor ensures the market has sufficient liquidity for the vault to enter and exit without materially impacting rates, and that adequate withdrawal liquidity exists under normal conditions.

The floor differs by protocol because market structure differs:

**Aave V3 — pooled market:** The floor is a market health sanity check. At $175M+ pool size on Base, even a large vault is a small fraction of total supply. The higher floor for Steady reflects the capital preservation mandate — the market should be comfortably large, not marginal.

**Compound V3 — pooled single-asset market:** The floor reflects both absolute pool size and the shared solvency risk. A smaller pool at the same utilization rate leaves less residual liquidity per dollar of exposure. The Steady floor of $25M effectively excludes the current Compound Base market (~$9M) until it grows. Balanced's $10M floor accepts it as a small-pool market, comparable in liquidity risk to a mid-size Morpho market.

**Morpho Blue — isolated market:** The floor is lower because isolation means withdrawal liquidity risk is market-specific and is primarily managed by the utilization gate (Section 3.3), not pool depth. A $10M Morpho market with 60% utilization has $4M immediately withdrawable — adequate for most vault sizes. The lower floor reflects that the utilization gate is the primary liquidity protection mechanism for Morpho.

| Protocol | Steady floor | Balanced floor |
|---|---|---|
| Aave V3 | $50M | $25M |
| Compound V3 | $25M | $10M |
| Morpho Blue | $25M | $10M |

### 3.2 APY Ceiling

The APY ceiling is a risk-exit gate, not a rebalance filter. If the current position's APY exceeds the ceiling, the agent exits the position regardless of whether a better alternative exists. An APY spike above the ceiling signals market stress — typically extreme borrowing demand causing utilization to overheat — and indicates the position is in a market with elevated withdrawal risk.

Normal USDC supply rates across all three protocols range from 3–8% under stable conditions. Rates above 8% consistently indicate elevated utilization pressure.

| Protocol | Steady ceiling | Balanced ceiling |
|---|---|---|
| Aave V3 | 800 bps (8%) | 1,500 bps (15%) |
| Compound V3 | 800 bps (8%) | 1,500 bps (15%) |
| Morpho Blue | 800 bps (8%) | 1,500 bps (15%) |

Steady exits at 8% — above the normal range, the agent treats it as overheating and protects capital. Balanced tolerates up to 15% before exiting, allowing it to capture genuine sustained demand spikes while still exiting before extreme stress conditions.

### 3.3 Morpho Utilization Gate

Morpho's Adaptive Curve Interest Rate Model targets 90% utilization — rates rise sharply above this point to attract new supply. Withdrawal liquidity is directly tied to utilization: available liquidity = (1 − utilization) × market supply. Above 90%, withdrawal liquidity is thin and the rate environment is volatile.

The utilization gate is the primary withdrawal liquidity protection mechanism for Morpho. It is more important than the TVL floor for Morpho markets because isolation means there is no cross-market liquidity backstop.

| Tier | Utilization gate |
|---|---|
| Steady | < 85% |
| Balanced | < 92% |

Steady requires a 15-percentage-point buffer before Morpho's IRM target, ensuring comfortable withdrawal headroom. Balanced operates closer to the IRM inflection point, relying on the market size floor to ensure adequate residual liquidity even at higher utilization.

### 3.4 Morpho LLTV Cap

LLTV (Liquidation Loan-to-Value) is the exact LTV at which a borrower's position becomes eligible for liquidation. On Morpho, liquidation triggers when a borrower's LTV reaches or exceeds the market's LLTV — not at 100% LTV. LLTV is set immutably at market creation from a governance-approved list: 77%, 86%, 91.5%, 94.5%, 96%, 98%.

A lower LLTV is safer for lenders. When LLTV is lower, liquidation is triggered earlier — when more collateral value remains relative to the debt — giving liquidators a larger margin to repay the debt and seize collateral cleanly. A higher LLTV means the borrower's position degrades further before liquidation can occur. In a fast-moving market, a high-LLTV market may breach its liquidation threshold before liquidator bots can act, leaving bad debt that is socialised across lenders.

The LLTV cap sets the maximum LLTV the agent will accept in a Morpho market. Markets with LLTV above the cap are excluded regardless of APY.

| Tier | LLTV cap | Markets captured |
|---|---|---|
| Steady | ≤ 86% | WETH/USDC (86%), wstETH/USDC (86%), cbETH/USDC (86%) |
| Balanced | ≤ 94.5% | Steady markets + cbBTC/USDC (91.5%) and any future ≤ 94.5% markets |

Steady restricts to ETH-correlated assets with 86% LLTV — markets with well-tested liquidation mechanisms and deep liquidator bot coverage. Balanced additionally accepts cbBTC at 91.5% LLTV, justified by cbBTC's deep Coinbase liquidity and reliable Chainlink oracle, which enable fast liquidation execution even at the tighter margin.

### 3.5 Morpho Collateral Allowlist

Morpho markets are defined by their collateral asset. The agent only supplies to markets whose collateral asset is on the allowlist. This ensures:

- The collateral has a reliable, live Chainlink price feed on Base (required for liquidations to function)
- The collateral is a recognised, liquid asset with an established liquidation track record

Excluded assets:
- **rsETH** — removed due to Kelp DAO exploit (April 2026) and oracle deprecation
- **rETH** — removed due to oracle deprecation on Base

| Tier | Allowed collateral assets |
|---|---|
| Steady | WETH, wstETH, cbETH, cbBTC, weETH, WBTC, USDC, USDT, USDS |
| Balanced | Steady list + ezETH |

ezETH is included in Balanced only — it has confirmed Chainlink support on Base but is a more complex LRT collateral type not appropriate for the Steady capital preservation mandate.

---

## 4. Rebalance Decision Logic

### 4.1 Cycle

The agent runs every 6 hours. Each cycle follows the same pipeline:

1. Read current vault state and active position
2. Fetch APY and market data for all whitelisted protocols
3. Apply eligibility gates — produce the set of eligible markets for this cycle
4. Apply trigger logic — decide whether to move, hold, top up, or exit
5. Execute if required; hold otherwise

### 4.2 Rebalance Trigger

The agent does not move for small rate differences. Frequent rebalancing for marginal gains is counterproductive — it introduces execution risk and disrupts the compounding of a stable position. The trigger requires a meaningful, sustained improvement before moving.

The minimum improvement required to trigger a rebalance is:

```
effectiveThreshold = max(10 bps, currentApy × relativeFactor)
```

Where `relativeFactor` is tier-specific:

| Tier | Relative factor | Example at 5% current APY |
|---|---|---|
| Steady | 15% | Threshold = 75 bps — must find a market paying at least 5.75% |
| Balanced | 10% | Threshold = 50 bps — must find a market paying at least 5.50% |

The absolute floor of 10 bps prevents movement on noise when rates are near zero.

The relative structure is intentional: at higher current APY, the required improvement scales proportionally. A 50 bps improvement matters more when current APY is 2% than when it is 8%.

### 4.3 Anomaly Filter

A single-cycle rate spike on an alternative market is not a reason to move. Short-term borrowing demand surges cause APY to spike on one protocol but typically revert within one or two cycles. Chasing a spike risks moving capital to a market that normalises before the next rebalance, locking in a worse long-term position.

The agent compares this cycle's alternative market APY against the previous cycle's APY for the same market. If the improvement is driven by a spike — the alternative's APY has risen more than the ceiling in a single cycle — the agent holds and flags the anomaly rather than executing.

| Tier | Anomaly ceiling |
|---|---|
| Steady | 200 bps spike vs prior cycle |
| Balanced | 300 bps spike vs prior cycle |

The ceiling is applied to the direction of the spike only. If the current position's APY has dropped — and the alternative's APY is stable or only moderately higher — the agent treats this as a genuine deterioration and rebalances without applying the ceiling. The ceiling exists to filter artificially elevated alternatives, not to trap the vault in a deteriorating position.

### 4.4 First Cycle and Idle Deployment

On the first cycle, or when the vault holds idle USDC with no active position, the agent deploys to the best eligible market immediately. The anomaly filter does not apply — there is no prior position to protect, and idle USDC earns nothing. Deployment to the best eligible market is always the right action when idle.

### 4.5 Top-Up

If the agent's decision is to hold the current position (HOLD), and the vault also holds idle USDC above the dust threshold, the agent top-ups the current position with the idle balance rather than leaving it undeployed. This applies to both tiers.

The dust threshold is `min($5, max($0.01, vault_tvl × 0.05%))` — a small buffer to absorb rounding and gas settlement differences without triggering unnecessary transactions.

### 4.6 No-Passing-Alternative Behaviour

When all markets fail the eligibility gates in a given cycle, the agent has no eligible destination. The tiers handle this differently:

| Tier | Behaviour | Rationale |
|---|---|---|
| Steady | Exit to idle immediately. Re-evaluate next cycle. | Capital preservation takes priority. If no market passes, the vault holds USDC at 0% rather than remaining in a market with no eligible peers as a benchmark. |
| Balanced | Hold current position for one cycle. Flag prominently. Re-evaluate next cycle. | Accepts short-term market disruptions without forcing an exit. A single cycle of no eligible alternatives is not sufficient reason to unwind — the market may normalise within 6 hours. If no alternative passes again next cycle, the agent re-evaluates whether to exit. |

---

## 5. Risk Exit

The risk exit is unconditional. It overrides all other decisions — it cannot be bypassed by a high current APY, the absence of an eligible alternative, or a HOLD decision. When a risk-exit condition is met, the agent withdraws the full position to idle USDC in the same cycle.

Risk-exit conditions:

1. **Market size drops below the TVL floor** — the market has shrunk to a size where withdrawal liquidity is no longer adequate under the strategy's standards
2. **APY exceeds the ceiling** — the current position is in an overheating market; continued supply carries elevated withdrawal risk
3. **Unexpected borrow position detected** — the vault should never hold a borrow; any borrow is an anomaly requiring immediate attention
4. **Morpho only — LLTV or utilization breach** — if the market's LLTV rises above the cap (not possible on Morpho due to immutability, but checked defensively) or utilization reaches or exceeds the utilization gate

After a risk exit, funds are held in idle USDC. The agent scans for eligible markets next cycle and redeploys when conditions permit.

---

## 6. Parameter Reference

### 6.1 Risk Gate Summary

| Parameter | Steady | Balanced |
|---|---|---|
| Aave V3 TVL floor | $50M | $25M |
| Compound V3 TVL floor | $25M | $10M |
| Morpho Blue TVL floor | $25M | $10M |
| APY ceiling — all protocols | 800 bps (8%) | 1,500 bps (15%) |
| Morpho LLTV cap | ≤ 86% | ≤ 94.5% |
| Morpho utilization gate | < 85% | < 92% |
| Morpho collateral allowlist | WETH, wstETH, cbETH, cbBTC, weETH, WBTC, USDC, USDT, USDS | + ezETH |

### 6.2 Rebalance Logic Summary

| Parameter | Steady | Balanced |
|---|---|---|
| Rebalance relative threshold | 15% of currentApy | 10% of currentApy |
| Rebalance absolute floor | 10 bps | 10 bps |
| Anomaly ceiling | 200 bps | 300 bps |
| No-passing-alternative | Exit to idle | Hold + flag one cycle |
| Cycle frequency | 6H | 6H |

---

## 7. What This Strategy Does Not Do

Regardless of tier, the lending strategy never:

- Borrows against supplied assets
- Uses leverage of any kind
- Swaps or trades assets
- Holds any token other than USDC or recognised lending receipt tokens (aUSDC, cUSDCv3, Morpho supply position tokens)
- Deploys to any protocol not in the whitelisted set
- Deploys to any Morpho market whose collateral is not on the allowlist
- Deploys to Morpho markets on Arbitrum (all Arbitrum Morpho USDC markets are below $2M and self-exclude by TVL gate, but are hard-excluded at the platform level)

---

## 8. Key Risks

This section describes the material risks a user should understand before depositing. It is not exhaustive.

**Smart contract risk.** The strategy interacts with Aave V3, Compound V3, and Morpho Blue smart contracts. All three protocols have undergone multiple independent security audits and have sustained significant TVL over multiple years, but no smart contract is risk-free. An exploit in any protocol could result in partial or total loss of supplied capital.

**Oracle risk.** Morpho markets rely on Chainlink price feeds to value collateral for liquidations. If an oracle fails or is manipulated, liquidations may not execute correctly, potentially allowing bad debt to accumulate. The collateral allowlist and LLTV cap are designed to limit exposure to markets where this risk is elevated, but they do not eliminate it.

**Liquidity risk.** In extreme market conditions, utilization across all markets may spike simultaneously. The risk-exit conditions and utilization gate provide protection, but if all markets reach the gate thresholds before the agent can withdraw, capital may be temporarily locked. This risk is mitigated by the 6-hour cycle frequency and the conservative utilization gate for Steady.

**Rate risk.** APY is variable and can change materially between cycles. The agent optimises within each cycle but cannot guarantee a specific return. The estimated APY ranges reflect typical market conditions, not guaranteed yields.

**Bad debt risk on Morpho.** If a Morpho market experiences a collateral price crash faster than liquidators can act, lenders may absorb bad debt proportional to their share of that market's supply. The LLTV cap and collateral allowlist reduce but do not eliminate this risk. Isolated market architecture ensures losses are contained to the affected market.

**Compound V3 shared solvency risk.** On Compound V3, all USDC suppliers share exposure to the solvency of the entire collateral pool. A collateral failure on any of the accepted collateral assets affects all suppliers. The TVL floor and APY ceiling provide indirect protection but not direct isolation.
