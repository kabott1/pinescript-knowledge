# PINESCRIPT KNOWLEDGE BASE
## Shared Knowledge for Claude Models (Opus & Sonnet)

**Version:** 3.0  
**Last Updated:** January 4, 2025  
**Maintainers:** Claude Opus, Claude Sonnet  
**Owner:** Lucas (Trader/Quant Developer)  
**Purpose:** Centralized, collaborative reference for Pine Script conventions, patterns, discoveries, and best practices

---

## HOW TO USE THIS DOCUMENT

This is a **living document** that any Claude model can read and update. When working with Lucas on Pine Script:

1. **READ** this document at the start of relevant sessions
2. **APPLY** conventions consistently across all code
3. **UPDATE** when new patterns, fixes, or conventions are discovered
4. **LOG** changes in the CHANGELOG section

**Location:** `/mnt/user-data/outputs/PINESCRIPT_KNOWLEDGE_BASE.md` (read/write)  
**Legacy Reference:** `/mnt/project/LUCAS_CODING_CONVENTIONS_v2.md` (read-only, archived)

---

## TABLE OF CONTENTS

1. [Version Preference](#1-version-preference)
2. [Self-Referential Variables](#2-self-referential-variables)
3. [Debugging Rules](#3-debugging-rules)
4. [Anti-Repainting Best Practices](#4-anti-repainting-best-practices)
5. [Signal Architecture](#5-signal-architecture)
6. [Color Palette](#6-color-palette)
7. [Multitimeframe System](#7-multitimeframe-system)
8. [MTF Calculation Rules](#8-mtf-calculation-rules)
9. [Code Formatting](#9-code-formatting)
10. [Language Rules](#10-language-rules)
11. [Risk Management Templates](#11-risk-management-templates)
12. [Strategy Development Standards](#12-strategy-development-standards)
13. [Discoveries & Lessons Learned](#13-discoveries--lessons-learned)
14. [Changelog](#changelog)

---

## 1. VERSION PREFERENCE

### Default: Pine Script v4 for Legacy Code

When modifying existing v4 indicators or creating new ones based on v4 code:
- **STAY IN v4** unless explicitly requested to migrate
- v4 syntax for cumulative/self-referential variables works reliably with `security()`
- Migration to v5/v6 can break `var` behavior inside functions passed to `request.security()`

### When to use v5/v6:
- User explicitly requests migration
- New v5/v6-only features are required (arrays, maps, methods, etc.)
- Starting from scratch with no legacy base
- Strategy development requiring v5/v6 improvements
- **Anti-repainting strategies** - v6 offers better `barstate.isconfirmed` integration

### Migration Checklist (if required):
```pine
// v4 -> v5/v6 Key Changes:
study()          -> indicator()
security()       -> request.security()
input()          -> input.int() / input.float() / input.bool() / input.string()
input.resolution -> input.timeframe
iff()            -> ternary operator ? :
nz()             -> nz() (same)
roc()            -> ta.roc()
ema()            -> ta.ema()
pivotlow()       -> ta.pivotlow()
valuewhen()      -> ta.valuewhen()
barssince()      -> ta.barssince()
crossover()      -> ta.crossover()
```

---

## 2. SELF-REFERENTIAL VARIABLES

### The Problem

Functions with self-referential (cumulative) variables behave differently across Pine versions when passed to `security()` / `request.security()`.

### v4 Pattern (WORKS RELIABLY):
```pine
PV() =>
    PV = 0.0
    PV := iff(volume > volume[1], nz(PV[1], 0) + roc(close, 1), nz(PV[1], 0))

// Usage - works correctly
PVI = security(syminfo.tickerid, resolution, PV())
```

### v5/v6 Pattern (PROBLEMATIC):
```pine
// This may NOT work correctly with request.security()
f_cumulative() =>
    var float val = 0.0
    val := condition ? val + something : val
    val

// The 'var' inside function + request.security() = unreliable
result = request.security(syminfo.tickerid, tf, f_cumulative())  // MAY FAIL
```

### v5/v6 Workaround (if migration required):
```pine
// Calculate OUTSIDE function, at global scope
var float PVI_acc = 0.0
PVI_acc := volume > volume[1] ? PVI_acc + ta.roc(close, 1) : PVI_acc

// Then pass the calculated value
result = request.security(syminfo.tickerid, tf, PVI_acc)
```

### Rule of Thumb:
- **v4 code with cumulative functions + security()** → Keep in v4
- **Need v5/v6** → Move cumulative logic to global scope before `request.security()`

---

## 3. DEBUGGING RULES

### Blank Output with No Errors

When migrated or modified code produces blank/empty output with no compilation errors:

1. **STOP** - Don't add more complexity trying to fix it
2. **REVERT** - Return to the last working version
3. **ISOLATE** - Make ONE minimal change at a time
4. **TEST** - Verify after EACH change before proceeding

### Common Causes of Blank Output:
- `var` inside functions passed to `request.security()` (v5/v6)
- Wrong timeframe string format
- Conditions that never evaluate to true
- Offset/lookback values exceeding available bars
- `na` propagation through calculations

### Debug Strategy:
```pine
// Step 1: Verify base data exists
plot(osc, color=color.red)  // Can you see the oscillator?

// Step 2: Verify pivots are detected
plotshape(plFound, style=shape.cross, location=location.bottom)  // Are pivots firing?

// Step 3: Verify conditions individually
plotshape(priceLL, style=shape.circle, location=location.top)  // Is price condition met?
plotshape(oscHL, style=shape.diamond, location=location.top)   // Is osc condition met?
```

### Prevention:
- Test original code BEFORE making changes
- Commit/save working versions frequently
- When in doubt, stay in v4 if source is v4

---

## 4. ANTI-REPAINTING BEST PRACTICES

### Core Principle
All signals must be generated using **confirmed (closed) bars only**. Never use `bar_index` (current bar) for signal generation.

### Strategy Settings (MANDATORY):
```pine
strategy("Name", 
    overlay=true, 
    calc_on_every_tick=false,  // CRITICAL: Only calculate on bar close
    process_orders_on_close=true
)
```

### Signal Generation Pattern:
```pine
// WRONG - Can repaint
if ta.crossover(k, d)
    strategy.entry("Long", strategy.long)

// CORRECT - Uses confirmed bars only
if barstate.isconfirmed
    if k[1] > d[1] and k[2] <= d[2]  // Cross happened on PREVIOUS closed bar
        strategy.entry("Long", strategy.long)
```

### Price Reference Rules:
```pine
// WRONG - References current bar
stopLoss = ta.lowest(low, 5)

// CORRECT - References only closed bars
stopLoss = ta.lowest(low[1], 5)  // Lowest of bars [1] through [5]
```

### The `[1]` Rule:
- Entry signals: Always check conditions on `[1]` and `[2]` (closed bars)
- Stop loss: Use `low[1]` or calculated from closed bars
- Take profit: Calculate from entry price (known value)

---

## 5. SIGNAL ARCHITECTURE

### 5.1 FRACTAL HA (Proprietary Heiken Ashi)

**Concept:** Detects confirmed direction change using 3 consecutive bar pattern.

**NOT traditional Heiken Ashi:**
```pine
// Smoothed with linear regression (not traditional HA calculation)
HaClose = linreg(close, 6, 0)
HaOpen  = linreg(open, 6, 0)
```

**Bullish Pattern:**
```pine
HaOpen[3] > HaClose[3]  and  // Bar 3: BEARISH candle
HaOpen[2] > HaClose[2]  and  // Bar 2: BEARISH candle (confirmation)
HaOpen[1] < HaClose[1]       // Bar 1: BULLISH candle <- SIGNAL
```

**Bearish Pattern:**
```pine
HaOpen[3] < HaClose[3]  and  // Bar 3: BULLISH candle
HaOpen[2] < HaClose[2]  and  // Bar 2: BULLISH candle (confirmation)
HaOpen[1] > HaClose[1]       // Bar 1: BEARISH candle <- SIGNAL
```

**Alternative versions (more concise):**
```pine
// Option 1: Helper function
isBearishCandle(i) => HaOpen[i] > HaClose[i]
isBullishCandle(i) => HaOpen[i] < HaClose[i]

fractal_bull = isBearishCandle(3) and isBearishCandle(2) and isBullishCandle(1)
fractal_bear = isBullishCandle(3) and isBullishCandle(2) and isBearishCandle(1)

// Option 2: Direction-based
haDirection(i) => HaOpen[i] < HaClose[i] ? 1 : -1

fractal_bull = haDirection(3) == -1 and haDirection(2) == -1 and haDirection(1) == 1
fractal_bear = haDirection(3) == 1  and haDirection(2) == 1  and haDirection(1) == -1
```

**Advantages:**
- No noise with linear regression smoothing
- No repaint (uses closed bars [1], [2], [3])
- Double confirmation (2 candles same direction before change)
- Detects momentum shift, not just level cross

---

### 5.2 K&D SIGNAL (Oscillators)

**Concept:** Confirmed K over D cross in extreme zone, using closed bars (no repaint).

**Bullish Pattern:**
```pine
k[2] < oversold  and  // K was in oversold (bar 2)
k[2] < d[2]      and  // K was BELOW D (bar 2)
k[1] > d[1]           // K crossed ABOVE D (bar 1) <- SIGNAL
```

**Bearish Pattern:**
```pine
k[2] > overbought  and  // K was in overbought (bar 2)
k[2] > d[2]        and  // K was ABOVE D (bar 2)
k[1] < d[1]             // K crossed BELOW D (bar 1) <- SIGNAL
```

**Advantages:**
- No repaint (uses [1] and [2] = closed bars)
- Zone confirmation (verifies extreme at [2])
- Position confirmation (K on correct side before cross)
- Clean cross (detects when already completed)

**Difference vs ta.crossover():**
```pine
// ta.crossover() - CAN repaint on current bar
ta.crossover(k, d)  // WARNING: Changes while bar is open

// Lucas method - NO repaint
k[2] < d[2] and k[1] > d[1]  // SAFE: Only closed bars
```

**Comparison: Fractal HA vs K&D Signal**

| Aspect | Fractal HA | K&D Signal |
|---------|------------|-----------|
| Confirmation | 3 bars | 2 bars |
| Context | Open vs Close | K vs D + zone |
| Application | Smoothed price action | Oscillators |
| Timing | Slower, safer | Faster |
| Repaint | NO (uses [3],[2],[1]) | NO (uses [2],[1]) |

---

## 6. COLOR PALETTE

### Base Configuration
- **Theme:** TradingView Light (white background)
- **Candles:** Green (bullish) / Red (bearish) - standard

### Pivot Markers (Overlay Indicators)

**By Timeframe:**
```pine
// DCL - Daily Cycle Low
dclColor = color.new(#26a69a, 0)  // Green
plotshape(..., color=dclColor, style=shape.triangleup, size=size.tiny)

// WCL - Weekly Cycle Low
wclColor = color.new(#2962ff, 0)  // Blue
plotshape(..., color=wclColor, style=shape.triangleup, size=size.small)
```

**By State/Type:**
```pine
rtColor = color.new(#4caf50, 0)       // Right Translation (Strong Green)
ltColor = color.new(#f44336, 0)       // Left Translation (Red)
failureColor = color.new(#ff5722, 0)  // Cycle Failure (Alert Red)
dssCrossColor = color.new(#9c27b0, 0) // DSS Cross (Purple)
```

**Visual Hierarchy:**

| Element | Color | Size | Purpose |
|---------|-------|------|---------|
| DCL | Green #26a69a | size.tiny | Daily pivots |
| WCL | Blue #2962ff | size.small | Weekly pivots (more important) |
| RT | Green #4caf50 | Label/Line | Bullish cycles |
| LT | Red #f44336 | Label/Line | Bearish cycles |
| Failure | Alert Red #ff5722 | Highlighted | Failure alert |
| DSS Cross | Purple #9c27b0 | size.tiny | Alternative method |

### Oscillators

**K & D Lines:**
```pine
kColor = color.new(#000000, 0)  // K: Black
dColor = color.new(#ff0000, 0)  // D: Red

plot(dssK, "K", color=kColor, linewidth=2)
plot(dssD, "D", color=dColor, linewidth=2)
```

**Levels:**
```pine
levelColor = color.new(#808080, 0)  // Gray

hline(upperLevel, "Overbought", color=levelColor, linestyle=hline.style_dotted)
hline(50, "Middle", color=levelColor, linestyle=hline.style_dotted)
hline(lowerLevel, "Oversold", color=levelColor, linestyle=hline.style_dotted)
```

### Unified Levels System

**Simplified Input (single parameter):**
```pine
// User enters ONE number
bandDistance = input.float(20.0, "Level Distance (from 50)", 
                          minval=0.0, maxval=50.0,
                          tooltip="Distance from middle line (50).\nEx: 20 = levels at 30 and 70")

// Automatic calculation
midLevel = 50.0
upperLevel = midLevel + bandDistance  // 50 + 20 = 70 (overbought)
lowerLevel = midLevel - bandDistance  // 50 - 20 = 30 (oversold)
```

**Examples:**

| Input | Oversold | Middle | Overbought | Use |
|-------|----------|--------|------------|-----|
| 20 | 30 | 50 | 70 | Standard |
| 25 | 25 | 50 | 75 | Moderate |
| 30 | 20 | 50 | 80 | Extremes |
| 15 | 35 | 50 | 65 | Conservative |

**Advantages:**
- Simplicity (one input controls both levels)
- Symmetry (always balanced around 50)
- Speed (adjust sensitivity with one parameter)
- Intuitive ("20" = zones 20 points from middle)

### Complete Oscillator Template

```pine
//@version=5
indicator("Oscillator - Lucas Style", format=format.price, precision=2)

// SETTINGS
periodK = input.int(8, "%K Length", minval=1, maxval=50)
smoothK = input.int(7, "%K Smoothing", minval=1, maxval=20)
periodD = input.int(3, "%D Smoothing", minval=1, maxval=20)
bandDistance = input.float(20.0, "Level Distance", minval=0.0, maxval=50.0)

// COLORS
kColor = color.new(#000000, 0)
dColor = color.new(#ff0000, 0)
levelColor = color.new(#808080, 0)

// LEVELS
midLevel = 50.0
upperLevel = midLevel + bandDistance
lowerLevel = midLevel - bandDistance

// CALCULATIONS
// [your K and D calculations here]

// PLOTS
plot(k, "K", color=kColor, linewidth=2)
plot(d, "D", color=dColor, linewidth=2)

hline(upperLevel, "Overbought", color=levelColor, linestyle=hline.style_dotted)
hline(midLevel, "Middle", color=levelColor, linestyle=hline.style_dotted)
hline(lowerLevel, "Oversold", color=levelColor, linestyle=hline.style_dotted)
```

---

## 7. MULTITIMEFRAME SYSTEM

### METHOD 1: COMPLETE Selector (13 options)

**Use:** Indicators with maximum flexibility (oscillators, moving averages, etc.)

```pine
// INPUT
tfLabel = input.string("1D", "Timeframe", options=["Chart", "1H", "4H", "6H", "12H", "1D", "2D", "3D", "4D", "1W", "2W", "1M"], group="Timeframe")

// CONVERSION (ALL IN ONE LINE - NO MULLETS)
string tf = tfLabel == "Chart" ? timeframe.period : tfLabel == "1H" ? "60" : tfLabel == "4H" ? "240" : tfLabel == "6H" ? "360" : tfLabel == "12H" ? "720" : tfLabel == "1D" ? "D" : tfLabel == "2D" ? "2D" : tfLabel == "3D" ? "3D" : tfLabel == "4D" ? "4D" : tfLabel == "1W" ? "W" : tfLabel == "2W" ? "2W" : tfLabel == "1M" ? "M" : "D"

// USAGE
data = request.security(syminfo.tickerid, tf, close, barmerge.gaps_off, barmerge.lookahead_off)
```

**Alternative for very long conversions - use switch:**
```pine
string tf = switch tfLabel
    "Chart" => timeframe.period
    "1H" => "60"
    "4H" => "240"
    "6H" => "360"
    "12H" => "720"
    "1D" => "D"
    "2D" => "2D"
    "3D" => "3D"
    "4D" => "4D"
    "1W" => "W"
    "2W" => "2W"
    "1M" => "M"
    => "D"
```

**Conversion Table:**

| User Label | Pine String | Minutes | Main Use |
|------------|-------------|---------|----------|
| Chart | timeframe.period | Variable | Adapts to current chart |
| 1H | "60" | 60 | Short intraday |
| 4H | "240" | 240 | Medium intraday |
| 6H | "360" | 360 | Long intraday |
| 12H | "720" | 720 | Short swing |
| 1D | "D" | 1440 | **Standard swing** |
| 2D | "2D" | 2880 | Medium swing |
| 3D | "3D" | 4320 | Long swing |
| 4D | "4D" | 5760 | Very long swing |
| 1W | "W" | 10080 | **Position trading** |
| 2W | "2W" | 20160 | Long position |
| 1M | "M" | ~43200 | **Macro trend** |

---

### METHOD 2: SIMPLE Selector (4 options)

**Use:** Cycle indicators, macro analysis where user doesn't need much detail

```pine
// INPUT
analysisTfOpt = input.string("1D", "Analysis TF", options=["Chart", "1D", "1W", "1M"], group="Timeframe")

// CONVERSION
string tfAnalysis = analysisTfOpt == "Chart" ? timeframe.period : analysisTfOpt == "1D" ? "D" : analysisTfOpt == "1W" ? "W" : "M"

// USAGE
cycleLow = request.security(syminfo.tickerid, tfAnalysis, low, barmerge.gaps_off, barmerge.lookahead_off)
```

**Conversion Table:**

| User Label | Pine String | Main Use |
|------------|-------------|----------|
| Chart | timeframe.period | Flexible, uses current TF |
| 1D | "D" | Daily cycles (DCL) |
| 1W | "W" | Weekly cycles (WCL) |
| 1M | "M" | Monthly cycles (MCL) |

---

### METHOD 3: HIERARCHICAL System (Cycle Finder)

**Use:** When you need multiple related timeframes automatically

```pine
// PRIMARY INPUT
analysisTfOpt = input.string("1D", "Analysis TF (DCL)", options=["Chart", "1D", "1W"], group="Timeframe")

// HIERARCHICAL CONVERSION
string tfDCL = analysisTfOpt == "Chart" ? timeframe.period : analysisTfOpt == "1D" ? "D" : "W"
string tfWCL = analysisTfOpt == "Chart" ? "D" : analysisTfOpt == "1D" ? "W" : "M"

// SECONDARY INPUTS (optional)
dssTfOption = input.string("Same as DCL", "DSS Timeframe", options=["Same as DCL", "12H", "1D", "1W"], group="DSS Bressert")
string tfDSS = dssTfOption == "Same as DCL" ? tfDCL : dssTfOption == "12H" ? "720" : dssTfOption == "1D" ? "D" : "W"

// USAGE
dclLow = request.security(syminfo.tickerid, tfDCL, low, barmerge.gaps_off, barmerge.lookahead_off)
wclLow = request.security(syminfo.tickerid, tfWCL, low, barmerge.gaps_off, barmerge.lookahead_off)
dssK = request.security(syminfo.tickerid, tfDSS, f_dss_k(close, high, low, 8, 7), barmerge.gaps_off, barmerge.lookahead_off)
```

**Automatic Hierarchy:**

| DCL Input | tfDCL | tfWCL (auto) | Relationship |
|-----------|-------|--------------|--------------|
| Chart | timeframe.period | "D" | Current -> Daily |
| 1D | "D" | "W" | Daily -> Weekly |
| 1W | "W" | "M" | Weekly -> Monthly |

---

### HELPER FUNCTION (Reusable)

```pine
f_resolveTf(string tfLabel) =>
    tfLabel == "Chart" ? timeframe.period : tfLabel == "1H" ? "60" : tfLabel == "4H" ? "240" : tfLabel == "6H" ? "360" : tfLabel == "12H" ? "720" : tfLabel == "1D" ? "D" : tfLabel == "2D" ? "2D" : tfLabel == "3D" ? "3D" : tfLabel == "4D" ? "4D" : tfLabel == "1W" ? "W" : tfLabel == "2W" ? "2W" : tfLabel == "1M" ? "M" : "D"
```

---

## 8. MTF CALCULATION RULES

### GOLDEN RULE: Calculate INSIDE the Timeframe

**WRONG:**
```pine
float htfClose = request.security(syminfo.tickerid, "D", close)
float htfEma = ta.ema(htfClose, 20)  // WRONG
```

**CORRECT:**
```pine
float htfEma = request.security(syminfo.tickerid, "D", ta.ema(close, 20))  // CORRECT
```

---

## 9. CODE FORMATTING

### NO MULLETS Rule

**WRONG:**
```pine
string tf = tfLabel == "Chart" ? timeframe.period :
            tfLabel == "1H" ? "60" :  // ERROR
            "D"
```

**CORRECT - One line:**
```pine
string tf = tfLabel == "Chart" ? timeframe.period : tfLabel == "1H" ? "60" : "D"
```

**CORRECT - Switch:**
```pine
string tf = switch tfLabel
    "Chart" => timeframe.period
    "1H" => "60"
    => "D"
```

---

## 10. LANGUAGE RULES

### English Only - 100%

```pine
// NEVER
senal_alcista = true

// ALWAYS
bullish_signal = true
```

**Applies to:** Variables, functions, comments, inputs, tooltips, labels, groups, ALL text.

**Exception:** None.

---

## 11. RISK MANAGEMENT TEMPLATES

### Fixed USD Risk System

**Concept:** Risk a fixed USD amount per trade, dynamically calculate position size based on stop distance.

```pine
// INPUTS
riskPerTradeUSD = input.float(100.0, "Risk per Trade (USD)", minval=1.0, group="Risk Management")
riskRewardRatio = input.float(3.0, "Risk:Reward Ratio", minval=0.5, step=0.5, group="Risk Management")
slBarsLookback = input.int(5, "SL Lookback Bars", minval=1, maxval=20, group="Risk Management")
slOffset = input.float(0.5, "SL Offset (%)", minval=0.0, maxval=5.0, group="Risk Management") / 100

// STOP LOSS CALCULATION (Market Structure Based)
recentLow = ta.lowest(low[1], slBarsLookback)  // [1] = only closed bars
stopLossLong = recentLow * (1 - slOffset)      // Below the low with offset

recentHigh = ta.highest(high[1], slBarsLookback)
stopLossShort = recentHigh * (1 + slOffset)    // Above the high with offset

// POSITION SIZE CALCULATION
entryPrice = close[1]  // Entry at previous bar close (confirmed)
stopDistanceLong = entryPrice - stopLossLong
stopDistanceShort = stopLossShort - entryPrice

positionSizeLong = riskPerTradeUSD / stopDistanceLong
positionSizeShort = riskPerTradeUSD / stopDistanceShort

// TAKE PROFIT CALCULATION
takeProfitLong = entryPrice + (stopDistanceLong * riskRewardRatio)
takeProfitShort = entryPrice - (stopDistanceShort * riskRewardRatio)
```

### Strategy Entry Template (Anti-Repaint)

```pine
// ENTRY CONDITIONS (on confirmed bars only)
longCondition = barstate.isconfirmed and [your_signal_logic]
shortCondition = barstate.isconfirmed and [your_signal_logic]

// ENTRIES
if longCondition and strategy.position_size == 0
    strategy.entry("Long", strategy.long, qty=positionSizeLong)
    strategy.exit("Long Exit", "Long", stop=stopLossLong, limit=takeProfitLong)

if shortCondition and strategy.position_size == 0
    strategy.entry("Short", strategy.short, qty=positionSizeShort)
    strategy.exit("Short Exit", "Short", stop=stopLossShort, limit=takeProfitShort)
```

### Important Notes:
- **NO trailing stops** - Based on Lucas's experience, fixed TP/SL is preferred
- Stop loss based on **market structure** (actual lows/highs), not arbitrary percentages
- Position size adapts to volatility via stop distance
- All calculations use `[1]` (closed bars) to prevent repainting

---

## 12. STRATEGY DEVELOPMENT STANDARDS

### Naming Convention
All strategies must begin with: `---Strat`  
Example: `---Strat_Cycle_Finder_DCL_v1`

### Required Strategy Settings
```pine
//@version=6
strategy("---Strat_[Name]_v[X]", 
    overlay=true,
    initial_capital=10000,
    default_qty_type=strategy.cash,
    default_qty_value=100,
    calc_on_every_tick=false,     // MANDATORY: Anti-repaint
    process_orders_on_close=true,  // MANDATORY: Anti-repaint
    commission_type=strategy.commission.percent,
    commission_value=0.1,
    slippage=2
)
```

### Backtesting Philosophy
- **Objective over subjective**: Test with code, not eyes
- **Fixed rules**: Same entry/exit logic for every trade
- **Realistic costs**: Always include commission and slippage
- **No curve fitting**: Avoid over-optimization to historical data

### Converting Indicator to Strategy
1. Start with working indicator
2. Define clear, objective entry rules
3. Add risk management (fixed USD risk system)
4. Add `barstate.isconfirmed` to all signal conditions
5. Use `[1]` references for all price data in signals
6. Test on multiple timeframes and assets

---

## 13. DISCOVERIES & LESSONS LEARNED

### Averaging Multiple MAs
**Discovery:** Averaging multiple moving averages with different periods produces superior results compared to single-period averages.

```pine
// Instead of single EMA
ema_single = ta.ema(close, 20)

// Use averaged MAs
ema_fast = ta.ema(close, 10)
ema_mid = ta.ema(close, 20)
ema_slow = ta.ema(close, 30)
ema_averaged = (ema_fast + ema_mid + ema_slow) / 3
```

### Pivot Detection vs Market Structure
**Learning:** `ta.pivotlow()` and `ta.pivothigh()` are useful for visual marking but unreliable for stop loss calculation in strategies. Use actual bar lows/highs with lookback instead.

```pine
// For strategies - more reliable
stopLoss = ta.lowest(low[1], 5)

// For visual indicators - OK to use pivots
pivotLow = ta.pivotlow(low, 5, 5)
```

### DSS Bressert Optimization
**Observation:** Default DSS parameters (8, 7, 3) work well for daily timeframe. For faster timeframes, consider reducing smoothing.

---

## GOLDEN RULES SUMMARY

1. **Version:** Stay in v4 for legacy code with cumulative functions + security()
2. **Debug:** Stop, revert, isolate, test one change at a time
3. **Anti-Repaint:** `barstate.isconfirmed` + `calc_on_every_tick=false` + `[1]` references
4. **Signals:** Fractal HA (3 bars) and K&D Signal (2 bars) - no repaint
5. **Colors:** Green/Blue pivots, Black/Red oscillators, Gray levels
6. **Levels:** Unified single input system
7. **TF:** Centralized conversion, calculate INSIDE request.security()
8. **Format:** No mullets - ternary in one line or switch
9. **Language:** English only - 100%
10. **Risk:** Fixed USD, market structure stops, fixed R:R
11. **Strategies:** Name with `---Strat`, objective rules, no trailing stops

---

## CHANGELOG

### v3.0 (January 4, 2025)
- **MIGRATED** from project file to shared outputs location
- **ADDED** Section 4: Anti-Repainting Best Practices
- **ADDED** Section 11: Risk Management Templates
- **ADDED** Section 12: Strategy Development Standards
- **ADDED** Section 13: Discoveries & Lessons Learned
- **RESTRUCTURED** for collaborative editing by multiple Claude models
- **UPDATED** Golden Rules Summary with new sections

### v2.0 (December 29, 2024) - Legacy
- Added VERSION PREFERENCE section (v4 vs v5/v6 guidelines)
- Added SELF-REFERENTIAL VARIABLES section
- Added DEBUGGING RULES section

### v1.0 (December 2024) - Legacy
- Initial document

---

## NOTES FOR CLAUDE MODELS

When updating this document:
1. Always increment version in the header
2. Add entry to CHANGELOG with date and changes
3. Maintain consistent formatting
4. Test any code examples before adding
5. Cross-reference with existing conventions to avoid conflicts

**Last validated:** January 4, 2025 by Claude Opus

---

**END OF DOCUMENT**
