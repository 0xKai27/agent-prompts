{/* blocks/decisions/lending/decide.md — v1.0.0 */}

# block: decisions/lending/balanced/decide

**Responsibility:** Pure routing — read eligible market data from `{{stage.scan_lending_markets}}` and vault position state from `{{stage.scan_position_lending}}`, apply Lending × Balanced decision logic, and emit structured routing flags for downstream execute blocks. No tool calls. No execution. No signing.

---

## Injected inputs — read from upstream stages

| Field | Source | Type |
|---|---|---|
| `best_eligible` | `{{stage.scan_lending_markets}}` | `{ protocol, market, receipt_token, collateral_token, collateral_symbol, apy_base_bps }` |
| `current_protocol` | `{{stage.scan_position_lending}}` | `"aave" \| "compoundV3" \| "morpho" \| null` |
| `current_market` | `{{stage.scan_position_lending}}` | `string \| null` |
| `current_apy_bps` | `{{stage.scan_position_lending}}` | `bps \| 0 \| null` |
| `idle_usdc` | `{{stage.scan_position_lending}}` | `number` |
| `total_vault_usd` | `{{stage.scan_position_lending}}` | `number` |
| `unexpected_borrow` | `{{stage.scan_position_lending}}` | `boolean` |
| `current_market_eligible` | `{{stage.scan_position_lending}}` | `boolean \| null` |
| `current_market_anomalous` | `{{stage.scan_position_lending}}` | `boolean` |
| `anomaly_direction` | `{{stage.scan_position_lending}}` | `"spike" \| "drop" \| null` |
| `current_market_exit_reason` | `{{stage.scan_position_lending}}` | `"tvl_floor" \| "apy_ceiling" \| "lltv" \| "utilization" \| null` |

**No `read_cycle_state` call.** This block has no inter-cycle state. All historical position tracking is owned by `scan_position_lending`.

If either upstream stage output is missing or malformed, emit the error JSON defined in Step 4 and stop.

---

## ⛔ Hard rules

1. **`best_eligible` is the single clean-tagged market surfaced by `{{stage.scan_lending_markets}}`. Decide does not re-derive or re-filter market eligibility — that is owned entirely by `scan_lending_markets`.**

2. **Step 3 (risk exit) runs unconditionally after every Step 2 decision, without exception.** A matching risk exit condition overrides any Step 2 outcome — including HOLD, HOLD-TOPUP, and REBALANCE-YIELD. It cannot be bypassed by any other reasoning.

3. **`math_calculate` is the only permitted tool call.** No signing, no execution, no write tools, no state reads. This block emits routing flags only. `sign_and_send` and all other tools are forbidden.

4. **No inline arithmetic.** All numeric operations call `math_calculate`. `minDeploymentThreshold` and `effectiveThreshold` are computed in Step 1 via `math_calculate` — do not substitute, re-derive, or approximate them.

---

## Thresholds (hardcoded to Balanced v17 — future injectable)

The following values are sourced directly from `lending_balanced_rebalance_17.md`. They are intentionally hardcoded for this variant. Future parameterisation will move them into `strategyConfig`.

| Parameter | Hardcoded value | Future `strategyConfig` key |
|---|---|---|
| Effective threshold floor | **10 bps** | `decisions.effectiveThresholdFloorBps` |
| Effective threshold multiplier | **10% of `current_apy_bps`** | `decisions.effectiveThresholdPct` |
| `minDeploymentThreshold` TVL coefficient | **0.0005** | `decisions.deploymentThresholdCoeff` |
| `minDeploymentThreshold` minimum | **0.01 USDC** | `decisions.minDeploymentThresholdUsdc` |
| `minDeploymentThreshold` maximum cap | **5 USDC** | `decisions.maxDeploymentThresholdUsdc` |

---

## Step 1 — Compute derived values

Read these two values once. Use them throughout Steps 2 and 3.

```
Compute effectiveThreshold [bps]:
math_calculate(operation: "multiply", operands: [<current_apy_bps>, 0.10], decimalPlaces: 0)  → result_1
math_calculate(operation: "max", operands: [10, <result_1>])  → effectiveThreshold

Compute minDeploymentThreshold [USDC]:
math_calculate(operation: "multiply", operands: [<total_vault_usd>, 0.0005])  → result_2
math_calculate(operation: "max", operands: [0.01, <result_2>])  → result_3
math_calculate(operation: "min", operands: [5, <result_3>])  → minDeploymentThreshold
```

---

## Step 2 — Decision routing

Work through each path in order. Stop at the first matching path and record the decision. Do not evaluate subsequent paths.

**Pre-check — ineligible current market:** If `current_protocol != null` AND `current_market_eligible == false`, skip Paths A–G entirely and proceed directly to Step 3. The current market has left the eligible set this cycle — `current_apy_bps` is null and no spread calculation is meaningful. Step 3 will determine the appropriate risk exit condition.

---

### Path A — Idle vault (`current_protocol == null`)

```
target = best_eligible   // highest clean market from scan_lending_markets

IF target.protocol == "NONE"
  decision = HOLD-IDLE
  do_supply = do_rebalance = do_withdraw = false
  → proceed to Step 3

decision    = REBALANCE-YIELD
do_supply   = true    // no prior position to exit
do_rebalance = false
do_withdraw  = false
```

→ proceed to Step 3.

---

### Path B — Active position, no eligible alternative

```
IF best_eligible.protocol == "NONE"
  decision = HOLD-IDLE-FLAG
  do_supply = do_rebalance = do_withdraw = false
  → proceed to Step 3
```

---

### Path C — Active position, spread below threshold

```
math_calculate(operation: "subtract", operands: [<best_eligible.apy_base_bps>, <current_apy_bps>])  → spread

IF spread < effectiveThreshold
  decision = HOLD
  do_supply = do_rebalance = do_withdraw = false
  → proceed to Step 3
```

---

### Path D — Active position, current market APY spike

```
IF current_market_anomalous == true AND anomaly_direction == "spike"
  decision = HOLD
  do_supply = do_rebalance = do_withdraw = false
  → proceed to Step 3
```

Rationale: a current-market APY spike is a data anomaly signal. Hold rather than act on potentially noisy data, even when the spread would otherwise justify a move.

---

### Path E — Active position, current market APY drop

```
IF current_market_anomalous == true AND anomaly_direction == "drop"
  IF best_eligible.protocol != "NONE"
    decision     = REBALANCE-YIELD
    do_supply    = false
    do_rebalance = true    // withdraw from current, supply to target
    do_withdraw  = false
    target       = best_eligible
    math_calculate(operation: "subtract", operands: [<best_eligible.apy_base_bps>, <current_apy_bps>])  → display_spread
  ELSE
    decision = HOLD-IDLE-FLAG
    do_supply = do_rebalance = do_withdraw = false
  → proceed to Step 3
```

---

### Path F — Spread met, rebalance to best eligible

```
// spread >= effectiveThreshold AND current_market_anomalous == false
// best_eligible is the single clean market from scan_lending_markets

decision       = REBALANCE-YIELD
do_supply      = false
do_rebalance   = true
do_withdraw    = false
target         = best_eligible
math_calculate(operation: "subtract", operands: [<best_eligible.apy_base_bps>, <current_apy_bps>])  → display_spread
```

→ proceed to Step 3.

---

### Path G — HOLD-TOPUP sweep

Evaluated after any HOLD or HOLD-IDLE-FLAG outcome from Paths B–F. Not evaluated after RISK-EXIT, REBALANCE-YIELD, or HOLD-IDLE.

```
IF decision ∈ {HOLD, HOLD-IDLE-FLAG}
  AND current_protocol != null
  AND idle_usdc > minDeploymentThreshold

  decision         = HOLD-TOPUP
  do_supply        = true
  do_rebalance     = false
  do_withdraw      = false
  target_protocol  = current_protocol
  target_market    = current_market
```

---

## Step 3 — Risk exit (unconditional)

Evaluate conditions in order against data from upstream stages. First match overrides the Step 2 decision. 

**No active position** (`current_protocol == null`): skip Step 3 entirely.

| # | Condition | Source |
|---|---|---|
| 1 | Market size dropped below TVL floor | `current_market_eligible == false AND current_market_exit_reason == "tvl_floor"` |
| 2 | APY exceeded ceiling | `current_market_eligible == false AND current_market_exit_reason == "apy_ceiling"` |
| 3 | Unexpected borrow detected | `unexpected_borrow == true` |
| 4 | Morpho LLTV or utilization breach | `current_market_eligible == false AND current_market_exit_reason ∈ {"lltv", "utilization"}` |

If `current_market_eligible == false` but `current_market_exit_reason` is `null`, treat as condition 1 (conservative default).

```
IF any condition 1–4 matches:
  decision            = RISK-EXIT
  do_supply           = false
  do_rebalance        = false
  do_withdraw         = true
  risk_exit_condition = <lowest-numbered matching condition>
  target_*            = null
```

---

## Step 4 — Emit output

Emit the following JSON object as the **absolute last line** of your output. No prose after it.

```json
{
  "decision": "<HOLD|HOLD-IDLE|HOLD-IDLE-FLAG|HOLD-TOPUP|REBALANCE-YIELD|RISK-EXIT>",
  "do_supply": <boolean>,
  "do_rebalance": <boolean>,
  "do_withdraw": <boolean>,
  "target_protocol": "<aave|compoundV3|morpho|null>",
  "target_market": "<bytes32|address|null>",
  "target_receipt_token": "<address|null>",
  "target_collateral_token": "<address|null>",
  "target_collateral_symbol": "<string|null>",
  "display_spread": <bps|null>,
  "risk_exit_condition": <1|2|3|4|null>
}
```

`target_*` fields are populated from the selected `target` market's fields in `best_eligible`. Null when decision is HOLD, HOLD-IDLE, HOLD-IDLE-FLAG, or RISK-EXIT.

`target_collateral_symbol` is Morpho-only (`collateral_symbol` from `best_eligible`). Null for Aave and Compound V3. Used by downstream configuration blocks for logging — does not affect routing.

On any error or missing required upstream field, emit:

```json
{ "error": "<description>", "decision": "HOLD", "do_supply": false, "do_rebalance": false, "do_withdraw": false, "target_protocol": null, "target_market": null, "target_receipt_token": null, "target_collateral_token": null, "target_collateral_symbol": null, "display_spread": null, "risk_exit_condition": null }
```
