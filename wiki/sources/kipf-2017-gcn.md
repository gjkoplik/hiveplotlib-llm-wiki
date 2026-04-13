---
title: "Kipf & Welling 2017 — Semi-Supervised Classification with GCNs"
type: source
created: 2026-04-13
updated: 2026-04-13
sources: []
tags: [graph-neural-networks, gcn, foundational-paper]
---

# Kipf & Welling 2017 — Semi-Supervised Classification with Graph Convolutional Networks

> **Stub:** This page is based on web research, not a full reading of the paper. Flagged for future full ingest.

## Citation

Thomas N. Kipf, Max Welling. "Semi-Supervised Classification with Graph Convolutional Networks." *ICLR 2017*.

## Summary

The foundational paper for Graph Convolutional Networks (GCNs). Kipf & Welling approximate spectral graph convolutions with a simple, efficient layer-wise propagation rule: normalize the adjacency matrix, multiply by node features, and apply a learned weight matrix. This simplification enables scalable [[graph-neural-networks|graph neural network]] training on standard benchmarks.

## Key Contributions

- **Spectral-to-spatial bridge:** Derives a first-order approximation of spectral graph convolutions that is efficient and localized
- **Simple architecture:** Two-layer GCN achieves strong results on node classification
- **Benchmark establishment:** Cora (2,708 nodes, 7 classes), CiteSeer (3,327 nodes, 6 classes), and PubMed (19,717 nodes, 3 classes) become the standard evaluation suite
- **Evaluation methodology:** Reports aggregate accuracy on fixed train/val/test splits — the same methodology the [[gnn-heterogeneity-hive-plots|GNN heterogeneity proposal]] critiques

## Relevance to This Wiki

This paper establishes both the baseline architecture and the aggregate evaluation methodology that the [[gnn-heterogeneity-hive-plots|hive plot evaluation proposal]] builds on. The GCN is the natural first model to evaluate with a [[hive-plot-matrix|HivePlotMatrix]] because:

1. It is the simplest and best-understood GNN architecture
2. Its neighborhood aggregation (normalized mean) has known limitations under [[structural-heterogeneity]] (degree-dependent smoothing)
3. Cora/CiteSeer/PubMed are small enough for matplotlib-backend hive plots

## See Also

- [[graph-neural-networks]] — GCN in context of the broader architecture family
- [[gnn-evaluation]] — The evaluation methodology established here
- [[gnn-heterogeneity-hive-plots]] — Research proposal critiquing aggregate evaluation
- [[ma-2021-subgroup-fairness]] — Later work showing disparities in GCN performance across subgroups
