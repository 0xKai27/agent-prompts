{/* blocks/decisions/trading/swing/decide.md — v1.6.0 */}

# block: decisions/trading/swing/decide

**Responsibility:** Pure routing — read market signals from `{{stage.scan_trading_signals}}` and vault position state from `{{stage.scan_position_trading}}`, evaluate exit rules independently for every open lot and entry conditions in parallel, and write a structured decision signal to cycle state for downstream execute blocks. Read-only queries (`compute_token_amount`, `simulate_exit`, `simulate_enter`) permitted. No execution. No signing.

---

## Injected inputs — read from upstream stages

| Field | Source | Type | Description |
|---|---|---|---|
| `current.price` | `{{stage.scan_trading_signals}}` | number | Current token price |
| `current.rsi_1h` | `{{stage.scan_trading_signals}}` | number | RSI on 1H bar |
| `current.rsi_4h` | `{{stage.scan_trading_signals}}` | number | RSI on 4H bar |
| `current.regime` | `{{stage.scan_trading_signals}}` | string | Market regime |
| `current.vol_ratio` | `{{stage.scan_trading_signals}}` | number | Volume ratio vs rolling average |
| `previous_cycle.rsi_1h` | `{{stage.scan_trading_signals}}` | number \| null | Prior-cycle RSI 1H — null on first cycle |
| `usdc_bal` | `{{stage.scan_position_trading}}` | number | Idle USDC in vault |
| `credit_usd` | `{{stage.scan_position_trading}}` | number | USDC deposited in Aave |
| `positions` | `{{stage.scan_position_trading}}` | array | Held trading token balances (empty = fully in cash) |
| `open_positions` | `{{stage.scan_position_trading}}` | array | Open lots with per-lot exit plan metadata (see schema below) |
| `total_vault_usd` | `{{stage.scan_position_trading}}` | number | Total vault USD value |
| `buy_cooldown_active` | `{{stage.scan_position_trading}}` | boolean | True if within 120 min of last buy |

**`open_positions[]` lot schema** (written by execute block, surfaced by `scan_position_trading`):

```
{
  "lot_index":        <number>,      // stable identifier — used in exits[] output
  "cost_basis_usd":   <number>,      // copy literally into simulate_exit — do not recompute
  "scalp_target_usd": <number>,      // price threshold — compare directly against current.price
  "tp_target_usd":    <number>,      // price threshold — compare directly against current.price
  "stop_loss_usd":    <number>,      // price threshold — compare directly against current.price
  "scalp_net_pct":        <number>,  // NET% target — pass directly as simulate_exit targetPnlPct
  "tp_net_pct":           <number>,  // NET% target — pass directly as simulate_exit targetPnlPct
  "scalp_cost_basis_usd": <number>,  // pre-computed cost basis for scalp simulate_exit call — copy literally
  "trade_count":          <number>,
  "entry_regime":         "<bull|normal>"
}
```

If either upstream stage is absent or contains an `error` key → write the error fallback defined in Step 4 and stop.

---

## ⛔ Hard rules

1. **No `sign_and_send` and no write tools.** `compute_token_amount`, `simulate_exit`, and `simulate_enter` are read-only queries — permitted. All mutations are forbidden.

2. **Exit and entry evaluations run independently every cycle.** Open lots do not block entry evaluation. Exposure is enforced by `compute_token_amount` (via `exposureTokenAddress` + `maxExposurePct`), not by routing logic.

3. **`simulate_exit` mandatory before every SELL except Rule 1 stop loss.** Stop loss bypasses simulation — the SELL GUARD handles it at the swap layer. Gating on `simulate_exit` risks `BLOCKED_LOSS` trapping the position past the stop.

4. **`compute_token_amount(includeOpenPositions=true)` runs unconditionally in Step 1 every cycle.** `error` stops execution and writes the error fallback. `exposureCapExceeded` sets `entry_amount = null` and short-circuits entry to HOLD in Step 3. **`simulate_enter` mandatory when `entry_amount` is non-null and all four entry conditions pass.** `BLOCKED_*` or `WARN_HIGH_FEES` collapses entry to HOLD.

5. **Never override a `simulate_exit` or `simulate_enter` verdict.**

6. **`previous_cycle.rsi_1h = null` means first cycle.** Crossover cannot be confirmed — entry decision is HOLD.

7. **Default is HOLD. When in doubt, HOLD.**

---

## ⚠️ AMBITIOUS PROFILE THRESHOLDS — READ LITERALLY, DO NOT CHANGE

- **Entry regime**: `regime ∈ {bull, normal}`
- **Entry signal**: `previous_cycle.rsi_1h < 48` AND `current.rsi_1h ≥ 48`
- **Entry guards**: `current.rsi_4h > 45` AND `current.vol_ratio ≥ 0.8`
- **Exit — stop**: `current.price ≤ lot.stop_loss_usd`
- **Exit — regime fire**: `lot.entry_regime = bull` AND `current.regime ∈ {normal, caution, bear}` — OR — `lot.entry_regime = normal` AND `current.regime ∈ {caution, bear}`
- **Entry size**: 55% of `buying_power_usd` | `maxExposurePct: 85`
- **BUY cooldown**: 2 hours, enforced via `buy_cooldown_active`

---

## Step 1 — Compute buying power and entry amount

```
compute_token_amount(vault=<vault>,
  tokenAddress="0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
  percentage=55,
  includeOpenPositions=true,
  exposureTokenAddress=<tradingTokenAddress>,
  maxExposurePct=85)
```

- `error` → write the error fallback (Step 4) and stop.
- `exposureCapExceeded` → `buying_power_usd = 0`, `entry_amount = null`.
- Otherwise → `buying_power_usd = balance.immediatelyAvailable`, `entry_amount = computed.fromImmediatelyAvailable`.

`buying_power_usd` and `entry_amount` are referenced throughout Steps 3 and 4. Do not recompute them inline.

---

## Step 2 — EXIT evaluation (per lot)

Evaluate Rules 1–4 independently for each lot in `open_positions[]` using that lot's own fields. Stop at the first rule that fires per lot. Collect all results into `exits[]`.

If `open_positions[]` is empty → `exits = []`. Proceed to Step 3.

---

**Rule 1 — Stop loss**
`current.price ≤ lot.stop_loss_usd`?
→ YES: append `{ lot_index: <n>, decision: "SELL", sell_pct: 100, exit_reason: "stop_loss" }`. Next lot.
  Do NOT call `simulate_exit`.
→ NO: Rule 2.

---

**Rule 2 — Regime downgrade**
Look up `lot.entry_regime` in the table below and check if `current.regime` appears in the fire set:

| `lot.entry_regime` | Fire exit when `current.regime` is |
|---|---|
| `bull`   | `normal`, `caution`, or `bear` |
| `normal` | `caution` or `bear` |

If `lot.entry_regime` is null or absent → skip this rule, proceed to Rule 3.

→ FIRE:
```
simulate_exit(percentage:100, costBasisUsd:<lot.cost_basis_usd — copy literally, do not recompute>, targetPnlPct:0)
```
- `SELL_CONFIRMED`: append `{ lot_index: <n>, decision: "SELL", sell_pct: 100, exit_reason: "regime_drop" }`. Next lot.
- `BLOCKED_LOSS`: append `{ lot_index: <n>, decision: "HOLD", sell_pct: null, exit_reason: null }`. Next lot.
  Do not sell at a loss via this rule — wait for stop or TP.
→ NO MATCH: Rule 3.

---

**Rule 3 — Take profit**
`current.price ≥ lot.tp_target_usd`?
→ YES:
```
simulate_exit(percentage:100,
  costBasisUsd:<lot.cost_basis_usd — copy literally, do not recompute>,
  targetPnlPct:<lot.tp_net_pct — copy literally>)
```
- `SELL_CONFIRMED`: append `{ lot_index: <n>, decision: "SELL", sell_pct: 100, exit_reason: "take_profit" }`. Next lot.
- `BELOW_TARGET` or `BLOCKED_LOSS`: append `{ lot_index: <n>, decision: "HOLD", sell_pct: null, exit_reason: null }`. Next lot.
→ NO: Rule 4.

---

**Rule 4 — Scalp**
`current.price ≥ lot.scalp_target_usd`?
→ YES: determine `sell_pct` and `costBasisUsd` from this table — copy both literally, no arithmetic:

| `lot.trade_count` | `sell_pct` | `costBasisUsd` for `simulate_exit` |
|---|---|---|
| `1` | `50` | `lot.scalp_cost_basis_usd` |
| `2` or more | `100` | `lot.cost_basis_usd` |

```
simulate_exit(percentage:<sell_pct from table above>,
  costBasisUsd:<costBasisUsd from table above — copy literally>,
  targetPnlPct:<lot.scalp_net_pct — copy literally>)
```
- `SELL_CONFIRMED`: append `{ lot_index: <n>, decision: "SELL", sell_pct: <sell_pct>, exit_reason: "scalp" }`. Next lot.
- `BELOW_TARGET` or `BLOCKED_LOSS`: append `{ lot_index: <n>, decision: "HOLD", sell_pct: null, exit_reason: null }`. Next lot.
→ NO: append `{ lot_index: <n>, decision: "HOLD", sell_pct: null, exit_reason: null }`. Next lot.

---

## Step 3 — ENTRY evaluation

**Exposure cap** (from Step 1): `entry_amount = null` → `entry = { decision: "HOLD", needs_withdraw: false, buying_power_usd: 0, entry_regime: null }`. Proceed to Step 4.

**Cooldown**: `buy_cooldown_active = true` → `entry = { decision: "HOLD", needs_withdraw: false, buying_power_usd: <buying_power_usd>, entry_regime: null }`. Proceed to Step 4.

**First-cycle guard**: `previous_cycle.rsi_1h = null` → `entry = { decision: "HOLD", needs_withdraw: false, buying_power_usd: <buying_power_usd>, entry_regime: null }`. Proceed to Step 4.

**Entry conditions (ALL four must be true)**:
1. `current.regime ∈ {bull, normal}`
2. `previous_cycle.rsi_1h < 48` AND `current.rsi_1h ≥ 48`
3. `current.rsi_4h > 45`
4. `current.vol_ratio ≥ 0.8`

Any condition false → `entry = { decision: "HOLD", needs_withdraw: false, buying_power_usd: <buying_power_usd>, entry_regime: null }`. Proceed to Step 4.

All four true → gate with simulate_enter:
```
simulate_enter(vaultAddress=<vault>,
  tokenIn="0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
  tokenOut=<tradingTokenAddress>,
  amountFormatted=<entry_amount from Step 1>,
  scalpTargetPct=1.5, tpTargetPct=3.0)
```

- `PROCEED`:
  `entry = { decision: "BUY", needs_withdraw: (credit_usd > 1), buying_power_usd: <buying_power_usd>, entry_regime: <current.regime> }`.
- `WARN_HIGH_FEES` or any `BLOCKED_*`:
  `entry = { decision: "HOLD", needs_withdraw: false, buying_power_usd: <buying_power_usd>, entry_regime: null }`.

Proceed to Step 4.

---

## Step 4 — Write cycle state

```
write_cycle_state(vaultAddress=<vault>, state={
  "entry": {
    "decision":         "<BUY | HOLD>",
    "needs_withdraw":   <true | false>,
    "buying_power_usd": <number>,
    "entry_regime":     "<bull | normal | null>"
  },
  "exits": [
    {
      "lot_index":   <number>,
      "decision":    "<SELL | HOLD>",
      "sell_pct":    <50 | 100 | null>,
      "exit_reason": "<stop_loss | regime_drop | take_profit | scalp | null>"
    }
  ]
})
```

Error fallback (either upstream stage absent or contains `error`):
```
write_cycle_state(vaultAddress=<vault>, state={
  "entry": { "decision": "HOLD", "needs_withdraw": false, "buying_power_usd": 0, "entry_regime": null },
  "exits": []
})
```

---

## Decision summary (output after writing state)

```
DECIDE
  regime: <current.regime> | rsi_1h: <previous_cycle.rsi_1h>→<current.rsi_1h> | buying_power: $<buying_power_usd>

  ENTRY → <BUY | HOLD>
    [BUY]  entry_regime: <entry_regime> | needs_withdraw: <true/false>
    [HOLD] <one-sentence reason>

  EXITS (<n> lots evaluated)
    [lot <n>] <SELL exit_reason sell_pct% | HOLD reason>
    ...
```
