---
title: "Jia & Benson 2020 — Residual Correlation in GNN Regression"
type: source
created: 2026-07-06
updated: 2026-07-06
sources: []
tags: [gnn-evaluation, residual-correlation, regression]
---

# Jia & Benson 2020 — Residual Correlation in Graph Neural Network Regression

> Web-research source (0 full ingests). Verified against the arXiv abstract.

## Citation

Junteng Jia, Austin R. Benson. "Residual Correlation in Graph Neural Network Regression." *KDD 2020*. arXiv:2002.08274. [arXiv](https://arxiv.org/abs/2002.08274)

## Summary

Argues the standard GNN pipeline wrongly assumes node outcomes are conditionally independent given neighborhood features. Models the joint distribution of a GNN's **regression residuals** across vertices as a parameterized multivariate Gaussian whose correlation structure follows the graph's edges, estimated by maximizing marginal likelihood with linear-time estimators. Uses the fitted residual correlation to produce corrected point predictions and calibrated intervals.

## Why it matters here

The canonical "GNN residuals are edge-correlated" paper, and the regression companion to [[huang-2021-correct-and-smooth|Correct & Smooth]]. For [[gnn-heterogeneity-findings]] it matters because the residual screen's edge-level premise (both endpoints co-fail more than independence predicts) is the classification, diagnostic reframing of what Jia & Benson model for regression, prediction-improvement purposes.

How the screen differs: (1) classification (binary correct/wrong) rather than continuous residuals; (2) diagnostic (flag surprising edges) rather than corrective (improve predictions); (3) the "expected" part is a separate covariate model of *who fails*, not the GNN's own prediction. The independence null on paired failures is not something Jia & Benson form.

## See Also

- [[gnn-heterogeneity-findings]] — Positions the residual screen against this
- [[huang-2021-correct-and-smooth]] — Classification companion (also edge error-correlation)
