---
title: "Chen et al. 2018 — Same Stats, Different Graphs (Kobourov group)"
type: source
created: 2026-07-04
updated: 2026-07-04
sources: [kobourov-2018-same-stats-different-graphs]
tags: [graph-drawing, graph-statistics, anscombe, datasaurus, network-visualization]
---

# Chen et al. 2018 — Same Stats, Different Graphs (Kobourov group)

**Citation:** Chen, H., Soni, U., Lu, Y., Huroyan, V., Maciejewski, R. and Kobourov, S., 2018. Same Stats, Different Graphs (Graph Statistics and Why We Need Graph Drawings). *Graph Drawing and Network Visualization (GD 2018)*, arXiv:1808.09913. Extended version 2019, arXiv:1911.01527.

## Summary

The network analog of Anscombe's quartet: sets of graphs that are identical across common graph statistics (degree distribution, clustering, and so on) yet structurally distinct, with the difference revealed only by drawing them. The point is that summary statistics under-determine graph structure, so a picture is needed to tell matched-statistic graphs apart. The reveal medium is a **spring / node-link** layout; there is no hive or radial encoding. Anscombe-motivated by the authors' own framing, and part of a celebrated device lineage (Anscombe 1973, the Datasaurus, Gelman's causal quartets), with no credible "gimmick" dismissal found.

## Bearing on the bioinformatics-examples record

Establishes that the "matched statistics, structurally different, revealed by drawing" device already exists *for networks*. This bounds hiveplotlib's honest claim: the Datasaurus-for-networks idea is not the library's to claim. The narrow, defensible delta is the **deterministic hive-plot reveal medium**, a layout that is a function of the data (a degree-rank ordering applied reproducibly) rather than a seed-dependent node-link drawing. See [[hiveplotlib-bioinformatics-examples]] and [[same-stats-different-graphs]].

## See Also

- [[hiveplotlib-bioinformatics-examples]] — The exploration that positions its delta against this device
- [[same-stats-different-graphs]] — The Datasaurus-for-networks plan (the hiveplotlib hero)
- [[same-stats-different-graphs-grounding]] — The grounding run behind that plan
- [[force-directed-layout]] — The spring / node-link family this work uses as its reveal
