---
title: GNN Evaluation
type: concept
created: 2026-04-13
updated: 2026-04-13
sources: [kipf-2017-gcn, ma-2021-subgroup-fairness, subramonian-2024-degree-bias, gnnfairviz-2025]
tags: [gnn-evaluation, metrics, heterogeneity]
---

# GNN Evaluation

Standard evaluation of [[graph-neural-networks|graph neural networks]] relies on aggregate metrics that summarize performance into a single number. This approach is simple and comparable across papers, but it **masks performance variation across graph structure** — a gap that [[gnn-heterogeneity-hive-plots|hive plot matrices]] could address.

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

## Existing Explainability Tools

Current GNN interpretability tools focus on **individual predictions**, not systematic patterns:

| Tool | What It Does | What It Doesn't Do |
|------|-------------|-------------------|
| **GNNExplainer** (Ying et al. 2019) | Identifies important edges/features for a single node's prediction | No aggregate structural analysis |
| **InteractiveGNNExplainer** | Interactive version of the above | Same limitation — node-level only |
| **Attention visualization** | Shows learned attention weights in GAT | Per-edge, not per-structure |
| **Grad-CAM for graphs** | Gradient-based importance | Individual prediction focus |

None of these tools answer: "Which *types* of nodes does this model systematically get wrong?"

## The Hive Plot Opportunity

A [[hive-plot-matrix|HivePlotMatrix]] can fill this gap by:

1. Partitioning classified nodes by structural properties ([[node-assignment]])
2. Sorting by prediction confidence or error magnitude
3. Color-coding edges by correct vs. misclassified
4. Sweeping partitioning schemes across the matrix to find which structural decomposition reveals the most performance variation

This produces a **visual model card** — a single artifact showing where a GNN succeeds and fails across graph structure. See [[gnn-heterogeneity-hive-plots]] for the full proposal.

## See Also

- [[graph-neural-networks]] — The models being evaluated
- [[structural-heterogeneity]] — The performance variation that metrics miss
- [[gnn-heterogeneity-hive-plots]] — Research proposal using hive plots to expose this gap
- [[ma-2021-subgroup-fairness]] — Fairness analysis confirming structural disparities
- [[subramonian-2024-degree-bias]] — Degree bias survey; inverse collision probability and prediction homogeneity as new metrics
- [[gnnfairviz-2025]] — Visual analytics for GNN demographic-attribute fairness
- [[hive-plot-matrix]] — The visualization tool for systematic comparison
