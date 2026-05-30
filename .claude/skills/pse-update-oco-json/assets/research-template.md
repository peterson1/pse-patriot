# OCO Sell-Order Research — <TICKER> (<Company Name>)

- **Exchange:** Philippine Stock Exchange (PSE) — quote confirmed in PHP (₱)
- **Research date:** <DD-Mon-YYYY>
- **Prepared by:** <model> via <tool>
- **Entry type:** New entry | Updated entry

## 1. Market data gathered

| Metric | Value | As of | Source |
|---|---|---|---|
| Last close | ₱<price> | <date> | <url> |
| Recent daily % moves | <e.g. -1.2%, +0.8%, ...> | <range> | <url> |
| Recent N-day high / low | ₱<hi> / ₱<lo> | <range> | <url> |
| Avg daily value turnover (liquidity) | ₱<value>/day | <period> | <url> |
| Volatility note | <gaps? spikes? steady?> | <range> | <url> |
| 14-day ATR (published, if used) | ₱<atr> (<Wilder/simple>) | <date> | <url> |

### PSE disambiguation
<State how you confirmed this is the PSE listing and not a same-symbol stock on another
exchange: the company's full Philippine name, the currency (PHP), the exchange tag/source,
and any sanity check on the price magnitude.>

## 2. 14-day ATR

<If you used a published ATR: state the value, the method (Wilder's vs simple), and the
source. If you computed it, fill the table below and show the average.>

| Date | High | Low | Prev Close | True Range |
|---|---|---|---|---|
| <d1> | | | | |
| <…>  | | | | |
| <d14>| | | | |

- **Sum of 14 True Ranges:** ₱<sum>
- **14-day ATR = sum ÷ 14 =** ₱<atr>
- **Current price:** ₱<price>
- **ATR% = ATR ÷ price × 100 =** **<atr_pct>%**

## 3. Trailing Offset computation

- Baseline (tight posture): `2.0 × ATR% = 2.0 × <atr_pct>% = <raw>%`
- Adjustments applied: <liquidity / gappiness / floor / ceiling — or "none">
- Reason for each adjustment: <…>
- Rounding: `<raw>% → ` **`<final>%`**
- **Trailing Offset = <final>%**

## 4. Limit Price (execution buffer) computation

- Baseline: `Trailing Offset ÷ 5 = <offset> ÷ 5 = <raw>%`
- Adjustments applied: <spread / liquidity / tick floor — or "none">
- Reason for each adjustment: <…>
- PSE tick check: at ₱<price>, one tick ≈ ₱<tick> (<tick_pct>%); buffer ≥ 1 tick + spread? <yes/no>
- Rounding: `<raw>% → ` **`<final>%`**
- **Limit Price = <final>%**

## 5. Why these values are better than the previous entry
<Delete this whole section for a brand-new ticker.>

| Field | Previous | New | Reason it's better |
|---|---|---|---|
| trailingOffsetPercent | <old> | <new> | <what changed in the data> |
| limitPricePercent | <old> | <new> | <what changed in the data> |

<Narrative: what shifted in volatility, liquidity, or price level since the previous entry
(<previous suggestedBy date>) that makes the new values a better fit.>

## 6. Values written to ocoSellOrderValues.json

```json
"<TICKER>": {
    "trailingLeg": {
        "trailingOffsetPercent": <offset>,
        "limitPricePercent": <limit>,
        "suggestedBy": "<model> via <tool> (<DD-Mon-YYYY>)"
    }
}
```

## 7. Sources
- <url 1 — what it provided>
- <url 2 — what it provided>

## 8. Verification checklist (for the human)
- [ ] Confirm the data is the PSE listing (PHP), not a same-symbol foreign stock
- [ ] Spot-check the last close and the ATR against the cited sources
- [ ] Re-derive ATR% and the 2.0× offset
- [ ] Confirm the limit buffer covers the typical spread / ≥ 1 PSE tick
