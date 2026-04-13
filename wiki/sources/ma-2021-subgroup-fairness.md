---
title: "Ma, Deng & Mei 2021 — Subgroup Generalization and Fairness of GNNs"
type: source
created: 2026-04-13
updated: 2026-04-13
sources: [NeurIPS-2021-subgroup-generalization-and-fairness-of-graph-neural-networks-Paper.pdf]
tags: [gnn-evaluation, fairness, heterogeneity, generalization-theory, pac-bayesian]
---

# Ma, Deng & Mei 2021 — Subgroup Generalization and Fairness of Graph Neural Networks

## Citation

Jiaqi Ma, Junwei Deng, Qiaozhu Mei. "Subgroup Generalization and Fairness of Graph Neural Networks." *NeurIPS 2021* (Spotlight, top 3%). University of Michigan.

Code: `github.com/TheaperDeng/GNN-Generalization-Fairness`

## Summary

This paper provides the first PAC-Bayesian generalization analysis for GNNs on **non-IID node-level tasks** (semi-supervised node classification). The key theoretical result (Theorem 3) shows that a GNN's generalization error on a subgroup of test nodes depends on **ε_m — the distance from that subgroup to the training set in aggregated feature space**. Subgroups farther from training nodes have weaker generalization guarantees, producing theoretically predictable accuracy disparity.

## Key Contributions

1. **PAC-Bayesian framework for non-IID semi-supervised learning** — General theorems (Theorems 1–2) for stochastic and deterministic classifiers, then a GNN-specific bound (Theorem 3) incorporating aggregated-feature distance ε_m.
2. **Subgroup accuracy disparity is theoretically predictable** — The bound suggests nodes far from training set (in aggregated feature space) will have lower accuracy. Empirically confirmed.
3. **Three subgroup definitions tested empirically:**
   - **Aggregated-feature distance** to training set — strong monotonic accuracy decrease (Figure 1). This is the theoretically motivated variable.
   - **Geodesic distance** (shortest path) to training set — similar monotonic trend, simpler to compute (Figure 2).
   - **Node centrality** (degree, closeness, betweenness, PageRank) — **no clear monotonic trend** (Figure 3). Centrality captures intrinsic node properties but not the relationship to the training set.
4. **Biased training node selection** (Section 5.2) — When training nodes are biased toward high-centrality nodes of one class, GNNs (but not MLPs) develop strong false positive bias toward that class, because feature aggregation spreads the influence of high-centrality training nodes.

## Experimental Setup

- **Models:** GCN, GAT, SGC, APPNP, MLP (baseline)
- **Datasets:** Cora (2,708 nodes, 7 classes), Citeseer (3,327 nodes, 6 classes), Pubmed (19,717 nodes, 3 classes). OGB datasets in appendix.
- **Protocol:** 20 nodes per class for training, 500 validation, 1,000 test. 40 independent trials per setting.
- **Subgroup construction:** Sort test nodes by distance metric, split into 5 equal-sized quintiles.

## What the Paper Does NOT Do

- **No network visualization.** All results are presented as grouped bar charts (accuracy by quintile). No node-link diagrams, no structural layouts.
- **Single-variable-at-a-time analysis.** Each subgroup definition is tested separately — no multi-dimensional decomposition or comparison across partitioning schemes.
- **Node-level only.** No analysis of edge-level patterns — which edges connect misclassified nodes, whether errors propagate along specific edge types.
- **No mitigation methods.** The paper diagnoses disparity but does not propose fairness-aware training algorithms (unlike some follow-up work).
- **No visual diagnostic framework.** The paper calls for attention to training node selection but provides no tool for practitioners to inspect their own models.

## Key Finding for Hive Plot Research

The finding that **node centrality does not monotonically predict accuracy** (Figure 3) does not mean centrality is uninformative — it means the relationship is more complex than a simple monotonic trend. Bar charts with 5 bins may flatten non-linear patterns that a continuous hive plot axis (sorting by centrality or confidence within centrality-based partitions) could reveal.

More importantly, the paper's exclusive focus on **node-level** accuracy leaves **edge-level heterogeneity entirely unexplored**. A [[hive-plot|hive plot]] shows edges as first-class visual elements, enabling analysis of whether misclassification errors cluster along specific edge types or between specific structural groups — a dimension Ma et al. cannot address with bar charts.

The [[gnn-heterogeneity-hive-plots|HivePlotMatrix proposal]] extends this work by providing: (1) multi-dimensional decomposition in a single view, (2) edge-aware visualization, (3) continuous rather than binned structural gradients, and (4) systematic comparison across partitioning schemes including training-set distance (their key variable).

## See Also

- [[graph-neural-networks]] — The models analyzed
- [[gnn-evaluation]] — The evaluation gap this paper documents
- [[structural-heterogeneity]] — The phenomenon studied
- [[gnn-heterogeneity-hive-plots]] — Research proposal building on these findings
- [[kipf-2017-gcn]] — Baseline architecture used in experiments
- [[subramonian-2024-degree-bias]] — Follow-up survey of 38 papers on degree bias
- [[gnnfairviz-2025]] — Visual analytics tool for GNN fairness (different framing: demographic attributes)
