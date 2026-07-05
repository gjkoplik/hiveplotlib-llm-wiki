---
title: "Atkins et al. 1998 — Spectral Algorithm for Seriation"
type: source
created: 2026-07-04
updated: 2026-07-04
sources: []
tags: [spectral-clustering, seriation, fiedler-vector, ordering, foundational-paper]
---

# Atkins, Boman & Hendrickson 1998 — A Spectral Algorithm for Seriation and the Consecutive Ones Problem

> **Stub:** based on web research (deep-research pass, 2026-07-04), not a full reading. Flagged for future full ingest.

## Citation

J.E. Atkins, E.G. Boman, B. Hendrickson. "A Spectral Algorithm for Seriation and the Consecutive Ones Problem." *SIAM Journal on Computing* 28(1), 1998. Modern treatment and robustness analysis: Fogel, Jenatton, Bach & d'Aspremont, "Spectral Ranking using Seriation," *JMLR* 17 (2016).

## Summary

The foundational, provably-correct result for **spectral seriation**: given a similarity matrix, sorting the entries of the **Fiedler vector** (the eigenvector of the second-smallest [[spectral-clustering|Laplacian]] eigenvalue) recovers the correct 1-D linear ordering of items in the noiseless (Robinson / pre-R matrix) case. Seriation is the canonical formalization of similarity-driven 1-D ordering: place items on a chain so that similarity decreases with distance.

## Relevance here

This is the prior art for the **within-axis ordering** component of [[spectral-hive-plots]]. Ordering nodes along an axis by a Laplacian eigenvector is not novel; it is exactly Fiedler-vector seriation, and "a 1998 theorem does not go stale." The spectral hive plot's twist is using a *higher* eigenvector (beyond the ones that define the cut) as the ordering, applied *per cluster*, which seriation does not itself prescribe.

## See Also

- [[spectral-hive-plots]] — uses eigenvector ordering as the within-axis sort
- [[spectral-clustering]] — Fiedler vector and Laplacian bookkeeping
- [[node-assignment]] — "where on the axis" is a 1-D ordering problem
