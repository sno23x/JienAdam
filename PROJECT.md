# Samurai Trading System — Project Reference

## Overview

A Pine Script v6 trading system for **XAUUSD (Gold)** on TradingView.
Two files: a live indicator and a full backtest strategy.

| File | Purpose |
|------|---------|
| `samurai_trading_system.pine` | Live indicator — signals, alerts, on-chart levels |
| `samurai_strategy.pine` | Backtest strategy — multi-TP partial close, trail SL, dashboard |

---

## Signal Logic

### MA Ribbon (per manual)
| MA | Type | Length | Source |
|----|------|--------|--------|
| MA1 | EMA | 100 | Close |
| MA2 | EMA | 133 | Close |
| MA3 | EMA | 233 | Close |
| MA4 | WMA | 720 | (H+L+C+C)/4 |

**Trend rules per manual:**
- **Uptrend:** Close above ALL MAs + breaking new highs + not breaking last swing low
- **Downtrend:** Close below ALL MAs + breaking new lows + not breaking last swing high
- **Sideway:** EMA Ribbon overlapping → no trade
- **Always read trend from higher TF** (H4 / D / W)

### Confluence Score (max 5/5)
Each factor scores 1 point toward a BUY or SELL signal:

| # | Factor | Source |
|---|--------|--------|
| 1 | Zone (Box / FVG / BOS) alignment | Current TF |
| 2 | MA Ribbon (price above/below all 4) | Current TF |
| 3 | RSI vs 50 midline | Current TF |
| 4 | At Fib retracement zone | Current TF |
| 5 | Momentum direction | Current TF |

**Minimum threshold:** 4/5 (default, was 3/5). Signal fires when score ≥ threshold.

### Signal Filters (all must pass)
- **ATR filter** — minimum volatility (atrPct ≥ 0.1%)
- **Session gates** — per-session toggles (Asia OFF / London ON / NY ON by default)
- **Market structure** — optional BOS alignment required
- **RSI extreme block** — prevents counter-trend entries at extremes:
  - RSI ≤ 30 → BUY is blocked (market chose DOWN = sell bias)
  - RSI ≥ 70 → SELL is blocked (market chose UP = buy bias)
  - After RSI exits extreme, hold block for N bars (reclaim & hold, default 3)
- **H4 Trend Gate** ⭐ (new) — entire MA Ribbon evaluated on H4; trade only with H4 trend
- **Sideway block via EMA overlap** (new) — when EMA 100/133/233 spread <0.3% → no trade
- **HTF Swing structure** (new) — block trades that break last H4 swing low (BUY) / swing high (SELL)
- **EMA Touch Entry** (new, optional) — require first MA touch after N bars away
- **Corner (มุม) levels** (new) — H4/D candle close levels drawn as S/R; optional gate

### Multi-TF Confirmation
Pulls RSI and PA signals from 15M / 30M / 1H / 4H simultaneously via `request.security()`.
Big-TF-only RSI mode: RSI filter applied only when current TF ≥ 1H.

---

## Entry / TP / SL

### SL Placement
- ATR-based: `SL = entry ± atr × slMult`
- Default multiplier: **1.5×**

### TP Levels (RR-based mode, default)
| TP | RR Ratio | Close % |
|----|----------|---------|
| TP1 | 1.5 × SL distance | 25% |
| TP2 | 2.0 × SL distance | 25% |
| TP3 | 3.0 × SL distance | 25% |
| TP4 | 4.0 × SL distance | 25% |

### Trail SL Modes
| Mode | Behavior |
|------|---------|
| Step (TP→prior TP) | After TP1 → move SL to entry (BE); after TP2 → move SL to TP1; etc. |
| BE only | Move SL to entry after TP1 only |
| Off | SL stays fixed until exit |

---

## Risk Sizing

```
Risk amount = equity × riskPct%       (default 1%)
Lot size    = riskAmt / (SL_distance × contractSize)
Minimum lot = 0.01 (fallback if calculated lot < min)
```

**Contract spec:** `contractSize = 100` (XAUUSD, 100 oz per standard lot)

### ⚠️ Leverage 1:1000 — Capital Warning

With $100 capital and 0.01 lot minimum:

| Item | Value |
|------|-------|
| Notional value (Gold @3000) | $3,000 |
| Margin required (1:1000) | **$3** per trade |
| SL distance typical 1H gold | 500–750 points |
| Actual risk at 0.01 lot | **$50–75 = 50–75% of capital** |
| 1% risk sizing yields | ~0.0002 lot → below minimum |
| Therefore lot used | 0.01 (fallback) = ~50% actual risk |

**Conclusion:** $100 capital is too small for 0.01 lot on gold with proper risk management.
Minimum recommended capital for 1% risk with 500-pt SL on 0.01 lot: **$500+**

---

## Alerts (Live Indicator)

8 alert conditions set up in TradingView:

| Alert | Trigger |
|-------|---------|
| BUY Signal (15M) | Score ≥ threshold, all filters pass, 15M TF |
| SELL Signal (15M) | Same, SELL direction |
| BUY Signal (30M) | 30M TF |
| SELL Signal (30M) | 30M TF |
| BUY Signal (1H) | 1H TF |
| SELL Signal (1H) | 1H TF |
| BUY Signal (4H) | 4H TF |
| SELL Signal (4H) | 4H TF |

**Recommended trigger:** "Once Per Bar" (not Once Per Bar Close) to avoid late alerts.  
Alerts persist until manually deleted — no need to recreate unless chart is reset.

---

## Backtest Results (1H XAUUSD, 1 Year)

| Metric | Value |
|--------|-------|
| Initial capital | $100 |
| Period | ~1 year |
| Total trades | 67 |
| Wins | 31 (46.3%) |
| Losses | 36 (53.7%) |
| Commission | $0.07/contract |
| Slippage | 2 points |

**Break-even win rate** at pure TP1 (1.5 RR): ~40%.  
At 46.3% WR the system is theoretically +EV at TP1 RR, but partial-close dynamics and partial TP hits can reduce effective RR. Current 46.3% WR needs improvement toward 50%+ for consistent profitability with this RR profile.

### Ideas to Improve Win Rate
- Tighten score threshold back to 4/5 on higher-volatility periods
- Add 4H trend filter (only take trades in 4H trend direction)
- Widen SL slightly (reduce premature SL hits)
- Test London/NY session only (filter out low-liquidity Asia entries)

---

## Settings Quick Reference

### Recommended Settings (v3 — capital + LTF preset)
```
Initial Capital: $1000  (was $100 — too small for 0.01 lot)
Min Score:       4/5
TP Mode:         RR (SL-based)
RR:              1.5 / 2.0 / 3.0 / 4.0
Trail SL:        Step (TP→prior TP)
Risk %:          1% per trade
Min Lot:         0.01
Sessions:        London + NY only (Asia OFF)
RSI Block:       ON (reclaim hold = 3 bars)
ATR Filter:      0.1% min
H4 Trend Gate:   ON  (4H MA Ribbon)
EMA Overlap:     ON  (block when spread <0.3%)
HTF Swing:       ON  (auto-bypassed on 15M/30M via LTF preset)
EMA Touch:       OFF (auto-forced ON on LTF via LTF preset)
Corner Lines:    ON  (H4 visual S/R)
Corner Gate:     OFF (optional)
LTF Preset:      ON  (auto-tune 15M/30M/1H — Plan B recovery)
  • Bypass HTF swing on 15M/30M
  • Force EMA touch on LTF
  • SL padding 50pt (was 100pt) on LTF
```

### Backtest Results (Nov 2025 – Apr 2026, ~6 months)

| TF | Trades | WR | PF | MaxDD | Net% |
|----|--------|----|----|-------|------|
| 15M | 12 | 16.7% | 0.09 | 3.7% | -306% |
| 30M | 24 | 29.2% | 0.59 | 6.5% | -85% |
| 1H | 23 | 21.7% | 0.23 | 5.5% | -251% |
| **4H** | **14** | **50%** | **1.11** | **4.4%** | **+18.2%** ✅ |

**Key insight:** 4H is the only profitable TF — system aligns with manual ("ตี TF สูงเท่านั้น"). LTF losses are amplified by oversized lot (0.01 minLot too big for $100 capital). v3 increases capital to $1000 + LTF preset to recover sub-4H performance.

### Recommended 4H XAUUSD Settings
```
Min Score:      3/5
TP Mode:        RR (SL-based)
RR:             2.0 / 3.0 / 4.0 / 5.0
SL Multiplier:  2.0× ATR
Trail SL:       Step (TP→prior TP)
RSI Big-TF:     ON (RSI filter only on ≥1H)
```

---

## File Summary

```
/home/user/JienAdam/
├── samurai_trading_system.pine   # Live indicator (Pine v6)
├── samurai_strategy.pine         # Backtest strategy (Pine v6)
├── PROJECT.md                    # This file
└── CLAUDE.md                     # AI coding guidelines
```

---

## Key RSI Logic (Manual Reference)

> RSI < 30 = market is choosing DOWN direction = **SELL bias, block BUY**  
> RSI > 70 = market is choosing UP direction = **BUY bias, block SELL**  
> These are trend confirmations, NOT reversal signals.  
> After RSI exits the extreme zone, wait N bars before allowing opposite-direction entry (reclaim & hold).
