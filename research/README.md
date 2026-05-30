# research/

Human-verifiable research notes backing each entry in `../ocoSellOrderValues.json`.

One note is created per run of the **pse-update-oco-json** skill, named
`<TICKER>-<YYYY-MM-DD>.md` (e.g. `ICT-2026-05-30.md`). Each note records the market data
gathered (with source URLs and as-of dates), proof the data is the PSE listing, the full ATR
/ Trailing Offset / Limit Price arithmetic, and — when an existing entry was changed — why
the new values beat the old ones.

The goal is that every value in `ocoSellOrderValues.json` can be checked by hand from its
note.
