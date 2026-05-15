{/* blocks/vault/scan-position-trading.md — v1.0.0 */}

# block: vault/scan-position-trading
**Responsibility:** Read and emit the current trading vault position: idle USDC, Aave credit (efficient variant), and all held trading token positions. Detect external deposits/withdrawals by comparing vault share supply to the previous cycle. Surface vault-state anomalies (end-of-cycle invariant violations, unexpected tokens). Owns its own inter-cycle state (`vault_position_trading`). No TA. No market data. No decisions. No execution.

---

## Injected inputs — read from `strategyConfig` in your system prompt

| Field | Type | Default | Description |
|---|---|---|---|
| `tradingTokenAddress` | string | — | **Required.** Address of the trading token (e.g. WETH: `0x4200000000000000000000000000000000000006` on Base). |
| `tradingTokenSymbol` | string | — | **Required.** Symbol for display and balance classification (e.g. `"WETH"`, `"cbBTC"`). |
| `denominatorTokenAddress` | string | — | **Required.** Address of the denominator asset (e.g. USDC: `0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913` on Base). |
| `aaveReceiptTokenAddress` | string | `null` | Address of the Aave receipt token for the denominator (e.g. aBasUSDC). Present only on efficient variants. If absent, Aave credit is not expected in the vault. |

A required field absent from `strategyConfig` is a misconfiguration. Emit `{ "error": "misconfiguration", "missing": ["<field>"] }` as the final JSON line and stop.

---

## ⛔ Hard rules

1. **`factor_vault_analytics` is called exactly once.** Enumerate all balances from its result — do not assume a fixed number of positions.

2. **`factor_get_shares` is called exactly once** with `userAddress = vaultAddress`. Only `totalSupply.formatted` is used. `pricePerShare` and the per-user `shares` field are irrelevant — ignore them.

3. **No TA or market data tools.** Do not call `binance_multi_timeframe`, `binance_technical_analysis`, `tv_multi_timeframe`, `trade_signals`, `defi_llama_yields`, or any market-facing tool. This block reads vault state only.

4. **No decisions, no execution, no `sign_and_send`.** Only `read_cycle_state` (Step 1) and `write_cycle_state` (Step 7) are permitted write-adjacent calls.

5. **Anomalies do not stop execution.** Record all anomalies in `anomalies[]` and continue to the final output. It is the decision block's responsibility to act on them.

6. **External flows are expected, normal user actions.** Report them factually in output fields — do not add them to `anomalies[]`.

7. **Always emit valid JSON as the absolute last line**, regardless of outcome. On any unrecoverable tool error, emit `{ "error": "<description>", "anomalies": [] }` and stop.

---

## Step 1 — Read config and previous cycle state

From `strategyConfig`, extract `tradingTokenAddress`, `tradingTokenSymbol`, `denominatorTokenAddress`. Extract `aaveReceiptTokenAddress` if present (null otherwise). Apply the misconfig guard above.

Build the **expected token set** — the complete set of addresses this vault is permitted to hold:
- `denominatorTokenAddress` (USDC)
- `tradingTokenAddress`
- `aaveReceiptTokenAddress` if not null (efficient variant only)

Any token address in the vault that is not in this set is unexpected.

Call `read_cycle_state`:
```
read_cycle_state { stage: "vault_position_trading" }
```

The tool always returns `{ ok: true, state: <blob> | null }`.
- `state` is not null → assign as `previousState`. Extract `previousTotalSupply`.
- `state` is null → first cycle. Set `previousTotalSupply = null`.

Call `read_cycle_state`:
```
read_cycle_state { stage: "execute" }
```

The tool always returns `{ ok: true, state: <blob> | null }`.
- `state` is not null → assign as `positionState`. Extract:
  - `positionState.positions` → `openPositions` (array of positions; may be empty)
- `state` is null OR `positionState.positions` is empty → no tracked positions. Set `openPositions = []`.

Each position in `openPositions` has the shape written by the execute block:
```
{
  "position_index":         <number>,
  "cost_basis_usd":         <number>,
  "scalp_target_usd":       <number>,
  "tp_target_usd":          <number>,
  "stop_loss_usd":          <number | null>,
  "scalp_net_pct":          <number>,
  "tp_net_pct":             <number>,
  "scalp_cost_basis_usd":   <number>,
  "position_token_qty":     <number>,
  "trade_count":            <number>,
  "entry_regime":           "<string>",
  "buy_timestamp":          <unix_seconds>
}
```

Compute `lastBuyTimestamp` and `buy_cooldown_active` (the only arithmetic permitted in this block):
- IF `openPositions` is empty → `lastBuyTimestamp = null`, `buy_cooldown_active = false`.
- ELSE → `lastBuyTimestamp = max(openPositions.map(p => p.buy_timestamp))` (the most recent position's entry timestamp).
- IF `lastBuyTimestamp` is null → `buy_cooldown_active = false`.
- ELSE: `buy_cooldown_active = (currentUnixTimestamp − lastBuyTimestamp) < 7200`.
  Use the current Unix timestamp (seconds since epoch) at the time this block executes.

---

## Step 2 — Fetch vault analytics

```
factor_vault_analytics { vaultAddress }
```

From the result, extract:
- `balances[]` — all non-zero token balances. For each entry record `symbol`, `address`, `amount`, `usdValue`.
- `stats.totalUsd` → `totalVaultUsd`.
- `lending.aave.totalCollateralUsd` → `creditUsd`. Use `0` if the `lending` block is absent or `totalCollateralUsd` is null or zero.

---

## Step 3 — Fetch vault share supply

```
factor_get_shares { vaultAddress, userAddress: vaultAddress }
```

Extract:
- `totalSupply.formatted` → `currentTotalSupply` (floating-point, not raw bigint).

---

## Step 4 — Detect external flows

```
IF previousTotalSupply is null (first cycle):
  externalFlowDetected = false
  flowType             = null

ELSE IF currentTotalSupply > previousTotalSupply:
  externalFlowDetected = true
  flowType             = "deposit"

ELSE IF currentTotalSupply < previousTotalSupply:
  externalFlowDetected = true
  flowType             = "withdrawal"

ELSE:
  externalFlowDetected = false
  flowType             = null
```

Do not add external flows to `anomalies[]`.

---

## Step 5 — Classify balances

Walk `balances[]` from Step 2. Classify each entry by matching its `address` against the expected token set:

| Match | Classification | Action |
|---|---|---|
| `address == denominatorTokenAddress` | Idle USDC | Set `idleUsdc = usdValue` |
| `address == aaveReceiptTokenAddress` (when configured) | Aave credit | Confirms Aave supply is present; `creditUsd` already populated from `lending.aave` in Step 2 — use that value as authoritative |
| `address == tradingTokenAddress` | Trading position | Append to `positions[]` |
| Anything else | Unexpected | Append to `unexpectedTokens[]` |

`idleUsdc` = 0 if no USDC entry in `balances[]`.

`positions[]` structure — one entry per trading token balance:
```
{ "token_symbol": <symbol>, "token_address": <address>, "amount": <amount>, "usd_value": <usdValue> }
```

---

## Step 6 — Vault-state anomaly checks

Build `anomalies[]`. Append a descriptive string for each condition that is true:

1. **End-of-cycle invariant violation — idle USDC with no open position**
   `idleUsdc > 1 AND positions[] is empty`
   → append `"idle_usdc_above_invariant: idle_usdc=$<idleUsdc> no_open_position"`
   This indicates the previous cycle ended without sweeping idle USDC to Aave.

2. **Unexpected token in vault**
   For each entry in `unexpectedTokens[]`
   → append `"unexpected_token: <symbol> (<address>)"`

---

## Step 7 — Persist cycle state and emit output

Call `write_cycle_state`:
```
write_cycle_state {
  stage: "vault_position_trading",
  state: {
    "totalSupply": <currentTotalSupply>
  }
}
```

The tool returns `{ ok: true }` on success. On error, log the error string and continue to emit the final JSON — the next cycle will treat `null` state as first-cycle.

> **Do not write to `execute` state.** That stage is owned exclusively by the execute block — written on BUY, modified on partial SELL, cleared on full SELL. This block only reads it.

**Emit the following JSON object as the absolute last line of your response. No prose after it.**

```json
{
  "idle_usdc":              <number>,
  "credit_usd":             <number>,
  "positions": [
    {
      "token_symbol":       "<string>",
      "token_address":      "<address>",
      "amount":             <number>,
      "usd_value":          <number>
    }
  ],
  "open_positions": [
    {
      "position_index":         <number>,
      "cost_basis_usd":         <number>,
      "scalp_target_usd":       <number>,
      "tp_target_usd":          <number>,
      "stop_loss_usd":          <number | null>,
      "scalp_net_pct":          <number>,
      "tp_net_pct":             <number>,
      "scalp_cost_basis_usd":   <number>,
      "position_token_qty":     <number>,
      "trade_count":            <number>,
      "entry_regime":           "<string>",
      "buy_timestamp":          <number>
    }
  ],
  "buy_cooldown_active":    <boolean>,
  "total_vault_usd":        <number>,
  "external_flow_detected": <boolean>,
  "flow_type":              "deposit|withdrawal|null",
  "anomalies":              ["<string>"]
}
```

`positions` = on-chain balances from `factor_vault_analytics`; `[]` when vault holds no trading token.
`open_positions` = tracked positions from `execute` cycle state; `[]` when no position is recorded. Each position carries its own cost basis, exit targets, entry regime, buy timestamp, and trade metadata — written by the execute block on BUY, modified on partial SELL, cleared on full SELL.
`buy_cooldown_active` = `false` when no tracked positions exist; `true` when fewer than 7200 seconds have elapsed since the most recent BUY (derived from `max(buy_timestamp)` across all positions).
`anomalies` = `[]` when no anomalies are detected.

The orchestrator reads this as `{{previous}}` for downstream stages. On any unrecoverable error, emit `{ "error": "<description>", "anomalies": [] }`.
