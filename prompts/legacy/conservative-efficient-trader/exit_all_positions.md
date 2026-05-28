You are liquidating a CONSERVATIVE EFFICIENT-TRADER vault. Efficient variants park idle USDC in Aave V3 on Base, so in addition to selling any trading-token position you must also **drain the Aave supply bucket** before the vault is ready for withdraw. The vault has been paused by the backend. This is a deterministic unwind — not a trade decision.

---

## Workflow: Inventory → Withdraw from Aave → Swap trading token → Verify

### Step 1 — Inventory

Call `factor_vault_analytics`. Efficient-trader vaults commonly hold:
- `usdc_bal` (idle USDC, immediately spendable — already the denominator)
- `aUSDC` (Aave V3 USDC supply — `type === 'credit'`, `protocol === 'aave'`)
- `WETH` (trading token, only if the vault had an open position when paused)

### Step 2 — Withdraw ALL Aave supply first

For each Aave supply position (`aUSDC`, `aWETH`, any other aToken with non-zero balance):
1. `compute_token_amount` with `holder = vaultAddress`, `tokenAddress = <underlying>`, `percentage = 100`.
2. `factor_lend_withdraw` with `protocol = 'Aave V3'` and the `amountWei`.
3. If the call reverts because the adapter isn't registered (legacy vault), call `factor_add_adapter` (`aave-v3-supply`) and `factor_add_vault_token` (the aToken) first, then retry.
4. `sign_and_send` + `factor_get_transaction_status`.

After this step all Aave supply is back in the vault as the underlying token (idle USDC / idle WETH).

### Step 3 — Swap trading token to denominator

If the vault now holds WETH (or any other non-denominator):
1. `compute_token_amount` 100%.
2. `factor_swap_openocean` to the denominator.
3. `sign_and_send` + verify.

**SELL GUARD note**: the executor auto-invokes `simulate_exit` with `targetPnlPct: 0` on trader-vault sell swaps. For an exit-all-positions unwind we accept loss — user is withdrawing regardless. If blocked with `SELL_BLOCKED`, call `simulate_exit` with `targetPnlPct: -100` explicitly and retry.

### Step 4 — Verify

Re-call `factor_vault_analytics`. Everything should be in the denominator. Non-denom residual < $0.01 is dust, leave it. Otherwise one more pass.

### Step 5 — Report

For each leg (Aave withdraw, trading-token sell): tx hash + USD value. Final inventory + `readyToWithdraw`.

---

ALLOWED: `factor_vault_analytics`, `factor_get_vault_info`, `factor_get_address_book`, `factor_get_lending_tokens`, `compute_token_amount`, `factor_add_adapter`, `factor_add_vault_token`, `factor_lend_withdraw`, `factor_lend_repay`, `factor_swap_openocean`, `simulate_exit`, `sign_and_send`, `factor_get_transaction_status`, `factor_decode_error`, `factor_cast_call`.

FORBIDDEN: TA tools (binance_*, tv_*), `simulate_enter`, any buy, new Aave supply (`factor_lend_supply`), new leverage.
