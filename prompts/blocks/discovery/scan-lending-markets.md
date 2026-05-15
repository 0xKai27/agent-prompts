{/* blocks/discovery/scan-lending-markets.md — v1.0.0 */}

# block: discovery/scan-lending-markets
**Responsibility:** Fetch live APYs across whitelisted lending protocols, apply risk gates, detect market-level APY anomalies against the previous cycle, and surface the single best clean market. Owns its own inter-cycle state (`scan_markets`). No position awareness. No decisions. No execution.

---

## Injected inputs — read from `strategyConfig` in your system prompt

| Field | Type | Default | Description |
|---|---|---|---|
| `protocols` | string[] | — | **Required.** Protocols to scan. Valid values: `"aave"`, `"compoundV3"`, `"morpho"`. Any protocol not in this list is silently skipped. |
| `token` | string | — | **Required.** Asset symbol to scan for (e.g. `"USDC"`, `"USDT"`, `"WETH"`). Used as the DefiLlama `symbol` filter and the Morpho `asset` param. |
| `chain` | string | — | **Required.** Chain name as used by DefiLlama (e.g. `"Base"`, `"Arbitrum"`). Determines Morpho availability. |
| `riskGates.aave.minMarketSizeUsd` | number | `25000000` | Minimum gross supply size (USD) for Aave V3. |
| `riskGates.compound.minMarketSizeUsd` | number | `10000000` | Minimum gross supply size (USD) for Compound V3. |
| `riskGates.maxApyBaseBps` | number | `1500` | Maximum allowed `apyBaseBps` across all protocols. Markets above this are dropped. |
| `riskGates.morpho.minSupplyAssetsUsd` | number | `10000000` | Minimum `supplyAssetsUsd` for Morpho Blue markets. |
| `riskGates.morpho.maxLltvPct` | number | `94.5` | Maximum allowed `lltvPct` for Morpho Blue markets. |
| `riskGates.morpho.maxUtilizationBps` | number | `9200` | Maximum allowed `utilizationBps` for Morpho Blue markets. |
| `riskGates.morpho.collateralAllowlist` | string[] | `["WETH","wstETH","cbETH","cbBTC","weETH","WBTC","ezETH","USDC","USDT","USDS"]` | Exact case-sensitive symbols allowed as Morpho collateral. Candidates not in this list are dropped silently. |
| `riskGates.anomalySpikeBps` | number | `300` | An eligible market is tagged `anomalous` if its `apyBaseBps` this cycle exceeds the prior cycle's stored `anomaly_ceiling` (= prior `apy_base_bps + anomalySpikeBps`). |

If a `riskGates` field is absent from `strategyConfig`, use the default shown above. Never infer or substitute values from context.

A required field (`protocols`, `token`, `chain`) absent from `strategyConfig` is a misconfiguration. Emit `{ "error": "misconfiguration", "missing": ["<field>"] }` as the final JSON line and stop.

> **Testing note:** The logic below uses hardcoded values from the Balanced Lending v17 strategy (Base, USDC, all three protocols). The injected inputs table above is preserved for future parameterisation.

---

## ⛔ Hard rules

1. **APY sources are fixed per protocol — do not mix them.**
   - Aave V3 APY → `defi_llama_yields` only.
   - Compound V3 APY → `defi_llama_yields` only.
   - Morpho APY, `supplyAssetsUsd`, `lltvPct`, `utilization` → `factor_get_morpho_markets` only. Never call `defi_llama_yields` for Morpho.

2. **`factor_get_lending_tokens` is always called unconditionally** for every protocol in `protocols` — it is the sole source of receipt token and collateral token addresses. Do not rely on DefiLlama for addresses.

3. **No inline arithmetic.** All numeric calculations use `math_calculate`. Bps conversions call `math_calculate` (see Step 2 — APY conversion). The anomaly ceiling is computed at Step 8 via `math_calculate`. All other comparisons are ordinal.

4. **Morpho Blue is only available on Base.** If `chain != "Base"`, skip Morpho regardless of `protocols`. Write `SKIPPED (chain: <chain>)` in SCAN RESULTS.

5. **No position awareness.** Do not call `factor_vault_analytics`, do not read current protocol or current APY from context. That belongs to `scan-position`.

6. **No decisions, no execution, no `sign_and_send`.** Only `read_cycle_state` (Step 1) and `write_cycle_state` (Step 8) are permitted write-adjacent calls. All other writes are forbidden.

7. **Do not evaluate any protocol outside of Aave V3, Compound V3, and Morpho Blue.** Ignore all other results from any tool regardless of APY.

8. **If `supplyApy` is null for a Morpho candidate, drop it silently.** Do not estimate or substitute.

9. **`best_eligible` is selected from `clean` markets only.** Anomalous and new markets are never surfaced as `best_eligible`. If no clean markets exist, `best_eligible.protocol = "NONE"`.

---

## Step 1 — Read config and previous cycle state

Apply the default values from the injected inputs table above.

Call `read_cycle_state` to load the previous cycle's state:

```
read_cycle_state { stage: "scan_markets" }
```

The tool always returns `{ ok: true, state: <blob> | null }`.
- `state` is a JSON object → assign it as `previousScanState`.
- `state` is `null` → `previousScanState = null` (first cycle — tag all markets as `new` in Step 5).

---

## Step 2 — Fetch APYs

**DefiLlama project name mapping (fixed — do not substitute):**

| `protocols` value | DefiLlama `project` param |
|---|---|
| `"aave"` | `"aave-v3"` |
| `"compoundV3"` | `"compound-v3"` |

---

### APY percentage → bps conversion

DefiLlama returns `apyBase` as a raw percentage (e.g. `3.25944` = 3.25944%). `factor_get_morpho_markets` returns `supplyApy` and `utilization` as raw percentages likewise. Convert each value to bps using `math_calculate` — one field at a time, truncating (do not round):

math_calculate(operation: "multiply", operands: [<value>, 100], decimalPlaces: 0)

Apply to these fields and store under the target names below:
- Aave V3 / Compound V3: `apyBase` → `apyBaseBps`
- Morpho Blue: `supplyApy` → `apyBaseBps`
- Morpho Blue: `utilization` → `utilizationBps`

---

### Aave V3

```
defi_llama_yields { chain: "Base", project: "aave-v3", stablecoin: true }
factor_get_lending_tokens { protocol: "aave" }
```

From `defi_llama_yields`: keep rows where `symbol` contains `"USDC"`. Extract:
- `marketSizeUsd` = `totalSupplyUsd` if not null, else `tvlUsd`
- Apply the bps split defined in **§ APY percentage → bps conversion** above: `apyBase` → `apyBaseBps`.

From `factor_get_lending_tokens`: extract `aToken` address by matching `underlyingSymbol == "USDC"`. Record unconditionally — do not rely on DefiLlama for addresses.

---

### Compound V3

```
defi_llama_yields { chain: "Base", project: "compound-v3", stablecoin: true }
factor_get_lending_tokens { protocol: "compoundV3" }
```

From `defi_llama_yields`: keep rows where `symbol` contains `"USDC"`. Extract:
- `marketSizeUsd` = `totalSupplyUsd` if not null, else `tvlUsd`
- Apply the bps split defined in **§ APY percentage → bps conversion** above: `apyBase` → `apyBaseBps`.

From `factor_get_lending_tokens`: extract `baseAssetAddress` (Comet contract address) unconditionally. Compound V3 has no receipt ERC-20 — do not look for a cToken.

---

### Morpho Blue (skip if `chain != "Base"`)

**Call M1 — market discovery:**
```
factor_get_morpho_markets { asset: "USDC" }
```
Fields used: `marketId`, `collateralSymbol`, `supplyAssetsUsd`, `supplyApy`, `utilization`, `lltvPct`.

**M1a — Allowlist filter.** Keep only records where `collateralSymbol` exactly matches (case-sensitive) one of the values in `collateralAllowlist`. Drop all others silently.

**M1b** — Drop any record where `supplyApy` is null.

**M1c** — Sort by `supplyAssetsUsd` descending. Take top 10.

**M1d — Apply the bps split defined in § APY percentage → bps conversion above, one field at a time:**
- `supplyApy` → `apyBaseBps`
- `utilization` → `utilizationBps`

**Call M2 — collateral address lookup:**
```
factor_get_lending_tokens { protocol: "morpho" }
```
For each M1 survivor, match `marketId` exactly to resolve `collateralToken` (address). No match → drop silently.

Each Morpho survivor now carries: `marketId`, `collateralSymbol`, `collateralToken`, `lltvPct`, `supplyAssetsUsd`, `apyBaseBps`, `utilizationBps`.

---

## Step 3 — Write SCAN RESULTS

```
SCAN RESULTS:
  aave:     apyBaseBps=<bps>, marketSizeUsd=<USD>, aToken=<address>  [PASS | FAIL: <reason> | SKIPPED]
  compound: apyBaseBps=<bps>, marketSizeUsd=<USD>, market=<baseAssetAddress>  [PASS | FAIL: <reason> | SKIPPED]
  morpho candidates (top 10, allowlist-filtered, apyBaseBps confirmed):
    - collateral=<symbol>, apyBaseBps=<bps>, supplyAssetsUsd=<USD>, utilizationBps=<bps>, marketId=<bytes32>, lltvPct=<pct>, collateralToken=<address>  [PASS | FAIL: <reason> | SKIPPED]
    (or: NO CANDIDATES — state which step produced zero results)
```

`SKIPPED` = protocol absent from `protocols`, or Morpho skipped due to chain. `FAIL: <reason>` = specific gate failure.

---

## Step 4 — Apply risk gates → build `eligibleMarketsSnapshot`

Initialise `failedMarketsSnapshot = []`. For each market that fails a gate, record the FIRST gate it fails and drop it. Markets that fail only due to a null/missing value are dropped silently with no entry in `failedMarketsSnapshot`.

**Aave V3** — evaluate in order, stop at first failure:
- `marketSizeUsd < 25000000` → add `{ protocol: "aave", market: null, reason: "tvl_floor" }` and drop
- `apyBaseBps` is null → drop silently
- `apyBaseBps > 1500` → add `{ protocol: "aave", market: null, reason: "apy_ceiling" }` and drop
- Otherwise → eligible

**Compound V3** — evaluate in order, stop at first failure:
- `marketSizeUsd < 10000000` → add `{ protocol: "compoundV3", market: <baseAssetAddress>, reason: "tvl_floor" }` and drop
- `apyBaseBps` is null → drop silently
- `apyBaseBps > 1500` → add `{ protocol: "compoundV3", market: <baseAssetAddress>, reason: "apy_ceiling" }` and drop
- Otherwise → eligible

**Morpho Blue** — evaluate each candidate in order, stop at first failure:
- `supplyAssetsUsd < 10000000` → add `{ protocol: "morpho", market: <marketId>, reason: "tvl_floor" }` and drop
- `apyBaseBps > 1500` → add `{ protocol: "morpho", market: <marketId>, reason: "apy_ceiling" }` and drop
- `lltvPct > 94.5` → add `{ protocol: "morpho", market: <marketId>, reason: "lltv" }` and drop
- `utilizationBps >= 9200` → add `{ protocol: "morpho", market: <marketId>, reason: "utilization" }` and drop
- Any other null/missing value → drop silently
- Otherwise → eligible

---

## Step 5 — Tag markets against previous cycle

For each market in `eligibleMarketsSnapshot`, assign a `tag`:

**If `previousScanState == null` (first cycle):** tag all markets as `new`.

**Otherwise:** match each market against `previousScanState.markets` by `protocol` + `market` identifier.
- **Match found:** retrieve the matched entry from `previousScanState.markets`. The stored `anomaly_ceiling` equals that entry's `apy_base_bps + anomalySpikeBps`, pre-computed during the prior cycle's Step 8 — it is the upper bound above which the current APY represents a spike relative to the prior cycle's value.
  - `currentApyBaseBps > matchedEntry.anomaly_ceiling` → tag `anomalous`
  - Otherwise → tag `clean`
- **No match:** tag `new`

---

## Step 6 — Select `best_eligible`

**If `previousScanState == null` (first cycle):** select `best_eligible` from the full `eligibleMarketsSnapshot` — all markets will be tagged `new` but the anomaly filter does not apply on first cycle and deployment is required. Pick the market with the highest `apyBaseBps` across all eligible markets.

**Otherwise:** consider only markets tagged `clean`.

In both cases, if multiple markets share the highest `apyBaseBps`, prefer in this order: `aave` → `compoundV3` → `morpho`.

If no eligible markets exist → `best_eligible.protocol = "NONE"`.

---

## Step 7 — Write ELIGIBLE MARKETS

```
ELIGIBLE MARKETS:
  best_eligible:
    protocol=<aave|compoundV3|morpho|NONE>
    market=<marketId|baseAssetAddress|null>
    receipt_token=<address|null>
    collateral_token=<address|null>
    collateral_symbol=<symbol|null>
    apy_base_bps=<bps|null>
  all:
    - protocol=<aave|compoundV3|morpho>, market=<marketId|baseAssetAddress|null>, receipt_token=<address|null>, collateral_token=<address|null>, collateral_symbol=<symbol|null>, apy_base_bps=<bps>, tag=<clean|anomalous|new>
    (or: NONE)
```

Field rules:
- `market`: Morpho → bytes32 `marketId`. Compound V3 → `baseAssetAddress`. Aave V3 → `null`.
- `receipt_token`: `aToken` address for Aave. `null` for Compound V3 (no receipt ERC-20 — Comet address is in `market`) and Morpho.
- `collateral_token`: Morpho only. `null` for Aave and Compound.
- `collateral_symbol`: Morpho only. `null` for Aave and Compound.

---

## Step 8 — Persist cycle state and emit final output

For each market in `eligibleMarketsSnapshot`, compute `anomaly_ceiling` via `math_calculate`:

math_calculate(operation: "add", operands: [<apyBaseBps>, <anomalySpikeBps>])

Call `write_cycle_state` to persist the state for the next cycle's anomaly comparison:

```
write_cycle_state {
  stage: "scan_markets",
  state: {
    "markets": [
      { "protocol": "<aave|compoundV3|morpho>", "market": "<marketId|baseAssetAddress|null>", "apy_base_bps": <bps>, "anomaly_ceiling": <bps> }
    ]
  }
}
```

If `eligibleMarketsSnapshot` is empty, write `{ "markets": [] }` — do not omit the call.

The tool returns `{ ok: true }` on success. On error, log the error string in your summary and continue to emit the final JSON — the next cycle will treat `null` state as first-cycle and re-baseline.

**Emit the following JSON object as the absolute last line of your summary.** No prose after it.

```json
{
  "best_eligible": {
    "protocol": "<aave|compoundV3|morpho|NONE>",
    "market": "<marketId|baseAssetAddress|null>",
    "receipt_token": "<address|null>",
    "collateral_token": "<address|null>",
    "collateral_symbol": "<string|null>",
    "apy_base_bps": <bps|null>
  },
  "all": [
    { "protocol": "<aave|compoundV3|morpho>", "market": "<marketId|baseAssetAddress|null>", "receipt_token": "<address|null>", "collateral_token": "<address|null>", "collateral_symbol": "<string|null>", "apy_base_bps": <bps>, "tag": "<clean|anomalous|new>" }
  ],
  "failed": [
    { "protocol": "<aave|compoundV3|morpho>", "market": "<address|bytes32|null>", "reason": "<tvl_floor|apy_ceiling|lltv|utilization>" }
  ]
}
```

The orchestrator reads this as `{{previous}}` for downstream stages. It must be valid JSON regardless of outcome — on any error, emit `{ "error": "<description>", "best_eligible": { "protocol": "NONE" }, "all": [], "failed": [] }`.
