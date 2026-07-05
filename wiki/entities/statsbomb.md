---
title: StatsBomb
type: entity
created: 2026-07-04
updated: 2026-07-04
sources: []
tags: [statsbomb, dataset, soccer, football, event-data, open-data]
---

# StatsBomb

**StatsBomb** is a soccer analytics data company. What matters for this wiki is **StatsBomb Open Data**: a free, publicly released set of detailed event data (JSON) at https://github.com/statsbomb/open-data, accessed in Python through the `statsbombpy` client (`sb.competitions()`, `sb.matches()`, `sb.events()`, `sb.lineups()`). It is the data behind the [[soccer-passing-hive-plots]] exploration.

## Event data

Event data records every on-ball action (passes with start *and* end locations, shots, carries, pressures, duels, and more) on a **120×80** pitch, normalized so both teams attack toward `x=120`. StatsBomb also ships **"360"** freeze-frame data for some matches, and the proprietary **OBV** possession-value metric (see [[expected-threat]] for the possession-value idea in its open form).

## Licensing

Free for **non-commercial use with attribution** ("Data provided by StatsBomb"), and **not for betting**.

## Open-data coverage (checked 2026)

- Men's World Cups (including 2022)
- UEFA Euro 2024
- Copa América 2024
- Champions League **finals** 1999/2000–2018/2019 (including the Messi-era Barcelona finals)
- Messi-era La Liga (Barcelona-only matches)
- Various women's competitions

**No Premier League**, and **no live / current-tournament data** (that is StatsBomb's commercial product). Because open La Liga is Barcelona-only, "opponents" in any Barcelona study means teams that played Barcelona.

## Reusable gotchas

Hard-won from the [[soccer-passing-hive-plots]] prototype, kept because they are the real lessons:

- **`statsbombpy` returns events TYPE-GROUPED, not in match order.** Any sequence logic must sort by the `index` column first, or motif and turnover code silently reads the wrong order.
- **NaN conventions:** `pass_outcome` NaN = completed; `pass_type` NaN = open play; `under_pressure` is `True`-or-NaN, **never `False`**.
- **`possession` / `possession_team`** columns enable turnover analysis (possession switches team) and chain analysis (linking consecutive passes within a possession).
- **Carries** carry their own start *and* end locations, so pass-only figures under-tell how the ball actually moves.

## See Also

- [[soccer-passing-hive-plots]] — The exploration built on StatsBomb open data
- [[expected-threat]] — Possession value (xT), and the OBV commercial sibling StatsBomb ships
- [[flow-motifs]] — The passing-style fingerprint built from StatsBomb pass sequences
