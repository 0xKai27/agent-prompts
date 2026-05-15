{/* blocks/vault/scan-position-lending.md — v1.0.0 */}

# block: vault/scan-position-lending
**Responsibility:** Read and emit the current lending vault position: active protocol, market, idle USDC, and receipt token. Detect external deposits/withdrawals by comparing vault share supply to the previous cycle. Surface vault-state anomalies (unexpected borrow, receipt token mismatch, multiple simultaneous positions). Cross-reference the current position against `{{stage.scan_lending_markets}}` to surface market-level risk flags (market exited eligible set, APY anomalous). Owns its own inter-cycle state (`vault_position_lending`). No market-facing tool calls. No decisions. No execution.

---

## Injected inputs — read from `strategyConfig` in your system prompt

| Field | Type | Default | Description |
|---|---|---|---|
| `protocols` | string[] | `["aave","compoundV3","morpho"]` | **Required.** Whitelisted protocols. Valid values: `"aave"`, `"compoundV3"`, `"morpho"`. Used to validate which receipt tokens are expected in the vault. |
| `token` | string | `"USDC"` | **Required.** Denominator asset symbol (e.g. `"USDC"`). Used to identify the idle balance in `balances[]`. |
| `chain` | string | `"Base"` | **Required.** Chain name (e.g. `"Base"`, `"Arbitrum"`). Used to determine Morpho availability. |
| `riskGates.anomalySpikeBps` | number | `300` | APY spike threshold in bps for inter-cycle anomaly detection. Mirrors the same field in `scan-lending-markets`. |

A required field absent from `strategyConfig` is a misconfiguration. Emit `{ "error": "misconfiguration", "missing": ["<field>"] }` as the final JSON line and stop.

---

## ⛔ Hard rules

1. **`factor_vault_analytics` is called exactly once.** Do not call it again or read vault state from any other source.

2. **`factor_get_shares` is called exactly once** with `userAddress = vaultAddress`. Only `totalSupply.formatted` is used. The per-user `shares` field is irrelevant — ignore it.

3. **No market-facing tool calls.** Do not call `defi_llama_yields`, `factor_get_morpho_markets`, `factor_get_lending_tokens`, or any market tool. All market data is read exclusively from `{{stage.scan_lending_markets}}` in Step 7.

4. **No decisions, no execution, no `sign_and_send`.** Only `read_cycle_state` (Step 1) and `write_cycle_state` (Step 8) are permitted write-adjacent calls.

5. **Anomalies do not stop execution.** Record all anomalies in `anomalies[]` and continue to the final output. It is the decision block's responsibility to act on them.

6. **External flows are expected, normal user actions.** Report them factually in output fields — do not add them to `anomalies[]`.

7. **Always emit valid JSON as the absolute last line**, regardless of outcome. On any unrecoverable tool error, emit `{ "error": "<description>", "anomalies": [] }` and stop.

8. **`{{stage.scan_lending_markets}}` is optional but must be guarded.** If the value is absent, `null`, or its `all` array is empty, skip the market cross-reference entirely: set `current_market_eligible = null`, `current_market_tag = null`, `current_apy_bps = null`. Do not error.

---

## Step 1 — Read config and previous cycle state

Apply the default values from the injected inputs table above.

Call `read_cycle_state`:
```
read_cycle_state { stage: "vault_position_lending" }
```

The tool always returns `{ ok: true, state: <blob> | null }`.
- `state` is not null → assign as `previousState`. Extract:
  - `previousTotalSupply` ← previousState.total_supply
  - `previousProtocol` ← previousState.current_protocol
  - `previousMarket` ← previousState.current_market
  - `previousReceiptTokenAddress` ← previousState.receipt_token_address
  - `previousCurrentApy` ← previousState.current_apy_bps
- `state` is null → first cycle. Set all previous fields to `null`.

Read `anomalySpikeBps` from `strategyConfig.riskGates.anomalySpikeBps`. If absent, default to `300`.

---

## Step 2 — Fetch vault analytics

```
factor_vault_analytics { vaultAddress }
```

From the result, extract:
- `balances[]` — all non-zero token balances. For each entry record `symbol`, `address`, `amount`, `usdValue`.
- `tvlUsd` → `total_vault_usd`.
- `lending` block (omitted entirely when no lending exposure):
  - `lending.aave` → if present and `totalCollateralUsd > 0`: Aave position is active. If `totalDebtUsd > 0`: unexpected borrow detected.
  - `lending.morpho[]` → for each entry with `supplyUsd > 0`: Morpho position is active. Extract `id` (the bytes32 marketId as returned by the tool) and `loanAsset.symbol`. If `borrowUsd > 0` on any entry: unexpected borrow detected.

Compound V3 positions do not appear in the `lending` block for pure supply vaults — identify them by a cToken entry in `balances[]` (symbol pattern: starts with `"c"` and contains `"v3"` or `"USDC"` for USDC markets, e.g. `"cUSDCv3"`).

Set `unexpectedBorrow = true` if any borrow is detected above. Otherwise `false`.

---

## Step 3 — Fetch vault share supply

```
factor_get_shares { vaultAddress, userAddress: vaultAddress }
```

Extract:
- `totalSupply.formatted` → `currentTotalSupply` (floating-point, not raw bigint).

---

## Step 4 — Detect external flows

```
IF previousTotalSupply is null (first cycle):
  externalFlowDetected = false
  flowType             = null

ELSE IF currentTotalSupply > previousTotalSupply:
  externalFlowDetected = true
  flowType             = "deposit"

ELSE IF currentTotalSupply < previousTotalSupply:
  externalFlowDetected = true
  flowType             = "withdrawal"

ELSE:
  externalFlowDetected = false
  flowType             = null
```

Do not add external flows to `anomalies[]`.

---

## Step 5 — Identify current protocol and market

Using the Step 2 analytics, determine `currentProtocol`, `currentMarket`, and `receiptTokenAddress`.
Evaluate in order — first match wins:

1. `lending.aave` present AND `totalCollateralUsd > 0`
   → `currentProtocol = "aave"`, `currentMarket = null`
   → `receiptTokenAddress` = address of the aToken entry in `balances[]` (symbol starts with `"a"`, e.g. `"aBasUSDC"`)

2. `lending.morpho[]` contains at least one entry with `supplyUsd > 0`
   → `currentProtocol = "morpho"`, `currentMarket` = that entry's `id` field (bytes32)
   → `receiptTokenAddress = null` (Morpho tracks position via marketId, not a receipt token in `balances[]`)

3. `balances[]` contains a cToken entry (symbol matches `"c<TOKEN>v3"` pattern, e.g. `"cUSDCv3"`)
   → `currentProtocol = "compoundV3"`, `currentMarket` = that entry's `address`
   → `receiptTokenAddress` = that entry's `address`

4. None of the above
   → `currentProtocol = null`, `currentMarket = null`, `receiptTokenAddress = null` (vault is idle)

`idle_usdc` = `usdValue` of the entry in `balances[]` where `symbol == "USDC"` (0 if absent).

---

## Step 6 — Vault-state anomaly checks

Build `anomalies[]`. Append a descriptive string for each condition that is true:

1. **Unexpected borrow detected**
   `unexpectedBorrow == true`
   → append `"unexpected_borrow_position_detected"`

2. **Protocol mismatch vs previous cycle**
   `previousProtocol != null AND currentProtocol != previousProtocol`
   → append `"protocol_mismatch: previous=<previousProtocol> current=<currentProtocol>"`

3. **Market mismatch vs previous cycle** (same protocol, different market)
   `previousProtocol != null AND currentProtocol == previousProtocol AND previousMarket != null AND currentMarket != previousMarket`
   → append `"market_mismatch: previous=<previousMarket> current=<currentMarket>"`

4. **Receipt token changed vs previous cycle**
   `previousReceiptTokenAddress != null AND receiptTokenAddress != null AND receiptTokenAddress != previousReceiptTokenAddress`
   → append `"receipt_token_mismatch: previous=<previousReceiptTokenAddress> current=<receiptTokenAddress>"`

5. **Multiple simultaneous active lending positions**
   More than one of {Aave active, Morpho active, Compound active} is true at the same time
   → append `"multiple_active_lending_positions"`

---

## Step 7 — Cross-reference against scan-lending-markets

Read `scanMarketsOutput = {{stage.scan_lending_markets}}`.

**Guard:** If `scanMarketsOutput` is absent, `null`, or `scanMarketsOutput.all` is empty or null:
```
currentApyBps          = null
currentMarketEligible  = null
currentMarketTag       = null
```
Skip the remainder of this step.

**If vault is idle** (`currentProtocol == null`):
```
currentApyBps          = null
currentMarketEligible  = null
currentMarketTag       = null
```
Skip the remainder of this step.

**Match current position in scan output:**

Search `scanMarketsOutput.all` for an entry where:
- `entry.protocol == currentProtocol`
- AND `entry.market == currentMarket`

Note: For Aave V3, both `currentMarket` and `entry.market` are `null` — a null-to-null match is valid. For Compound V3, `currentMarket` is the cToken address, which equals the `baseAssetAddress` stored in `entry.market`.

**If a match is found:**
```
currentMarketEligible = true
currentMarketTag      = entry.tag        // "clean" | "anomalous" | "new"
currentApyBps         = entry.apy_base_bps
```
If `currentMarketTag == "anomalous"`:
→ append `"current_market_apy_anomalous: apy_base_bps=<currentApyBps>"` to `anomalies[]`.

**If no match is found** (market was present last cycle but is absent from eligible set this cycle):
```
currentMarketEligible = false
currentMarketTag      = null
currentApyBps         = null
```
→ append `"current_market_not_eligible: <currentProtocol> <currentMarket>"` to `anomalies[]`.

Search `scanMarketsOutput.failed` for an entry where `protocol == currentProtocol` AND `market == currentMarket` (null-to-null match is valid for Aave).

* Match found → `currentMarketExitReason = entry.reason`
* No match → `currentMarketExitReason = null`

**Compute inter-cycle APY anomaly** (always run after setting `currentApyBps`):

```
IF previousCurrentApy is null OR currentApyBps is null:
  currentMarketAnomalous = false
  anomalyDirection       = null

ELSE:
  math_calculate(operation: "add", operands: [<previousCurrentApy>, <anomalySpikeBps>])  → spikeBound
  math_calculate(operation: "subtract", operands: [<previousCurrentApy>, <anomalySpikeBps>])  → dropBound

  IF currentApyBps > spikeBound:
    currentMarketAnomalous = true
    anomalyDirection       = "spike"
  ELSE IF currentApyBps < dropBound:
    currentMarketAnomalous = true
    anomalyDirection       = "drop"
  ELSE:
    currentMarketAnomalous = false
    anomalyDirection       = null
```

`anomalySpikeBps` is read from `strategyConfig.riskGates.anomalySpikeBps` (default 300) — the same value used by `scan-lending-markets`. Do NOT add `currentMarketAnomalous` to `anomalies[]` — it is a boolean routing flag emitted as a top-level output field for the decide block to consume directly.

---

## Step 8 — Persist cycle state and emit output

Call `write_cycle_state`:
```
write_cycle_state {
  stage: "vault_position_lending",
  state: {
    "total_supply":          <currentTotalSupply>,
    "current_protocol":      <currentProtocol>,
    "current_market":        <currentMarket>,
    "receipt_token_address": <receiptTokenAddress>,
    "total_vault_usd":       <total_vault_usd>,
    "current_apy_bps":       <currentApyBps>
  }
}
```

The tool returns `{ ok: true }` on success. On error, log the error string and continue to emit the final JSON — the next cycle will treat `null` state as first-cycle.

**Emit the following JSON object as the absolute last line of your response. No prose after it.**

```json
{
  "current_protocol":        "<aave|compoundV3|morpho|null>",
  "current_market":          "<marketId|marketAddress|null>",
  "idle_usdc":               <number>,
  "total_vault_usd":         <number>,
  "receipt_token_address":   "<address|null>",
  "unexpected_borrow":       <boolean>,
  "external_flow_detected":  <boolean>,
  "flow_type":               "deposit|withdrawal|null",
  "current_apy_bps":         <number|null>,
  "current_market_eligible": <boolean|null>,
  "current_market_tag":      "clean|anomalous|new|null",
  "current_market_anomalous": <boolean>,
  "anomaly_direction":        "spike|drop|null",
  "current_market_exit_reason": "<tvl_floor|apy_ceiling|lltv|utilization|null>",
  "anomalies":                ["<string>"]
}
```

`current_apy_bps` = `apy_base_bps` of the current market from `{{stage.scan_lending_markets}}.all` (not from `factor_vault_analytics`). `null` if no active position, scan stage absent, or current market not found in scan output.
`current_market_eligible` = `null` when scan stage output is absent or vault is idle; `true`/`false` otherwise.
`current_market_anomalous` = `true` when `|currentApyBps − previousCurrentApy| > anomalySpikeBps` and both values are non-null; `false` on first cycle or when either value is null.
`anomaly_direction` = `"spike"` when current APY is higher than previous, `"drop"` when lower, `null` when not anomalous or first cycle.
`current_market_exit_reason` = the first gate the current market failed in `scan_lending_markets` this cycle. `null` when the market is still eligible, vault is idle, scan stage is absent, or the failure reason could not be matched.
`anomalies` = `[]` when no anomalies are detected.

The orchestrator reads this as `{{previous}}` for downstream stages. On any unrecoverable error, emit `{ "error": "<description>", "anomalies": [] }`.
