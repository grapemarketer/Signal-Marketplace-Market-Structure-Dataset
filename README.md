# Signal-Marketplace-Market-Structure-Dataset
Annotated BTC/USDT 15-minute market session data with labeled support/resistance levels, auction ranges, and structural candle events — plus a schema data dictionary and signal-weighting guide for AI-assisted market structure analysis.

# Market Context Data Dictionary

**Schema version:** 5
**File:** `sessions_combined.json`
**Format:** JSON array of session objects, sorted chronologically by `range.start_time`

---

## Overview

Each element in `sessions_combined.json` represents one annotated trading session for BTC/USDT on a 15-minute timeframe. A session contains two categories of data:

- **Objective data** — values derived mechanically from market feeds (price, volume, indicators). These are fixed and reproducible given the same source data and parameters.
- **Subjective data** — analyst annotations (levels, auction ranges, significant candles, interactions). These represent interpretive judgments and are not reproducible algorithmically.

The boundary between the two is deliberate: objective fields describe *what happened*; subjective fields describe *what it means*.

---

## Top-Level Fields

| Field | Type | Category | Description |
|---|---|---|---|
| `schema_version` | `integer` | Objective | Version of this schema. Always `5`. Used for forward-compatibility checks. |
| `title` | `string` | Objective | Descriptor for the data type. Always `"MarketContext15m"`. |
| `symbol` | `string` | Objective | Trading pair in exchange format. Always `"BTCUSDT"`. |
| `timeframe` | `string` | Objective | Candle interval. Always `"15m"`. |
| `range` | `object` | Objective | The UTC time window covered by this session. See [range](#range). |
| `indicator_params` | `object` | Objective | Parameters used to compute technical indicators. See [indicator_params](#indicator_params). |
| `candles` | `array` | Objective | Ordered list of 15-minute OHLCV candles with computed indicators. See [candles](#candles). |
| `manual_levels` | `object` | Subjective | Analyst-identified support and resistance price levels. See [manual_levels](#manual_levels). |
| `auction_ranges` | `object` | Subjective | Analyst-identified price ranges where the market was in a balanced, two-sided auction. See [auction_ranges](#auction_ranges). |
| `significant_candles` | `array` | Subjective | Candles flagged for notable market structure events. See [significant_candles](#significant_candles). Empty array `[]` when none identified. |
| `level_interactions` | `array` | Subjective | Annotated instances where price interacted meaningfully with a defined level. See [level_interactions](#level_interactions). Empty array `[]` when none recorded. |

---

## `range`

The session window. All candle timestamps fall within `[start_time, end_time)`.

| Field | Type | Description |
|---|---|---|
| `start_time` | `string` (ISO 8601) | UTC datetime of the first candle open. |
| `end_time` | `string` (ISO 8601) | UTC datetime of the last candle close (exclusive upper bound). |

---

## `indicator_params`

Parameters used to compute all indicator values in `candles`. Constant across all sessions.

| Field | Type | Description |
|---|---|---|
| `atr_length` | `integer` | Lookback period for Average True Range. Value: `14`. |
| `ema25_length` | `integer` | Period for the Exponential Moving Average. Value: `25`. |
| `rsi_length` | `integer` | Lookback period for the Relative Strength Index. Value: `14`. |

---

## `candles`

Ordered array (ascending by time) of 15-minute candle objects.

| Field | Type | Category | Description |
|---|---|---|---|
| `idx` | `integer` | Objective | Zero-based position of the candle within this session. The first candle is always `0`. |
| `t` | `string` (ISO 8601) | Objective | UTC open timestamp of the candle. |
| `o` | `float` | Objective | Open price (USD). |
| `h` | `float` | Objective | High price (USD). |
| `l` | `float` | Objective | Low price (USD). |
| `c` | `float` | Objective | Close price (USD). |
| `v` | `float` | Objective | Total traded volume (BTC) over the candle interval. |
| `volume_delta` | `float` | Objective | Net taker-initiated volume: taker buy volume minus taker sell volume, in BTC. Positive values indicate net buying aggression; negative values indicate net selling aggression. |
| `atr` | `float` | Objective | 14-period Average True Range at candle close, in USD. Measures recent volatility. |
| `ema25` | `float` | Objective | 25-period Exponential Moving Average of close prices at candle close, in USD. |
| `rsi` | `float` | Objective | 14-period Relative Strength Index at candle close. Range: 0–100. Values above 70 indicate overbought conditions; values below 30 indicate oversold conditions. |

> **Note on `volume_delta`:** This field was named `clv_volume` in pre-schema-v5 sessions. All sessions in this file have been standardised to `volume_delta`.

---

## `manual_levels`

Contains two arrays: `support` and `resistance`. Each entry represents a price level the analyst identified as a meaningful reference point based on price action and volume behaviour.

### `manual_levels.support[]`

A horizontal price level at which downward price movement previously stalled, reversed, or was absorbed. Implies potential future buying interest.

| Field | Type | Category | Description |
|---|---|---|---|
| `id` | `string` | Subjective | Unique identifier for this level within the session. Typically `"Support 1"`, `"Support 2"`, etc. |
| `label` | `string` | Subjective | Display label. Usually identical to `id`. |
| `price` | `float` | Subjective | The exact price (USD) that defines the level. Chosen by the analyst to represent the meaningful price boundary, often aligned to a candle high, low, or close. |
| `candle_idx_range` | `object` | Subjective | The candle range over which the level is considered active or relevant. See [candle_idx_range](#candle_idx_range-object). |
| `conviction` | `string` | Subjective | Analyst's confidence in the significance of this level. See [Conviction values](#conviction-values). |
| `notes` | `string` | Subjective | Free-text rationale explaining why this level was identified and what evidence supports it. Empty string when no annotation provided. |
| `level_converted_to_macro_auction_lower_bound` | `string` | Subjective | Records whether this support level was later used as the lower bound of a macro auction range. Format: `""` (undetermined), `"False"`, or `"True, {candle_idx}"` where `candle_idx` is the candle at which the conversion was established. |
| `level_converted_to_micro_auction_lower_bound` | `string` | Subjective | Same as above, for micro auction ranges. |

### `manual_levels.resistance[]`

A horizontal price level at which upward price movement previously stalled, reversed, or was distributed. Implies potential future selling interest.

| Field | Type | Category | Description |
|---|---|---|---|
| `id` | `string` | Subjective | Unique identifier. Typically `"Resistance 1"`, `"Resistance 2"`, etc. |
| `label` | `string` | Subjective | Display label. Usually identical to `id`. |
| `price` | `float` | Subjective | The exact price (USD) that defines the level. |
| `candle_idx_range` | `object` | Subjective | The candle range over which the level is considered active. See [candle_idx_range](#candle_idx_range-object). |
| `conviction` | `string` | Subjective | Analyst's confidence in the significance of this level. See [Conviction values](#conviction-values). |
| `notes` | `string` | Subjective | Free-text rationale. Empty string when no annotation provided. |
| `level_converted_to_macro_auction_upper_bound` | `string` | Subjective | Records whether this resistance level became the upper bound of a macro auction range. Same format as the support field equivalent. |
| `level_converted_to_micro_auction_upper_bound` | `string` | Subjective | Same as above, for micro auction ranges. |

---

## `auction_ranges`

Contains two arrays: `macro` and `micro`. An **auction** is a price range within which the market was in a balanced, two-sided state — neither strongly trending up nor down. Price oscillating within a bounded range signals that buyers and sellers have reached temporary equilibrium. Auctions are identified by observing price returning to the same zone repeatedly, with neither side able to push through.

**Macro auctions** span a larger price and/or time range and represent the dominant value area of the session. **Micro auctions** are smaller, nested within or adjacent to the macro context, and often form at the bounds of macro auctions as the market tests whether it can break out of the larger range.

### `auction_ranges.macro[]`

| Field | Type | Category | Description |
|---|---|---|---|
| `id` | `string` | Subjective | Unique identifier. Typically `"Macro 1"`, `"Macro 2"`, etc. |
| `label` | `string` | Subjective | Display label. Usually identical to `id`. |
| `instance` | `integer` | Subjective | Sequential number (1-based) distinguishing multiple instances of the same auction range (e.g. price returning to the same range in a second visit). |
| `low` | `float` | Subjective | Lower boundary of the auction range (USD). Corresponds to a previously identified support level. |
| `high` | `float` | Subjective | Upper boundary of the auction range (USD). Corresponds to a previously identified resistance level. |
| `candles_within` | `array` | Subjective | List of candle index spans during which price was trading *within* the auction boundaries. Each entry is an object: `{start_idx, end_idx, confirmation_candle_idx}`. See [candle range objects](#candle-range-objects). |
| `conviction` | `string` | Subjective | Analyst's confidence in the auction identification. See [Conviction values](#conviction-values). |
| `notes` | `string` | Subjective | Free-text rationale. Empty string when no annotation provided. |

### `auction_ranges.micro[]`

| Field | Type | Category | Description |
|---|---|---|---|
| `id` | `string` | Subjective | Unique identifier. Typically `"Micro 1"`, `"Micro 2"`, etc. |
| `label` | `string` | Subjective | Display label. Usually identical to `id`. |
| `low` | `float` | Subjective | Lower boundary of the micro auction range (USD). |
| `high` | `float` | Subjective | Upper boundary of the micro auction range (USD). |
| `candle_idx_range` | `object` | Subjective | The single candle span during which the micro auction was active. Object: `{start_idx, end_idx, confirmation_candle_idx}`. See [candle range objects](#candle-range-objects). |
| `conviction` | `string` | Subjective | Analyst's confidence in the micro auction identification. See [Conviction values](#conviction-values). |
| `notes` | `string` | Subjective | Free-text rationale. Empty string when no annotation provided. |

---

## `significant_candles`

Array of individual candles flagged for notable structural events. An empty array means no candles were flagged for this session.

| Field | Type | Category | Description |
|---|---|---|---|
| `candle_idx` | `integer` | Subjective | The `idx` of the flagged candle within `candles[]`. |
| `event` | `string` | Subjective | The type of structural event observed. See [Event values](#event-values). |
| `bias` | `string` | Subjective | The directional implication of the event. `"bullish"` or `"bearish"`. |
| `conviction` | `string` | Subjective | Analyst's confidence in the event classification. See [Conviction values](#conviction-values). |
| `notes` | `string` | Subjective | Free-text explanation of the specific evidence that supports the event classification. |

---

## `level_interactions`

Array recording specific instances where price interacted meaningfully with a defined level or auction boundary. Serves as supporting evidence for level classifications. An empty array means no interactions were explicitly annotated for this session.

| Field | Type | Category | Description |
|---|---|---|---|
| `level_id` | `string` | Subjective | The `id` of the level or auction range being interacted with (references a `manual_levels` or `auction_ranges` entry). |
| `level_type` | `string` | Subjective | Category of the referenced level. One of: `"support"`, `"resistance"`, `"auction_macro"`, `"auction_micro"`. |
| `candle_idx` | `array[integer]` | Subjective | One or more candle indices (from `candles[].idx`) at which the interaction occurred. Always an array, even for single-candle interactions. |
| `interaction` | `string` | Subjective | The nature of the price interaction. See [Interaction values](#interaction-values). |
| `side` | `string` | Subjective | The direction from which price approached the level. `"from_above"` or `"from_below"`. |
| `notes` | `string` | Subjective | Free-text explanation of the specific evidence that classifies this as a meaningful interaction. |

---

## Shared Sub-Objects

### `candle_idx_range` object

Used in `manual_levels` entries and `auction_ranges.micro` entries to define a single contiguous candle span.

| Field | Type | Description |
|---|---|---|
| `start_idx` | `integer \| null` | The `idx` of the first candle in the range. `null` if not yet established at time of annotation. |
| `end_idx` | `integer \| null` | The `idx` of the last candle in the range. `null` if the level/auction is still open at the end of the session. |
| `confirmation_candle_idx` | `integer \| null` | The `idx` of the candle that definitively confirmed the level or auction boundary (e.g. a clear rejection or breakout candle). `null` if no specific confirmation candle was identified. |

### Candle range objects

Used inside `auction_ranges.macro[].candles_within[]`. Same fields as `candle_idx_range`, representing one visit of price to the macro auction zone.

| Field | Type | Description |
|---|---|---|
| `start_idx` | `integer \| null` | First candle of this visit within the auction range. |
| `end_idx` | `integer \| null` | Last candle of this visit. `null` if still active at end of session. |
| `confirmation_candle_idx` | `integer \| null` | Candle that confirmed the auction boundaries on this visit. `null` if not identified. |

---

## Enumeration Values

### Conviction values

Applies to `manual_levels.support[].conviction`, `manual_levels.resistance[].conviction`, `auction_ranges.macro[].conviction`, `auction_ranges.micro[].conviction`, and `significant_candles[].conviction`.

| Value | Meaning |
|---|---|
| `"very_strong"` | Multiple high-quality interactions with clear, high-volume evidence. High probability that the market will respect this level again. |
| `"strong"` | Clear evidence from price action and/or volume. The level is well-defined and defensible. |
| `"neutral"` | Reasonable evidence exists but is incomplete, mixed, or contradicted by other factors. The level is plausible but uncertain. |
| `"weak"` | Thin or ambiguous evidence. The level is speculative. |
| `""` | Conviction not yet assessed by the analyst. |

### Event values

Applies to `significant_candles[].event`.

| Value | Meaning |
|---|---|
| `"absorption"` | Large taker volume failed to produce a proportionally large price move. Suggests that passive orders on the opposing side absorbed the aggressive flow, implying the opposing side is in control. |

Additional event types may be added in future schema versions.

### Interaction values

Applies to `level_interactions[].interaction`.

| Value | Meaning |
|---|---|
| `"bounce"` | Price approached the level and reversed without breaking through, confirming the level is holding. |
| `"rejection"` | Price tested the level from one side, failed to close through it, and reversed. Often characterised by wicks or doji candles at the level. |
| `"breakout"` | Price closed convincingly beyond the level, potentially invalidating it as support/resistance and converting it to the opposite. |

---

## Signal Parsing and Weighting Guide for AI Agents

This section describes a recommended approach for an AI agent or external model to parse, prioritise, and combine the structural signals present in each session object. The goal is to produce a coherent directional bias and confidence level from the annotated data.

---

### Parsing Order

Always process the session in this sequence. Each layer provides context that constrains the interpretation of the next.

```
1. Establish macro auction context        → auction_ranges.macro
2. Identify micro auction context         → auction_ranges.micro
3. Locate active support/resistance       → manual_levels
4. Identify structural candle events      → significant_candles
5. Read per-candle flow signals           → candles[].volume_delta, rsi, atr
6. Apply level interaction evidence       → level_interactions
```

Work top-down: macro context sets the regime, micro context narrows the zone, levels provide the specific trigger price, and candle-level data provides real-time confirmation or denial.

---

### Step 1 — Determine Macro Auction Regime

The macro auction defines whether the market is in a **range** or **trending** state, which governs how all other signals should be interpreted.

| Price position relative to macro auction | Regime | Interpretation bias |
|---|---|---|
| Inside `[low, high]` | Range | Fade extremes. Expect price to mean-revert toward the midpoint. Treat `high` as resistance and `low` as support. |
| Outside `[low, high]`, above `high` | Breakout up | Follow price. The upper bound becomes support on retests. Reduce weight on resistance levels within the old range. |
| Outside `[low, high]`, below `low` | Breakout down | Follow price. The lower bound becomes resistance on retests. Reduce weight on support levels within the old range. |
| Re-entering range after breakout attempt | Failed breakout | High-conviction fade. The failed side gains additional weight as the opposing level strengthens. |

If `candles_within` contains multiple entries (i.e. `instance > 1`), the auction has already been revisited. A second or third visit to the same range typically signals weakening resolve on the breakout side and increases the probability of continued range behaviour.

---

### Step 2 — Determine Micro Auction Context

Micro auctions represent concentrated equilibrium zones, usually forming at one of the macro bounds. They narrow the area of interest and signal which side is building pressure.

| Micro auction position | Signal |
|---|---|
| Micro auction at macro `low` | Bearish pressure accumulating. A confirmed break below the micro low triggers a macro breakdown. |
| Micro auction at macro `high` | Bullish pressure accumulating. A confirmed break above the micro high triggers a macro breakout. |
| Micro auction mid-range | Indecision. Reduce directional conviction until the micro auction resolves. |

Check `confirmation_candle_idx` in `candle_idx_range`. If non-null, the auction boundaries have been explicitly confirmed and carry more weight than unconfirmed auctions.

---

### Step 3 — Score Active Levels

For each level in `manual_levels.support[]` and `manual_levels.resistance[]`, compute a **level weight** using the following base conviction scores and multipliers.

#### Base conviction score

| `conviction` value | Base score |
|---|---|
| `"very_strong"` | 1.00 |
| `"strong"` | 0.70 |
| `"neutral"` | 0.40 |
| `"weak"` | 0.15 |
| `""` (unassessed) | 0.10 |

#### Multipliers (multiplicative, applied to base score)

| Condition | Multiplier |
|---|---|
| Level is also a macro auction boundary (`level_converted_to_macro_auction_*` = `"True, …"`) | ×1.60 |
| Level is also a micro auction boundary (`level_converted_to_micro_auction_*` = `"True, …"`) | ×1.30 |
| Level has one or more entries in `level_interactions[]` | ×1.15 per recorded interaction, capped at ×1.45 |
| Current price is within 0.5× ATR of the level | ×1.20 (proximity bonus — the level is immediately relevant) |
| Level's `candle_idx_range.end_idx` is null (still open at session end) | ×1.10 (level is unresolved and therefore still active) |

**Effective level score = base score × all applicable multipliers**, capped at 1.00.

A support level with score ≥ 0.60 should be treated as a high-confidence demand zone. A resistance level with score ≥ 0.60 should be treated as a high-confidence supply zone.

---

### Step 4 — Apply Significant Candle Signals

Significant candles provide structural context at specific price points. Weight them as follows.

| `event` | `bias` | Signal weight added to the corresponding directional score |
|---|---|---|
| `"absorption"` | `"bullish"` | +0.25 toward bullish at the candle's price level |
| `"absorption"` | `"bearish"` | +0.25 toward bearish at the candle's price level |

Scale this weight by the `conviction` of the significant candle using the same base conviction scores in Step 3 (e.g. a `"strong"` absorption adds 0.25 × 0.70 = +0.175 net).

Absorption at or near a level that already has a high effective score is strong confirmation. Absorption in open space, away from any level, is weaker and should be treated as a flag rather than a primary signal.

---

### Step 5 — Read Per-Candle Flow Signals

These signals are objective and should be used for **confirmation** of the structural picture, not as primary drivers.

#### `volume_delta`

Compute a rolling sum of `volume_delta` over the most recent N candles (recommended N = 3–5 for 15m data).

| Rolling `volume_delta` | Near a support level | Near a resistance level |
|---|---|---|
| Strongly positive (sustained buying) | Confirms support hold. Adds +0.15 to bullish score. | Warns of breakout attempt. Reduce bearish conviction by 0.10. |
| Strongly negative (sustained selling) | Warns of support failure. Reduce bullish conviction by 0.10. | Confirms resistance hold. Adds +0.15 to bearish score. |
| Near zero (balanced) | No confirmation. Hold prior score. | No confirmation. Hold prior score. |

"Strongly positive/negative" is contextual: use the session's average absolute `volume_delta` per candle as the baseline. A candle's delta is "strong" if its absolute value exceeds 1.5× the session mean.

#### `rsi`

Use RSI as a regime filter, not a trigger.

| RSI range | Effect |
|---|---|
| RSI < 25 | Multiply bullish level scores by 1.15. Market is statistically stretched to the downside. |
| 25 ≤ RSI < 40 | Mild bullish lean. No score adjustment; use as background context. |
| 40 ≤ RSI ≤ 60 | Neutral. No adjustment. |
| 60 < RSI ≤ 75 | Mild bearish lean. No score adjustment; use as background context. |
| RSI > 75 | Multiply bearish level scores by 1.15. Market is statistically stretched to the upside. |

RSI alone should never override a structural signal. Its role is to tilt probability when multiple signals are otherwise equal.

#### `atr`

Use ATR for **proximity thresholds** and **expected move sizing**, not for directional signals.

- A level is "in play" if the current candle's close is within 0.5× ATR of the level price.
- A level is "nearby but not yet active" if within 1.0–2.0× ATR.
- Breakout confirmation requires a candle close beyond a level by at least 0.25× ATR (to filter false breakouts from wicks).

---

### Step 6 — Incorporate Level Interaction Evidence

Each entry in `level_interactions[]` documents a historically observed price behaviour at a level. Use it to adjust forward expectations.

| `interaction` | `side` | Interpretation |
|---|---|---|
| `"bounce"` | `from_above` | The level held as support. Adds credibility to the level holding again on the next approach from above. |
| `"bounce"` | `from_below` | The level held as resistance. Adds credibility to the level holding again on the next approach from below. |
| `"rejection"` | either | Strong level defence. Higher weight than a bounce; adds more to the level score (already captured in the multiplier in Step 3). |
| `"breakout"` | either | The level was breached. If the same level appears in `manual_levels` with `end_idx` still null, treat it with reduced conviction — it survived the breakout or is now acting in the opposite role. |

---

### Combining Signals into a Directional Score

After running Steps 1–6, compute a net directional score in the range [−1.0, +1.0]:

```
directional_score = (sum of bullish signal weights) − (sum of bearish signal weights)
```

Where weights contributed by each step are:

| Source | Max contribution per side |
|---|---|
| Active support/resistance level score (Step 3) | Up to 1.00 (primary signal) |
| Significant candle at or near level (Step 4) | Up to 0.25 |
| Rolling volume_delta confirmation (Step 5) | Up to 0.15 |
| RSI extreme filter (Step 5) | Multiplicative on level scores only |
| Macro regime alignment (Step 1) | Multiplicative — signals against the macro regime are discounted by 0.50 |

**Confidence thresholds:**

| `directional_score` | Interpretation |
|---|---|
| > 0.65 | High-confidence directional signal. Structure is clear and confirmed. |
| 0.35 – 0.65 | Moderate signal. Wait for one additional candle of confirmation (e.g. a volume_delta spike or close through the level). |
| −0.35 – 0.35 | Low confidence. No actionable directional bias; treat as ranging. |
| < −0.65 | High-confidence in the opposing direction. Apply the same logic symmetrically. |

---

### Key Parsing Rules and Failure Modes

**Always follow these rules to avoid misinterpretation:**

1. **Auction context overrides levels.** A support level inside a broken-down macro auction carries significantly less weight than a fresh support below the auction. Adjust level scores down by 50% if the auction has resolved against the level's side.

2. **Unconfirmed auctions are hypotheses, not facts.** If `confirmation_candle_idx` is null, treat the auction boundaries as provisional. Do not anchor strongly to them until a confirmation candle has been observed.

3. **`end_idx: null` means the level/auction is unresolved.** This is the normal state for the current session's annotations. Do not interpret it as missing data.

4. **Multiple macro auction instances signal weakening.** If `instance` ≥ 2, the market has returned to the same range. The probability of a clean breakout decreases; range-fade strategies gain relative weight.

5. **`notes` fields are narrative, not numeric.** Read them to understand the analyst's reasoning, but do not attempt to parse them programmatically for signal values. Use them as qualitative context to sanity-check the structured signal output.

6. **Conviction `""` means do not rely on the level.** An unassessed conviction indicates the level was recorded but not evaluated. Assign minimal weight (base score 0.10) and do not allow it to be the primary driver of a directional call.

7. **`level_interactions` corroborate, they do not create signals.** A level with three bounce interactions but low conviction is still a lower-quality level than one with a single interaction and `"very_strong"` conviction. The interaction multiplier amplifies the base; it does not replace it.

---

## Schema Evolution Notes

The following changes were made when standardising sessions to schema version 5:

| Change | Affected sessions |
|---|---|
| Added `schema_version: 5` | Jan-08 session (previously absent) |
| Renamed `clv_volume` → `volume_delta` in candle objects | Jan-08 session |
| Renamed `symbol` `"BTC-USD"` → `"BTCUSDT"` | Jan-02, Jan-08, Jan-12 sessions |
| Added `conviction` field to `manual_levels` entries | Jan-08 session |
| Added `level_converted_to_*` fields to `manual_levels` entries | Jan-08 session |
| Converted `auction_ranges.macro[].candle_idx_range` (object) → `candles_within` (array of objects) | Jan-08 session |
| Added `instance` field to `auction_ranges.macro[]` entries | Jan-08 session |
| Added `conviction` field to `auction_ranges.macro[]` and `auction_ranges.micro[]` entries | Jan-08 session |
| Added `auction_ranges.micro` key (was entirely absent) | Jan-08 session |
| Added `confirmation_candle_idx: null` to all `candles_within` and `candle_idx_range` objects that lacked it | Jan-02, Jan-08, Jan-12, Jan-18 sessions |
| Added `significant_candles: []` | Jan-02, Jan-08, Jan-12, Jan-18 sessions |
| Added `level_interactions: []` | Jan-02, Jan-12, Jan-18, Jan-21 sessions |
| Normalised `level_interactions[].candle_idx` to always be an array | Jan-08 session (was inconsistently int or array) |
