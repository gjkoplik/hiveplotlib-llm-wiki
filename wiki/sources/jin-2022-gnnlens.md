---
title: "Jin et al. 2022 — GNNLens (Visual Analytics for GNN Error Diagnosis)"
type: source
created: 2026-07-06
updated: 2026-07-06
sources: []
tags: [gnn-evaluation, visual-analytics, error-diagnosis]
---

# Jin et al. 2022 — GNNLens: A Visual Analytics Approach for Prediction Error Diagnosis of GNNs

> Web-research source (0 full ingests). Verified against the arXiv full text (first posted 2020 as "GNNVis").

## Citation

Zhihua Jin, Yong Wang, Qianwen Wang, Yao Ming, Tengfei Ma, Huamin Qu. "GNNLens: A Visual Analytics Approach for Prediction Error Diagnosis of Graph Neural Networks." *IEEE TVCG* 29(6):3024-3038, 2023. arXiv:2011.11048. [arXiv](https://arxiv.org/abs/2011.11048)

## Summary

An interactive visual-analytics system for diagnosing where a GNN misclassifies nodes. Computes per-node metrics that overlap heavily with structural risk factors: degree, "center-neighbor consistency rate" (local homophily), shortest-path distance to training nodes, and label distribution/consistency of nearest training and feature-similar nodes. Users explore correlations with wrong predictions via parallel-sets and projection views, then drill into individual nodes. Disentangles structure vs feature influence by comparing GNN, MLP (features only), and structure-only variants.

## Why it matters here

This is the **nearest visual-analytics prior art** for the hive-plot approach in [[gnn-heterogeneity-findings]], and it is both an anchor and a differentiator.

- **Anchor / threat:** its covariate list is nearly identical to the residual screen's, and it targets exactly the same task (per-node GNN error diagnosis on citation graphs). It also already visualized the accuracy half of the calibration finding: misclassified nodes tend to be far from labeled nodes. Any "hive plots for GNN error diagnosis" claim must cite and differentiate it.
- **Differentiator:** GNNLens externalizes correlation-finding to the human eye (parallel sets, projections). It fits **no** statistical baseline, computes **no** expected-error probability, has **no** residual concept, and does **no** edge-level joint-failure analysis. It also omits clustering coefficient and true class as covariates. The residual screen internalizes the correlation in a fitted baseline and then studies what the baseline cannot explain, at the edge level.

Honest ceiling: GNNLens already occupies the "interactive visual GNN error diagnosis" space more richly than a static hive plot does. The defensible hive-plot angle is a layout-first, static complement, not a replacement.

## See Also

- [[gnn-heterogeneity-findings]] — Positions the hive-plot approach against this
- [[hsu-2022-gnn-calibration]] — GNNLens shows the accuracy half; Hsu the calibration half
- [[gnnfairviz-2025]] — Other GNN visual analytics (demographic fairness framing)
- [[gnn-evaluation]] — The diagnosis gap both address
