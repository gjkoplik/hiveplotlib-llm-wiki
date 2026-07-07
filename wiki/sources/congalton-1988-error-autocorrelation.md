---
title: "Congalton 1988 — Spatial Autocorrelation of Classification Errors"
type: source
created: 2026-07-06
updated: 2026-07-06
sources: []
tags: [spatial-statistics, error-analysis, remote-sensing, residual-correlation]
---

# Congalton 1988 — Spatial Autocorrelation Analysis of Map Classification Errors

> Web-research source (0 full ingests). Verified against citation records and the remote-sensing accuracy-assessment literature.

## Citation

Russell G. Congalton. "Using spatial autocorrelation analysis to explore the errors in maps generated from remotely sensed data." *Photogrammetric Engineering & Remote Sensing* 54(5):587-592, 1988. Related: Congalton (1991), "A review of assessing the accuracy of classifications of remotely sensed data," *Remote Sensing of Environment*. Antecedent: Campbell (1981) first reported that misclassified pixels cluster spatially.

## Summary

Treats classification errors as a **binary field over spatial units** (each pixel correct or wrong) and tests whether misclassifications **cluster beyond chance** using spatial-autocorrelation statistics, notably join-count statistics on the error map. The "black-black" join count is the number of adjacent unit pairs where **both** are misclassified, compared against an expectation under spatial randomization.

This launched a ~40-year remote-sensing tradition of spatially explicit accuracy assessment: localized confusion matrices, LISA quadrants (Anselin 1995), Moran's I on residuals, and kriged error surfaces.

## Why it matters here

This is the **sharpest single antecedent** for the residual screen in [[gnn-heterogeneity-findings]], and the most dangerous for a novelty claim. The join-count "both-endpoints-wrong" statistic is structurally identical to the prototype's `both_wrong` edge count: observed-versus-expected joint failure on an adjacency structure, used diagnostically. A remote-sensing reviewer would say "this is join-count analysis of classification error, moved from a spatial lattice to a GNN graph."

The honest differentiator, and it is narrow: the classic baseline is a **spatial-permutation / randomization** null, whereas the prototype conditions the independence null on a **covariate-fitted per-node failure model** (homophily, degree, distance-to-train, and so on) and applies it to arbitrary GNN graphs rather than a regular lattice. Any writeup should cite and distinguish this head-on rather than let a reviewer raise it.

## See Also

- [[gnn-heterogeneity-findings]] — The residual screen this most closely anticipates
- [[huang-2021-correct-and-smooth]] / [[jia-benson-2020-residual-correlation]] — The GNN-side edge-error-correlation prior art
