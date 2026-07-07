---
title: "Huang et al. 2021 — Correct & Smooth (Label Propagation + Simple Models)"
type: source
created: 2026-07-06
updated: 2026-07-06
sources: []
tags: [gnn-evaluation, label-propagation, residual-correlation]
---

# Huang et al. 2021 — Combining Label Propagation and Simple Models Out-performs GNNs (Correct & Smooth)

> Web-research source (0 full ingests). Verified against the arXiv abstract and method description.

## Citation

Qian Huang, Horace He, Abhay Singh, Ser-Nam Lim, Austin R. Benson. "Combining Label Propagation and Simple Models Out-performs Graph Neural Networks." *ICLR 2021*. arXiv:2010.13993. [arXiv](https://arxiv.org/abs/2010.13993)

## Summary

Shows a shallow model plus two label-propagation post-processing steps rivals or beats GNNs on standard node-classification benchmarks. The relevant step is **"Correct"**: it builds a residual/error matrix (base-model residual on training nodes, zero elsewhere) and **propagates those errors along edges** by label-propagation smoothing, then adds the smoothed errors back to base predictions. The explicit operating assumption: "errors in the base prediction are positively correlated along edges. An error at node i increases the chance of a similar error at neighboring nodes."

## Why it matters here

This is the **mandatory citation** for [[gnn-heterogeneity-findings]]. It establishes, as textbook and exploited-on-purpose, that GNN classification errors are positively autocorrelated along edges. Two consequences for the prototype:

1. The proposal's "errors cluster on edges, maybe they propagate" framing was never a discovery. C&S already assumes and profits from exactly this regularity.
2. The residual screen's contribution has to be positioned against C&S precisely: C&S *assumes* edge error-correlation to smooth-and-fix predictions; the screen instead *tests for* excess joint failure against a covariate-conditioned independence null, as a diagnostic, without altering predictions. "We test what they assume" is the honest one-liner.

Companion prior art for the regression case is [[jia-benson-2020-residual-correlation|Jia & Benson 2020]].

## See Also

- [[gnn-heterogeneity-findings]] — Uses this to refute the edge-contagion claim and position the residual screen
- [[jia-benson-2020-residual-correlation]] — Regression analog (models residual correlation on edges)
- [[congalton-1988-error-autocorrelation]] — The diagnostic (rather than corrective) treatment of edge-correlated errors
