---
name: pse-update-oco-json
description: >-
  Compute and record the bottom (stop-loss) leg of an OCO sell order for a
  Philippine Stock Exchange (PSE) stock — the Trailing Offset % and Limit Price
  % — by researching up-to-date PSE price action, liquidity, volatility, and the
  14-day ATR, writing a human-verifiable research note under research/, and
  adding or updating that ticker's entry in ocoSellOrderValues.json. Use this
  whenever the user gives one or more PSE tickers and wants OCO / trailing-stop
  / stop-loss sell values computed or refreshed, asks to update
  ocoSellOrderValues.json, or wants PSE stocks researched for a swing-trade exit
  — e.g. "update OCO for ICT", "compute the trailing stop for ALI", "refresh
  sell values for JFC", "add SCC to the oco json", "update OCO for ICT, ALI and
  SCC". Triggers on one or more bare tickers plus "OCO" / "trailing offset" /
  "limit price" / "sell order" even when the JSON file is not named.
---

# PSE OCO Sell-Order Updater

This skill prepares the **bottom leg of an OCO (one-cancels-the-other) sell order** for a
swing trade on the Philippine Stock Exchange. The bottom leg is a **trailing stop-limit**:

- **Trailing Offset %** — how far below the high-water mark the stop *triggers*.
- **Limit Price %** — the *execution buffer*: how far below the triggered stop the limit
  sits so the order actually fills in a fast drop.

The deliverables are (1) a dated, human-verifiable research note in `research/`, and
(2) an added or updated entry in `ocoSellOrderValues.json`.

## The arguments

This skill takes **one or more PSE ticker symbols** (e.g. `ICT`, or `ICT ALI SCC`). Accept
them however the user supplies them — space-, comma-, or "and"-separated — and **normalize
each to uppercase**. De-duplicate repeated tickers. If the user supplied no ticker, ask for
at least one before doing anything else — every step depends on it.

When more than one ticker is given, run the **full per-ticker workflow (steps 1–7) once for
each ticker, independently**: each ticker gets its own research note and its own entry in
`ocoSellOrderValues.json`. Do **step 0 (the date) once** up front and reuse the same date for
every ticker. Process the tickers in the order given. If one ticker can't be verified
(wrong-exchange ambiguity, no reliable data, halted), **note it, skip it, and continue with
the rest** — don't abort the whole batch because one failed. The final report (step 8) then
covers all tickers together.

## Files this skill touches

- `ocoSellOrderValues.json` (repo root) — the data file you add to or update.
- `research/<TICKER>-<YYYY-MM-DD>.md` — a new research note you create each run.
- `.claude/skills/pse-update-oco-json/references/methodology.md` — the detailed formula
  rationale, worked examples, and edge-case guidance. **Read it before computing** so your
  adjustments are consistent and well-justified.
- `.claude/skills/pse-update-oco-json/assets/research-template.md` — the skeleton to copy
  into the new research note.

## Workflow

**For a single ticker, run steps 0–9 once.** For **multiple tickers**, run step 0 once, then
run steps 1–7 once per ticker (fully completing one ticker — research note + JSON entry —
before moving to the next keeps each audit trail self-contained), and run steps 8–9 once at
the end to summarize the whole batch and push it.

### 0. Get today's date (reliably) — once for the whole batch

Don't guess the date. Run this once and reuse the result for every ticker:

```powershell
Get-Date -Format "d-MMM-yyyy"   # e.g. 30-May-2026  → used in suggestedBy and the note
Get-Date -Format "yyyy-MM-dd"   # e.g. 2026-05-30   → used in the research filename
```

### 1. Research the stock — and confirm it is the PSE listing

Search the web for the **most recent daily price action** for the ticker. Because the same
symbol often exists on other exchanges, **disambiguation is mandatory** — getting this wrong
poisons every downstream number:

- Always include "PSE" / "Philippine Stock Exchange" / the company's Philippine name in
  queries. The official source `edge.pse.com.ph` is authoritative for quotes.
- **Verify the quote is in Philippine pesos (₱ / PHP)** and the company is the Philippine
  one before trusting any figure.
- Watch for collisions, e.g. `AP` = Aboitiz Power (PSE) vs Ampco-Pittsburgh (NYSE: AP);
  `AC` = Ayala Corp (PSE) vs Associated Capital (NYSE: AC). When a number looks off by an
  order of magnitude, suspect a wrong-exchange match.

Gather and record (with a source URL and as-of date for each):

- **Last close** and the date of that close.
- **Recent daily OHLC** for ~15 trading days (needed to compute the 14-day ATR).
- **Liquidity** — average daily value turnover (₱ traded/day) or average volume.
- **Volatility** read — recent daily % swings, any gaps, news-driven spikes.
- **14-day ATR** if a source publishes one (note whether it's Wilder's or simple).

Cross-check across **at least two sources** where you can. If reliable data genuinely can't
be found (thin/illiquid name, halted, recent IPO), say so plainly in the note, be more
conservative, and flag low confidence rather than inventing numbers.

### 2. Establish ATR% — the engine of both values

`ATR% = (14-day ATR ÷ current price) × 100`

- If a source publishes a trustworthy 14-day ATR, use it and cite it.
- Otherwise compute it from the OHLC series and **show the work in the note** (a human must
  be able to re-derive it). True Range per day = max of: `High − Low`, `|High − PrevClose|`,
  `|Low − PrevClose|`. The 14-day ATR is the **simple average of the last 14 daily TRs**
  (the easiest form to verify by hand; Wilder's smoothing is acceptable if that's what a
  cited source uses — just say which).

### 3. Compute the Trailing Offset %

Baseline (tight posture, per the configured risk preference):

```
Trailing Offset % = 2.0 × ATR%
```

Then apply **judgment** (this is a hybrid methodology, not a rigid formula) and **log every
deviation from the 2.0× baseline in the note with its reason**:

- Widen for thin liquidity, wide spreads, or gappy/news-driven action.
- Respect a soft floor of ~2% and a soft ceiling of ~12.5% (the observed range in the data).
- Round to the nearest **0.5** (finer granularity like 3.2 or 9.5 is fine when it better
  reflects the computed value).

See `references/methodology.md` for the reasoning and worked examples.

### 4. Compute the Limit Price % (execution buffer)

Baseline:

```
Limit Price % = Trailing Offset ÷ 5     (≈ 20% of the offset; the 15–20% band is normal)
```

Then adjust and log reasons:

- **Widen** for illiquid / wide-spread names so the stop-limit still fills when price is
  falling fast; **tighten** for very liquid blue chips.
- Ensure the buffer in pesos is **at least one PSE tick plus the typical bid–ask spread** at
  the current price (PSE tick size widens with price band; if the price sits near a band
  edge, verify the tick against PSE's official schedule). A buffer thinner than the spread
  may never fill.
- Floor of **0.25%**, round to the nearest **0.25**.

### 5. Write the research note

Copy `assets/research-template.md` to `research/<TICKER>-<YYYY-MM-DD>.md` and fill in every
section: gathered data with source URLs and as-of dates, the PSE-disambiguation evidence,
the full ATR / offset / limit arithmetic, and the verification checklist. The goal is that
**a human can later check each value by hand** — leave nothing as an unexplained jump.

If a note for this ticker and date already exists, append `-v2`, `-v3`, … rather than
overwriting.

### 6. Add or update ocoSellOrderValues.json

Read the file first to confirm its current shape. Each entry is:

```json
"TICKER": {
    "trailingLeg": {
        "trailingOffsetPercent": <number>,
        "limitPricePercent": <number>,
        "suggestedBy": "<model> via <tool> (<DD-Mon-YYYY>)"
    }
}
```

- **New ticker** → insert the block in its correct **alphabetical position** among its
  neighbors. Use a targeted edit at the right spot; do **not** rewrite or re-sort the whole
  file (that creates a noisy diff and can disturb other entries). Match the existing 4-space
  indentation exactly, and keep the trailing comma rules valid.
- **Existing ticker** → edit just the three values in place. Additionally, fill in the
  **"Why these values are better than the previous entry"** section of the research note:
  old vs new for each field, and what changed in the data (volatility / liquidity) that
  justifies the change.

When processing several tickers, still apply **one targeted edit per ticker** at that
ticker's own alphabetical spot — don't batch them into a single rewrite of the file. Re-read
the file between inserts if a prior insert shifted line numbers.

### 7. Set `suggestedBy`

Format: `"<your model name> via <tool> (<DD-Mon-YYYY>)"` using the date from step 0. When
running in this environment that is, e.g., `"Claude Opus 4.8 via Claude Code (30-May-2026)"`.
Use whatever model is actually running — don't hardcode a model you aren't.

### 8. Report back

Give the user a concise summary **for each ticker processed**: company name + confirmation
it's the PSE listing, the key data (last close, ATR%, liquidity read), the computed
**Trailing Offset %** and **Limit Price %**, whether the entry was **added or updated**
(old → new if updated), and the path to the research note. Remind them each note is the audit
trail for manual verification.

When you processed **multiple tickers**, lead with a compact summary table (one row per
ticker: ticker, offset %, limit %, added/updated) and then the per-ticker detail. **Call out
any ticker you skipped** and why (couldn't confirm PSE listing, no reliable data, halted), so
the batch result is unambiguous.

### 9. Commit and push directly to `main`

This repo's workflow lands OCO updates **straight on the `main` branch** — no feature branch,
no PR. As the final step, stage the research note(s) and the JSON together and push:

```powershell
git add ocoSellOrderValues.json research/
git commit -m "Update OCO sell-order values for ICT, ALI, SCC (2-Jun-2026)"
git push origin main
```

- **Commit the data and its audit trail in one commit** — the `ocoSellOrderValues.json`
  change and the matching `research/<TICKER>-<DATE>.md` note(s) belong together.
- **Commit message** — follow the existing history: name the tickers and the date, e.g.
  `Update OCO sell-order values for ICT, ALI, SCC (2-Jun-2026)`, or `Add ...` when the run
  introduces brand-new tickers. End the message with the standard co-author trailer
  (`Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>`).
- **Only commit the files this run actually changed.** Don't sweep in unrelated working-tree
  edits — stage `ocoSellOrderValues.json` and the specific note(s) explicitly.
- If `git push` is rejected because `main` advanced upstream, `git pull --rebase origin main`
  and push again.
- If any ticker was skipped, still push the ones that succeeded; just reflect that in the
  commit message.

## Guardrails

- **Right exchange or nothing.** If you cannot positively confirm the data is the PSE,
  PHP-denominated listing, stop and say so. A confident wrong number is worse than "I
  couldn't verify this."
- **Never fabricate market data.** Every figure in the note needs a source. Gaps get flagged,
  not filled with guesses.
- **Verifiability over cleverness.** Show all arithmetic; a value the user can't re-derive
  from the note is a bug.
