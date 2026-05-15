{/* blocks/execute/trading/swing/execute.md — v1.0.0 */}

# block: execute/trading/swing/execute

**Responsibility:** Mechanical execution of the decisions written by the `decide` block. Read cycle state, check Aave readiness, execute all SELL/BUY orders via OpenOcean, compute and persist per-position exit metadata, sweep idle USDC to Aave on every cycle, write the updated `positions[]` to cycle state with buy_timestamp on BUY cycles, and emit a cycle summary. No routing logic — only deterministic execution of pre-computed decisions. No inline arithmetic — use `math_calculate` for every calculation.

---

## Injected inputs — read from cycle state and upstream stages

| Field | Source | Type | Description |
|---|---|---|---|
| `entry.decision` | `{{stage.decide}}` (in-memory injection) | string | `BUY` or `HOLD` |
| `entry.needs_withdraw` | `{{stage.decide}}` (in-memory injection) | boolean | True if Aave withdrawal required before BUY |
| `entry.entry_regime` | `{{stage.decide}}` (in-memory injection) | string\|null | `bull`, `normal`, or null |
| `entry.atr_4h_pct` | `{{stage.decide}}` (in-memory injection) | number \| null | 4H ATR as % of price — used to compute ATR-based stop distance. Null on Binance fallback. |
| `exits[]` | `{{stage.decide}}` (in-memory injection) | array | Per-position exit decisions |
| `positions[]` | `read_cycle_state({ stage: "execute" })` (DB read from prior cycle) | array | Open position metadata from the prior cycle — empty array on first cycle |
| `idle_usdc` | `{{stage.scan_position_trading}}` (in-memory injection) | number | Idle USDC held in the vault (not in Aave) |
| `denominatorTokenAddress` | system prompt | `address` | Address of the denominator token (e.g. USDC) |

If the decide stage state is absent or contains an `error` key → write the error fallback (Step 6) and STOP.

---

## Open position schema

Written by this block to cycle state. Read back next cycle from `positions[]`.

```
{
  "position_index":            <number>,   // stable identifier — assigned at BUY time
  "cost_basis_usd":       <number>,   // USDC spent on entry — copy literally into simulate_exit
  "scalp_target_usd":     <number>,   // from simulate_enter targets.scalpPrice
  "tp_target_usd":        <number>,   // from simulate_enter targets.tpPrice
  "stop_loss_usd":        <number | null>,   // ATR-based stop: entryPrice - (entryPrice × atr_4h_pct × 0.025); null if atr_4h_pct unavailable
  "scalp_net_pct":        <number>,   // 1.5 — copy of scalpTargetPct param, literal
  "tp_net_pct":           <number>,   // 3.0 — copy of tpTargetPct param, literal
  "scalp_cost_basis_usd": <number>,   // cost_basis_usd / 2 — computed via math_calculate
  "position_token_qty":        <number>,   // token quantity in human-readable units (e.g. 0.05 for WETH); actual tokens received in BUY swap; halved after each partial scalp
  "trade_count":          <number>,   // 1 on first BUY; incremented after each partial scalp
  "entry_regime":         "<bull|normal>",
  "buy_timestamp":        <number>    // server epoch seconds from write_cycle_state written_at — set at BUY time, preserved on scalp updates, never changed on HOLD cycles
}
```

---

## ⛔ Hard rules

1. **SELLs always execute before BUY.** Process every position in `exits[]` with `decision: "SELL"` first. Only then execute BUY.

2. **TOKEN GUARD — verify before every swap.** `tokenIn` and `tokenOut` must each be exactly one of:
   - USDC: `<denominatorTokenAddress>`
   - Trading token: `<tradingTokenAddress>` (injected from config)

   Any other address — including aTokens, receipt tokens, or any address not in this list — means something has gone wrong. Report `"TOKEN GUARD: unexpected address [X] — cycle aborted."` and STOP.

3. **Transaction status is a string, never a hex.** `factor_get_transaction_status` returns `"success"`, `"pending"`, or `"failed"`. Check `status == "success"`. On `"pending"`: retry once. On `"failed"` or second `"pending"`: log the failure and proceed to the error path for that workflow. On `"failed"`: call `factor_decode_error` with the transaction hash before logging — always surface the revert reason.

4. **`simulate_enter` mandatory before every BUY.** On `BLOCKED_*` or `WARN_HIGH_FEES`: abort BUY and proceed to Step 5 (SWEEP handles resupply).

5. **No `simulate_exit` for stop_loss SELLs.** Stop-loss is a price-based trigger, not a PnL gate. The backend SELL GUARD code has an explicit bypass that allows stop-loss swaps through even when they realize a loss (`pnl_pct ≤ stopLossPct`). Calling `simulate_exit` with any `targetPnlPct` would gate the stop behind a PnL check, which can trap the position:
   - `targetPnlPct ≥ 0` → `BLOCKED_LOSS` verdict (position held at the exact stop threshold)
   - `targetPnlPct < 0` → fragile to fees/slippage variance (NET PnL may not align with the GROSS price trigger)
   - Tool error (OpenOcean unavailable) → stop cannot execute
   
   Proceed directly to swap. SELL GUARD handles the loss-blocking bypass at code layer.

6. **All arithmetic via `math_calculate`.** Never compute amounts, prices, percentages, or indices inline. Every calculation — including `position_index` assignment — must go through the tool.

7. **SWEEP mandatory on every cycle.** At cycle end (Step 5), supply all idle USDC to Aave if `idle_usdc > 1` OR a SELL executed successfully this cycle. This ensures post-SELL USDC is swept even when the pre-cycle idle balance was zero.

8. **`write_cycle_state` for BUY cycles uses a two-write pattern** (capture server epoch in Step 4f, then flush final state in Step 6 for HOLD/SELL-only). **For HOLD and SELL-only cycles, `write_cycle_state` runs once at the end (Step 6)**, with the final `positions[]` state reflecting all successful executions.

---

## Step 1 — Read cycle state

This step reads data from three sources: in-memory stage injections (same pipeline) and a cross-cycle DB read. **Important:** Each source has a separate read mechanism.

**From `{{stage.decide}}` (in-memory, pre-injected by orchestrator):**
```
{{stage.decide}} contains:
  entry: { decision, needs_withdraw, entry_regime }
  exits[]: [ { position_index, decision, sell_pct, exit_reason } ]
```
Use these values directly. They are populated by the **decide** block earlier in this cycle.

**From `{{stage.scan_position_trading}}` (in-memory, pre-injected by orchestrator):**
```
{{stage.scan_position_trading}} contains:
  idle_usdc: <number>
```
Use this value directly for the current idle USDC balance.

**From DB (prior-cycle state persistence):**
```
read_cycle_state({ stage: "execute" })
```
Returns: `{ positions: [...] }` — array of open position metadata from the prior cycle. On the very first cycle, `positions[]` will be empty. **Vault address is auto-injected** from the agent context; do not pass it as a parameter.

**Error handling:** If `{{stage.decide}}` is absent or contains an `error` key → write the error fallback (Step 6) and STOP.

---

## Step 2 — Aave bootstrap check

```
factor_get_vault_info(vaultAddress=<vault>)
```

Inspect the result for:
1. Is `factor_aave_adapter_pro` present in `adapters.manager[]`?
2. Is the aUSDC token present in `assets.supported[]`?

**If BOTH present → vault is Aave-ready. Proceed to Step 3.**

**If EITHER is missing → write the error fallback (Step 6) and STOP.** Do NOT attempt bootstrap — vault setup is the configuration block's responsibility.

---

## Step 3 — SELL execution (per position)

**SKIP: If no `exits[]` entry has `decision = "SELL"` → set `sell_results = []` and proceed to Step 4. The SELL loop below only executes when at least one SELL decision exists.**

Iterate over `exits[]`. For each entry where `decision = "SELL"`, execute in array order. Collect results into `sell_results[]`.

---

**For each SELL position:**

**SKIP: Look up the position in `positions[]` by matching `exit.position_index`. If not found → log `"position <n> not found in positions[] — skipping SELL"` and proceed to next position. Skip all steps 3a–3c below for this position.**

**TOKEN GUARD** (before every swap): verify `tokenIn = <tradingTokenAddress>`, `tokenOut = <denominatorTokenAddress>`. Both must match exactly.

**3a. Compute sell amount**

Sell amount is derived from `position.position_token_qty` — not from the total vault balance, which may contain tokens from other open positions.

If `exit.sell_pct == 50`:
```
math_calculate(operation: "divide",
  operands: [<position.position_token_qty as string>, "2"])
```
→ `sell_formatted = result`

If `exit.sell_pct == 100`: `sell_formatted = position.position_token_qty`

Convert to wei:
```
compute_token_amount(vault=<vault>,
  tokenAddress=<tradingTokenAddress>,
  formattedAmount=<sell_formatted>)
```

- `error` → log failure, mark `{ position_index: <n>, status: "failed" }` in `sell_results[]`, proceed to next position.
- Otherwise → `sell_amount_wei = computed.fromIdleWei`

**3b. Swap**

```
factor_swap_openocean(vaultAddress=<vault>,
  tokenIn=<tradingTokenAddress>,
  tokenOut=<denominatorTokenAddress>,
  amount=<sell_amount_wei>, slippage=0.5)
```

→ `sign_and_send` → `factor_get_transaction_status`

Check the returned `status` string (never a hex):
- `status == "success"` → swap confirmed on-chain. Proceed to 3c.
- `status == "pending"` → transaction not yet finalized. Retry `factor_get_transaction_status` once (small delay acceptable).
  - Retry result is `"success"` → proceed to 3c.
  - Retry result is `"pending"` or `"failed"` → transaction failed or timed out. Call `factor_decode_error` with the transaction hash to get the revert reason, log the failure including the decoded error string, mark `{ position_index: <n>, status: "failed" }`, proceed to next position.
- `status == "failed"` → transaction reverted on-chain. Call `factor_decode_error` with the transaction hash to get the revert reason, log the failure including the decoded error string, mark `{ position_index: <n>, status: "failed" }`, proceed to next position.

**3c. Update position metadata in `positions[]`**

**⛔ Only execute this step when the swap in 3b returned `status == "success"` (including after a successful retry). A failed or pending swap must NEVER modify `positions[]` — the position must remain intact so the next cycle can re-attempt the sell. Pruning a position after an unconfirmed swap permanently loses the position context.**

| `exit.exit_reason` | `exit.sell_pct` | Action on `positions[]` |
|---|---|---|
| `stop_loss` | 100 | Remove position from `positions[]` |
| `regime_drop` | 100 | Remove position from `positions[]` |
| `take_profit` | 100 | Remove position from `positions[]` |
| `scalp` | 50 | Update position fields (see below) |
| `scalp` | 100 | Remove position from `positions[]` |

For the `scalp` 50% case, update three fields via `math_calculate`:

**(i) Halve `position_token_qty`:**
```
math_calculate(operation: "divide",
  operands: [<position.position_token_qty as string>, "2"])
```
Store returned `result` as `position_token_qty`.

**(ii) Update `cost_basis_usd`:** `cost_basis_usd ← position.scalp_cost_basis_usd`

**(iii) Increment `trade_count`:**
```
math_calculate(operation: "add",
  operands: [<position.trade_count as string>, "1"])
```
Store returned `result` as `trade_count`.

Mark `{ position_index: <n>, exit_reason: <exit_reason>, sell_pct: <sell_pct>, status: "executed" }` in `sell_results[]`.

---

## Step 4 — BUY execution

If `entry.decision = "HOLD"` → proceed to Step 5 (SWEEP).

**4a. Withdraw USDC from Aave (only if `entry.needs_withdraw = true`)**

```
factor_lend_withdraw(vaultAddress=<vault>, protocol="aave",
  assetAddress=<denominatorTokenAddress>, amount="all")
```

→ `sign_and_send` → `factor_get_transaction_status`

Check the returned `status` string:
- `status == "success"` → withdrawal confirmed on-chain. Proceed to 4b.
- `status == "pending"` → retry `factor_get_transaction_status` once.
  - Retry result is `"success"` → proceed to 4b.
  - Retry result is `"pending"` or `"failed"` → withdrawal failed/timed out. Call `factor_decode_error` with the transaction hash to get the revert reason, log the failure including the decoded error string, write error fallback (Step 6), STOP.
- `status == "failed"` → withdrawal reverted on-chain. Call `factor_decode_error` with the transaction hash to get the revert reason, log the failure including the decoded error string, write error fallback (Step 6), STOP.

**4b. Compute entry amount**

```
compute_token_amount(vault=<vault>,
  tokenAddress=<denominatorTokenAddress>,
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

**4c. TOKEN GUARD** — verify `tokenIn = <denominatorTokenAddress>`, `tokenOut = <tradingTokenAddress>`.

**4d. Pre-flight — gate with `simulate_enter`**

```
simulate_enter(vaultAddress=<vault>,
  tokenIn=<denominatorTokenAddress>,
  tokenOut=<tradingTokenAddress>,
  amountFormatted=<entry_amount_formatted>,
  scalpTargetPct=1.5, tpTargetPct=3.0)
```

- `PROCEED` → store `effective_entry_price = entryQuote.effectiveEntryPrice`, `scalp_target_usd = targets.scalpPrice`, `tp_target_usd = targets.tpPrice`. Proceed to 4e.
- `WARN_HIGH_FEES` or any `BLOCKED_*` → abort BUY, proceed to Step 5 (SWEEP at end).

**4e. Swap**

```
factor_swap_openocean(vaultAddress=<vault>,
  tokenIn=<denominatorTokenAddress>,
  tokenOut=<tradingTokenAddress>,
  amount=<entry_amount_wei>, slippage=0.5)
```

→ `sign_and_send` → `factor_get_transaction_status`

Check the returned `status` string:
- `status == "success"` → swap confirmed on-chain. Proceed to 4f.
- `status == "pending"` → retry `factor_get_transaction_status` once.
  - Retry result is `"success"` → proceed to 4f.
  - Retry result is `"pending"` or `"failed"` → swap failed/timed out. Call `factor_decode_error` with the transaction hash to get the revert reason, log the failure including the decoded error string, proceed to Step 5 (SWEEP at end).
- `status == "failed"` → swap reverted on-chain. Call `factor_decode_error` with the transaction hash to get the revert reason, log the failure including the decoded error string, proceed to Step 5 (SWEEP at end).

**4f. Compute position metadata (all arithmetic via `math_calculate`)**

**(i) Assign `position_index`**

**SKIP: If `positions[]` is empty → `position_index = 0`. Proceed to (ii). The math_calculate calls below only apply when positions already exist.**

Otherwise:
```
math_calculate(operation: "max",
  operands: [<all position.position_index values as strings>])
```
Then:
```
math_calculate(operation: "add",
  operands: [<max result>, "1"])
```
Use the returned `result` as `position_index`.

**(ii) Compute `stop_loss_usd`**

If `entry.atr_4h_pct` is null: set `stop_loss_usd = null`. Log `"atr_4h_pct unavailable — stop_loss_usd not set for this position"` and proceed.

Otherwise, three `math_calculate` calls:

**Step A — ATR multiplier:**
```
math_calculate(operation: "multiply",
  operands: [<entry.atr_4h_pct as string>, "0.025"])
```

Store returned `result` as `atr_multiplier`.

**Step B — ATR distance in USD:**
```
math_calculate(operation: "multiply",
  operands: [<effective_entry_price as string>, <atr_multiplier as string>])
```

Store returned `result` as `atr_distance_usd`.

**Step C — Stop level:**
```
math_calculate(operation: "subtract",
  operands: [<effective_entry_price as string>, <atr_distance_usd as string>])
```

Store returned `result` as `stop_loss_usd`.

**(iii) Compute `scalp_cost_basis_usd`**

```
math_calculate(operation: "divide",
  operands: [<entry_amount_formatted as string>, "2"])
```

Store returned `result` as `scalp_cost_basis_usd`.

**(iv) Compute `position_token_qty` (actual tokens received)**

**SKIP: If `positions[]` was empty before this BUY → `position_token_qty = total_token_balance_formatted` (no subtraction needed). Proceed to "Assemble the new position". The steps below only apply when positions already exist.**

Read the current total trading token balance post-swap:
```
compute_token_amount(vault=<vault>,
  tokenAddress=<tradingTokenAddress>,
  percentage=100)
```
→ `total_token_balance_formatted = computed.fromIdleFormatted`

Subtract the sum of all *existing* positions' `position_token_qty` (i.e. every position already in `positions[]` before this new position is appended):
```
math_calculate(operation: "subtract",
  operands: [<total_token_balance_formatted as string>, <sum of existing position.position_token_qty values as string>])
```

Store returned `result` as `position_token_qty`. This captures the exact tokens received after slippage rather than using the pre-swap quote.

**Assemble the new position:**

```json
{
  "position_index":            <position_index>,
  "cost_basis_usd":       <entry_amount_formatted>,
  "scalp_target_usd":     <scalp_target_usd>,
  "tp_target_usd":        <tp_target_usd>,
  "stop_loss_usd":        <stop_loss_usd>,
  "scalp_net_pct":        1.5,
  "tp_net_pct":           3.0,
  "scalp_cost_basis_usd": <scalp_cost_basis_usd>,
  "position_token_qty":   <position_token_qty>,
  "trade_count":          1,
  "entry_regime":         <entry.entry_regime>,
  "buy_timestamp":        null
}
```

Assemble the new position with `"buy_timestamp": null` and append it to `positions[]`.

**Write 1 — capture server epoch:**
Call `write_cycle_state({ stage: "execute", state: { "positions": <positions[] with the new position buy_timestamp=null> } })`.
The tool returns `{ ok: true, written_at: <epoch_seconds> }`.
Store `written_at` as `buy_ts`.

**Write 2 — persist with timestamp:**
Set the new position's `buy_timestamp = buy_ts`.
Call `write_cycle_state({ stage: "execute", state: { "positions": <positions[] with buy_timestamp now set> } })`.

Skip to the execution summary — do NOT call `write_cycle_state` again in Step 6 on this cycle.

---

## Step 5 — SWEEP (unified end-of-cycle)

Runs at the end of every cycle, regardless of `entry.decision` or outcome. This consolidates all idle USDC resupply logic from both BUY and HOLD paths into a single unified operation.

If `idle_usdc > 1` OR `sell_results` contains at least one entry with `status: "executed"`:

```
factor_lend_supply(vaultAddress=<vault>, protocol="aave",
  assetAddress=<denominatorTokenAddress>, amount="all")
```

→ `sign_and_send` → `factor_get_transaction_status`

Check the returned `status` string:
- `status == "success"` → supply confirmed on-chain. Proceed to Step 6.
- `status == "pending"` → retry `factor_get_transaction_status` once.
  - Retry result is `"success"` → proceed to Step 6.
  - Retry result is `"pending"` or `"failed"` → supply failed/timed out. Call `factor_decode_error` with the transaction hash to get the revert reason, log the failure including the decoded error string. Proceed to Step 6 (write state with unchanged positions).
- `status == "failed"` → supply reverted on-chain. Call `factor_decode_error` with the transaction hash to get the revert reason, log the failure including the decoded error string. Proceed to Step 6 (write state with unchanged positions).

If `idle_usdc ≤ 1`: invariant already satisfied — skip supply, proceed directly to Step 6.

---

## Step 6 — Write cycle state

**SKIP: If `entry.decision` was BUY and the swap in Step 4e succeeded,** cycle state was already written in Step 4f (two-write pattern for server epoch timestamp). Skip this entire step — do not write again.

For all other paths (HOLD, SELL-only, BUY aborted, error fallback), proceed with the write below. Existing positions carry their `buy_timestamp` untouched (or `buy_timestamp = null` if set on previous HOLD cycles before BUY is executed).

```
write_cycle_state({ stage: "execute", state={
  "positions": [
    {
      "position_index":            <number>,
      "cost_basis_usd":       <number>,
      "scalp_target_usd":     <number>,
      "tp_target_usd":        <number>,
      "stop_loss_usd":        <number | null>,
      "scalp_net_pct":        <number>,
      "tp_net_pct":           <number>,
      "scalp_cost_basis_usd": <number>,
      "position_token_qty":   <number>,
      "trade_count":          <number>,
      "entry_regime":         "<bull|normal>",
      "buy_timestamp":        <number>
    }
  ]
} })
```

**Important:** The tool signature is `write_cycle_state({ stage, state })`. The `vaultAddress` is **auto-injected** from the agent context and is NOT a parameter. The `stage` field MUST be the string `"execute"` and the `state` object contains the positions array.

`positions[]` reflects the final state after all SELL updates and any new BUY position appended. Fully-closed positions (100% SELL) are absent. Open positions (including partially-scalped positions) are present with updated fields.

**Error fallback** (decide state absent, `error` key present, or Aave bootstrap failed):

```
write_cycle_state({ stage: "execute", state={
  "positions": <existing positions[] unchanged>
} })
```

Do not mutate positions on error — preserve the prior state for the next cycle.

---

## Output

Emit the following JSON as the **absolute last line** of output. No prose before or after it.

```json
{
  "status": "<SUCCESS|ERROR>",
  "decision": "<BUY|HOLD>",
  "executed": <boolean>,
  "sell_results": [
    { "position_index": <n>, "exit_reason": "<str>", "sell_pct": <n>, "status": "<executed|failed>" }
  ],
  "buy_executed": <true|false>,
  "buy_position_index": <number|null>,
  "sweep_executed": <true|false>,
  "positions_open": <number>,
  "stop_loss_null_positions": [<position_index>, ...],
  "action_summary": "<string — see format below>"
}
```

| Field | Value |
|---|---|
| `status` | `"SUCCESS"` when the cycle completed without an error fallback; `"ERROR"` when decide stage was absent, Aave bootstrap failed, or Aave withdrawal failed |
| `decision` | `entry.decision` from `{{stage.decide}}` |
| `executed` | `true` when any on-chain transaction occurred this cycle: `buy_executed`, any `sell_results` entry with `status: "executed"`, or `sweep_executed`. `false` on a pure HOLD cycle where no swap, supply, or withdraw was sent. |
| `sell_results` | Every SELL attempt this cycle with its outcome — empty `[]` when no SELLs were attempted |
| `buy_executed` | `true` only when the BUY swap in Step 4e returned `status == "success"` |
| `buy_position_index` | `position_index` of the newly created position when `buy_executed = true`; else `null` |
| `sweep_executed` | `true` when the Aave supply in Step 5 returned `status == "success"` |
| `positions_open` | Count of positions in `positions[]` after all updates this cycle |
| `stop_loss_null_positions` | Array of `position_index` values where `stop_loss_usd` is `null` after this cycle — empty `[]` when all positions have a stop set |
| `action_summary` | Human-readable cycle summary — use the format below |

**`action_summary` format:**

```
EXECUTE
  positions open: <n> | sells: <n executed> / <n attempted> | buy: <BUY executed | HOLD | ABORTED reason>

  SELLS
    [position <n>] <exit_reason> <sell_pct>% → <executed | failed>
    ...

  BUY
    [executed] position_index: <n> | entry_price: $<effectiveEntryPrice> | stop: $<stop_loss_usd> | scalp: $<scalp_target_usd> | tp: $<tp_target_usd> | regime: <entry_regime>
    [hold] <one-sentence reason>

  SWEEP → <supplied $<amount> to Aave | skipped (idle_usdc ≤ $1 and no SELL executed)>
```
