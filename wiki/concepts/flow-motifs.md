---
title: Flow Motifs
type: concept
created: 2026-07-04
updated: 2026-07-04
sources: []
tags: [flow-motifs, passing-networks, team-style, soccer, football, tiki-taka]
---

# Flow Motifs

**Flow motifs** are a method for fingerprinting a soccer team's passing style by the *player-identity pattern* of short pass sequences. The idea comes from László Gyarmati, Haewoon Kwak, and Pablo Rodriguez, "Searching for a Unique Style in Soccer" (2014, arXiv 1409.0308).

## The method

Take consecutive **3-pass sequences** (overlapping windows within a single possession), each involving up to four players. Relabel the players in order of first appearance to letters. That relabeling collapses every 3-pass sequence into exactly **five motif classes**:

| Motif | Pattern | Character |
| --- | --- | --- |
| ABAB | two players, ball bounced back and forth | maximal reuse |
| ABAC | returns to A, then out to a new C | reuse |
| ABCA | out through B and C, back to A | circulation |
| ABCB | returns to B | reuse |
| ABCD | four distinct players, no repeat | maximal spread |

A team's **distribution over these five classes** is a style fingerprint. The paper's headline finding: Barcelona's *tiki-taka* is a statistical outlier, skewed toward ball-reuse motifs (ABAB / ABAC / ABCB) rather than the four-player ABCD spread.

The literature is **aspatial**. It uses motif *frequencies* only and never asks *where on the pitch* a motif happens. A player-level extension of the idea exists in the same literature (fingerprinting individual players' passing styles, e.g. asking which players are stylistically interchangeable), noted here as related rather than asserted in detail (the specific citation was not independently verified).

## Mapping to hive plots

Drawing each motif class's passes on a **fixed [[hive-plot|hive-plot]] layout** adds the spatial dimension the frequency-only literature lacks: you see not just *how often* a team reuses the ball but *where* it does. The [[soccer-passing-hive-plots]] prototype renders a two-team-by-five-motif grid (its "motifs in space" figure) and replicates the Gyarmati outlier finding at scale across 45 Guardiola-era Barcelona matches: a reuse-motif share of **36.3% for Barcelona vs 29.7% for opponents**, mirroring the single-match Euro-final split. Because the layout is fixed by pitch third, the motif panels for two teams sit on the same frame and read apples-to-apples (see [[fixed-layout-comparability]]).

## See Also

- [[soccer-passing-hive-plots]] — The exploration that renders motifs spatially
- [[expected-threat]] — The other soccer-analytics concept the prototype maps onto edges
- [[fixed-layout-comparability]] — Why the fixed layout makes motif panels comparable across teams
- [[statsbomb]] — The event-data source the motif sequences are built from
