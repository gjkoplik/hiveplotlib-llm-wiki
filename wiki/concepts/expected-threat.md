---
title: Expected Threat (xT)
type: concept
created: 2026-07-04
updated: 2026-07-04
sources: []
tags: [expected-threat, possession-value, soccer, football, xt, obv]
---

# Expected Threat (xT)

**Expected Threat (xT)** is a grid-based possession-value model for soccer, introduced by Karun Singh in 2018 ("Introducing Expected Threat (xT)", https://karun.in/blog/expected-threat.html). It assigns every location on the pitch a scalar "threat" value: how likely the team in possession is to score soon, given they have the ball there. An action's *xT gained* is the difference between where it ended and where it started, which is why xT separates dangerous progression from sterile circulation.

## The model

Divide the pitch into a grid of cells. The value of a cell is defined recursively:

```
value(cell) = P(shoot here) × P(goal | shoot here)
            + P(move the ball) × Σ_j P(move to cell j) × value(cell j)
```

A cell's value reflects both the chance of scoring directly from it and the chance of passing or carrying onward into a more valuable cell. Because `value` appears on both sides, this is a Markov model **solved by iteration** to a fixed point. Once the grid converges, the xT gained by a pass or carry is simply:

```
xT_gained = value(end cell) − value(start cell)
```

Singh published an open **12×8 grid** as JSON, which is the one the [[soccer-passing-hive-plots]] prototype fetched, cached, and used (orientation asserted empirically at load).

## Commercial sibling: On-Ball Value (OBV)

StatsBomb's proprietary **On-Ball Value (OBV)** is a similar possession-value model (see https://statsbomb.com/soccer-metrics/). The main conceptual difference is that OBV also **credits and debits defensive actions**, not just on-ball attacking moves, so it can value a tackle or interception that a pure xT grid does not touch. It is the commercial counterpart to the open xT idea; see [[statsbomb]] for the data company and its open-data offering.

## Mapping to hive plots

The classic complaint about unweighted passing networks is that **all passes look equal**: a sterile square ball across the back line draws the same edge as a line-breaking pass into the box. Weighting or coloring each pass arc by xT gained is the native fix. The [[soccer-passing-hive-plots]] prototype uses two complementary xT lenses on the fixed [[hive-plot|hive-plot]] layout:

- **Total-xT (accumulated threat).** Arcs weighted by positive xT gained (via count-replication), so back-line circulation fades out and threat-creating corridors light up. In both showcase finals the winning side created roughly twice the threat of the loser.
- **Mean-xT (danger per pass).** Brightness is the *average* xT per pass through a corridor, computed through hiveplotlib's density-corrected column-reduction datashader path. This can reveal a team that was as dangerous *per pass* despite far lower passing volume.

## See Also

- [[soccer-passing-hive-plots]] — The exploration that value-weights pass arcs by xT
- [[flow-motifs]] — The other soccer-analytics concept the prototype renders spatially
- [[fixed-layout-comparability]] — Why value-weighted edges on a fixed layout compare across teams
- [[statsbomb]] — The data source, and home of the OBV commercial sibling
