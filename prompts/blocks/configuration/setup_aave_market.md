## Block: setup_aave_market

Ensures a Factor vault has the **Aave V3** adapter and aToken registered and ready for supply. Designed to run at the start of every cycle as part of the protocol configuration chain. Idempotent — in steady state the pre-flight check confirms both are already registered and exits immediately with a single `factor_get_vault_info` call and no transactions.

This block is protocol-agnostic and can be plugged into any strategy that includes Aave V3 in its `strategyConfig.protocols` list.

**Aave V3 has no marketId concept.** It is an account-level protocol: one adapter covers all assets on a chain. Never call `addMarketToAssetAndDebt` for Aave — there is no such function, and attempting it will revert.

---

## Parameters

| Parameter | Type | Description |
|---|---|---|
| `{{vault_address}}` | address | The vault to configure |
| `{{asset_address}}` | address | ERC20 address of the token to supply (e.g. USDC, WETH) |
| `{{asset_symbol}}` | string | Human-readable symbol for logging (e.g. `USDC`, `WETH`) |
| `{{a_token_address}}` | address | Aave receipt token for this asset on this chain (e.g. aBasUSDC `0x4e65fE4DbA92790696d040ac24Aa414708F5c0AB` on Base) |

---

## Pre-flight check (READ-ONLY — do this first)

Call `factor_get_vault_info` with `vaultAddress = {{vault_address}}`.

Record two flags independently:
- `adapter_missing` = `true` if `managerAdapters` does NOT contain `factor_aave_adapter_pro`; `false` if it does
- `atoken_missing` = `true` if `assets` does NOT include `{{a_token_address}}`; `false` if it does

If **both conditions are already satisfied** (adapter present AND aToken registered) → **STOP immediately. Do not call `factor_get_address_book`, `factor_add_adapter`, `factor_add_vault_token`, or any other tool.** This is the expected outcome on every cycle after the first setup run. Emit `{"configured":true,"skipped":true,"reason":"already_configured","protocol":"aave","asset":"{{asset_symbol}}"}` as the final output and exit.

Otherwise, proceed to Step 1. Execute only the steps whose flag is set.

---

## Steps

### Step 1 — Resolve adapter address

Call `factor_get_address_book` and record:
- `addr_aave_adapter` = address of `factor_aave_adapter_pro`

**Do NOT guess or hardcode this address.**

### Step 2 — Register the Aave V3 adapter (only if `adapter_missing`)

```
factor_add_adapter({
  vaultAddress: "{{vault_address}}",
  adapterAddress: <addr_aave_adapter>
})
→ sign_and_send → factor_get_transaction_status (must be 0x1)
```

If the transaction reverts with `Already exists`, treat as a no-op and continue.

### Step 3 — Register the receipt token (aToken) (only if `atoken_missing`)

Aave issues an aToken per asset per chain. Register it so the vault accounting includes the supplied balance.

```
factor_add_vault_token({
  vaultAddress: "{{vault_address}}",
  tokenAddress: "{{a_token_address}}",
  type: "asset"
})
→ sign_and_send → factor_get_transaction_status (must be 0x1)
```

If the transaction reverts with `Already exists`, treat as a no-op.

### Step 4 — Verify

Call `factor_get_vault_info` with `vaultAddress = {{vault_address}}` and confirm:
- `managerAdapters` contains `factor_aave_adapter_pro`.
- `assets` contains `{{a_token_address}}`.

If both are present, the vault is correctly configured.

---

## Output (last line, single-line JSON, no fences)

```
{"configured":bool,"skipped":bool,"protocol":"aave","asset":"{{asset_symbol}}","adapter_registered":bool,"token_registered":bool,"errors":[str]}
```

- `configured = true` only when both adapter and aToken are present in `factor_get_vault_info` after this block runs.
- `skipped = true` when the pre-flight found everything already present (no-op path).
- `errors`: empty `[]` on clean success; otherwise concise tags (`"addr_book_failed"`, `"adapter_revert"`, `"token_revert"`, `"verify_failed"`, `"sign_and_send_failed"`).

---

## Tool whitelist (only)

`factor_get_vault_info`, `factor_get_address_book`, `factor_add_adapter`, `factor_add_vault_token`, `sign_and_send`, `factor_get_transaction_status`, `factor_decode_error`

## Hard rules

1. **`factor_add_adapter` takes `adapterAddress` only** — resolve `factor_aave_adapter_pro` from `factor_get_address_book` first (Step 1). There is no `adapterType` field on this tool.
2. **`type: "asset"` is required on `factor_add_vault_token`** — omitting it will fail schema validation.
3. **Pre-flight checks `{{a_token_address}}` in `assets[]`, NOT `{{asset_address}}`** — the aToken is what gets registered, not the underlying ERC20.
4. **NEVER call `addMarketToAssetAndDebt` or `factor_execute_manager`** — Aave V3 has no market registry. Any call to `factor_execute_manager` from this block is forbidden.
5. Every `sign_and_send` call **must** be immediately followed by `factor_get_transaction_status`. **A transaction can be confirmed (included in a block) and still fail on-chain.** `status = "0x0"` means an EVM revert — the tx was mined but its execution reverted. Only `status = "0x1"` means the operation succeeded. Do not proceed past any step until you observe `"0x1"`.
6. On any non-`0x1` status (including `"0x0"` EVM revert): call `factor_decode_error` once, set `errors`, emit final JSON, and **STOP** — do not retry or attempt recovery.
