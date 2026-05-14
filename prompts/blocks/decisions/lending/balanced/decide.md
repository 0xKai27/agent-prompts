# block: decisions/lending/balanced/decide

**Responsibility:** Pure routing — read eligible market data from `{{stage.scan_lending_markets}}` and vault position state from `{{stage.scan_position_lending}}`, apply Lending × Balanced decision logic, and emit structured routing flags for downstream execute blocks. No tool calls. No execution. No signing.

---

## Injected inputs — read from upstream stages

| Field | Source | Type |
|---|---|---|
| `best_eligible` | `{{stage.scan_lending_markets}}` | `{ protocol, market, receiptToken, collateralToken, collateralSymbol, apyBase }` |
| `current_protocol` | `{{stage.scan_position_lending}}` | `"aave" \| "compoundV3" \| "morpho" \| null` |
| `current_market` | `{{stage.scan_position_lending}}` | `string \| null` |
| `current_apy_bps` | `{{stage.scan_position_lending}}` | `bps \| 0 \| null` |
| `idle_usdc` | `{{stage.scan_position_lending}}` | `number` |
| `total_vault_usd` | `{{stage.scan_position_lending}}` | `number` |
| `unexpected_borrow` | `{{stage.scan_position_lending}}` | `boolean` |
| `current_market_eligible` | `{{stage.scan_position_lending}}` | `boolean \| null` |
| `current_market_anomalous` | `{{stage.scan_position_lending}}` | `boolean` |
| `anomaly_direction` | `{{stage.scan_position_lending}}` | `"spike" \| "drop" \| null` |

**No `read_cycle_state` call.** This block has no inter-cycle state. All historical position tracking is owned by `scan_position_lending`.

If either upstream stage output is missing or malformed, emit the error JSON defined in Step 4 and stop.

---

## ⛔ Hard rules

1. **`best_eligible` is the single clean-tagged market surfaced by `{{stage.scan_lending_markets}}`. Decide does not re-derive or re-filter market eligibility — that is owned entirely by `scan_lending_markets`.**

2. **Step 3 (risk exit) runs unconditionally after every Step 2 decision, without exception.** A matching risk exit condition overrides any Step 2 outcome — including HOLD, HOLD-TOPUP, and REBALANCE-YIELD. It cannot be bypassed by any other reasoning.

3. **No tool calls, no signing, no execution.** This block emits routing flags only. `sign_and_send` and all write tools are forbidden.

4. **No arithmetic.** All comparisons are ordinal. `minDeploymentThreshold` and `effectiveThreshold` are derived from the hardcoded values below using the specified formulas — do not substitute, re-derive, or approximate them.

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
effectiveThreshold     = max(10, current_apy_bps × 0.10)     [bps]
minDeploymentThreshold = min(5, max(0.01, total_vault_usd × 0.0005))   [USDC]
```

---

## Step 2 — Decision routing

Work through each path in order. Stop at the first matching path and record the decision. Do not evaluate subsequent paths.

---

### Path A — Idle vault (`current_apy_bps == 0 or null`)

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
spread = best_eligible.apyBase - current_apy_bps

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
    display_spread = best_eligible.apyBase - current_apy_bps
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
display_spread = best_eligible.apyBase - current_apy_bps
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

Evaluate both conditions in order against data already available from Steps 1 and 2. First match overrides the Step 2 decision. Market-level risk gates (size, APY cap, LLTV, utilization) are the responsibility of `scan_lending_markets` — decide does not re-check them. Their outcome is already encoded in `current_market_eligible`.

| # | Condition | How to evaluate |
|---|---|---|
| 1 | `current_market_eligible == false` | Current market has left the eligible set this cycle (failed a size, APY, LLTV, or utilization gate in `scan_lending_markets`). Read `current_market_eligible` from `{{stage.scan_position_lending}}`. |
| 2 | `unexpected_borrow == true` | Vault holds debt it should not. Read `unexpected_borrow` from `{{stage.scan_position_lending}}`. |

**No active position** (`current_protocol == null`): `current_market_eligible` is `null` — condition 1 does not fire. `unexpected_borrow` cannot be true on an idle vault. Skip Step 3 entirely.

```
IF any condition 1–2 matches:
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
  "risk_exit_condition": <1|2|null>
}
```

`target_*` fields are populated from the selected `target` market's fields in `best_eligible`. Null when decision is HOLD, HOLD-IDLE, HOLD-IDLE-FLAG, or RISK-EXIT.

`target_collateral_symbol` is Morpho-only (`collateralSymbol` from `best_eligible`). Null for Aave and Compound V3. Used by downstream configuration blocks for logging — does not affect routing.

On any error or missing required upstream field, emit:

```json
{ "error": "<description>", "decision": "HOLD", "do_supply": false, "do_rebalance": false, "do_withdraw": false, "target_protocol": null, "target_market": null, "target_receipt_token": null, "target_collateral_token": null, "target_collateral_symbol": null, "display_spread": null, "risk_exit_condition": null }
```
