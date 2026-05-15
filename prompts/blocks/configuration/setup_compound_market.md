{/* blocks/configuration/setup_compound_market.md — v1.0.0 */}

## Block: setup_compound_market

Ensures a Factor vault has both **Compound V3** (Comet) adapters, the cToken, and the market registration in place and ready for supply. Designed to run at the start of every cycle as part of the protocol configuration chain. Idempotent — in steady state the pre-flight check confirms all three conditions are met and exits immediately with a single `factor_get_vault_info` call and no transactions.

This block is protocol-agnostic and can be plugged into any strategy that includes Compound V3 in its `strategyConfig.protocols` list.

Compound V3 uses a **market address** (Comet contract / cToken address, e.g. cUSDCv3 on Base: `0xb125E6687d4313864e53df431d5425969c15Eb2`) rather than a marketId. Each Comet instance is a separate lending market for one base asset.

Compound V3 requires **two** manager adapters:
- `factor_compound_v3_adapter_pro` — handles supply and withdraw
- `factor_compound_v3_market_adapter_pro` — handles market registration

Both must be present in `adapters.manager` before any supply will succeed.

---

## Parameters

| Parameter | Type | Description |
|---|---|---|
| `{{vault_address}}` | address | The vault to configure |
| `{{market_address}}` | address | Compound V3 cToken / Comet address for the target market (e.g. cUSDCv3) |
| `{{asset_address}}` | address | ERC20 address of the base asset in this market (e.g. USDC) |
| `{{asset_symbol}}` | string | Human-readable symbol for logging (e.g. `USDC`) |
| `{{c_token_address}}` | address | Same as `{{market_address}}` — the Comet contract is also the receipt token |

---

## Pre-flight check (READ-ONLY — do this first)

Call `factor_get_vault_info` with `vaultAddress = {{vault_address}}`.

Record three flags independently:
- `adapter_pro_missing` = `true` if `adapters.manager` does NOT contain `factor_compound_v3_adapter_pro`; `false` if it does
- `market_adapter_missing` = `true` if `adapters.manager` does NOT contain `factor_compound_v3_market_adapter_pro`; `false` if it does
- `ctoken_missing` = `true` if `assets.supported` does NOT include `{{c_token_address}}`; `false` if it does

If **all three conditions are already satisfied** (both adapters present AND cToken registered) → **STOP immediately. Do not call `factor_get_address_book`, `factor_add_adapter`, `factor_add_vault_token`, `factor_execute_manager`, or any other tool.** This is the expected outcome on every cycle after the first setup run. Emit `{"configured":true,"skipped":true,"reason":"already_configured","protocol":"compound","asset":"{{asset_symbol}}"}` as the final output and exit.

Otherwise, proceed to Step 1. Execute only the steps whose flag is set; always run Step 5.

---

## Steps

### Step 1 — Resolve adapter addresses

Call `factor_get_address_book` and record:
- `addr_supply_adapter` = address of `factor_compound_v3_adapter_pro`
- `addr_market_adapter` = address of `factor_compound_v3_market_adapter_pro`

These addresses are required for all `factor_add_adapter` calls below. **Do NOT guess or hardcode them.**

### Step 2 — Register the supply adapter (only if `adapter_pro_missing`)

```
factor_add_adapter({
  vaultAddress: "{{vault_address}}",
  adapterAddress: <addr_supply_adapter>
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

### Step 4 — Register the receipt token / cToken (only if `ctoken_missing`)

```
factor_add_vault_token({
  vaultAddress: "{{vault_address}}",
  tokenAddress: "{{c_token_address}}",
  type: "asset"
})
→ sign_and_send → factor_get_transaction_status (must be "success"; retry once on "pending")
```

If the transaction reverts with `Already exists`, treat as a no-op and continue.

### Step 5 — Register the market (ALWAYS run — idempotent)

This call goes through `factor_compound_v3_market_adapter_pro` and is required before any supply or withdraw will work. It is safe to run every time — calling it again when the market is already registered is a no-op on-chain. Without this step `factor_lend_supply` will broadcast but revert silently.

```
factor_execute_manager({
  vaultAddress: "{{vault_address}}",
  steps: [{
    protocol: "compoundV3",
    action: "addMarketToAsset",
    params: {
      marketAddress: "{{market_address}}",
      assetAddress: "{{asset_address}}"
    }
  }]
})
→ sign_and_send → factor_get_transaction_status (must be "success"; retry once on "pending")
```

### Step 6 — Verify

Call `factor_get_vault_info` with `vaultAddress = {{vault_address}}` and confirm:
- `adapters.manager` contains `factor_compound_v3_adapter_pro`.
- `adapters.manager` contains `factor_compound_v3_market_adapter_pro`.
- `assets.supported` contains `{{c_token_address}}`.

If all three are present, the vault is correctly configured for Compound V3.

---

## Output (last line, single-line JSON, no fences)

```
{"configured":bool,"skipped":bool,"protocol":"compound","asset":"{{asset_symbol}}","market_address":"{{market_address}}","adapter_registered":bool,"market_adapter_registered":bool,"token_registered":bool,"market_registered":bool,"errors":[str]}
```

- `configured = true` only when both adapters and cToken are present in `factor_get_vault_info` after this block runs.
- `skipped = true` when the pre-flight found everything already present (no-op path).
- `market_registered = true` when Step 5 (`addMarketToAsset`) completed without revert.
- `errors`: empty `[]` on clean success; otherwise concise tags (`"addr_book_failed"`, `"adapter_revert"`, `"market_adapter_revert"`, `"token_revert"`, `"market_register_revert"`, `"verify_failed"`, `"sign_and_send_failed"`).

---

## Tool whitelist (only)

`factor_get_vault_info`, `factor_get_address_book`, `factor_add_adapter`, `factor_add_vault_token`, `factor_execute_manager`, `sign_and_send`, `factor_get_transaction_status`, `factor_decode_error`

## Hard rules

1. **`factor_add_adapter` takes `adapterAddress` only** — the schema accepts `vaultAddress` + `adapterAddress` and nothing else. There is no `adapterType` field and no `marketAddress` field on this tool. Resolve the adapter address from `factor_get_address_book` first (Step 1); do not hardcode or guess it.
2. **Both adapters are required and each is a separate transaction** — `factor_compound_v3_adapter_pro` and `factor_compound_v3_market_adapter_pro` must both be in `adapters.manager`. Registering only one will cause supply to fail silently on-chain.
3. **`factor_execute_manager` with `addMarketToAsset` is mandatory every run** — without Step 5, `factor_lend_supply` broadcasts but reverts silently. The call is idempotent; always execute it regardless of whether the other steps were skipped.
4. **`addMarketToAsset` (Compound) ≠ `addMarketToAssetAndDebt` (Morpho)** — do not confuse these. The Compound action is idempotent and safe to call unconditionally. The Morpho `addMarketToAssetAndDebt` is NOT — calling it twice causes phantom asset registration. Never substitute one for the other.
5. Every `sign_and_send` call **must** be immediately followed by `factor_get_transaction_status`. The tool returns `"success"`, `"pending"`, or `"failed"` — never hex. **A transaction can be confirmed on-chain and still fail (EVM revert).** Only `status == "success"` means the operation succeeded. On `"pending"`: retry `factor_get_transaction_status` once. If the retry returns `"pending"` or `"failed"`, treat as failure. Do not proceed past any step until you observe `"success"`.
6. On `"failed"` or a second `"pending"`: call `factor_decode_error` once, set `errors`, emit final JSON, and **STOP** — do not retry or attempt recovery.
