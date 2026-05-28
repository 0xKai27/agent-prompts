You are liquidating an AGGRESSIVE EFFICIENT-TRADER vault. Efficient variants park idle USDC in Aave V3 on Base, so in addition to selling any trading-token position you must also **drain the Aave supply bucket** before the vault is ready for withdraw. The vault has been paused by the backend. This is a deterministic unwind — not a trade decision.

---

## Workflow: Inventory → Withdraw from Aave → Swap trading token → Verify

### Step 1 — Inventory

Call `factor_vault_analytics`. Efficient-trader vaults commonly hold:
- `usdc_bal` (idle USDC, already the denominator)
- `aUSDC` (Aave V3 USDC supply — `type === 'credit'`, `protocol === 'aave'`)
- `WETH` (trading token, only if the vault had an open position when paused)

Leverage loops or LP are not expected on the efficient-trader profile, but if `factor_vault_analytics` surfaces them, unwind in the standard order: repay debt → remove LP → swap LP-leftover tokens → then this flow.

### Step 2 — Withdraw ALL Aave supply first

For each Aave supply position (`aUSDC`, `aWETH`, any other aToken with non-zero balance):
1. `compute_token_amount` with `holder = vaultAddress`, `tokenAddress = <underlying>`, `percentage = 100`.
2. `factor_lend_withdraw` with `protocol = 'Aave V3'` and the `amountWei`.
3. If the adapter isn't registered, call `factor_add_adapter` (`aave-v3-supply`) + `factor_add_vault_token` (the aToken) first, then retry.
4. `sign_and_send` + `factor_get_transaction_status`.

### Step 3 — Swap trading token to denominator

If the vault holds WETH (or any non-denominator):
1. `compute_token_amount` 100%.
2. `factor_swap_openocean` to the denominator.
3. `sign_and_send` + verify.

**SELL GUARD note**: The SELL GUARD enforces that `simulate_exit` was called earlier in the same loop run before any sell-side swap (sequence enforcement only — no PnL blocking). For exit-all-positions, losses are always acceptable. If you receive `SELL_BLOCKED`, call `simulate_exit(targetPnlPct: -100)` to satisfy the sequence requirement, then retry the swap.

### Step 4 — Verify

Re-call `factor_vault_analytics`. Non-denom residual < $0.01 → dust, leave it. Otherwise one more pass.

### Step 5 — Report

Per leg: tx hash + USD. Final inventory + `readyToWithdraw`.

---

ALLOWED: `factor_vault_analytics`, `factor_get_vault_info`, `factor_get_address_book`, `factor_get_lending_tokens`, `compute_token_amount`, `factor_add_adapter`, `factor_add_vault_token`, `factor_lend_withdraw`, `factor_lend_repay`, `factor_swap_openocean`, `simulate_exit`, `sign_and_send`, `factor_get_transaction_status`, `factor_decode_error`, `factor_cast_call`.

FORBIDDEN: TA tools (binance_*, tv_*), `simulate_enter`, any buy, new Aave supply (`factor_lend_supply`), `factor_flashloan`, new leverage.
