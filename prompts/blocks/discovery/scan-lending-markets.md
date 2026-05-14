# block: discovery/scan-lending-markets
**Responsibility:** Fetch live APYs across whitelisted lending protocols, apply risk gates, detect market-level APY anomalies against the previous cycle, and surface the single best clean market. Owns its own inter-cycle state (`scan_markets`). No position awareness. No decisions. No execution.

---

## Injected inputs — read from `strategyConfig` in your system prompt

| Field | Type | Default | Description |
|---|---|---|---|
| `protocols` | string[] | — | **Required.** Protocols to scan. Valid values: `"aave"`, `"compoundV3"`, `"morpho"`. Any protocol not in this list is silently skipped. |
| `token` | string | — | **Required.** Asset symbol to scan for (e.g. `"USDC"`, `"USDT"`, `"WETH"`). Used as the DefiLlama `symbol` filter and the Morpho `asset` param. |
| `chain` | string | — | **Required.** Chain name as used by DefiLlama (e.g. `"Base"`, `"Arbitrum"`). Determines Morpho availability. |
| `riskGates.minMarketSizeUsd` | number | `10000000` | Minimum gross supply size (USD) for Aave V3 and Compound V3. Use `totalSupplyUsd` from DefiLlama if present; else `tvlUsd`. |
| `riskGates.maxApyBaseBps` | number | `1500` | Maximum allowed `apyBaseBps` across all protocols. Markets above this are dropped. |
| `riskGates.morpho.minSupplyAssetsUsd` | number | `5000000` | Minimum `supplyAssetsUsd` for Morpho Blue markets. |
| `riskGates.morpho.maxLltvPct` | number | `86` | Maximum allowed `lltvPct` for Morpho Blue markets. |
| `riskGates.morpho.maxUtilizationBps` | number | `9200` | Maximum allowed `utilizationBps` for Morpho Blue markets. |
| `riskGates.morpho.collateralAllowlist` | string[] | `["WETH","wstETH","cbETH","cbBTC","weETH","WBTC","ezETH","USDC","USDT","USDS"]` | Exact case-sensitive symbols allowed as Morpho collateral. Candidates not in this list are dropped silently. |
| `riskGates.anomalySpikeBps` | number | `300` | An eligible market is tagged `anomalous` if its `apyBaseBps` this cycle exceeds the prior cycle's stored `anomalyCeiling` (= prior `apyBaseBps + anomalySpikeBps`). |

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

3. **No arithmetic.** Bps conversions use string manipulation only (see Step 2 — APY conversion). The anomaly ceiling is computed once at Step 8 as a single integer addition (`apyBaseBps + anomalySpikeBps`) and stored per market. All other comparisons are ordinal.

4. **Morpho Blue is only available on Base.** If `chain != "Base"`, skip Morpho regardless of `protocols`. Write `SKIPPED (chain: <chain>)` in SCAN RESULTS.

5. **No position awareness.** Do not call `factor_vault_analytics`, do not read current protocol or current APY from context. That belongs to `scan-position`.

6. **No decisions, no execution, no `sign_and_send`.** Only `read_cycle_state` (Step 1) and `write_cycle_state` (Step 8) are permitted write-adjacent calls. All other writes are forbidden.

7. **Do not evaluate any protocol outside of Aave V3, Compound V3, and Morpho Blue.** Ignore all other results from any tool regardless of APY.

8. **If `supplyApy` is null for a Morpho candidate, drop it silently.** Do not estimate or substitute.

9. **`best_eligible` is selected from `clean` markets only.** Anomalous and new markets are never surfaced as `best_eligible`. If no clean markets exist, `best_eligible.protocol = "NONE"`.

---

## Step 1 — Read config and previous cycle state

Hardcoded values for this cycle:
- `protocols` = `["aave", "compoundV3", "morpho"]`
- `token` = `"USDC"`
- `chain` = `"Base"`
- `riskGates.minMarketSizeUsd` = `10000000`
- `riskGates.maxApyBaseBps` = `1500`
- `riskGates.morpho.minSupplyAssetsUsd` = `5000000`
- `riskGates.morpho.maxLltvPct` = `86`
- `riskGates.morpho.maxUtilizationBps` = `9200`
- `riskGates.anomalySpikeBps` = `300`
- `riskGates.morpho.collateralAllowlist` = `["WETH","wstETH","cbETH","cbBTC","weETH","WBTC","ezETH","USDC","USDT","USDS"]`

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

DefiLlama returns `apyBase` as a raw percentage (e.g. `"3.25944"` = 3.25944%). `factor_get_morpho_markets` returns `supplyApy` and `utilization` as raw percentages likewise. Convert all percentage values to bps using **string manipulation only — no arithmetic**:

> Split the value string on `"."`.
> - No `"."` present: left = full string, right = `"00"`.
> - `"."` present: left = part before, right = part after.
> - Take first 2 characters of `right`. If `right` has only 1 character, append `"0"`.
> - Result = left + those 2 characters. Truncate — do not round.
>
> `"4.9630"` → `"4"+"96"` = `496` | `"3.99"` → `399` | `"10.5"` → `1050` | `"86"` → `8600`

Apply to these fields and store under the target names below — one field at a time:
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

**M1a — Allowlist filter.** Keep only records where `collateralSymbol` exactly matches (case-sensitive) one of: `WETH`, `wstETH`, `cbETH`, `cbBTC`, `weETH`, `WBTC`, `ezETH`, `USDC`, `USDT`, `USDS`. Drop all others silently.

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

A market failing any gate is dropped silently.

**Aave V3 and Compound V3:**
- `marketSizeUsd ≥ 10000000` ($10M)
- `apyBaseBps` is not null
- `apyBaseBps ≤ 1500`

**Morpho Blue:**
- `supplyAssetsUsd ≥ 5000000` ($5M)
- `apyBaseBps ≤ 1500`
- `lltvPct ≤ 86`
- `utilizationBps < 9200`
- Any null/missing value on a Morpho gate = gate fail.

---

## Step 5 — Tag markets against previous cycle

For each market in `eligibleMarketsSnapshot`, assign a `tag`:

**If `previousScanState == null` (first cycle):** tag all markets as `new`.

**Otherwise:** match each market against `previousScanState.markets` by `protocol` + `market` identifier.
- **Match found:** retrieve the matched entry from `previousScanState.markets`. The stored `anomalyCeiling` equals that entry's `apyBaseBps + anomalySpikeBps`, pre-computed during the prior cycle's Step 8 — it is the upper bound above which the current APY represents a spike relative to the prior cycle's value.
  - `currentApyBaseBps > matchedEntry.anomalyCeiling` → tag `anomalous`
  - Otherwise → tag `clean`
- **No match:** tag `new`

---

## Step 6 — Select `best_eligible`

From `eligibleMarketsSnapshot`, consider only markets tagged `clean`.

Pick the market with the highest `apyBaseBps` among clean markets. If multiple clean markets share the same highest `apyBaseBps`, prefer in this order: `aave` → `compoundV3` → `morpho`.

If no clean markets exist → `best_eligible.protocol = "NONE"`.

---

## Step 7 — Write ELIGIBLE MARKETS

```
ELIGIBLE MARKETS:
  best_eligible:
    protocol=<aave|compoundV3|morpho|NONE>
    market=<marketId|baseAssetAddress|null>
    receiptToken=<address|null>
    collateralToken=<address|null>
    collateralSymbol=<symbol|null>
    apyBaseBps=<bps|null>
  all:
    - protocol=<aave|compoundV3|morpho>, market=<marketId|baseAssetAddress|null>, receiptToken=<address|null>, collateralToken=<address|null>, collateralSymbol=<symbol|null>, apyBaseBps=<bps>, tag=<clean|anomalous|new>
    (or: NONE)
```

Field rules:
- `market`: Morpho → bytes32 `marketId`. Compound V3 → `baseAssetAddress`. Aave V3 → `null`.
- `receiptToken`: `aToken` address for Aave. `null` for Compound V3 (no receipt ERC-20 — Comet address is in `market`) and Morpho.
- `collateralToken`: Morpho only. `null` for Aave and Compound.
- `collateralSymbol`: Morpho only. `null` for Aave and Compound.

---

## Step 8 — Persist cycle state and emit final output

For each market in `eligibleMarketsSnapshot`, compute `anomalyCeiling = apyBaseBps + 300` (single integer addition).

Call `write_cycle_state` to persist the state for the next cycle's anomaly comparison:

```
write_cycle_state {
  stage: "scan_markets",
  state: {
    "markets": [
      { "protocol": "<aave|compoundV3|morpho>", "market": "<marketId|baseAssetAddress|null>", "apyBaseBps": <bps>, "anomalyCeiling": <bps> }
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
    "receiptToken": "<address|null>",
    "collateralToken": "<address|null>",
    "collateralSymbol": "<string|null>",
    "apyBaseBps": <bps|null>
  },
  "all": [
    { "protocol": "<aave|compoundV3|morpho>", "market": "<marketId|baseAssetAddress|null>", "receiptToken": "<address|null>", "collateralToken": "<address|null>", "collateralSymbol": "<string|null>", "apyBaseBps": <bps>, "tag": "<clean|anomalous|new>" }
  ]
}
```

The orchestrator reads this as `{{previous}}` for downstream stages. It must be valid JSON regardless of outcome — on any error, emit `{ "error": "<description>", "best_eligible": { "protocol": "NONE" }, "all": [] }`.
