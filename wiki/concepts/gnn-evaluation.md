---
title: GNN Evaluation
type: concept
created: 2026-04-13
updated: 2026-07-06
sources: [kipf-2017-gcn, ma-2021-subgroup-fairness, subramonian-2024-degree-bias, gnnfairviz-2025, hsu-2022-gnn-calibration, jin-2022-gnnlens]
tags: [gnn-evaluation, metrics, heterogeneity, calibration]
---

# GNN Evaluation

Standard evaluation of [[graph-neural-networks|graph neural networks]] relies on aggregate metrics that summarize performance into a single number. This approach is simple and comparable across papers, but it **masks performance variation across graph structure**. Hive plots were proposed as one way to expose that variation; a prototype ran and produced mostly negative results, documented honestly in [[gnn-heterogeneity-findings]]. This page describes the evaluation gap and where the prototype landed.

## Standard Metrics

| Metric | What It Measures | Limitation |
|--------|-----------------|------------|
| **Accuracy** | Fraction of correct predictions | Insensitive to class imbalance |
| **F1 Score** | Harmonic mean of precision/recall | Usually macro-averaged across classes, not structure |
| **AUC-ROC** | Ranking quality across thresholds | Aggregate over all nodes |
| **Precision / Recall** | Error type tradeoffs | Per-class at best, never per-structure |

### Per-Class Reporting

Some papers report per-class accuracy or F1, which reveals class-level disparities. But even per-class metrics ignore **within-class structural variation** — a high-degree node and a low-degree node in the same class are treated identically.

## The Gap: Structural Performance Heterogeneity

What standard evaluation does NOT measure:

- **Performance by degree** — Do hub nodes classify better than peripheral nodes?
- **Performance by community structure** — Does the model succeed in dense communities but fail in sparse ones?
- **Performance by homophily** — Does local label agreement affect accuracy?
- **Performance by centrality** — Are bottleneck nodes (high betweenness) harder to classify?

The GNN fairness literature (e.g., [[ma-2021-subgroup-fairness|Ma et al. 2021]]) confirms these disparities exist but are rarely reported in standard benchmarks.

### Calibration heterogeneity, not just accuracy

Accuracy is not the only thing that varies across structure; **confidence calibration** does too. [[hsu-2022-gnn-calibration|Hsu et al. 2022]] show GNNs are generally under-confident and that per-node calibration depends on distance to the training set and on agreement with neighbors. The [[gnn-heterogeneity-findings|prototype]] confirmed this and added a narrow error-conditional refinement: among the errors, near-training ones are confidently wrong while distant ones fail quietly. Calibration-by-structure is a real evaluation axis, but it is now a documented one, not an open gap.

## Existing Explainability Tools

Current GNN interpretability tools focus on **individual predictions**, not systematic patterns:

| Tool | What It Does | What It Doesn't Do |
|------|-------------|-------------------|
| **GNNExplainer** (Ying et al. 2019) | Identifies important edges/features for a single node's prediction | No aggregate structural analysis |
| **InteractiveGNNExplainer** | Interactive version of the above | Same limitation — node-level only |
| **Attention visualization** | Shows learned attention weights in GAT | Per-edge, not per-structure |
| **Grad-CAM for graphs** | Gradient-based importance | Individual prediction focus |

These per-prediction tools do not answer "which *types* of nodes does this model systematically get wrong?" But one dedicated tool does: [[jin-2022-gnnlens|GNNLens]] (Jin et al., IEEE TVCG 2022) is an interactive visual-analytics system for GNN error diagnosis that correlates misclassification with degree, local homophily, and distance-to-training. Any hive-plot approach to this question must be positioned as a complement to GNNLens, not as filling an empty space.

## The Hive Plot Opportunity (and its honest ceiling)

A [[hive-plot-matrix|HivePlotMatrix]] can partition classified nodes by structural properties ([[node-assignment]]), sort by confidence, and color edges by correctness, sweeping partitions to find the most informative structural decomposition. The [[gnn-heterogeneity-findings|prototype that did this]] reached a sober conclusion: the statistics (groupby tables, chi-squared, permutation nulls) did the finding, and the hive plot contributed positional *layout*, not discovery. The one method with defensible novelty is a **residual screen**: fit a covariate baseline for per-node error, form an edge-level independence null, and flag joint failures the baseline cannot explain. Even that is novel only as a composition; its parts are prior art ([[congalton-1988-error-autocorrelation|Congalton 1988]] join-count on classification errors, [[huang-2021-correct-and-smooth|edge-correlated errors]], slice discovery). See [[gnn-heterogeneity-findings]] for the full scorecard.

## See Also

- [[graph-neural-networks]] — The models being evaluated
- [[structural-heterogeneity]] — The performance variation that metrics miss
- [[gnn-heterogeneity-hive-plots]] — Research proposal using hive plots to expose this gap
- [[ma-2021-subgroup-fairness]] — Fairness analysis confirming structural disparities
- [[subramonian-2024-degree-bias]] — Degree bias survey; inverse collision probability and prediction homogeneity as new metrics
- [[gnnfairviz-2025]] — Visual analytics for GNN demographic-attribute fairness
- [[jin-2022-gnnlens]] — Dedicated visual analytics for GNN error diagnosis (the nearest prior art)
- [[hsu-2022-gnn-calibration]] — Calibration heterogeneity across graph structure
- [[gnn-heterogeneity-findings]] — Empirical results and honest prior-art positioning
- [[hive-plot-matrix]] — The visualization tool for systematic comparison
