{/* blocks/execute/trading/swing/verify.md — v1.0.0 */}

# block: execute/trading/swing/verify

**Responsibility:** Independent post-execution sanity check. Read the execute stage's structured output from `{{previous}}`, re-fetch live vault state via `factor_vault_analytics`, and confirm the live position is consistent with what execute reported. Thin check only — the goal is to catch hallucinated successes, surface SELL failures that need retry, and flag positions without stop protection. Emits `verified` boolean and a user-facing `summary`. No execution, no writes.

---

## Input

Read exclusively from `{{previous}}` (the execute stage JSON output). Required fields:

| Field | Type | Used for |
|---|---|---|
| `executed` | `boolean` | Gate — if `false`, no on-chain transaction occurred this cycle; skip tool calls |
| `status` | `string` | `"SUCCESS"` or `"ERROR"` — whether the cycle completed without an error fallback |
| `decision` | `string` | `BUY` or `HOLD` — context for summary |
| `sell_results` | `array` | Detect failed SELL attempts |
| `buy_executed` | `boolean` | Whether a BUY swap succeeded this cycle |
| `buy_position_index` | `number\|null` | Index of the new position when `buy_executed = true` |
| `sweep_executed` | `boolean` | Whether Aave SWEEP succeeded this cycle |
| `positions_open` | `number` | Expected count of open positions after the cycle |
| `stop_loss_null_positions` | `array` | Positions with `stop_loss_usd = null` — no stop protection active |
| `action_summary` | `string` | Passed through as `summary` on clean pass |

Also reads `vaultAddress` from the system prompt.

**No `read_cycle_state`, `write_cycle_state`, or signing tool calls.** Verify is strictly read-only.

---

## ⛔ Hard rules

1. **One `factor_vault_analytics` call only.** Do not call any write tools, signing tools, swap tools, or cycle state tools.
2. **`verified: true` means live state is consistent with `previous`.** It does not mean execution was perfect — `verified: true` with warnings in `summary` is valid and expected on SELL-failure cycles.
3. **On any `factor_vault_analytics` error**, emit `verified: false` with `summary` explaining the tool failure. Do not retry.
4. **SELL failures are warnings, not `verified: false`.** A failed SELL is preserved in `positions[]` for the next cycle to retry — this is the correct behaviour, not an inconsistency.

---

## Step 1 — Gate on `executed`

If `previous.executed == false`:

```json
{ "verified": true, "summary": "<previous.action_summary>" }
```

Stop. No tool calls.

---

## Step 2 — Fetch live vault state

```
factor_vault_analytics { vaultAddress }
```

On error → emit `{ "verified": false, "summary": "⚠️ factor_vault_analytics failed — could not verify live state: <error message>" }` and stop.

Derive from the response:

| Name | Derivation |
|---|---|
| `live_idle_usdc` | `usdValue` of the entry in `balances[]` where `address == denominatorTokenAddress`; `0` if absent |
| `live_trading_token_usd` | `usdValue` of the entry in `balances[]` where `address == tradingTokenAddress`; `0` if absent |
| `live_aave_usdc_usd` | `lending.aave.totalCollateralUsd` if present, else `0` |

---

## Step 3 — Checks

Run all checks below. Collect all failure messages into `issues[]`. `verified` starts as `true` and is set to `false` only by a hard assertion failure.

### 3a — Failed SELLs

Inspect `previous.sell_results`. Collect any entry where `status == "failed"` into `failed_sells[]`.

If `failed_sells[]` is non-empty → append to `issues[]`:

```
⚠️ SELL failed for position(s) [<indices>] — preserved in positions[] for retry next cycle
```

This does **not** set `verified: false`. A failed SELL is the expected safe fallback.

---

### 3b — Partial SELL consistency

If `previous.sell_results` contains at least one entry with `status == "executed"` AND `previous.positions_open > 0`:

Assert `live_trading_token_usd > 0`.

- Pass → no issue.
- Fail → append to `issues[]`: `"⚠️ Partial SELL reported success but live vault shows zero trading token balance — position may not have been reduced correctly."` Set `verified = false`.

---

### 3c — Full exit consistency

If `previous.positions_open == 0` AND `previous.sell_results` contains at least one entry with `status == "executed"`:

Assert ALL of:
1. `live_trading_token_usd < 1`
2. `live_aave_usdc_usd > 0` OR `live_idle_usdc > 0`

- All pass → no issue.
- `live_trading_token_usd ≥ 1` → append to `issues[]`: `"⚠️ All positions reported closed but live vault still shows $<live_trading_token_usd> in trading token — residual balance may require manual review."` Set `verified = false`.
- Both USDC balances zero → append to `issues[]`: `"⚠️ All positions reported closed but no USDC recovered in vault or Aave — proceeds may not have settled."` Set `verified = false`.

---

### 3d — BUY consistency

If `previous.buy_executed == true`:

Assert `live_trading_token_usd > 0`.

- Pass → no issue.
- Fail → append to `issues[]`: `"⚠️ BUY reported success (position_index: <buy_position_index>) but live vault shows zero trading token balance — swap may not have settled."` Set `verified = false`.

---

### 3e — SWEEP consistency

If `previous.sweep_executed == true`:

Assert `live_idle_usdc ≤ 1`.

- Pass → no issue.
- Fail → append to `issues[]`: `"⚠️ SWEEP reported success but live vault shows $<live_idle_usdc> idle USDC — Aave supply may not have settled."` Set `verified = false`.

---

### 3f — ERROR path

If `previous.status == "ERROR"`:

No state assertion — `factor_vault_analytics` succeeded and the vault is reachable. Error was already reported by execute; vault is not in an unknown state.

→ `verified: true` (unless a hard assertion in 3b–3e already set `verified = false` for a partial execution that preceded the error).

---

### 3g — Missing stop protection

If `previous.stop_loss_null_positions` is non-empty → append to `issues[]`:

```
⚠️ Position(s) [<indices>] have no stop_loss_usd (atr_4h_pct was unavailable at entry) — exits will rely on regime_drop and take_profit signals only
```

This does **not** set `verified: false`. It is an advisory flag for the next decide cycle.

---

## Step 4 — Emit output

Compose `summary`:
- Start with `previous.action_summary`.
- If `issues[]` is non-empty, append each issue on a new line.

Emit the following JSON as the **absolute last line** of output. No prose after it.

```json
{
  "verified": <boolean>,
  "summary": "<string>"
}
```

`verified` is `true` when no hard assertion in Steps 3b, 3c, 3d, or 3e failed. SELL failures (3a), ERROR path (3f), and missing stop warnings (3g) do not affect `verified`.
