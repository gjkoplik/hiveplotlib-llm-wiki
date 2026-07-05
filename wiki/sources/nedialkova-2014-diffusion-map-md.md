---
title: "Nedialkova et al. 2014 — Diffusion Maps for Peptide Folding"
type: source
created: 2026-07-04
updated: 2026-07-04
sources: []
tags: [diffusion-maps, molecular-dynamics, spectral-clustering, cheminformatics, foundational-paper]
---

# Nedialkova et al. 2014 — Diffusion Maps, Clustering and Fuzzy Markov Modeling in Peptide Folding Transitions

> **Stub:** based on web research (deep-research pass, 2026-07-04), not a full reading. Flagged for future full ingest.

## Citation

L.V. Nedialkova, M.A. Amat, I.G. Kevrekidis, G. Hummer. "Diffusion maps, clustering and fuzzy Markov modeling in peptide folding transitions." *Journal of Chemical Physics* 141, 114102 (2014). PMC4169379. Related: Noë & Clementi, "Kinetic distance and kinetic maps from molecular dynamics simulation," *JCTC* 2015 (arXiv 1506.06259).

## Why this is the load-bearing precedent for [[spectral-hive-plots]]

This paper independently validates the one move in the spectral hive plot that felt like a leap: **reading within-cluster structure off a HIGHER eigenvector**. In diffusion-map analysis of alanine pentapeptide, "v1 appears to organize points according to a folded/unfolded criterion, v2 distinguishes the points in state U2 and v9 does the same for the points in state U1," and the authors generalize that "different eigenvectors resolve different portions of the data."

That is exactly the spectral hive plot's split: the first non-trivial eigenvector defines the top-level cut (the axes), and later, distinct eigenvectors resolve substructure inside individual clusters (the within-axis order). The mechanism is not speculative; it is established computational-chemistry practice.

## Corroborating the rw-normalization choice

Noë & Clementi (2015) show the eigenfunctions of the backward Markov propagator are the optimal reaction coordinates, and rescaled eigenvectors form a **kinetic map** where Euclidean distance equals kinetic distance. These are diffusion / transfer-operator (random-walk-family) coordinates, which is why the `hiveplotlib-spectral` prototype sorts by **random-walk** eigenvectors rather than the sqrt(degree)-scaled symmetric ones.

## See Also

- [[spectral-hive-plots]] — the direction this precedent validates
- [[spectral-clustering]] — diffusion maps as a spectral-clustering relative
