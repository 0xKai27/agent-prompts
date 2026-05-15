{/* blocks/configuration/setup_morpho_market.md — v1.0.0 */}

## Block: setup_morpho_market

Ensures a Factor vault has both **Morpho Blue** adapters, both the collateral and loan tokens registered with Chainlink accounting, and the market registered via `addMarketToAssetAndDebt`. Designed to run at the start of every cycle as part of the protocol configuration chain. Idempotent — in steady state the pre-flight check confirms all four conditions are met and exits immediately with a single `factor_get_vault_info` call and no transactions.

This block is protocol-agnostic and can be plugged into any strategy that includes Morpho Blue in its `strategyConfig.protocols` list.

Morpho Blue uses **per-market isolation**: each market is identified by a `marketId` (bytes32) and has a specific loan asset and collateral asset. Both tokens must be registered as vault assets with Chainlink accounting BEFORE `addMarketToAssetAndDebt` is called — without this, the call reverts with `INVALID_ASSET`.

Morpho Blue requires **two** manager adapters:
- `factor_morpho_adapter_pro` — handles supply, withdraw, borrow, repay
- `factor_morpho_market_adapter_pro` — handles market registration

---

## ⚠️ CRITICAL — READ THIS BEFORE EXECUTING

`addMarketToAssetAndDebt(marketId)` pushes a `marketId` into the vault's `assets[]` storage array. **On Studio Pro V1, this array CANNOT be shrunk** — there is no `removeMarketFromAssetAndDebt`. Each call appends a new entry. Calling it more than once for the same marketId duplicates the entry → `totalAssets()` multi-counts the position → the vault becomes structurally broken (observed 2026-05-05/06 incident: 8 phantom copies inflated TVL by $15,387 / 235% on vault `0x9446…8c`, requiring full liquidation to recover).

**Pre-flight check is mandatory. The gate is enforced as follows — do not proceed to Step 4 without consulting it:**

| `tokens_present` at pre-flight | Action for Step 4 |
|---|---|
| `true` — both tokens already in `assets[]` | **SKIP Step 4. Do NOT call `factor_execute_manager`.** The market was registered in a previous run. Calling it again permanently corrupts the vault. |
| `false` — one or both tokens absent | Proceed with Step 4, exactly once. |

If you are uncertain whether `tokens_present` is `true` or `false`, re-call `factor_get_vault_info` before proceeding. Do not guess.

---

## Parameters

| Parameter | Type | Description |
|---|---|---|
| `{{vault_address}}` | address | The vault to configure |
| `{{market_id}}` | bytes32 | Morpho Blue market identifier (e.g. `0x3a4048c64ba1b375330d376b1ce40e4047d03b47...`) |
| `{{loan_asset_address}}` | address | ERC20 address of the loan asset in this market (e.g. USDC) |
| `{{loan_asset_symbol}}` | string | Symbol of the loan asset (e.g. `USDC`) |
| `{{collateral_asset_address}}` | address | ERC20 address of the collateral asset (e.g. WETH) |
| `{{collateral_asset_symbol}}` | string | Symbol of the collateral asset (e.g. `WETH`) |

---

## Pre-flight check (READ-ONLY — do this first)

Call `factor_get_vault_info` with `vaultAddress = {{vault_address}}`.

Record FOUR flags independently:
- `morpho_adapter_missing` = `true` if `adapters.manager` does NOT contain `factor_morpho_adapter_pro`; `false` if it does
- `market_adapter_missing` = `true` if `adapters.manager` does NOT contain `factor_morpho_market_adapter_pro`; `false` if it does
- `collateral_missing` = `true` if `assets.supported` does NOT include `{{collateral_asset_address}}`; `false` if it does
- `loan_missing` = `true` if `assets.supported` does NOT include `{{loan_asset_address}}`; `false` if it does

Derive:
- `tokens_present` = collateral AND loan are both in `assets[]`

If **all four conditions are already satisfied** (both adapters present AND both tokens registered) → **STOP immediately. Do not call `factor_get_address_book`, `factor_add_adapter`, `factor_add_vault_token`, `factor_execute_manager`, or any other tool.** This is the expected outcome on every cycle after the first setup run. Emit `{"configured":true,"skipped":true,"reason":"already_configured","protocol":"morpho","market_id":"{{market_id}}"}` as the final output and exit.

Otherwise, proceed to Step 1. Execute only the steps whose flag is set; always run Step 4 if `tokens_present` is false, and SKIP Step 4 if `tokens_present` is true.

---

## Steps

### Step 1 — Resolve adapter addresses

Call `factor_get_address_book` and record:
- `addr_morpho_adapter` = address of `factor_morpho_adapter_pro`
- `addr_market_adapter` = address of `factor_morpho_market_adapter_pro`

**Do NOT guess or hardcode these addresses.**

### Step 2 — Register the Morpho adapter (only if `morpho_adapter_missing`)

```
factor_add_adapter({
  vaultAddress: "{{vault_address}}",
  adapterAddress: <addr_morpho_adapter>
})
→ sign_and_send → factor_get_transaction_status (must be "success"; retry once on "pending")
```

If the transaction reverts with `Already exists`, treat as a no-op and continue.

### Step 3 — Register the market adapter (only if `market_adapter_missing`)

This is a **separate transaction** from Step 2.

```
factor_add_adapter({
  vaultAddress: "{{vault_address}}",
  adapterAddress: <addr_market_adapter>
})
→ sign_and_send → factor_get_transaction_status (must be "success"; retry once on "pending")
```

If the transaction reverts with `Already exists`, treat as a no-op and continue.

### Step 3a — Register collateral token (only if `collateral_missing`)

Both the collateral token AND the loan token must be registered as vault assets with Chainlink accounting before `addMarketToAssetAndDebt` will work. If either is missing, the call will revert with `INVALID_ASSET`.

```
factor_add_vault_token({
  vaultAddress: "{{vault_address}}",
  tokenAddress: "{{collateral_asset_address}}",
  type: "asset"
})
→ sign_and_send → factor_get_transaction_status (must be "success"; retry once on "pending")
```

`accountingAddress` can be omitted — the tool auto-detects the Chainlink accounting adapter for the current chain.

### Step 3b — Register loan token (only if `loan_missing`)

```
factor_add_vault_token({
  vaultAddress: "{{vault_address}}",
  tokenAddress: "{{loan_asset_address}}",
  type: "asset"
})
→ sign_and_send → factor_get_transaction_status (must be "success"; retry once on "pending")
```

`Already exists` reverts are no-ops — proceed.

### Step 4 — Register the market with the vault

⛔ **DO NOT EXECUTE THIS STEP if `tokens_present = true` (both tokens already in `assets[]` at pre-flight).** The market was already registered. Calling `addMarketToAssetAndDebt` again appends a duplicate entry to the vault's storage array — an operation that cannot be undone. See the CRITICAL section above.

Only proceed if `tokens_present = false` (confirmed from pre-flight):

```
factor_execute_manager({
  vaultAddress: "{{vault_address}}",
  steps: [{
    protocol: "morpho",
    action: "addMarketToAssetAndDebt",
    params: { marketId: "{{market_id}}" }
  }]
})
→ sign_and_send → factor_get_transaction_status (must be "success"; retry once on "pending")
```

If this reverts: call `factor_decode_error`, set `errors: ["add_market_revert:<decoded>"]`, emit final JSON, and STOP. Do NOT retry — a partial revert leaves no duplicate; retrying would create one.

### Step 5 — Verify

Call `factor_get_vault_info` with `vaultAddress = {{vault_address}}` and confirm:
- `adapters.manager` contains `factor_morpho_adapter_pro`.
- `adapters.manager` contains `factor_morpho_market_adapter_pro`.
- `assets.supported` contains `{{collateral_asset_address}}`.
- `assets.supported` contains `{{loan_asset_address}}`.

If all four are present, the vault is correctly configured for this Morpho market.

---

## Output (last line, single-line JSON, no fences)

```
{"configured":bool,"skipped":bool,"protocol":"morpho","market_id":"{{market_id}}","loan_asset":"{{loan_asset_symbol}}","collateral_asset":"{{collateral_asset_symbol}}","adapter_registered":bool,"market_adapter_registered":bool,"collateral_registered":bool,"loan_registered":bool,"market_registered":bool,"errors":[str]}
```

- `configured = true` only when both adapters and both tokens are present in `factor_get_vault_info` after this block runs.
- `skipped = true` when the pre-flight found everything already present (no-op path).
- `market_registered = true` when `addMarketToAssetAndDebt` was called successfully in this run OR was already done at pre-flight (tokens already present).
- `errors`: empty `[]` on clean success; otherwise concise tags (`"addr_book_failed"`, `"adapter_revert"`, `"market_adapter_revert"`, `"collateral_token_revert"`, `"loan_token_revert"`, `"add_market_revert:<decoded>"`, `"verify_failed"`, `"sign_and_send_failed"`).

---

## Tool whitelist (only)

`factor_get_vault_info`, `factor_get_address_book`, `factor_add_adapter`, `factor_add_vault_token`, `factor_execute_manager`, `sign_and_send`, `factor_get_transaction_status`, `factor_decode_error`

## Hard rules

1. **`factor_add_adapter` takes `adapterAddress` only** — resolve `factor_morpho_adapter_pro` and `factor_morpho_market_adapter_pro` from `factor_get_address_book` first (Step 1). There is no `adapterType` field on this tool.
2. **Both adapters are required and each is a separate transaction** — registering only one will cause supply or market registration to fail.
3. **`type: "asset"` is required on every `factor_add_vault_token` call** — omitting it will fail schema validation.
4. **Both tokens must be in `assets[]` BEFORE `addMarketToAssetAndDebt`** — if either is missing, the call reverts with `INVALID_ASSET`. Steps 3a/3b must succeed before Step 4.
5. **`addMarketToAssetAndDebt` is called AT MOST ONCE per marketId per vault, ever.** The pre-flight gate (`tokens_present`) enforces this — if both tokens were already in `assets[]`, Step 4 is permanently skipped.
6. **`factor_execute_manager` format requires a `steps` array** — the correct call is `{ steps: [{ protocol: "morpho", action: "addMarketToAssetAndDebt", params: { marketId } }] }`. Passing `action` or `params` at the top level (outside `steps`) will fail validation.
7. **Morpho Blue has no mToken / receipt token** — do NOT attempt to look up or register a "Morpho shares token". Only the collateral and loan asset addresses are registered as vault assets.
8. Every `sign_and_send` call **must** be immediately followed by `factor_get_transaction_status`. The tool returns `"success"`, `"pending"`, or `"failed"` — never hex. **A transaction can be confirmed on-chain and still fail (EVM revert).** Only `status == "success"` means the operation succeeded. On `"pending"`: retry `factor_get_transaction_status` once. If the retry returns `"pending"` or `"failed"`, treat as failure. Do not proceed past any step until you observe `"success"`.
9. On `"failed"` or a second `"pending"`: call `factor_decode_error` once, set `errors`, emit final JSON, and **STOP** — do not retry.
