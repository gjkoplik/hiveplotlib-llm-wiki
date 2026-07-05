---
title: "Koren 2005 — Drawing Graphs by Eigenvectors"
type: source
created: 2026-07-04
updated: 2026-07-04
sources: []
tags: [spectral-clustering, graph-drawing, network-visualization, foundational-paper]
---

# Koren 2005 — Drawing Graphs by Eigenvectors: Theory and Practice

> **Stub:** based on web research (deep-research pass, 2026-07-04), not a full reading. Flagged for future full ingest.

## Citation

Yehuda Koren. "Drawing Graphs by Eigenvectors: Theory and Practice." *Computers & Mathematics with Applications* 49 (2005) 1867-1888. Attributes the earliest spectral graph-drawing algorithm to K.M. Hall, "An r-dimensional Quadratic Placement Algorithm," *Management Science* 17 (1970) 219-229.

## Summary

The canonical reference for **spectral graph drawing** (the "eigenprojection" method): lay a graph out by using the lowest positive [[spectral-clustering|Laplacian]] eigenvectors directly as continuous node coordinates. Koren: "the optimal layout is given by the lowest positive Laplacian eigenvectors v2, ..., v_{p+1}."

## The distinction that matters here

Koren is explicit that graph drawing uses the eigenvectors *raw*, whereas clustering, partitioning, and ordering "use discrete quantizations of the eigenvectors, unlike graph drawing, which employs the eigenvectors without any modification." So standard spectral layout places node *i* at Cartesian (v2_i, v3_i, ...) on **orthogonal** axes, one eigenvector per axis.

This is the prior art that [[spectral-hive-plots]] must be distinguished from: "use Laplacian eigenvectors as node coordinates" is 55-year-old technique and not novel. The spectral hive plot departs by quantizing the low eigenvectors into a **partition** that defines **radial cluster axes**, then using a *higher* eigenvector for within-axis order, which is not what eigenprojection does.

## See Also

- [[spectral-hive-plots]] — the recombination this source helps bound
- [[spectral-clustering]] — the shared Laplacian machinery
- [[hive-plot]] — the radial-axis alternative to Cartesian eigenprojection
