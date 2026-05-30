# OCO Trailing-Leg Methodology

This is the reasoning behind the two computed values. The approach is a **hybrid**: the
formulas below are the default, but you are expected to use judgment per stock and **record
any deviation and its reason in the research note**. The point of the formulas is consistency
and human-verifiability, not blind obedience.

## Why ATR drives everything

A swing-trade trailing stop should survive *normal* daily noise and only fire on a *real*
reversal. The 14-day Average True Range (ATR) is the market's own measure of that normal
daily noise, so both computed values are anchored to it. Expressing ATR as a percentage of
price (`ATR% = ATR ÷ price × 100`) makes the logic comparable across a ₱2 stock and a ₱400
stock.

### True Range and the 14-day ATR

For each day, True Range (TR) is the largest of:

1. `High − Low`
2. `|High − Previous Close|`  (captures an up-gap)
3. `|Low − Previous Close|`   (captures a down-gap)

The **14-day ATR** is the simple average of the last 14 daily TRs. This simple average is
preferred here because a human can re-add 14 numbers and divide — it is easy to audit.
Wilder's smoothed ATR (what many charting tools show) is also fine to *use* if you're citing
a source that publishes it; just note which method the number came from. You need ~15 trading
days of OHLC to get 14 TRs (day 1's TR needs the prior close).

## Trailing Offset %

**Risk posture: tight, baseline `2.0 × ATR%`.**

Two average days of adverse movement is roughly `2 × ATR`, so a 2× offset gives the trade
about two ATR-days of room before the stop triggers. Tight means it locks in gains quickly
and re-arms close to price; the trade-off the user has accepted is more risk of being
whipsawed out on an ordinary down day. Don't quietly widen it to "avoid whipsaw" — that
would undo the chosen posture. If you do widen, it must be for a structural reason (see
below), stated in the note.

### Adjustments (log each one you apply)

- **Liquidity / spread.** Thin names gap and have wide spreads; a stop placed too tight gets
  picked off by noise and fills poorly. Nudge the offset up for low turnover.
- **Gappiness / event risk.** Recent news-driven spikes or frequent gaps argue for a touch
  more room.
- **Soft bounds.** Floor ~2% (even the most liquid blue chips like SM sit near 2%), soft
  ceiling ~12.5% (the most speculative small caps). These are guides; a genuinely extreme
  name can sit outside, but justify it.

### Rounding

Round to the nearest **0.5**. Finer granularity (e.g. 3.2, 3.8, 9.5, 10.5) is acceptable when
the computed number clearly sits between halves and rounding would distort it.

## Limit Price % (execution buffer)

**Baseline `Trailing Offset ÷ 5`** (≈ 20% of the offset; the 15–20% band is the normal zone).

When the stop triggers, a plain stop-*limit* only fills if there are buyers at or above the
limit. In a fast drop the price can blow through a too-tight limit and leave you holding a
falling stock. The buffer is the insurance that the exit actually executes. It scales with
the offset because the offset already encodes the stock's volatility and liquidity.

### Adjustments (log each one you apply)

- **Widen for illiquid / wide-spread names.** The buffer must comfortably exceed the
  bid–ask spread, or the limit may sit inside the spread and never fill. Illiquid micro-caps
  can need 2–3%.
- **Tighten for very liquid blue chips.** Deep books fill at a tick or two; 0.25–0.5% is
  plenty (this is why SM/BDO/SMPH sit at 0.25 despite the formula suggesting more).
- **Tick awareness.** The buffer in pesos must be **≥ one PSE tick + typical spread** at the
  current price. PSE tick size increases with the price band, so a ₱500 stock needs a larger
  peso buffer than a ₱5 stock for the same percentage. If the price is near a band boundary,
  confirm the tick against the official PSE schedule.

### Rounding

Floor **0.25%**, round to the nearest **0.25**.

## Liquidity tiers (rough guide, not rigid)

- **Deep blue chip** (very high turnover): offset can sit at the floor; buffer 0.25–0.5%.
- **Mid cap** (moderate turnover): formula values usually apply as-is.
- **Thin / speculative** (low turnover, wide spread): widen both; flag confidence.

## Worked example (illustrative numbers)

A PSE stock trades at **₱100.00**; you computed a 14-day ATR of **₱2.50**, and turnover is
healthy (mid-cap).

1. `ATR% = 2.50 ÷ 100.00 × 100 = 2.5%`
2. `Trailing Offset = 2.0 × 2.5% = 5.0%` → no adjustment needed → **5.0%**
3. `Limit Price = 5.0 ÷ 5 = 1.0%` → spread is narrow, no widening → **1.0%**

Now a thinner name at **₱4.00** with ATR **₱0.26** and low turnover:

1. `ATR% = 0.26 ÷ 4.00 × 100 = 6.5%`
2. `Trailing Offset = 2.0 × 6.5% = 13.0%` → above the ~12.5% soft ceiling; the move is
   liquidity-driven noise, so cap at **12.5%** (logged).
3. `Limit Price = 12.5 ÷ 5 = 2.5%` → wide spread on a thin name confirms keeping it wide →
   **2.5%**. Sanity-check: at ₱4.00 the PSE tick is small, so 2.5% (₱0.10) clears it easily.

## Edge cases

- **No reliable ATR / OHLC.** Don't fabricate. State the limitation, lean conservative
  (wider buffer), and mark the entry low-confidence in the note.
- **Recently halted or just resumed.** Volatility is unstable; prefer a wider buffer and say
  why.
- **Recent IPO / short history.** Fewer than 14 sessions means the ATR is approximate —
  compute over the days available, label it, and treat the result as provisional.
- **Hit the floor/ceiling.** Fine to sit at 2% or 12.5%; just confirm the data supports it.
