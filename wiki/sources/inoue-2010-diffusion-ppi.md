---
title: "Inoue et al. 2010 — Diffusion Spectral Clustering of PPI Networks"
type: source
created: 2026-07-04
updated: 2026-07-04
sources: []
tags: [spectral-clustering, diffusion-maps, protein-interaction-network, network-visualization]
---

# Inoue, Li & Kurata 2010 — Diffusion Model Based Spectral Clustering for Protein-Protein Interaction Networks

> **Stub:** based on web research (deep-research pass, 2026-07-04), not a full reading. Flagged for future full ingest.

## Citation

K. Inoue, W. Li, H. Kurata. "Diffusion Model Based Spectral Clustering for Protein-Protein Interaction Networks." *PLoS ONE* 5(9): e12623 (2010). PMC2935381.

## Summary

Applies "a diffusion model-based spectral clustering algorithm, which analytically solves the cluster structure of PPI networks as a problem of random walks in the diffusion process." In the resulting spectral embedding, nodes "are actually distributed along the radial directions from the original point, forming cluster structure," and cluster membership is read from the angular distance between node vectors.

## Why it matters for [[spectral-hive-plots]] (and why it does not kill it)

This is the **closest existing spatial analog** to a radial-axis spectral layout: a spectral embedding where clusters appear as radial directions. But it is an angular-clustering *scatter*, not a hive plot: it does not lay nodes on discrete radial axes, and it uses no higher-eigenvector within-axis ordering. It supplies the spectral-embedding half of the recombination, not the whole, so it bounds the novelty claim without refuting it.

## See Also

- [[spectral-hive-plots]] — the recombination this source is the nearest analog to
- [[spectral-clustering]] — diffusion-process spectral clustering
