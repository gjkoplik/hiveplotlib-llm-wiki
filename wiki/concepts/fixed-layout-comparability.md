---
title: Fixed-Layout Comparability
type: concept
created: 2026-07-04
updated: 2026-07-04
sources: [krzywinski-2012]
tags: [hive-plot, comparability, network-comparison, small-multiples, design-principle]
---

# Fixed-Layout Comparability

**Fixed-layout comparability** is a general [[hive-plot]] design principle (not soccer-specific, but surfaced sharply by the [[soccer-passing-hive-plots]] work): because a hive plot places nodes on axes by *meaningful structural properties* rather than by a force-directed or data-derived embedding, you can fix the axis definitions and node ordering up front. Two **different** networks rendered on that same fixed frame are then directly comparable: the panels line up, and only the edges and their density differ.

This is what force-directed layouts cannot give. A [[force-directed-layout|force layout]] hands every graph its own idiosyncratic embedding, so no two plots share a frame and side-by-side comparison is meaningless. Fixing the frame by structure is the whole point (introduced with the hive plot itself in Krzywinski 2012; see [[hive-plot]]).

## The load-bearing example

Standard soccer passing networks place each node at that player's **average pitch position**, so no two teams' plots share a frame and you can never lay them apples-to-apples. Fixing the layout by **pitch third** (defensive / middle / final third as the three axes) makes the frame identical for every team, match, and league. That is the entire selling point of the [[soccer-passing-hive-plots]] exploration.

## A practical corollary: pin the axis ranges

A frame that is *nominally* fixed still lies if each panel infers its own axis value-range from its own data. A sparse panel then stretches its narrow band to fill the axis, and the comparison silently distorts. The soccer work learned to **pin axis value-ranges to fixed absolute bounds** at construction (`HivePlot(axis_kwargs=...)`), never per-panel from data. Pinning identical bounds on every cell also unifies a multi-panel matrix by construction.

## Related, but distinct

- [[differential-hive-plot]] renders the *difference* of two networks on a single plot (added / removed / preserved edges). It is a stronger, single-plot form of the same comparability goal: instead of asking the reader to compare two panels, it computes the comparison for them.
- [[hive-plot-matrix|HivePlotMatrix]] is the multi-panel structure that *operationalizes* side-by-side comparison. Fixed-layout comparability is the property that makes those panels worth putting next to each other.

Fixed-layout comparability is also why hive plots are the natural tool for exposing [[structural-heterogeneity|structural heterogeneity]]: the same structural axis definitions that let you compare two networks let you compare structural subgroups within one.

## See Also

- [[hive-plot]] — The method whose structural axes make the frame fixable (Krzywinski 2012)
- [[differential-hive-plot]] — The single-plot diff form of the same comparability goal
- [[hive-plot-matrix]] — The multi-panel structure comparability operationalizes
- [[soccer-passing-hive-plots]] — The exploration where this principle is load-bearing
- [[structural-heterogeneity]] — What a fixed structural frame is designed to expose
