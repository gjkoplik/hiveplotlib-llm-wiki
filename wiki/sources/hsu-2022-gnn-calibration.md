---
title: "Hsu et al. 2022 — What Makes Graph Neural Networks Miscalibrated?"
type: source
created: 2026-07-06
updated: 2026-07-06
sources: []
tags: [gnn-evaluation, calibration, confidence, heterogeneity]
---

# Hsu et al. 2022 — What Makes Graph Neural Networks Miscalibrated?

> Web-research source (0 full ingests). Verified against the arXiv abstract and Section 4 text.

## Citation

Hans Hao-Hsun Hsu, Yuesong Shen, Christian Tomani, Daniel Cremers. "What Makes Graph Neural Networks Miscalibrated?" *NeurIPS 2022*. arXiv:2210.06391. [arXiv](https://arxiv.org/abs/2210.06391) | [NeurIPS PDF](https://papers.neurips.cc/paper_files/paper/2022/file/5975754c7650dfee0682e06e1fec0522-Paper-Conference.pdf)

## Summary

A systematic study of which node-level factors govern GNN confidence calibration. Identifies five: a general **under-confident** tendency (opposite of the over-confidence typical of image CNNs), diversity of nodewise predictive distributions, **distance to training nodes**, relative confidence level, and **neighborhood similarity** (prediction agreement with neighbors). Proposes GATS, a per-node temperature-scaling model that encodes these factors.

## Why it matters here

This is the **primary refuter** for the distance-structured miscalibration finding in [[gnn-heterogeneity-findings]]. Section 4.3 ("Distance to training nodes") measures nodewise GCN calibration error against minimum shortest-path distance to training nodes, on Cora and CiteSeer, the same model, datasets, and distance metric used in the prototype. Verbatim: "nodes with shorter distances ... tend to be better calibrated." So "GNN calibration is structured by distance to the training set" is established prior art, not a new observation.

Two openings the prototype work exploits, both narrow:
- Hsu report **aggregate** nodewise calibration error over all nodes (near-training = better calibrated). They do not condition on misclassified nodes, so they do not report the complementary error-conditional fact (the errors that occur near training are confident errors, sitting above the reliability diagonal).
- They treat **distance** (factor 3) and **neighborhood similarity** (factor 5) as separate factors and never deconfound them, so a homophily-partialled distance effect is a step they did not take.

Section 4.5's "neighborhood similarity" (agreement with neighbors lowers calibration error) is also the calibration-side echo of the homophily accuracy result in [[gnn-heterogeneity-findings]].

## See Also

- [[gnn-heterogeneity-findings]] — Uses this as the prior art demoting the calibration finding
- [[ma-2021-subgroup-fairness]] — Accuracy-by-distance analog (Hsu is the calibration analog)
- [[jin-2022-gnnlens]] — Visualizes the accuracy half of the distance effect
- [[gnn-evaluation]] — Calibration heterogeneity as an evaluation dimension
