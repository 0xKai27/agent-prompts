# block: execution/lending/balanced/execute

**Responsibility:** Execute the routing decision from `{{stage.decide}}`. Perform a mandatory liveness check against the live vault before touching the chain, run the appropriate protocol operations with balance-delta verification after every transaction, write post-execution cycle state, and emit the structured result. This block combines execution and verification — balance delta is the only valid proof that a transaction landed.

---

## Injected inputs

| Field | Source | Type |
|---|---|---|
| `decision` | `{{stage.decide}}` | `HOLD\|HOLD-IDLE\|HOLD-IDLE-FLAG\|HOLD-TOPUP\|REBALANCE-YIELD\|FLAG_ANOMALY\|RISK-EXIT` |
| `do_supply` | `{{stage.decide}}` | `boolean` |
| `do_rebalance` | `{{stage.decide}}` | `boolean` |
| `do_withdraw` | `{{stage.decide}}` | `boolean` |
| `target_protocol` | `{{stage.decide}}` | `"aave"\|"compoundV3"\|"morpho"\|null` |
| `target_market` | `{{stage.decide}}` | `bytes32\|address\|null` |
| `target_receipt_token` | `{{stage.decide}}` | `address\|null` |
| `target_collateral_token` | `{{stage.decide}}` | `address\|null` |
| `target_collateral_symbol` | `{{stage.decide}}` | `string\|null` — Morpho only |
| `display_spread` | `{{stage.decide}}` | `bps\|null` |
| `risk_exit_condition` | `{{stage.decide}}` | `1\|2\|3\|4\|null` |
| `vaultAddress` | system prompt | `address` |
| `chain` | system prompt | `"base"\|"arbitrum"` |

If `{{stage.decide}}` is missing or `decision` is absent, emit `SAFETY_HALT` with `error_reason: "decide_stage_output_missing"` and stop.

---

## ⛔ Hard rules

1. **Liveness check is mandatory on every path.** Call `factor_vault_analytics` as the first tool call. If the live vault state is inconsistent with the decide routing (see Step 1), emit `SAFETY_HALT` and stop. No execution.

2. **TX hash = nothing.** `sign_and_send` returning a hash proves only that the tx was broadcast. `factor_get_transaction_status` returning `"confirmed"` proves only that it was mined — NOT that it succeeded. After every `factor_lend_supply` or `factor_lend_withdraw`, call `factor_vault_analytics` and verify the credit position state changed in the expected direction. That position check is the ONLY valid proof of success or failure.

4. **Adapter presence check is a last-resort safety guard.** In normal operation the `configure` stage (which runs before this block every cycle) ensures all required adapters are registered. If the check fails here, something went wrong upstream — emit `ERROR` and stop. Do not attempt to register adapters. Registration is the responsibility of the configure stage and the MND-763 configuration blocks.

5. **`write_cycle_state { stage: "execute" }` is mandatory on ALL paths**, including HOLD paths where no execution occurred. Skipping it leaves the next cycle's `scan_position` reading a stale position snapshot.

6. **No borrowing, swapping, leverage, or LP positions. Ever.** `factor_lend_borrow`, `factor_lend_repay`, `factor_swap_openocean`, and any write tool not in the allowed list below are forbidden.

7. **Gas is never a factor.** Gas is sponsored. Never cite gas as a reason for any decision or failure.

8. **No qualitative bias between protocols.** Protocol identity never influences execution decisions — only routing flags from `{{stage.decide}}`.

**Allowed tools:** `factor_vault_analytics`, `factor_lend_supply`, `factor_lend_withdraw`, `factor_get_vault_info`, `compute_token_amount`, `sign_and_send`, `factor_get_transaction_status`, `factor_decode_error`

**Forbidden:** `factor_lend_borrow`, `factor_lend_repay`, `factor_swap_openocean`, `factor_cast_call`, `factor_add_vault_token`, `factor_execute_manager`, `factor_add_adapter`, `factor_get_address_book`, any write tool not in the allowed list above.

---

## USDC addresses

<!-- TODO: replace with strategyConfig.usdcAddress once template is generalised beyond test usage -->

| Chain | USDC address |
|---|---|
| `base` | `0x833589fcd6edb6e08f4c7c32d4f71b54bda02913` |
| `arbitrum` | `0xaf88d065e77c8cC2239327C5EDb3A432268e5831` |

Resolve once from the `chain` field in the system prompt. Use the resolved address verbatim in every `factor_lend_supply` and `factor_lend_withdraw` call. Never substitute or re-derive.

---

## Protocol call signatures

```
Aave V3:     factor_lend_supply/withdraw { protocol: "aave",       assetAddress: <USDC>,            amount: <wei|"all"> }
Compound V3: factor_lend_supply/withdraw { protocol: "compoundV3", marketAddress: <target_market>,  assetAddress: <USDC>, amount: <wei|"all"> }
Morpho:      factor_lend_supply/withdraw { protocol: "morpho",     marketId: <target_market>,        amount: <wei|"all"> }
```

For Compound V3 withdrawal, `marketAddress` is `live_market` (from liveness check), not `target_market`. For RISK-EXIT, all protocol params come from the liveness check, not from `{{stage.decide}}`.

---

## Step 1 — Liveness check

Call `factor_vault_analytics { vaultAddress }` and derive the following from the response. Use these derived values throughout all subsequent steps.

| Name | Derivation |
|---|---|
| `live_protocol` | Priority order: (1) `lending.aave` present AND `totalCollateralUsd > 0` → `"aave"`; (2) any `lending.morpho[]` entry with `supplyUsd > 0` → `"morpho"`; (3) any `positions[]` entry with `protocol === "compound"` → `"compoundV3"`; (4) none → `null` |
| `live_market` | Aave: `null`. Morpho: `lending.morpho[0].id` (bytes32 marketId). Compound: `.address` of the cToken entry in `positions[]`. If vault is idle: `null`. |
| `live_apy_bps` | String-manipulate `stats.weightedApyCredit`: split on `"."`. If no `"."`, right = `"00"`. Take left + first 2 chars of right (pad with `"0"` if right is 1 char). Concatenate = bps. E.g. `"4.96"` → `496`, `"10.5"` → `1050`, `"5"` → `500`. `0` if no credit position. |
| `live_idle_usdc` | `stats.totalIdleUsd` (USD value — used for cycle state and action summary only, not for wei calculations) |
| `total_vault_usd` | `tvlUsd` |

Then apply the guard below based on `decision`:

| Decision | Guard |
|---|---|
| `HOLD-TOPUP` | Assert `live_protocol == target_protocol` AND `live_market == target_market`. Mismatch → `SAFETY_HALT`. |
| `REBALANCE-YIELD` where `do_rebalance == false` (idle vault, first deploy) | Assert `live_protocol == null`. Not null → `SAFETY_HALT`. |
| `REBALANCE-YIELD` where `do_rebalance == true` (moving from existing position) | No assertion — use `live_protocol` / `live_market` as the "from" side for withdrawal. |
| `RISK-EXIT` | No assertion — use `live_protocol` / `live_market` for withdrawal targets. |
| `HOLD`, `HOLD-IDLE`, `HOLD-IDLE-FLAG`, `FLAG_ANOMALY` | No assertion — `live_apy_bps` and `live_protocol` are used for output only. |

On `SAFETY_HALT`: write cycle state (Step 5) using `live_*` values, then emit the final JSON with `status: "SAFETY_HALT"` and stop. Do not proceed to Step 2.

---

## Step 2 — Route

| Decision | Next step |
|---|---|
| `HOLD` / `HOLD-IDLE` / `HOLD-IDLE-FLAG` / `FLAG_ANOMALY` | → Step 5 (skip execution) |
| `HOLD-TOPUP` | → Step 3a |
| `REBALANCE-YIELD` | → Step 3b |
| `RISK-EXIT` | → Step 3c |

---

## Step 3a — HOLD-TOPUP

**1. Idle check**

If `live_idle_usdc == 0` (from Step 1 liveness check) → no idle USDC available. Treat as HOLD, skip to Step 5.

**2. Adapter presence check**

```
factor_get_vault_info { vaultAddress } → adapters.manager[]
```

| `target_protocol` | Required adapters (match by `name` field) |
|---|---|
| `aave` | `factor_aave_adapter_pro` |
| `compoundV3` | `factor_compound_v3_adapter_pro` AND `factor_compound_v3_market_adapter_pro` |
| `morpho` | `factor_morpho_adapter_pro` AND `factor_morpho_market_adapter_pro` |

If any required adapter is missing from `adapters.manager`: record `error_reason: "adapter_not_registered: <target_protocol> requires <adapter_key> — configure stage should have registered this adapter this cycle"`, skip to Step 5 with status `ERROR`. Do not attempt to register adapters here.

**3. Supply**

```
factor_lend_supply { vaultAddress, protocol: target_protocol, [marketAddress|marketId]: target_market, assetAddress: <USDC>, amount: "all" }
→ sign_and_send → factor_get_transaction_status
→ record topUpTxHash, topUpTxStatus, topUpBlockNumber
```

Do NOT assess success from `topUpTxStatus`. Proceed to balance check.

**4. Mandatory balance check**

```
compute_token_amount { holder: vaultAddress, tokenAddress: <USDC>, percentage: 100 }
→ record amountWei as postSupplyUsdcWei
```

If `postSupplyUsdcWei != "0"` → USDC still in vault, supply did not fully land → status `ERROR`. Skip to Step 5.

**5. Post-supply analytics**

```
factor_vault_analytics { vaultAddress }
→ postTopUpApy   = string-manipulate stats.weightedApyCredit to bps (split on ".", left + first 2 chars of right padded to 2, concatenate)
→ total_vault_usd = tvlUsd
```

Status = `SUCCESS`. Proceed to Step 5.

---

## Step 3b — REBALANCE-YIELD

**1. Withdraw (only if `do_rebalance == true`)**

Skip this sub-step if `do_rebalance == false` (idle vault, no prior position).

```
factor_lend_withdraw { vaultAddress, protocol: live_protocol, [marketAddress|marketId]: live_market, assetAddress: <USDC>, amount: "all" }
→ sign_and_send → factor_get_transaction_status
→ record withdrawTxHash, withdrawTxStatus, withdrawBlockNumber
```

**Mandatory balance check:**

```
factor_vault_analytics { vaultAddress }
```

If `positions[]` still contains a credit entry with `protocol === live_protocol` and `valueUsd > 0` → withdrawal did not land → status `ERROR`. Stop. Do NOT proceed to supply.

**2. Adapter presence check**

Run the adapter presence check for `target_protocol` (same as Step 3a, sub-step 2). If any required adapter is missing → status `ERROR`, skip to Step 5.

**3. Supply**

```
factor_lend_supply { vaultAddress, protocol: target_protocol, [marketAddress|marketId]: target_market, assetAddress: <USDC>, amount: "all" }
→ sign_and_send → factor_get_transaction_status
→ record supplyTxHash, supplyTxStatus, supplyBlockNumber
```

Do NOT assess success from `supplyTxStatus`. Proceed to balance check.

**4. Mandatory balance check**

```
compute_token_amount { holder: vaultAddress, tokenAddress: <USDC>, percentage: 100 }
→ record amountWei as postSupplyUsdcWei
```

If `postSupplyUsdcWei != "0"` → USDC still in vault, supply did not fully land → status `PARTIAL` (withdrawal succeeded, supply failed — capital held idle). Skip to Step 5.

**5. Post-supply analytics**

```
factor_vault_analytics { vaultAddress }
→ postRebalanceApy = string-manipulate stats.weightedApyCredit to bps (split on ".", left + first 2 chars of right padded to 2, concatenate)
→ total_vault_usd  = tvlUsd
```

Status = `SUCCESS`. Proceed to Step 5.

---

## Step 3c — RISK-EXIT

For each active position identified by the liveness check (`live_protocol` / `live_market`):

**1. Withdraw**

```
factor_lend_withdraw { vaultAddress, protocol: live_protocol, [marketAddress|marketId]: live_market, assetAddress: <USDC>, amount: "all" }
→ sign_and_send → factor_get_transaction_status
→ record txHash, txStatus, blockNumber
```

**2. Mandatory balance check + post-exit analytics**

```
factor_vault_analytics { vaultAddress }
→ residualPositions = positions[] filtered where type == "credit" | "supply" and valueUsd > 0
→ total_vault_usd   = tvlUsd
→ postExitIdleUsdcUsd = stats.totalIdleUsd
```

Record this withdrawal as `confirmed` only if `residualPositions` contains no entry for `live_protocol`. Otherwise record as `failed` regardless of `txStatus`.

Determine overall status:
- All withdrawals balance-confirmed → `SUCCESS`
- Some balance-confirmed, some failed → `PARTIAL_EXIT`
- None balance-confirmed → `ERROR`

Proceed to Step 5.

---

## Step 4 — Post-execution analytics (HOLD paths only)

For `HOLD`, `HOLD-IDLE`, `HOLD-IDLE-FLAG`, `FLAG_ANOMALY`: call `factor_vault_analytics { vaultAddress }` → `total_vault_usd = tvlUsd`. Use `live_apy_bps` derived in Step 1 (same string-manipulation rule) as `post_apy_bps`.

Executing paths (`HOLD-TOPUP`, `REBALANCE-YIELD`, `RISK-EXIT`) get `total_vault_usd` from their own post-execution `factor_vault_analytics` call in Step 3.

---

## Step 5 — Write cycle state

Call `write_cycle_state { stage: "execute", state: <object> }` on ALL paths without exception.

The state object to write:

```json
{
  "decision": "<from stage.decide>",
  "current_protocol": "<resolved below>",
  "current_market": "<resolved below>",
  "post_apy_bps": <resolved below>
}
```

`written_at` is stamped automatically by the DB layer on every `write_cycle_state` upsert — the execute block does not need to supply a timestamp.

Field resolution by path:

| Path | `current_protocol` | `current_market` | `post_apy_bps` |
|---|---|---|---|
| `REBALANCE-YIELD` SUCCESS | `target_protocol` | `target_market` | `postRebalanceApy` |
| `REBALANCE-YIELD` PARTIAL | `null` (position not confirmed) | `null` | `null` |
| `REBALANCE-YIELD` ERROR (withdraw failed) | `live_protocol` | `live_market` | `live_apy_bps` |
| `HOLD-TOPUP` SUCCESS | `target_protocol` | `target_market` | `postTopUpApy` |
| `HOLD-TOPUP` ERROR | `live_protocol` | `live_market` | `live_apy_bps` |
| `RISK-EXIT` SUCCESS | `null` | `null` | `0` |
| `RISK-EXIT` PARTIAL_EXIT / ERROR | `live_protocol` | `live_market` | `live_apy_bps` |
| `HOLD` / `HOLD-IDLE` / `HOLD-IDLE-FLAG` / `FLAG_ANOMALY` | `live_protocol` | `live_market` | `live_apy_bps` |
| `SAFETY_HALT` | `live_protocol` | `live_market` | `live_apy_bps` |

---

## Step 6 — Emit output

Emit the following JSON object as the **absolute last line** of output. No prose after it.

```json
{
  "executed": <boolean>,
  "status": "<SUCCESS|PARTIAL|PARTIAL_EXIT|ERROR|SAFETY_HALT|HOLD>",
  "decision": "<from stage.decide>",
  "from_protocol": "<live_protocol on rebalance/risk-exit paths, else null>",
  "from_market": "<live_market on rebalance/risk-exit paths, else null>",
  "to_protocol": "<target_protocol on supply paths, else null>",
  "to_market": "<target_market on supply paths, else null>",
  "withdraw_tx_hash": "<string|null>",
  "withdraw_tx_status": "<confirmed|failed|pending|null>",
  "withdraw_block_number": <number|null>,
  "supply_tx_hash": "<string|null>",
  "supply_tx_status": "<confirmed|failed|pending|null>",
  "supply_block_number": <number|null>,
  "post_apy_bps": <number|null>,
  "total_vault_usd": <number|null>,
  "risk_exit_condition": <1|2|3|4|null>,
  "risk_exit_withdrawals": <[{protocol, market, tx_hash, tx_status, block_number, balance_confirmed}]|null>,
  "residual_positions": <array|null>,
  "post_exit_idle_usdc_usd": <number|null>,
  "action_summary": "<string>",
  "error_reason": "<string|null>"
}
```

`executed` is `true` for `REBALANCE-YIELD`, `HOLD-TOPUP`, and `RISK-EXIT` paths regardless of outcome. It is `false` for `HOLD`, `HOLD-IDLE`, `HOLD-IDLE-FLAG`, `FLAG_ANOMALY`, and `SAFETY_HALT`.

### Action summary strings

Resolve `action_summary` to one of the following. `postRebalanceApy/100` and `postTopUpApy/100` are the numeric APY percentages (e.g. `450 bps → 4.50%`).

| Decision | Status | `action_summary` |
|---|---|---|
| `REBALANCE-YIELD` | `SUCCESS` (first deploy, Aave) | `Vault now earning [postRebalanceApy/100]% APY on Aave V3 — your agent deployed capital.` |
| `REBALANCE-YIELD` | `SUCCESS` (first deploy, other) | `Vault now earning [postRebalanceApy/100]% APY on [to_protocol] — your agent deployed capital to [to_protocol] ([to_market]).` |
| `REBALANCE-YIELD` | `SUCCESS` (subsequent, moved from existing position) | `Vault now earning [postRebalanceApy/100]% APY on [to_protocol] — your agent captured a +[display_spread]bps improvement and moved from [from_protocol] to [to_protocol].` |
| `REBALANCE-YIELD` | `PARTIAL` | `⚠️ Withdrew from [from_protocol] but supply to [to_protocol] failed — capital held idle. Agent will retry next cycle.` |
| `REBALANCE-YIELD` | `ERROR` | `⚠️ Withdrawal from [from_protocol] failed — position unchanged.` |
| `HOLD-TOPUP` | `SUCCESS` | `Additional $[live_idle_usdc, 2dp] USDC earning [postTopUpApy/100]% APY on [to_protocol] — topped up with idle USDC.` |
| `HOLD-TOPUP` | `ERROR` | `⚠️ Top-up on [to_protocol] failed — idle balance remains undeployed. Agent will retry next cycle.` |
| `HOLD` | — | `Vault holding [live_apy_bps/100]% APY on [live_protocol] — confirmed best rate this cycle.` |
| `HOLD-IDLE` | — | `No market passed risk gates — capital preserved in USDC. Agent will re-evaluate next cycle.` |
| `HOLD-IDLE-FLAG` | — | `Vault holding [live_apy_bps/100]% APY on [live_protocol] — no eligible alternative this cycle.` |
| `FLAG_ANOMALY` | — | `⚠️ Vault holding [live_apy_bps/100]% APY on [live_protocol] — unusual rate spike detected on best alternative. Will re-evaluate next cycle.` |
| `RISK-EXIT` | `SUCCESS` | `⚠️ Risk condition [risk_exit_condition] triggered. Full position withdrawn — funds held in USDC. Agent will scan next cycle.` |
| `RISK-EXIT` | `PARTIAL_EXIT` | `⚠️ Risk condition [risk_exit_condition] triggered. Partial withdrawal only — remaining positions listed in residual_positions. Review required.` |
| `RISK-EXIT` | `ERROR` | `⚠️ Risk condition [risk_exit_condition] triggered but withdrawal failed — position unchanged. Immediate review required.` |
| `SAFETY_HALT` | — | `⚠️ Vault state changed between scan and execution — cycle aborted as a safety measure. No funds moved. Agent will re-evaluate next cycle.` |
