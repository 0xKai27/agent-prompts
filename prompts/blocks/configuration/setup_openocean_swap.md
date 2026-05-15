{/* blocks/configuration/setup_openocean_swap.md — v1.0.0 */}

## Block: setup_openocean_swap

Ensures a Factor vault has the **OpenOcean** DEX aggregator adapter registered and ready for swaps. Designed to run at the start of every cycle as part of the protocol configuration chain. Idempotent — in steady state the pre-flight check confirms the adapter is registered and exits immediately with a single `factor_get_vault_info` call and no transactions.

This block is protocol-agnostic and can be plugged into any strategy that includes OpenOcean in its `strategyConfig.protocols` list. Note: OpenOcean is a swap adapter, not a lending protocol — it does not belong in a lending-only configuration chain.

OpenOcean requires **only an adapter registration** — no receipt token, no market ID, no `addMarketToAssetAndDebt`. The vault's existing token registry (assets added at deploy or via lending setup blocks) is sufficient for swap output accounting.

---

## Parameters

| Parameter | Type | Description |
|---|---|---|
| `{{vault_address}}` | address | The vault to configure |

---

## Pre-flight check (READ-ONLY — do this first)

Call `factor_get_vault_info` with `vaultAddress = {{vault_address}}`.

- If `adapters.manager` already contains an entry whose name includes `"openocean"` → **STOP immediately. Do not call `factor_get_address_book`, `factor_add_adapter`, or any other tool.** This is the expected outcome on every cycle after the first setup run. Emit `{"configured":true,"skipped":true,"reason":"already_configured","protocol":"openocean"}` as the final output and exit.
- Otherwise, proceed to Step 1.

---

## Steps

### Step 1 — Resolve adapter address

Call `factor_get_address_book` and record:
- `addr_openocean_adapter` = address of `factor_openocean_adapter_pro`

**Do NOT guess or hardcode this address.**

### Step 2 — Register the OpenOcean adapter

```
factor_add_adapter({
  vaultAddress: "{{vault_address}}",
  adapterAddress: <addr_openocean_adapter>
})
→ sign_and_send → factor_get_transaction_status (must be "success"; retry once on "pending")
```

If the transaction reverts with `Already exists`, the adapter is already present — treat as a no-op.

### Step 3 — Verify

Call `factor_get_vault_info` with `vaultAddress = {{vault_address}}` and confirm:
- `adapters.manager` contains an entry whose name includes `"openocean"`.

If present, the vault is correctly configured for swaps.

---

## Output (last line, single-line JSON, no fences)

```
{"configured":bool,"skipped":bool,"protocol":"openocean","adapter_registered":bool,"errors":[str]}
```

- `configured = true` only when the openocean adapter is present in `factor_get_vault_info` after this block runs.
- `skipped = true` when the pre-flight found the adapter already present (no-op path).
- `errors`: empty `[]` on clean success; otherwise concise tags (`"adapter_revert"`, `"verify_failed"`, `"sign_and_send_failed"`).

---

## Tool whitelist (only)

`factor_get_vault_info`, `factor_get_address_book`, `factor_add_adapter`, `sign_and_send`, `factor_get_transaction_status`, `factor_decode_error`

## Hard rules

1. **`factor_add_adapter` takes `adapterAddress` only** — resolve `factor_openocean_adapter_pro` from `factor_get_address_book` first (Step 1). There is no `adapterType` field on this tool.
2. **Do NOT register any receipt token** — OpenOcean is a router, not a custodian. Tokens received from swaps land in the vault directly and are already covered by the vault's existing asset registry.
3. **Do NOT call `addMarketToAssetAndDebt`** — OpenOcean has no market concept.
4. Every `sign_and_send` call **must** be immediately followed by `factor_get_transaction_status`. The tool returns `"success"`, `"pending"`, or `"failed"` — never hex. **A transaction can be confirmed on-chain and still fail (EVM revert).** Only `status == "success"` means the operation succeeded. On `"pending"`: retry `factor_get_transaction_status` once. If the retry returns `"pending"` or `"failed"`, treat as failure. Do not proceed past any step until you observe `"success"`.
5. On `"failed"` or a second `"pending"`: call `factor_decode_error` once, set `errors`, emit final JSON, and **STOP** — do not retry.

## Note on token registration

OpenOcean swaps require **both the input token and the output token** to be in the vault's `assets.supported[]` registry:
- The **input token** must be present so the adapter can read the vault's balance before the swap.
- The **output token** must be present so vault accounting captures the tokens received from the swap.

This block does **not** register tokens — it registers only the OpenOcean adapter. Token registration is the responsibility of the execution prompt that calls `factor_swap_openocean`, which must ensure both sides are registered (at vault deploy time or via a lending setup block: `setup_aave_market`, `setup_compound_market`, `setup_morpho_market`) before invoking the swap.
