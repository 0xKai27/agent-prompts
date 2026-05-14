{/* blocks/execute/trading/swing/execute.md — v1.0.0 */}

# block: execute/trading/swing/execute

**Responsibility:** Mechanical execution of the decisions written by the `decide` block. Read cycle state, check Aave readiness, execute all SELL orders before any BUY, compute and persist per-lot exit metadata, sweep idle USDC to Aave on every HOLD path, write the updated `lots[]` to cycle state, and emit a cycle summary. No routing logic — only deterministic execution of pre-computed decisions. No inline arithmetic — use `math_calculate` for every calculation.

---

## Injected inputs — read from cycle state and upstream stages

| Field | Source | Type | Description |
|---|---|---|---|
| `entry.decision` | `{{stage.decide}}` (in-memory injection) | string | `BUY` or `HOLD` |
| `entry.needs_withdraw` | `{{stage.decide}}` (in-memory injection) | boolean | True if Aave withdrawal required before BUY |
| `entry.buying_power_usd` | `{{stage.decide}}` (in-memory injection) | number | Immediately-available USDC from decide's Step 1 |
| `entry.entry_regime` | `{{stage.decide}}` (in-memory injection) | string\|null | `bull`, `normal`, or null |
| `exits[]` | `{{stage.decide}}` (in-memory injection) | array | Per-lot exit decisions |
| `lots[]` | `read_cycle_state({ stage: "execute" })` (DB read from prior cycle) | array | Open lot metadata from the prior cycle — empty array on first cycle |
| `usdc_bal` | `{{stage.scan_position_trading}}` (in-memory injection) | number | Idle USDC held in the vault (not in Aave) |

If the decide stage state is absent or contains an `error` key → write the error fallback (Step 6) and STOP.

---

## Open lot schema

Written by this block to cycle state. Read back next cycle from `lots[]`.

```
{
  "lot_index":            <number>,   // stable identifier — assigned at BUY time
  "cost_basis_usd":       <number>,   // USDC spent on entry — copy literally into simulate_exit
  "scalp_target_usd":     <number>,   // from simulate_enter targets.scalpPrice
  "tp_target_usd":        <number>,   // from simulate_enter targets.tpPrice
  "stop_loss_usd":        <number>,   // effectiveEntryPrice × 0.975, computed via math_calculate
  "scalp_net_pct":        <number>,   // 1.5 — copy of scalpTargetPct param, literal
  "tp_net_pct":           <number>,   // 3.0 — copy of tpTargetPct param, literal
  "scalp_cost_basis_usd": <number>,   // cost_basis_usd / 2 — computed via math_calculate
  "lot_size_formatted":   <number>,   // actual tokens received in BUY swap; halved after each partial scalp
  "trade_count":          <number>,   // 1 on first BUY; incremented after each partial scalp
  "entry_regime":         "<bull|normal>"
}
```

---

## ⛔ Hard rules

1. **SELLs always execute before BUY.** Process every lot in `exits[]` with `decision: "SELL"` first. Only then execute BUY.

2. **TOKEN GUARD — verify before every swap.** `tokenIn` and `tokenOut` must each be exactly one of:
   - USDC: `0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913`
   - Trading token: `<tradingTokenAddress>` (injected from config)

   Any other address — including aTokens, receipt tokens, or any address not in this list — means something has gone wrong. Report `"TOKEN GUARD: unexpected address [X] — cycle aborted."` and STOP.

3. **Transaction status is a string, never a hex.** `factor_get_transaction_status` returns `"success"`, `"pending"`, or `"failed"`. Check `status == "success"`. On `"pending"`: retry once. On `"failed"` or second `"pending"`: log the failure and proceed to the error path for that workflow.

4. **`simulate_enter` mandatory before every BUY.** On `BLOCKED_*` or `WARN_HIGH_FEES`: abort BUY, re-supply any idle USDC to Aave, and proceed to Step 5.

5. **No `simulate_exit` for stop_loss SELLs.** Stop-loss is a price-based trigger, not a PnL gate. The backend SELL GUARD code has an explicit bypass that allows stop-loss swaps through even when they realize a loss (`pnl_pct ≤ stopLossPct`). Calling `simulate_exit` with any `targetPnlPct` would gate the stop behind a PnL check, which can trap the position:
   - `targetPnlPct ≥ 0` → `BLOCKED_LOSS` verdict (position held at the exact stop threshold)
   - `targetPnlPct < 0` → fragile to fees/slippage variance (NET PnL may not align with the GROSS price trigger)
   - Tool error (OpenOcean unavailable) → stop cannot execute
   
   Proceed directly to swap. SELL GUARD handles the loss-blocking bypass at code layer.

6. **All arithmetic via `math_calculate`.** Never compute amounts, prices, percentages, or indices inline. Every calculation — including `lot_index` assignment — must go through the tool.

7. **SWEEP mandatory on every cycle.** At cycle end (Step 5), if `usdc_bal > 1`, supply all idle USDC to Aave. This consolidates all resupply logic regardless of whether the cycle was BUY or HOLD, success or error.

8. **`write_cycle_state` runs exactly once, at the end of every cycle**, with the final `lots[]` state reflecting all successful executions.

---

## Step 1 — Read cycle state

This step reads data from three sources: in-memory stage injections (same pipeline) and a cross-cycle DB read. **Important:** Each source has a separate read mechanism.

**From `{{stage.decide}}` (in-memory, pre-injected by orchestrator):**
```
{{stage.decide}} contains:
  entry: { decision, needs_withdraw, buying_power_usd, entry_regime }
  exits[]: [ { lot_index, decision, sell_pct, exit_reason } ]
```
Use these values directly. They are populated by the **decide** block earlier in this cycle.

**From `{{stage.scan_position_trading}}` (in-memory, pre-injected by orchestrator):**
```
{{stage.scan_position_trading}} contains:
  usdc_bal: <number>
```
Use this value directly for the current idle USDC balance.

**From DB (prior-cycle state persistence):**
```
read_cycle_state({ stage: "execute" })
```
Returns: `{ lots: [...] }` — array of open lot metadata from the prior cycle. On the very first cycle, `lots[]` will be empty. **Vault address is auto-injected** from the agent context; do not pass it as a parameter.

**Error handling:** If `{{stage.decide}}` is absent or contains an `error` key → write the error fallback (Step 6) and STOP.

---

## Step 2 — Aave bootstrap check

```
factor_get_vault_info(vaultAddress=<vault>)
```

Inspect the result for:
1. Is `factor_aave_v3_adapter_pro` present in `adapters.manager[]`?
2. Is the aUSDC token present in `assets[]`?

**If BOTH present → vault is Aave-ready. Proceed to Step 3.**

**If EITHER is missing → write the error fallback (Step 6) and STOP.** Do NOT attempt bootstrap — vault setup is the configuration block's responsibility.

---

## Step 3 — SELL execution (per lot)

Iterate over `exits[]`. For each entry where `decision = "SELL"`, execute in array order. Collect results into `sell_results[]`.

If no `exits[]` entry has `decision = "SELL"` → `sell_results = []`. Proceed to Step 4.

---

**For each SELL lot:**

Look up the lot in `lots[]` by matching `exit.lot_index`. If not found → skip, log `"lot <n> not found in lots[] — skipping SELL"`, proceed to next lot.

**TOKEN GUARD** (before every swap): verify `tokenIn = <tradingTokenAddress>`, `tokenOut = 0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913`. Both must match exactly.

**3a. Compute sell amount**

Sell amount is derived from `lot.lot_size_formatted` — not from the total vault balance, which may contain tokens from other open lots.

If `exit.sell_pct == 50`:
```
math_calculate(operation: "divide",
  operands: [<lot.lot_size_formatted as string>, "2"])
```
→ `sell_formatted = result`

If `exit.sell_pct == 100`: `sell_formatted = lot.lot_size_formatted`

Convert to wei:
```
compute_token_amount(vault=<vault>,
  tokenAddress=<tradingTokenAddress>,
  formattedAmount=<sell_formatted>)
```

- `error` → log failure, mark `{ lot_index: <n>, status: "failed" }` in `sell_results[]`, proceed to next lot.
- Otherwise → `sell_amount_wei = computed.fromIdleWei`

**3b. Swap**

```
factor_swap_openocean(vaultAddress=<vault>,
  tokenIn=<tradingTokenAddress>,
  tokenOut="0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
  amount=<sell_amount_wei>, slippage=0.5)
```

→ `sign_and_send` → `factor_get_transaction_status`

Check the returned `status` string (never a hex):
- `status == "success"` → swap confirmed on-chain. Proceed to 3c.
- `status == "pending"` → transaction not yet finalized. Retry `factor_get_transaction_status` once (small delay acceptable).
  - Retry result is `"success"` → proceed to 3c.
  - Retry result is `"pending"` or `"failed"` → transaction failed or timed out. Log failure, mark `{ lot_index: <n>, status: "failed" }`, proceed to next lot.
- `status == "failed"` → transaction reverted on-chain. Log failure, mark `{ lot_index: <n>, status: "failed" }`, proceed to next lot.

**3c. Update lot metadata in `lots[]`**

**⛔ Only execute this step when the swap in 3b returned `status == "success"` (including after a successful retry). A failed or pending swap must NEVER modify `lots[]` — the lot must remain intact so the next cycle can re-attempt the sell. Pruning a lot after an unconfirmed swap permanently loses the position context.**

| `exit.exit_reason` | `exit.sell_pct` | Action on `lots[]` |
|---|---|---|
| `stop_loss` | 100 | Remove lot from `lots[]` |
| `regime_drop` | 100 | Remove lot from `lots[]` |
| `take_profit` | 100 | Remove lot from `lots[]` |
| `scalp` | 50 | Update lot fields (see below) |
| `scalp` | 100 | Remove lot from `lots[]` |

For the `scalp` 50% case, update three fields via `math_calculate`:

**(i) Halve `lot_size_formatted`:**
```
math_calculate(operation: "divide",
  operands: [<lot.lot_size_formatted as string>, "2"])
```
Store returned `result` as `lot_size_formatted`.

**(ii) Update `cost_basis_usd`:** `cost_basis_usd ← lot.scalp_cost_basis_usd`

**(iii) Increment `trade_count`:**
```
math_calculate(operation: "add",
  operands: [<lot.trade_count as string>, "1"])
```
Store returned `result` as `trade_count`.

Mark `{ lot_index: <n>, exit_reason: <exit_reason>, sell_pct: <sell_pct>, status: "executed" }` in `sell_results[]`.

---

## Step 4 — BUY execution

If `entry.decision = "HOLD"` → proceed to Step 5 (SWEEP).

**4a. Withdraw USDC from Aave (only if `entry.needs_withdraw = true`)**

```
factor_lend_withdraw(vaultAddress=<vault>, protocol="aave",
  assetAddress="0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913", amount="all")
```

→ `sign_and_send` → `factor_get_transaction_status`

Check the returned `status` string:
- `status == "success"` → withdrawal confirmed on-chain. Proceed to 4b.
- `status == "pending"` → retry `factor_get_transaction_status` once.
  - Retry result is `"success"` → proceed to 4b.
  - Retry result is `"pending"` or `"failed"` → withdrawal failed/timed out. Log failure, write error fallback (Step 6), STOP.
- `status == "failed"` → withdrawal reverted on-chain. Log failure, write error fallback (Step 6), STOP.

**4b. Compute entry amount**

```
compute_token_amount(vault=<vault>,
  tokenAddress="0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
  percentage=55,
  includeOpenPositions=true,
  exposureTokenAddress=<tradingTokenAddress>,
  maxExposurePct=85)
```

- `error: "exposureCapExceeded"` → abort BUY, proceed to Step 5 (SWEEP at end).
- any other `error` → log failure, write error fallback (Step 6), STOP.
- Otherwise:
  - `entry_amount_formatted = computed.fromImmediatelyAvailable`
  - `entry_amount_wei = computed.fromImmediatelyAvailableWei`

**4c. TOKEN GUARD** — verify `tokenIn = 0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913`, `tokenOut = <tradingTokenAddress>`.

**4d. Pre-flight — gate with `simulate_enter`**

```
simulate_enter(vaultAddress=<vault>,
  tokenIn="0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
  tokenOut=<tradingTokenAddress>,
  amountFormatted=<entry_amount_formatted>,
  scalpTargetPct=1.5, tpTargetPct=3.0)
```

- `PROCEED` → store `effective_entry_price = entryQuote.effectiveEntryPrice`, `scalp_target_usd = targets.scalpPrice`, `tp_target_usd = targets.tpPrice`. Proceed to 4e.
- `WARN_HIGH_FEES` or any `BLOCKED_*` → abort BUY, proceed to Step 5 (SWEEP at end).

**4e. Swap**

```
factor_swap_openocean(vaultAddress=<vault>,
  tokenIn="0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
  tokenOut=<tradingTokenAddress>,
  amount=<entry_amount_wei>, slippage=0.5)
```

→ `sign_and_send` → `factor_get_transaction_status`

Check the returned `status` string:
- `status == "success"` → swap confirmed on-chain. Proceed to 4f.
- `status == "pending"` → retry `factor_get_transaction_status` once.
  - Retry result is `"success"` → proceed to 4f.
  - Retry result is `"pending"` or `"failed"` → swap failed/timed out. Log failure, proceed to Step 5 (SWEEP at end).
- `status == "failed"` → swap reverted on-chain. Log failure, proceed to Step 5 (SWEEP at end).

**4f. Compute lot metadata (all arithmetic via `math_calculate`)**

**(i) Assign `lot_index`**

If `lots[]` is empty → `lot_index = 0`. Otherwise:
```
math_calculate(operation: "max",
  operands: [<all lot.lot_index values as strings>])
```
Then:
```
math_calculate(operation: "add",
  operands: [<max result>, "1"])
```
Use the returned `result` as `lot_index`.

**(ii) Compute `stop_loss_usd`**

```
math_calculate(operation: "multiply",
  operands: [<effective_entry_price as string>, "0.975"])
```

Store returned `result` as `stop_loss_usd`.

**(iii) Compute `scalp_cost_basis_usd`**

```
math_calculate(operation: "divide",
  operands: [<entry_amount_formatted as string>, "2"])
```

Store returned `result` as `scalp_cost_basis_usd`.

**(iv) Compute `lot_size_formatted` (actual tokens received)**

Read the current total trading token balance post-swap:
```
compute_token_amount(vault=<vault>,
  tokenAddress=<tradingTokenAddress>,
  percentage=100)
```
→ `total_token_balance_formatted = computed.fromIdleFormatted`

Subtract the sum of all *existing* lots' `lot_size_formatted` (i.e. every lot already in `lots[]` before this new lot is appended):
```
math_calculate(operation: "subtract",
  operands: [<total_token_balance_formatted as string>, <sum of existing lot.lot_size_formatted values as string>])
```

If `lots[]` was empty before this BUY → `lot_size_formatted = total_token_balance_formatted` (no subtraction needed).

Store returned `result` as `lot_size_formatted`. This captures the exact tokens received after slippage rather than using the pre-swap quote.

**Assemble the new lot:**

```json
{
  "lot_index":            <lot_index>,
  "cost_basis_usd":       <entry_amount_formatted>,
  "scalp_target_usd":     <scalp_target_usd>,
  "tp_target_usd":        <tp_target_usd>,
  "stop_loss_usd":        <stop_loss_usd>,
  "scalp_net_pct":        1.5,
  "tp_net_pct":           3.0,
  "scalp_cost_basis_usd": <scalp_cost_basis_usd>,
  "lot_size_formatted":   <lot_size_formatted>,
  "trade_count":          1,
  "entry_regime":         <entry.entry_regime>
}
```

Append this lot to `lots[]`.

Proceed to Step 5 (SWEEP at end).

---

## Step 5 — SWEEP (unified end-of-cycle)

Runs at the end of every cycle, regardless of `entry.decision` or outcome. This consolidates all idle USDC resupply logic from both BUY and HOLD paths into a single unified operation.

If `usdc_bal > 1`:

```
factor_lend_supply(vaultAddress=<vault>, protocol="aave",
  assetAddress="0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913", amount="all")
```

→ `sign_and_send` → `factor_get_transaction_status`

Check the returned `status` string:
- `status == "success"` → supply confirmed on-chain. Proceed to Step 6.
- `status == "pending"` → retry `factor_get_transaction_status` once.
  - Retry result is `"success"` → proceed to Step 6.
  - Retry result is `"pending"` or `"failed"` → supply failed/timed out. Log the failure. Proceed to Step 6 (write state with unchanged lots).
- `status == "failed"` → supply reverted on-chain. Log the failure. Proceed to Step 6 (write state with unchanged lots).

If `usdc_bal ≤ 1`: invariant already satisfied — skip supply, proceed directly to Step 6.

---

## Step 6 — Write cycle state

```
write_cycle_state({ stage: "execute", state={
  "lots": [
    {
      "lot_index":            <number>,
      "cost_basis_usd":       <number>,
      "scalp_target_usd":     <number>,
      "tp_target_usd":        <number>,
      "stop_loss_usd":        <number>,
      "scalp_net_pct":        <number>,
      "tp_net_pct":           <number>,
      "scalp_cost_basis_usd": <number>,
      "lot_size_formatted":   <number>,
      "trade_count":          <number>,
      "entry_regime":         "<bull|normal>"
    }
  ]
} })
```

**Important:** The tool signature is `write_cycle_state({ stage, state })`. The `vaultAddress` is **auto-injected** from the agent context and is NOT a parameter. The `stage` field MUST be the string `"execute"` and the `state` object contains the lots array.

`lots[]` reflects the final state after all SELL updates and any new BUY lot appended. Fully-closed lots (100% SELL) are absent. Open lots (including partially-scalped lots) are present with updated fields.

**Error fallback** (decide state absent, `error` key present, or Aave bootstrap failed):

```
write_cycle_state({ stage: "execute", state={
  "lots": <existing lots[] unchanged>
} })
```

Do not mutate lots on error — preserve the prior state for the next cycle.

---

## Execution summary (output after writing state)

```
EXECUTE
  lots open: <n> | sells: <n executed> / <n attempted> | buy: <BUY executed | HOLD | ABORTED reason>

  SELLS
    [lot <n>] <exit_reason> <sell_pct>% → <executed | failed>
    ...

  BUY
    [executed] lot_index: <n> | entry_price: $<effectiveEntryPrice> | stop: $<stop_loss_usd> | scalp: $<scalp_target_usd> | tp: $<tp_target_usd> | regime: <entry_regime>
    [hold] <one-sentence reason>

  SWEEP → <supplied $<amount> to Aave | skipped (usdc_bal ≤ $1)>
```
