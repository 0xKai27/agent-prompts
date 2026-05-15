{/* blocks/execute/lending/verify.md — v1.0.0 */}

# block: execute/lending/balanced/verify

**Responsibility:** Independent post-execution sanity check. Read the execute stage's output from `{{previous}}`, re-fetch live vault state via `factor_vault_analytics`, and confirm the live position is consistent with what execute reported. Thin check only — the goal is to catch hallucinated successes, not to re-verify every detail. Emits `verified` boolean and the user-facing `action_summary`.

---

## Input

Read exclusively from `{{previous}}` (the execute stage output). Required fields:

| Field | Type | Used for |
|---|---|---|
| `executed` | `boolean` | Gate — if false, skip all tool calls |
| `status` | `string` | Determines which live-state check to run |
| `decision` | `string` | Context for summary |
| `to_protocol` | `string\|null` | Expected live protocol after supply |
| `to_market` | `string\|null` | Expected live market after supply |
| `action_summary` | `string` | Passed through as `summary` on success |

Also reads `vaultAddress` from the system prompt.

**No `read_cycle_state` call.** Verify has no inter-cycle state.

---

## ⛔ Hard rules

1. **If `previous.executed == false`, emit output immediately — no tool calls.** This covers HOLD, HOLD-IDLE, HOLD-IDLE-FLAG, and SAFETY_HALT paths.
2. **One `factor_vault_analytics` call only.** Do not call any write tools, signing tools, or cycle state tools.
3. **`verified: true` means live state is consistent with `previous.status`.** It does not mean the execution was optimal or that every field matches exactly — only that the vault is not in a state that contradicts what execute reported.
4. **On any `factor_vault_analytics` error**, emit `verified: false` with `summary` explaining the tool failure. Do not retry.

---

## Step 1 — Gate on `executed`

If `previous.executed == false`:

```json
{ "verified": true, "summary": "<previous.action_summary>" }
```

Stop. No tool calls.

---

## Step 2 — Re-fetch live state

```
factor_vault_analytics { vaultAddress }
```

Derive from the response (same logic as execute.md Step 1):

| Name | Derivation |
|---|---|
| `live_protocol` | (1) `lending.aave` present AND `totalCollateralUsd > 0` → `"aave"`; (2) `lending.morpho[]` entry with `supplyUsd > 0` → `"morpho"`; (3) `positions[]` entry with `protocol === "compound"` → `"compoundV3"`; (4) none → `null` |
| `live_market` | Aave: `null`. Morpho: `lending.morpho[0].id`. Compound: cToken `.address` from `positions[]`. |
| `live_idle_usdc` | `stats.totalIdleUsd` |
| `positions[]` | All position entries — filter by `type == "credit" \| "supply"` to check for active lending positions |
| `total_vault_usd` | `tvlUsd` |

---

## Step 3 — Consistency check

Apply the check for `previous.status` and `previous.decision`:

### SUCCESS — REBALANCE-YIELD

Assert ALL of:
1. `live_protocol == previous.to_protocol`
2. `live_market == previous.to_market`
3. `live_idle_usdc ≤ dustThreshold` where `dustThreshold = min(5, max(0.01, total_vault_usd × 0.0005))` — funds considered fully deployed; small residuals from rounding are expected

- All pass → `verified: true`
- Protocol/market mismatch → `verified: false`, summary: `⚠️ Verification failed — vault shows [live_protocol]/[live_market] but execute reported move to [to_protocol]/[to_market].`
- Idle USDC exceeds threshold → `verified: false`, summary: `⚠️ Verification failed — vault shows [live_idle_usdc] USDC still idle after reported REBALANCE-YIELD success (dust threshold: [dustThreshold]).`

### SUCCESS — HOLD-TOPUP

Assert ALL of:
1. `live_protocol == previous.to_protocol`
2. `live_idle_usdc ≤ dustThreshold` where `dustThreshold = min(5, max(0.01, total_vault_usd × 0.0005))` — idle USDC considered fully deployed; small residuals from rounding are expected

- All pass → `verified: true`
- Protocol mismatch → `verified: false`, summary: `⚠️ Verification failed — vault protocol is [live_protocol], expected [to_protocol] after top-up.`
- Idle USDC exceeds threshold → `verified: false`, summary: `⚠️ Verification failed — vault shows [live_idle_usdc] USDC still idle after reported HOLD-TOPUP success (dust threshold: [dustThreshold]).`

### SUCCESS — RISK-EXIT

Assert no active lending position exists: `positions[]` filtered to `type == "credit" | "supply"` contains no entries with `valueUsd > 0` AND `live_idle_usdc > 0`.

- Consistent → `verified: true`
- Position still active → `verified: false`, summary: `⚠️ Verification failed — vault still holds an active position after reported RISK-EXIT success.`

### PARTIAL — REBALANCE-YIELD

Assert vault is idle: `live_protocol == null` OR no active receipt token balance. Withdrawal succeeded but supply failed — vault should be holding USDC.

- Idle → `verified: true`
- Active position → `verified: false`, summary: `⚠️ Verification failed — vault shows an active position; reported PARTIAL may be inaccurate.`

### PARTIAL_EXIT — RISK-EXIT

Assert at least one position was withdrawn: `live_idle_usdc > 0`. Some positions may still be active — that is consistent with PARTIAL_EXIT.

- Idle USDC > 0 → `verified: true`
- Zero idle USDC → `verified: false`, summary: `⚠️ Verification failed — no USDC recovered despite reported PARTIAL_EXIT.`

### ERROR — any decision

No specific assertion. `factor_vault_analytics` succeeded and the vault is reachable — that is sufficient.

- → `verified: true` (error was already reported by execute; vault is not in an unknown state)

---

## Step 4 — Emit output

Emit the following JSON object as the **absolute last line** of output. No prose after it.

```json
{
  "verified": <boolean>,
  "summary": "<string>"
}
```

`summary` is `previous.action_summary` when `verified: true`. When `verified: false`, it is the discrepancy message from Step 3.
