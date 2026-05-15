{/* blocks/discovery/scan-trading-signals.md — v1.0.0 */}

# block: discovery/scan-trading-signals
**Responsibility:** Surface pre-computed market indicators for the trading token, maintain previous-cycle signal state for crossover detection, and emit a structured signal snapshot for downstream decision stages. No thresholds. No decisions. No execution.

> **Multi-timeframe note:** This block assumes a 1H primary interval. All pre-computed signals (price, ATR, vol_ratio, candle anatomy) are derived from 1H bars by the executor. Support for 4H/1D primary intervals is tracked in MND-764 and will not require a breaking change to this block's output schema.

---

## Injected inputs — read from `strategyConfig` in your system prompt

| Field | Required | Default | Description |
|---|---|---|---|
| `tradingTokenSymbol` | Yes | — | Trading token symbol (e.g. `"WETH"`, `"cbBTC"`). Used only on the fallback path when the pre-computed block is unavailable. |

A missing `tradingTokenSymbol` with no pre-computed block is a misconfiguration. Emit `{ "error": "misconfiguration", "missing": ["tradingTokenSymbol"] }` as the final JSON line and stop.

---

## ⛔ Hard rules

1. **Read the pre-computed block first.** Do NOT call `binance_multi_timeframe`, `binance_technical_analysis`, or `tv_multi_timeframe` unless the system prompt explicitly shows "Market Indicators (unavailable this cycle)".

2. **No thresholds, no decisions.** This block surfaces signals verbatim. Crossover detection, regime filtering, and RSI guards belong in the `decisions/` block.

3. **No arithmetic.** Copy all numeric values exactly as shown in the pre-computed block. Do not round, scale, or re-derive.

4. **No execution.** Do not call `sign_and_send`, `compute_token_amount`, or any swap/lending tool.

5. **Always emit valid JSON as the last line**, regardless of outcome. On any failure, emit the error JSON structure and stop.

---

## Step 1 — Read previous cycle state

Call `read_cycle_state { stage: "scan_trading_signals" }`.

- `{ ok: true, state: null }` → first cycle. Set `previousSignals = null`. All `previous_cycle` fields in the output will be `null`.
- `{ ok: true, state: { ... } }` → store as `previousSignals`. You will use it in Step 3.
- Any other response → treat as first cycle. Proceed with `previousSignals = null`.

---

## Step 2 — Read current signals

### Primary path — pre-computed block (no tool call)

If the system prompt contains **"Market Indicators (pre-computed — treat as ground truth)"**, read the following fields verbatim:

```
price, rsi_1h, rsi_4h, rsi_daily, rsi_weekly, ema20_weekly,
atr_pct, atr_1h_pct, vol_ratio, body_pct_price, lower_wick_ratio,
is_green, is_red, at_lower_bb, regime
```

Do not call any tool. Proceed to Step 3.

### Fallback path — Binance (only when pre-computed block is unavailable)

If the system prompt shows **"Market Indicators (unavailable this cycle)"**:

```
binance_multi_timeframe { symbol: "<tradingTokenSymbol>USDT" }
```

Map the response to the signal set. Fields not returned by this tool (`vol_ratio`, `atr_pct`, `atr_1h_pct`, `body_pct_price`, `lower_wick_ratio`, `is_green`, `is_red`, `at_lower_bb`, `ema20_weekly`) are set to `null` in the output. The `regime` field, if not returned directly, is set to `"unavailable"`.

---

## Step 3 — Write SIGNAL SNAPSHOT

```
SIGNAL SNAPSHOT:
  source:          pre-computed | binance-fallback
  price:           <value>
  rsi_1h:          <value>
  rsi_4h:          <value>
  rsi_daily:       <value>
  rsi_weekly:      <value>
  ema20_weekly:    <value>
  atr_pct:         <value>   (4h-scaled)
  atr_1h_pct:      <value>   (per 1h bar)
  vol_ratio:       <value>
  body_pct_price:  <value>
  lower_wick_ratio:<value>
  is_green:        <value>
  is_red:          <value>
  at_lower_bb:     <value>
  regime:          <value>

  previous_cycle:
    rsi_1h:    <value | null (first cycle)>
    rsi_4h:    <value | null>
    rsi_daily: <value | null>
    regime:    <value | null>
    vol_ratio: <value | null>
    price:     <value | null>
```

---

## Step 4 — Write cycle state and emit final output

Persist the current signals so the next cycle can read them as `previous_cycle`:

```
write_cycle_state {
  stage: "scan_trading_signals",
  state: {
    "rsi_1h":    <current rsi_1h>,
    "rsi_4h":    <current rsi_4h>,
    "rsi_daily": <current rsi_daily>,
    "regime":    "<current regime>",
    "vol_ratio": <current vol_ratio>,
    "price":     <current price>
  }
}
```

On error, log the error string and continue to emit the final JSON — the next cycle will treat `null` state as first-cycle and re-baseline.

**Emit the following JSON object as the absolute last line of your summary.** No prose after it.

```json
{
  "current": {
    "price": <number>,
    "rsi_1h": <number>,
    "rsi_4h": <number>,
    "rsi_daily": <number>,
    "rsi_weekly": <number>,
    "ema20_weekly": <number | null>,
    "atr_pct": <number | null>,
    "atr_1h_pct": <number | null>,
    "vol_ratio": <number | null>,
    "body_pct_price": <number | null>,
    "lower_wick_ratio": <number | null>,
    "is_green": <boolean | null>,
    "is_red": <boolean | null>,
    "at_lower_bb": <boolean | null>,
    "regime": "<bull|normal|caution|bear|unavailable>"
  },
  "previous_cycle": {
    "rsi_1h": <number | null>,
    "rsi_4h": <number | null>,
    "rsi_daily": <number | null>,
    "regime": "<string | null>",
    "vol_ratio": <number | null>,
    "price": <number | null>
  }
}
```

The `previous_cycle.rsi_1h` field is the `rsi_1h_prev` value for crossover detection in downstream decision blocks. It is `null` on the first cycle — decision blocks must handle `null` as "no prior signal, crossover cannot be confirmed this cycle."

On any unrecoverable error, emit: `{ "error": "<description>", "current": null, "previous_cycle": null }`.
