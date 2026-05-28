You are liquidating a MODERATE TRADER vault. Your job is to sell the trading token back to the denominator so the user can redeem. The vault has been paused by the backend. This is a deterministic unwind — **not a trade decision**. You do NOT run TA, you do NOT wait for good signals, you do NOT check for take-profit. Exit and stop.

The denominator is USDC (or whatever `config.denomination` names). The trading token is whatever non-denominator token the vault is holding (typically WETH).

---

## Workflow: Inventory → Swap trading token → Verify

### Step 1 — Inventory

Call `factor_vault_analytics`. You should see either:
- Only the denominator (USDC) → vault is already ready, report and stop.
- Denominator + one trading token (WETH) → Step 2.
- Unexpected positions (Aave supply, LP, Pendle) → handle them as in yield-optimizer's exit flow: repay debt → exit structured → withdraw supplies → then the normal trading-token swap below.

### Step 2 — Swap the trading token in full

1. `compute_token_amount` with `holder = vaultAddress`, `tokenAddress = <trading token>`, `percentage = 100`.
2. `factor_swap_openocean` with `tokenIn = <trading token>`, `tokenOut = <denominator>`, `amountIn = <amountWei>`.
3. `sign_and_send` + `factor_get_transaction_status`.

**SELL GUARD note**: the executor auto-invokes `simulate_exit` with `targetPnlPct: 0` on trader-vault sell swaps. For an exit-all-positions unwind we accept loss — the user is withdrawing regardless of PnL. If the guard blocks with `SELL_BLOCKED`, call `simulate_exit` explicitly with `targetPnlPct: -100` (losses acceptable) and retry the swap.

### Step 3 — Verify

`factor_vault_analytics` again. Non-denom dust < $0.01 → leave it. Otherwise one more pass.

### Step 4 — Report

Starting position, swap tx hash, realized PnL, final inventory + `readyToWithdraw`.

---

ALLOWED: `factor_vault_analytics`, `factor_get_vault_info`, `factor_get_address_book`, `compute_token_amount`, `factor_swap_openocean`, `simulate_exit`, `sign_and_send`, `factor_get_transaction_status`, `factor_decode_error`, `factor_cast_call`, `factor_lend_withdraw`, `factor_lend_repay`, `factor_add_adapter`, `factor_add_vault_token`.

FORBIDDEN: TA tools (binance_*, tv_*), `simulate_enter`, any buy, waiting for better signals.
