---
title: "Soccer Passing Networks as Hive Plots (exploration)"
type: analysis
created: 2026-07-04
updated: 2026-07-04
sources: [hiveplotlib-python-repo]
tags: [hive-plot, soccer, football, passing-networks, datashader, expected-threat, flow-motifs, comparability, exploration, prototype]
---

# Soccer Passing Networks as Hive Plots (exploration)

> **Exploratory prototype, not a hiveplotlib feature.** This documents a throwaway repo
> (`hiveplotlib-futbol`, GitHub `gjkoplik/hiveplotlib-futbol`) that exercised [[hiveplotlib]] on
> soccer passing data. Nothing here shipped into the library; the canonical detail (round-by-round
> decisions, figure inventory, workstreams) is that repo's `PLAN.md` and its `notes/`. Filed here
> for the reusable takeaways, the hard-won design calls, and the prior-art framing, in case a figure
> graduates to a writeup. A sibling to [[nn-training-dynamics-p2cp-exploration]].

## The bet

Standard soccer passing networks place each node at that player's **average pitch position** and
draw passes as edges. That layout is spatially honest but produces a hairball, and crucially **two
teams' plots are not comparable**: the node positions differ from team to team and match to match,
so you can never lay two side by side and read them apples-to-apples. Analysts complain about
exactly this: average positions distort a deep-lying passing striker to look permanently deep, subs
and formation changes break the picture mid-match, every pass looks equally important, and no two
teams' plots share a frame.

A [[hive-plot]] throws away literal position and **[[fixed-layout-comparability|fixes the layout by structure]]**, which is
hiveplotlib's whole thesis. Soccer hands us that structure for free: bin the pitch into thirds. The
layout becomes identical for every team, match, and league, so plots compare directly. **Fixed-layout
comparability is the entire selling point**, and it maps cleanly onto the documented pass-network
complaints.

## The core model (the pass-endpoint model)

The encoding you can explain to a soccer person in one sentence: **every line is one pass, from where
it started to where it ended.**

- Two nodes per pass, its origin and its destination.
- Each node sits on its **third's axis** (defensive / middle / final third = the three axes).
- Position along the axis is **lateral distance from the pitch center** (`lat_abs = |y - 40|`): the
  inner end of the axis is play through the middle, the outer end is play near the touchlines.
- The pass is the edge.

No aggregation, no graph metrics, no player identity. Datashaded (each pass a curve, brightness =
density). This model **superseded** two earlier ones (see "What was tried and dropped"): a fixed
nine-zone-node model and a player-mean-node model. The "mean" in that second model was the tell that
it aggregated and locked nodes to people, which is not the point.

## Figure catalog

The repo's figures are gitignored, so they are described here by name, not embedded (the nn-viz
convention).

- **(a) Flagship two-team comparison** (`passes_wc2022.png`, `passes_euro2024.png`). Every pass as
  an arc, two teams side by side on one shared layout via a generic [[hive-plot-matrix|HivePlotMatrix]],
  datashaded nodes + edges, unified color, shared colorbars. Euro 2024 final is the stark one: Spain
  458 passes vs England 208, the possession gap quantified in the panel titles.
- **(b) Formation "lineups"** (`lineup_argentina.png`, `lineup_france.png`). A per-starter
  small-multiple grid laid out in the team's **actual formation** via a name-to-grid map
  (`position_to_grid`), so a 4-2-3-1 or 4-3-2-1 reads correctly. Only that player's passes per cell.
  The term was deliberately changed from "atlas" to **"lineups"** as the plainer word.
- **(c) Shot build-up** (`shot_buildup_wc2022.png`, `shot_buildup_euro2024.png`). Outcome-conditioned:
  only passes from shot-ending possessions. Spain 109 vs England 29; England's **defensive third is
  nearly empty**, a counter/set-piece fingerprint (danger from short, high possessions). The
  danger-possession ratio is 3.8:1 vs 2.2:1 overall, which is the direct answer to "volume does all
  the talking."
- **(d) Under-pressure split** (`pressure_wc2022.png`). Settled vs pressured passing as matrix rows.
  Argentina 5.6% of passes under pressure vs France 13.3%; France's pressured passing concentrates in
  midfield.
- **(e) Total-xT weighted** (`xt_wc2022.png`, `xt_euro2024.png`). Arcs weighted by expected threat
  gained (Karun Singh's 2018 grid), via count-replication proportional to positive xT. The winners
  created **~2x the threat** (Argentina 1.63 vs France 0.77; Spain 1.41 vs England 0.68); sterile
  back-line circulation fades out entirely.
- **(f) Mean-xT / danger-per-pass** (`xt_mean_wc2022.png`, `xt_mean_euro2024.png`). The complementary
  lens: brightness is the *average* xT per pass through a corridor, via the native column-reduction
  path (below). England 3.3 vs Spain 3.1 milli-xT per pass, so England was as **dangerous per pass**
  despite half the volume. The brightest arcs everywhere are the rare direct balls into and within
  the final third, not the busy midfield.
- **(g) Motifs in space** (`motifs_euro2024.png`). A 2x5 grid (two teams x five motif classes).
  Gyarmati-style 3-pass motifs (ABAB / ABAC / ABCA / ABCB / ABCD) on the fixed layout. Spain over
  England on every ball-reuse motif, England skews ABCD; loudest signal is volume (337 vs 118 clean
  triples). The **spatial dimension is the part the motif literature never shows** (see novelty).
- **(h) Turnovers** (`turnovers_wc2022.png`, `turnovers_euro2024.png`). Possession-loss points on the
  fixed layout. In both finals possession overwhelmingly dies in the **attacking third** (Argentina 44
  of 72, Spain 28 of 46), with deep own-third losses rare. The 2022 midfield is the one asymmetry
  (France coughed it up 27 times to Argentina's 19).
- **(i) Barcelona motif seasons** (`barca_motifs.png`). 45 peak-Guardiola La Liga matches (2008/09,
  2009/10, 2010/11), the Gyarmati "tiki-taka is an outlier" replication at scale. Barcelona's
  reuse-motif share 36.3% vs opponents' 29.7% (mirroring the single-match Euro figure, Spain 35.2 vs
  England 29.6). Volume is the bigger outlier: 337 triples per match to opponents' 67, a 5x rate.

## Design decisions with rationale (the hard-won ones)

These took many rounds to settle. Recorded so they are not re-litigated.

- **Within-axis sort is lateral distance-from-center (`lat_abs`), not raw left-right.** Handedness of
  StatsBomb's `y=0` was never verified, and left/right visually flips by axis quadrant, which is
  confusing. Distance-from-center makes the central-vs-wide read clean and is immune to the
  unresolved handedness. `lat_y` is kept on every node, so flipping to a flank-bias view is one token.
- **Height / vertical sort was tried and rejected as disingenuous.** Sorting within an axis by pitch
  `x` makes cross-third passes hug the axis boundaries, but that is mostly a **geometric artifact**:
  most passes are short, so a pass crossing any line straddles it closely, wherever the line sits.
  The thirds are our imposed bins, so the pattern shows our binning, not the team, and can't support
  a "players think in thirds" read. An empirical check confirms it: a 10 m histogram of pass
  x-positions is smooth and unimodal (no break at the halfway line), so **any vertical banding is a
  communication choice, not something in the data**. If "do players think in thirds" is ever worth
  chasing, the honest test is that continuous histogram, not a height sort.
- **Axis ranges are pinned to absolute pitch bounds at construction**, not inferred from data
  (`HivePlot(axis_kwargs=...)`). This was a real bug: hiveplotlib's default infers each axis's
  vmin/vmax from the data on it, so a centrally-playing team gets its narrow band stretched to the
  touchlines, breaking cross-team comparability. Pinning identical bounds on every cell also unifies
  the matrix by construction (so `unify_axes` is dropped).
- **Thirds are the default binning** because they match analyst vocabulary ("final third", "zone 14").
  A friend's half / third-quarter / final-quarter bands are kept as an experiment (`make bands`), and
  arguably better for a danger-focused story, but thirds stay the default.

## Datashader note worth recording

hiveplotlib's datashader default aggregator is **count** (what every figure here uses). Column-based
reductions are opt-in and, **by design**, density-corrected: passing `reduction=ds.sum("col")`
rasterizes the column sum and divides by a count raster, giving the per-pixel **mean** of the passes
crossing it, no overlap double-encoding. This is the intended way to color edges by a variable. The
**mean-xT figure (f)** uses that path directly; the **total-xT figure (e)** wants *accumulated*
threat, so it uses count-replication instead. Raw xT deltas (~0.001-0.26) sit far below the library's
default `LogNorm` `vmin=1`, so the mean-xT figure needs an explicit norm (values scaled to milli-xT).

## What analysts value, and how it maps

The practice-research synthesis (a web-research probe on what practitioners actually use):

| Analyst concept | Hive-plot mapping |
| --- | --- |
| Zone thinking (thirds, zone 14, half-spaces, juego de posición) | Our fixed grid, the strongest conceptual fit |
| Progressive passes (FBref) | Cross-axis arcs; a progressive-only filter is native |
| Possession value ([[expected-threat\|xT]] on a fixed grid, StatsBomb OBV) | Value-weighted edges, the native fix for "all passes look equal" |
| "High turnovers" (ball won within 40 m of goal) | Turnover location |
| [[flow-motifs\|Flow motifs]] (Gyarmati) | Spatial motifs (figure g) |

Poor fits, stated honestly: **PPDA / pressing intensity** (off-ball, not in passing geometry) and
**player-centrality "who is the hub"** (our layout erases identity by design; the lineup grid only
partially recovers it).

## Novelty and prior art (honest, none-located-not-provably-first)

Two defensibly-novel angles, on the same standard as nn-viz ("novel" = none located after a scan, not
provably first):

1. **Fixed-layout comparability for passing networks.** All located passing-network viz uses
   pitch-positioned node-link layouts (Peña & Touchette 2012, arXiv 1206.6904; `mplsoccer`;
   Soccermatics). None fixes the layout by structure to make teams comparable, which is the documented
   fix for the average-position complaints.
2. **Spatial flow-motifs.** The Gyarmati / Kwak / Rodriguez 2014 motif literature (arXiv 1409.0308) is
   entirely **aspatial**: nobody shows *where* motifs happen. Chains on our fixed layout are novel on
   exactly that axis.

Caveat: the underlying object is a readable rendering of the academic "pitch-passing network" (the
zone-to-zone matrix analysts already use), so this is a new *rendering*, not a new object. Inline
cites: Gyarmati et al. 2014 (flow motifs); Karun Singh 2018 (xT, karun.in/blog/expected-threat.html);
StatsBomb OBV; zone-14 practitioner references. URLs live in `PLAN.md`'s "Research round" and
"analyst-practice" sections.

## Data and methodology

[[statsbomb|StatsBomb]] open data via `statsbombpy`. Pitch 120x80, both teams normalized to attack toward x=120
(verified: shots originate at mean x ~104 for both sides). Open-play completed passes by default
(`pass_outcome` NaN = completed, `pass_type` NaN = open-play). The xT grid is Karun Singh's published
12x8 JSON, fetched and cached at runtime, orientation empirically asserted.

**Reusable gotchas (the real lessons):**

- **`statsbombpy` returns events TYPE-GROUPED, not in match order. Any sequence logic must sort by the
  `index` column first.** This bit the motif and turnover code.
- NaN conventions: `pass_outcome` NaN = completed, `pass_type` NaN = open-play, `under_pressure` is
  True-or-NaN, **never False**.
- `possession` / `possession_team` columns enable both turnovers (possession switches team) and chains
  (~93% of consecutive pass pairs have clean recipient == next-passer). The last event of a lost
  possession is usually the *winner's* Ball Receipt / Goal Keeper event, so "where lost" must use the
  losing team's final located action.
- Carries are ~940 per match with start + end locations, ~20% progressive, so pass-only figures
  **under-tell** how the ball actually moves.
- `statsbombpy` has no on-disk cache; the Barcelona 45-match study added a per-match parquet cache
  (`data/passes_cache/`, gitignored) to make reruns instant.

**Data availability** (drives every dataset choice): WC 2022, Euro 2024, Copa América 2024, Champions
League **finals** 1999/2000-2018/2019 (including Messi-era Barcelona finals), La Liga Messi seasons.
**No Premier League** (so no Thiago-at-Liverpool), **no live / 2026 data** (live is StatsBomb's
commercial product). Open La Liga is **Barcelona-only** matches, so "opponents" in the Barca motif
study means teams that played Barcelona.

## What was tried and dropped

So the next person does not re-litigate:

- **Channel-partition matrix** (`from_partition` on left/center/right): mostly short lateral passes,
  the "Other" collapse muddied it. Deleted.
- **Directional split as a sort** (progression vs recycle): collapses each axis to three spots.
  Deleted.
- **Height / vertical sort:** the disingenuous boundary artifact above. Removed along with its
  machinery.
- **Earlier zone-node and player-mean-node models:** both superseded by the pass-endpoint model.

## Status and open questions

Exploratory prototype in `hiveplotlib-futbol`, nothing shipped into the library. Canonical detail is
that repo's `PLAN.md`.

**Graduation candidates** for a gallery notebook or tutorial writeup: shot build-up, the two xT lenses
side by side, and the Barcelona motifs.

**Open:**

- Turnovers over a whole season (Barcelona La Liga is in open data).
- A marquee-player lineup (single-player filter, a one-call change today).
- The **time dimension**: this WC final's late momentum swing (France dormant until ~80', then two
  goals in 97 seconds) as a per-phase matrix. Cheap to add, changes no core machinery.
- Verifying `y=0` handedness before any "builds down the left" claim.

## See Also

- [[examples-and-applications]] — Catalog of hiveplotlib example/application explorations (this is one)
- [[nn-training-dynamics-p2cp-exploration]] — Sibling exploration (a tiny MLP learning MNIST as a P2CP movie)
- [[hive-plot]] — The fixed-layout method the whole bet rests on
- [[hiveplotlib]] — The library exercised
- [[hive-plot-matrix]] — The comparative grid every multi-panel figure here is built on
- [[edge-rendering]] — How hiveplotlib draws edges (and the datashader path the xT figures use)
- [[p2cp]] — The sibling encoding (used by the nn-viz exploration, not here)
- [[differential-hive-plot]] — The two-network diff method the comparison figures gesture at
- [[expected-threat]] — xT, the possession-value model behind the value-weighted figures
- [[flow-motifs]] — The 3-pass motif fingerprint the spatial-motif figure renders
