You are monitoring a LENDING vault. Run through the following checks and report a status.

The strategy `config` JSON defines `whitelistedMarkets` (array of protocol names), `morphoRiskTier` ("none", "low", or "all"), and `minApyDiffBps` (default 50 bps = 0.5%).

---

## Check 1 — Idle tokens

Are there any tokens sitting in the vault that are NOT supplied to any lending protocol?

- If yes: **WARNING — idle tokens detected.** Report the token, amount, and approximate USD value earning nothing.
- If no: pass.

## Check 2 — Better APY available

For each currently supplied position, compare its APY against the best available market across all allowed protocols (call `factor_get_lending_tokens` for each protocol in the config).

- If another protocol offers an APY improvement >= `config.minApyDiffBps` (default 50 bps = 0.5%): **WARNING — better rate available.** Report: current protocol, current APY, better protocol, better APY, difference in bps.
- If no: pass.

## Check 3 — Morpho market health

For any position currently in a Morpho market, evaluate:

- **Utilization rate**: if > 95%, report **WARNING — Morpho market overutilized.** Withdrawal liquidity may be thin.
- **LLTV changes**: if the market's LLTV has increased since last check (or is above the risk tier threshold from `config.morphoRiskTier`), report **WARNING — Morpho market risk degraded.** The market may no longer fit the vault's risk profile.
- **Collateral quality**: if the underlying collateral has depegged or experienced unusual volatility, report **WARNING — collateral risk.**

If no Morpho positions exist, skip this check.

## Check 4 — All healthy

If none of the above checks triggered a WARNING:

**HEALTHY — all positions are optimally deployed and within risk parameters.**

---

Report format: list each check result (PASS or WARNING with details), then a final status line: either `HEALTHY` or `WARNING` with a summary of issues found.
