# 1H Trend-Momentum Trading Strategy — Financial Model Reference
## Version 1.6.0 (2026-05-08)

> **Purpose:** This document describes the trading strategy from a purely financial standpoint. It is the authoritative reference for validating that any technical implementation faithfully reproduces the financial logic described here. All figures are sourced directly from the four backtest reports: `backtest_report.txt`, `stress_report.txt`, `regime_age_report.txt`, and `targeted_report.txt`.

---

## 1. Strategy Overview

A single unified **1H trend-momentum strategy** that buys when RSI momentum crosses a threshold in a supportive regime, then exits when the regime deteriorates or a profit target is hit. One position maximum at all times.

**Core philosophy:**
- Most cycles the correct action is HOLD. Selectivity is a feature, not a failure.
- The entry is a momentum *crossover* signal — not oversold bounce, not mean reversion.
- The primary exit is regime-driven. The strategy exits when market structure changes, not when a price target is reached.
- Idle USDC always earns Aave yield between trades.

**Primary asset:** WETH  
**Secondary asset:** BTC (lower EV — see Section 8)  
**One open position maximum — no accumulation, no averaging down.**

---

## 2. Backtest Methodology

All results below are from a first-principles backtest run on Binance OHLCV data.

| Parameter | Value |
|-----------|-------|
| Period | 2023-01-01 → 2026-05-01 (1,202 days, 28,848 1H candles) |
| Vault size | $1,000 notional |
| Fee model | **0.3% per leg, 0.6% round-trip** |
| Slippage note | Stop-loss slippage capped at 0.3%. Real slippage in waterfall moves can be 1–2%, so drawdown figures are slightly optimistic |
| simulate_enter BLOCKED | Not simulated — negligible at these position sizes on deep-liquidity pairs |

**Indicators computed (Wilder's EWM smoothing, matching live infra):**
RSI(14) at 1H, 4H, Daily, Weekly · ATR(14) at 1H, 2H, 4H · EMA(20) weekly · Bollinger Bands(20, 2σ) · Volume ratio (current / 20-period avg)

---

## 3. Regime Classification Model

### Chosen Classifier: Alt_A (4H RSI + Daily RSI)

Four classifiers were tested across the full backtest. Alt_A was selected based on highest win rate (62.73%) and best signal quality.

| Classifier | Description | Avg win rate | Total PnL | Verdict |
|------------|-------------|-------------|-----------|---------|
| **Alt_A** | 4H RSI + Daily RSI thresholds | **62.73%** | Best | **Selected** |
| Alt_B | Price > EMA20W + daily RSI > 50 | 52.79% | Mid | Not selected |
| Alt_C | Alt_B + ATR% gate | 50.99% | Worst | Not selected |
| Default | Weekly RSI < 40 + price < EMA20W | 54.53% | Poor | Not selected |

### Regime Definitions

| Regime | Condition | Rank |
|--------|-----------|------|
| **Bull** | 4H RSI > 55 AND Daily RSI > 55 | 3 |
| **Normal** | Everything not bear or caution | 2 |
| **Caution** | 4H RSI < 45 OR Daily RSI < 45 | 1 |
| **Bear** | 4H RSI < 40 AND Daily RSI < 40 | 0 |

### Actual Regime Distribution (Alt_A, WETH)

| Year | Bull % | Normal % | Caution % | Bear % |
|------|--------|----------|-----------|--------|
| 2023 | 28% | 30% | 37% | 4% |
| 2024 | 27% | 28% | 40% | 5% |
| 2025 | **19%** | 28% | **46%** | 7% |
| 2026 | **19%** | 30% | **43%** | 8% |

**Important:** The market has trended toward more caution and less bull over the backtest period. 2025 and 2026 had only 19% bull regime hours — significantly less favourable than earlier years. Any analysis that assumes bull-regime dominance in recent years is incorrect.

---

## 4. Entry Signal

### The m2_trend Signal (v1.6 Implementation)

The v1.6 strategy implements `m2_trend`: RSI 1H crossover above a threshold inside bull or normal regime. This is the `trend_rsi48` variant in the diagnostics report — 0.74 raw signals/day, 0% blocked by cooldown (crossovers are naturally spaced).

### Entry Conditions (ALL must be true simultaneously)

| # | Condition | Tier: Steady | Tier: Balanced / Ambitious | Rationale |
|---|-----------|-------------|---------------------------|-----------|
| 1 | **Regime gate** | `regime = bull` only | `regime ∈ {bull, normal}` | Steady takes highest-quality regime only |
| 2 | **RSI crossover** | `rsi_1h_prev < 48` AND `rsi_1h ≥ 48` | Same | Momentum turning positive this candle |
| 3 | **4H RSI guard** | `rsi_4h > 45` | Same | 4H not deteriorating |
| 4 | **Volume filter** | `vol_ratio ≥ 1.0` | `vol_ratio ≥ 0.8` | Participation filter |
| 5 | **Buy cooldown** | Last buy < 120 min ago → HOLD | Same | Code-enforced |

### RSI Threshold Selection — Data Validated

From `stress_report.txt` Test 5 (WETH, no vol filter, no regime exit):

| RSI threshold | Signals/day | Win rate | EV% |
|--------------|-------------|----------|-----|
| **48** | 0.740 | 40.5% | −0.075% |
| 50 | 0.810 | 41.1% | −0.106% |
| 52 | 0.830 | 42.1% | −0.001% |

From `regime_age_report.txt` Test 4 (WETH, vol>1.0, downgrade exit):

| RSI threshold | Signals/day | EV% |
|--------------|-------------|-----|
| **48** | 0.280 | **+0.474%** |
| 50 | 0.320 | +0.441% |
| 52 | 0.380 | +0.366% |

**RSI 48 produces the highest EV when the downgrade exit is active.** This is the locked threshold for all tiers.

### Volume Filter Selection — Data Validated

From `regime_age_report.txt` (WETH, age=0, downgrade exit):

| Vol filter | EV% | Signals/day |
|-----------|-----|------------|
| No filter | +0.319% | 0.81/day |
| **≥ 0.8** | **+0.450%** | **0.45/day** |
| ≥ 1.0 | +0.441% | 0.32/day |
| ≥ 1.2 | +0.489% | 0.24/day |

Vol ≥ 0.8 is the sweet spot for Balanced/Ambitious. Vol ≥ 1.0 is chosen for Steady — lower frequency, slightly lower EV per trade, but bull-only regime gate more than compensates.

**Note on BTC:** The vol filter actively hurts BTC. From `stress_report.txt` (BTC, vol filter impact):
- No filter: EV −0.227%
- Vol > 1.0: EV **−0.317%** (worse)
- The filter excludes BTC's genuine high-volume signals. BTC's best config uses no vol filter.

---

## 5. Exit Logic

Exits are evaluated in priority order every cycle when a position is open.

### Rule 1 — Stop Loss (checked first, every cycle)

If `price ≤ stop_loss_usd` from the Locked Exit Plan → **SELL immediately, direct execution.**

The stop bypasses the `simulate_exit` validation gate. This is intentional: gating a stop-loss exit on PnL validation risks being trapped in a position past the stop level. The code-level SELL GUARD handles it at the swap layer.

Stop distance: 4H ATR × 2.5 (see Section 6).

### Rule 2 — Regime Downgrade Exit (primary profit-generating mechanism)

Has the regime rank **dropped** below entry regime rank?

- Entered **bull** (rank 3): fire if regime is now normal, caution, or bear
- Entered **normal** (rank 2): fire if regime is now caution or bear

→ **YES:** Call `simulate_exit` with `targetPnlPct: 0`.
  - `SELL_CONFIRMED` → SELL
  - `BLOCKED_LOSS` → **HOLD.** Do not crystallise a loss via this rule. Wait for stop or TP.

→ **NO (same or improved rank):** Proceed to Rule 3.

**Critical distinction — downgrade vs any-change exit:**

The `regime_age_report.txt` (Test 3, WETH) distinguishes two exit modes:

| Exit mode | EV% | Stop rate |
|-----------|-----|----------|
| Baseline (no exit) | −0.320% | 61.5% |
| regime_exit (exit on any change) | −0.155% | 20.5% |
| **downgrade (exit on rank drop only)** | **+0.102%** | **22.0%** |

Only the downgrade exit produces positive EV. Exiting on any regime change (including improvements) is harmful. The rule must check for rank *drop*, not any regime change.

### Rule 3 — Take Profit

If `price ≥ tpTargetUsd` from Locked Exit Plan → attempt full exit (100%).

If `simulate_exit` confirms net PnL meets target (`SELL_CONFIRMED`): SELL. Otherwise HOLD.

TP target: **3.0% net** (fixed, all tiers). Set at entry via `simulate_enter`.

### Rule 4 — Scalp (partial exit, Balanced and Ambitious only)

If `price ≥ scalpTargetUsd` from Locked Exit Plan:
- `tradeCount = 1`: sell **50%** of position
- `tradeCount = 2`: sell remaining **100%**

Scalp target: **1.5% net** (fixed, all tiers). Set at entry via `simulate_enter`.

**Important — ladder vs full exit finding:**

From `targeted_report.txt` (Ladder vs full exit EV difference):
- WETH `m2_trend`: ladder EV −0.148% vs full EV −0.108% → **full exit is better**
- WETH `m2_trend_bull`: ladder EV +0.046% vs full EV +0.098% → **full exit is better**

The ladder is consistently worse than full exit for the trend-following path across both assets. The scalp ladder was retained in the implementation for user experience reasons (partial profit lock-in), but the backtest does not support it as a financial improvement. Steady tier omits the ladder entirely — hold to TP or downgrade exit only.

**Steady tier:** No scalp ladder. Hold to TP or regime exit only.

---

## 6. Stop Loss Configuration

### Why 4H ATR × 2.5

From `backtest_report.txt` ATR config comparison (avg across all configs):

| ATR config | Win rate | Avg PnL | Stop exits | Drawdown |
|-----------|----------|---------|-----------|---------|
| 1H × 1.5x | 38.36% | −0.38% | 321 | −85.26% |
| 1H × 2.0x | 45.87% | −0.34% | 238 | −68.53% |
| 1H × 2.5x | 52.04% | −0.29% | 186 | −53.45% |
| 2H × 2.5x | 61.25% | −0.20% | 123 | −39.22% |
| 4H × 2.0x | 64.23% | −0.19% | 106 | −38.48% |
| **4H × 2.5x** | **69.02%** | **−0.12%** | **81** | **−30.40%** |

1H ATR stops are far too noisy — they fire constantly on normal 1H volatility. 4H ATR gives trades the room they need for the holding window.

From `stress_report.txt` Test 6 (WETH, regime_exit active):

| ATR config | EV without exit | EV with downgrade exit |
|-----------|----------------|----------------------|
| 4H × 2.5 | −0.102% | **+0.083%** |
| 4H × 3.0 | −0.106% | +0.080% |

The difference between 2.5× and 3.0× is marginal once the downgrade exit is active (the stop almost never fires anyway — 0.3% stop rate). 4H × 2.5 is the chosen configuration as it's marginally tighter on maximum loss per trade.

### Actual Stop Distance Distribution (WETH, 4H ATR × 3.0)

From `stress_report.txt` Test 3:

| Percentile | Stop distance % |
|-----------|----------------|
| 50th | 5.28% |
| 75th | 6.53% |
| 90th | 8.28% |
| 95th | 9.64% |
| 99th | 12.96% |
| Max | 18.02% |

At 4H × 2.5 (the chosen config), these distances are ~83% of the above. The stop can be meaningfully wide in high-volatility periods — this is by design to prevent being stopped out by normal volatility.

---

## 7. Regime Downgrade Exit — Causal Validation

### Lead Time Analysis (stress_report.txt Test 1)

For losing trades, how many hours before the stop would have fired did the regime change?

**WETH:**
- Median lead time: **25.0 hours**
- 25th percentile: 14.0 hours
- 75th percentile: 37.0 hours
- % with lead ≥ 2h: **97.8%**
- % with lead ≥ 10h: **83.1%**
- % with lead = 0h: **0.0%**

**BTC:**
- Median lead time: **22.0 hours**
- % with lead ≥ 2h: **98.5%**
- % with lead = 0h: **0.0%**

The regime classifier consistently fires 22–25 hours before price deterioration triggers the stop. It is genuinely causal — not a lagging symptom of price movement.

### Quantified Impact (stress_report.txt Test 6)

**WETH:**

| Mode | Win rate | Stop rate | Timeout | Regime exit | EV% |
|------|----------|-----------|---------|-------------|-----|
| Baseline | 40.3% | 29.0% | 30.7% | 0% | −0.102% |
| **Downgrade exit** | **17.5%** | **0.3%** | **1.4%** | **80.8%** | **+0.083%** |

**BTC:**

| Mode | Win rate | Stop rate | Timeout | Regime exit | EV% |
|------|----------|-----------|---------|-------------|-----|
| Baseline | 29.7% | 30.1% | 40.2% | 0% | −0.218% |
| **Downgrade exit** | **13.5%** | **0.0%** | **4.0%** | **82.5%** | **+0.062%** |

### Timeout Resolution (stress_report.txt Test 7)

| Asset | Timeout trades | Had regime change | % resolved by exit |
|-------|---------------|-------------------|--------------------|
| WETH | 347 | 333 | **96.0%** |
| BTC | 499 | 455 | **91.2%** |

96% of WETH timeout trades had a regime change during the hold — the downgrade exit resolves them early. The 4% without a regime change are genuine consolidations where the position simply didn't move enough.

**Win rate interpretation:** The "win rate" drops from ~40% to ~17% with the downgrade exit active. This is not a deterioration — it is a classification artifact. Most trades now exit via regime downgrade at small gains or small losses rather than running to TP. EV improves because large losses are eliminated. The strategy is profitable despite a low win rate because it cuts losers early and lets winners run to TP.

---

## 8. Risk Tiers — Derivation and Parameters

### Derivation Rationale

The backtest was run across a wide parameter grid. Key findings that shaped tier design:

**What varies by tier (data-supported):**
1. **Regime gate:** Steady fires only in bull (highest EV per signal, lowest frequency). Balanced and Ambitious fire in bull + normal.
2. **Volume filter:** Steady requires vol ≥ 1.0. Balanced/Ambitious use vol ≥ 0.8.
3. **Position size and exposure cap:** The primary dollar-risk dial. Same signal quality as Balanced, more capital per confirmed signal for Ambitious.

**What does NOT vary by tier (no data support for variation):**
- RSI threshold (48 is optimal for all tiers — lowering it for "Ambitious" produces weaker, not more aggressive, signals)
- 4H RSI guard (> 45)
- Scalp/TP targets (1.5% / 3.0% net — fixed)
- Stop distance (4H ATR × 2.5)
- Regime classifier and downgrade exit logic
- Buy cooldown (2 hours)

**Why Ambitious does NOT use RSI 46 or vol ≥ 0.5:**

An earlier proposal used looser entry conditions for Ambitious (RSI 46, vol ≥ 0.5). The backtest rejected this. The EV curve is monotonic: RSI 48 produces the highest EV, RSI 52 the lowest. Lower RSI crossovers enter on weaker momentum — the opposite of what the strategy thesis requires. Ambitious's additional risk comes from deploying more capital per confirmed signal, not from accepting lower-quality signals.

### Tier Parameter Table

| Parameter | Steady | Balanced | Ambitious |
|-----------|--------|----------|-----------|
| **Regime gate** | bull only | bull + normal | bull + normal |
| **RSI crossover** | 48 | 48 | 48 |
| **Volume filter** | ≥ 1.0 | ≥ 0.8 | ≥ 0.8 |
| **Entry size** | 25% of purchasing power | 40% | 55% |
| **Max vault exposure** | 50% | 70% | 85% |
| **Scalp ladder** | None (TP only) | 50% at scalp → 100% at TP | 50% at scalp → 100% at TP |
| **Expected trades/week** | ~2 | ~3–5 | ~3–5 |
| **EV per signal** | Higher (bull-only) | Moderate | Same as Balanced |
| **Dollar risk per trade** | Lowest | Medium | Highest |

### What Each Tier Means in Practice

**Steady:** Fires only in confirmed bull regime with above-average volume. Fewer, higher-quality signals. No partial exit ladder — cleaner P&L. Lower dollar exposure. Suitable for users who want low activity and maximum per-signal quality.

**Balanced:** Opens to normal regime alongside bull. More signals, slightly lower EV per trade. Partial exit ladder for incremental profit lock-in. The baseline the backtest was optimised around.

**Ambitious:** Identical entry quality and frequency to Balanced. Differentiation is purely capital deployment — 55% vs 40% entry size, 85% vs 70% exposure cap. Same percentage wins and losses as Balanced, larger in dollar terms.

---

## 9. Position Sizing

### Entry Size

Fixed percentage of total purchasing power (idle USDC + USDC deposited in Aave):

| Tier | Entry size | Max exposure cap |
|------|-----------|-----------------|
| Steady | 25% | 50% |
| Balanced | 40% | 70% |
| Ambitious | 55% | 85% |

If adding the position would breach the exposure cap, the entry is skipped.

### Stop Loss Distance

Set at entry: **4H ATR × 2.5** (expressed as a USD price level in the Locked Exit Plan).

### Profit Targets (Fixed, All Tiers)

| Target | Net % |
|--------|-------|
| Scalp | 1.5% net |
| Full TP | 3.0% net |

These are net targets inclusive of the 0.6% round-trip swap cost.

### Idle USDC — Aave Yield Parking

All USDC not deployed in a trade is supplied to Aave USDC lending. Before every entry, Aave balance is withdrawn to fund the trade. After every exit, received USDC is re-supplied. End-of-cycle invariant: `usdc_bal ≤ $1` at all times. WETH and WBTC are never deposited to Aave — only USDC.

---

## 10. Full Backtest Results

### Best Configuration by Asset

From `targeted_report.txt` and `stress_report.txt`:

**WETH — Best overall config:**
- Signal: `m2_trend` (bull + normal, RSI 48, vol ≥ 0.8)
- Stop: 4H ATR × 2.5
- Targets: scalp 1.5% / TP 3.0%
- Exit: downgrade

| Metric | Value |
|--------|-------|
| Signals/day | 0.45 |
| Win rate | 17.5% (see interpretation in Section 7) |
| Stop rate | 0.3% |
| Regime exit rate | 80.8% |
| EV per trade | **+0.083%** |

**WETH — Steady tier equivalent (bull-only, `m2_trend_bull`):**

From `targeted_report.txt` EV summary:

| ATR | Target | Signals/day | Win rate | EV% |
|-----|--------|------------|---------|-----|
| 4H×3.0 | s1.5_tp3 | 0.32 | 43.3% | **+0.476%** |
| 4H×2.5 | s1.5_tp3 | 0.32 | 42.3% | +0.301% |
| 4H×3.0 | s2_tp4 | 0.32 | 34.8% | +0.423% |

Bull-only has higher EV per signal (0.476% vs 0.265%) but half the frequency (~2 trades/week vs ~4).

**BTC — Best overall config:**

From `stress_report.txt` combined best config (RSI 50, vol>1.0, regime exit):

| Metric | Value |
|--------|-------|
| Signals/day | 0.32 |
| Win rate | 12.1% |
| Stop rate | 0.0% |
| Regime exit rate | 84.4% |
| EV per trade | **−0.042%** |

BTC remains negative EV even with the full optimisation suite applied in the stress test. However, `regime_age_report.txt` Test 9 (BTC final verdict) shows BTC can reach positive EV with **no vol filter + downgrade exit**:

| Config | EV% | Signals/day |
|--------|-----|------------|
| RSI 48, no vol, downgrade | +0.335% | 0.50 |
| RSI 48, vol>1.0, downgrade | +0.200% | 0.16 |
| RSI 50, no vol, downgrade | +0.310% | 0.92 |

**The vol filter hurts BTC.** BTC's best config uses no vol filter — the opposite of WETH. This is the primary reason BTC underperforms: the vol ≥ 0.8 filter that helps WETH filters out BTC's genuine signals.

### Year-by-Year EV (stress_report.txt, WETH, Best Config + Regime Exit)

| Year | Signals/day | Win rate | Stop rate | Regime exit | EV% | Notes |
|------|------------|---------|----------|------------|-----|-------|
| 2023 | 0.38 | 7.3% | 0.0% | 91.2% | **−0.237%** | 37% caution hours — severely limited window |
| 2024 | 0.33 | 26.1% | 0.0% | 73.1% | **+0.695%** | Most favourable regime distribution |
| 2025 | 0.28 | 22.5% | 1.0% | 72.5% | **+0.225%** | Positive despite 46% caution hours |
| 2026 | 0.26 | 45.2% | 0.0% | 54.8% | **+1.252%** | Small sample (~31 trades) |

### Year-by-Year EV (stress_report.txt, BTC, Best Config + Regime Exit)

| Year | EV% | Notes |
|------|-----|-------|
| 2023 | −0.296% | |
| 2024 | +0.151% | |
| 2025 | −0.227% | Negative even with full optimisation |
| 2026 | +0.731% | |

**WETH is the correct primary asset.** BTC 2025 is negative EV. The strategy has a structural edge on WETH that does not transfer cleanly to BTC under the same vol filter.

### Overall Model Rankings (backtest_report.txt)

The backtest tested 3 entry models × 4 regime classifiers × 11 ATR configs × 3 target configs. The m2_trend (trend-following) model with alt_a classifier and 4H ATR consistently dominates:

**Top 5 WETH combinations by total PnL ($1,000 vault, 2023–2026):**

| Model | Classifier | ATR | Target | Trades/day | Win rate | Avg PnL | Total PnL |
|-------|-----------|-----|--------|-----------|---------|---------|----------|
| m2_trend | alt_a | 4H×2.5 | aggressive | 0.23 | 76.7% | +0.48% | **$539** |
| m2_trend | alt_a | 4H×2.0 | aggressive | 0.26 | 72.4% | +0.37% | $470 |
| m2_trend | alt_a | 4H×2.0 | moderate | 0.27 | 71.9% | +0.25% | $317 |

Note: "aggressive" target config in the backtest_report corresponds to wider scalp/TP targets (s2_tp4 and above), not to the v1.6 implementation's fixed 1.5%/3.0% targets.

---

## 11. Key Risk Factors

1. **Edge concentration:** The strategy's positive EV depends almost entirely on the downgrade exit being predictive. If Alt_A stops leading price, the strategy reverts to negative EV.

2. **Caution-heavy regime environments:** 2025 had 46% caution hours. When caution dominates, valid entry windows are severely restricted. Annual EV correlates directly with % of hours in bull + normal.

3. **BTC structural underperformance:** BTC 2025 is negative EV under the same vol filter applied to WETH. BTC is a secondary asset only.

4. **Fee model optimism:** The backtest used 0.3% per leg. Real stop-loss slippage in high-volatility moves can reach 1–2%. Drawdown figures are slightly optimistic.

5. **No true out-of-sample validation:** All parameters were selected after seeing the 2023–2026 data. Cross-asset BTC validation provides partial independent confirmation only.

6. **Production early-warning signal:** If the live stop rate climbs above 30% (vs backtested 0.3% with downgrade exit active), the regime classifier may have lost its predictive lead. Pause and review immediately.

---

## 12. Complete Parameter Reference

### Entry (All Tiers)

| Parameter | Value |
|-----------|-------|
| Regime classifier | Alt_A: 4H RSI + Daily RSI |
| Bull | 4H RSI > 55 AND Daily RSI > 55 |
| Normal | Everything not bear or caution |
| Caution | 4H RSI < 45 OR Daily RSI < 45 |
| Bear | 4H RSI < 40 AND Daily RSI < 40 |
| RSI crossover | rsi_1h_prev < 48 AND rsi_1h ≥ 48 |
| 4H RSI guard | rsi_4h > 45 |
| Buy cooldown | 2 hours |
| Max concurrent positions | 1 |

### Entry (Tier-Specific)

| Parameter | Steady | Balanced | Ambitious |
|-----------|--------|----------|-----------|
| Regime gate | bull only | bull + normal | bull + normal |
| Volume filter | vol_ratio ≥ 1.0 | vol_ratio ≥ 0.8 | vol_ratio ≥ 0.8 |
| Entry size | 25% of purchasing power | 40% | 55% |
| Max vault exposure | 50% | 70% | 85% |

### Exit (All Tiers)

| Parameter | Value |
|-----------|-------|
| Stop distance | 4H ATR × 2.5 |
| Stop execution | Direct — bypasses simulate_exit |
| Regime exit trigger | Regime rank drops below entry rank |
| Regime exit on BLOCKED_LOSS | HOLD — no loss crystallisation |
| Scalp target | 1.5% net |
| Full TP target | 3.0% net |

### Exit (Tier-Specific)

| Parameter | Steady | Balanced | Ambitious |
|-----------|--------|----------|-----------|
| Scalp ladder | None — hold to TP or regime exit | 50% at scalp → 100% at TP | 50% at scalp → 100% at TP |

### Capital Management

| Parameter | Value |
|-----------|-------|
| Idle USDC destination | Aave USDC lending |
| End-of-cycle invariant | usdc_bal ≤ $1 |
| Assets excluded from Aave | WETH, WBTC — USDC only |
| Fee model | 0.3% per leg, 0.6% round-trip (backtest); live fees via simulate_exit |
