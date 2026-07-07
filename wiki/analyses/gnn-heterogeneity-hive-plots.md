---
title: "GNN Heterogeneity Diagnosis via Hive Plot Matrices"
type: analysis
created: 2026-04-13
updated: 2026-06-25
sources: [krzywinski-2012, hiveplotlib-python-repo, nollenburg-2023, ma-2021-subgroup-fairness, kipf-2017-gcn, subramonian-2024-degree-bias, gnnfairviz-2025]
tags: [gnn-evaluation, hive-plot-matrix, heterogeneity, research-proposal, machine-learning, edge-heterogeneity]
---

# GNN Heterogeneity Diagnosis via Hive Plot Matrices

A research proposal for using [[hive-plot-matrix|HivePlotMatrix]] to expose classification performance heterogeneity that standard [[gnn-evaluation|GNN evaluation metrics]] mask — at both the **node level** and the **edge level**.

> **Status (2026-07-06): this is the original proposal; its novelty claims did not survive.** The prototype ran (Cora and CiteSeer), and an adversarial literature check demoted or refuted every candidate finding. The honest results and full prior-art positioning are in [[gnn-heterogeneity-findings]]. Read that page for what happened; the proposal below is kept as the original framing. In particular, the "edge-level heterogeneity is unexplored" premise in the Motivation is wrong (see the one-paragraph summary next).

## Findings status (post-counterfactual, 2026-07-06)

One-paragraph scorecard; details and citations in [[gnn-heterogeneity-findings]].

1. **Homophily is the strongest simple predictor (confirmed, known).** 31/64/91% accuracy across homophily bins on Cora, but this is established (Zhu 2020; [[ma-2022-homophily-necessity|Ma 2022]]; Loveland 2023; Mao 2023), not a discovery.
2. **Edge-level "error contagion" (refuted).** The 2.22x `both_wrong` enrichment collapses to ~1.16x under a community-x-homophily null (seed-stable 1.20). And the premise (errors correlate on edges) is textbook: [[huang-2021-correct-and-smooth|Correct & Smooth]] and [[jia-benson-2020-residual-correlation|Jia & Benson]] already assume/model it. The "edge-level heterogeneity is unexplored" claim below is therefore false.
3. **Calibration structured by distance-to-training (real, but prior art).** [[hsu-2022-gnn-calibration|Hsu et al. 2022]] already names distance-to-training as a GNN calibration factor, on the same model and datasets. Only a narrow error-conditional refinement survives.
4. **Residual screen finding intra-class failure pockets (novel only as a composition).** Every ingredient has prior art ([[congalton-1988-error-autocorrelation|Congalton 1988]] join-count; [[jin-2022-gnnlens|GNNLens]]; slice discovery); the mechanism is owned by [[ma-2022-homophily-necessity|Ma 2022]]; label noise is not yet ruled out.

The GCN accuracy figure sketched below (~97.5%) is also wrong: it is ~81% on Cora.

## Motivation

1. **GNN evaluation uses aggregate metrics** — accuracy, F1, AUC-ROC — that collapse performance across all nodes into a single number.
2. **Literature confirms [[structural-heterogeneity]] affects GNN accuracy** — [[ma-2021-subgroup-fairness|Ma, Deng & Mei (2021)]] prove via PAC-Bayesian analysis that subgroups of test nodes distant from the training set suffer lower accuracy. Follow-up work ([[subramonian-2024-degree-bias|Subramonian et al. 2024]]) surveys 38 papers on degree bias, finding many hypotheses "not rigorously validated and can even be contradictory."
3. **No standard visual tool exists** for analyzing systematic performance patterns across graph structure. Existing explainability tools (GNNExplainer, InteractiveGNNExplainer) explain *individual* predictions, not systematic patterns. The closest visual analytics work ([[gnnfairviz-2025|GNNFairViz]]) targets demographic-attribute fairness, not structural subgroup diagnosis.
4. **Edge-level heterogeneity is unexplored.** The entire GNN fairness literature analyzes node-level accuracy only. Nobody has examined whether misclassification errors concentrate along specific edge types, propagate through specific structural regions, or correlate with edge-level graph properties. Hive plots show edges as first-class visual elements — this is uniquely hive-plot territory.

**The punchline:** "Your GNN gets 95% accuracy, but decomposed by degree, it's 99% on hubs and 78% on peripheral nodes — and the misclassified nodes are connected to each other in ways that aggregate metrics can't show. A hive plot matrix reveals this at a glance."

## Related Work and Positioning

### Ma, Deng & Mei 2021 (NeurIPS)

[[ma-2021-subgroup-fairness|Ma et al.]] prove that GNN accuracy varies by subgroup and identify **distance to training set** (in aggregated feature space) as the key driver. They test three subgroup definitions:

- **Aggregated-feature distance to training set** — strong monotonic accuracy decrease
- **Geodesic distance to training set** — similar trend, simpler to compute
- **Node centrality (degree, closeness, betweenness, PageRank)** — no clear monotonic trend in their 5-bin bar charts

Their analysis is purely tabular (grouped bar charts), single-variable-at-a-time, and node-level only. This proposal extends their work in four ways:

1. **Multi-dimensional decomposition** — HivePlotMatrix sweeps across candidate partitioning schemes simultaneously, including Ma et al.'s training-set distance alongside intrinsic properties, to discover which structural lens is most informative for a given model/dataset.
2. **Edge-aware visualization** — Hive plots show the edge structure between misclassified nodes, revealing whether errors propagate along specific connection patterns.
3. **Continuous structural gradients** — Axis sorting provides continuous views rather than 5-bin quantile bar charts, which may flatten non-linear patterns (e.g., centrality may show complex non-monotonic relationships that bins miss).
4. **Visual model card** — A single HivePlotMatrix artifact that characterizes model behavior across graph structure, usable as a standard diagnostic output alongside accuracy/F1.

### GNNFairViz (Ye et al. 2025, IEEE TVCG)

[[gnnfairviz-2025|GNNFairViz]] is a multi-view visual analytics tool for GNN fairness, but it operates in a different space: **demographic-attribute fairness** (e.g., gender, race groups), not structural subgroup fairness. It uses standard interactive graph views, not rational/axis-based layouts. The hive plot approach complements GNNFairViz by targeting structural performance variation rather than protected-attribute equity.

### Degree Bias Literature

[[subramonian-2024-degree-bias|Subramonian et al. (NeurIPS 2024)]] survey 38 papers on GNN degree bias and find the field lacks rigorous validation of competing hypotheses. Mitigation methods include GraphPatcher (Ju et al., NeurIPS 2023; test-time augmentation for low-degree nodes), DegFairGNN (Liu et al., AAAI 2023; learnable debiasing), and FairACE (2025; contrastive training with Accuracy Distribution Gap metric). All are metrics/training-focused — none provide a visualization framework for structural diagnosis.

## Method: Hive Plot Matrix for GNN Evaluation

### Step 1: Train and Collect Predictions

Train a [[graph-neural-networks|GNN]] (e.g., GCN on Cora node classification). For each node, record:

- True label
- Predicted label
- Prediction confidence (softmax probability of predicted class)
- Structural properties: degree, clustering coefficient, betweenness centrality, community membership, local homophily ratio
- **Training-set relationship properties:** aggregated-feature distance to training set, geodesic distance to nearest training node (following [[ma-2021-subgroup-fairness|Ma et al.]]'s finding that these are strong predictors of accuracy disparity)

### Step 2: Build a HivePlotMatrix

Construct a [[hive-plot-matrix|HivePlotMatrix]] where:

- **Partition variable** sweeps across structural properties (degree bins, community, homophily bins, training-set distance bins)
- **Sorting variable** = prediction confidence (or error magnitude)
- **Edge color** = correct (green/blue) vs. misclassified (red/orange)
- **Node color** = true class label
- Matrix rows/columns = different partitioning schemes or different models

### Step 3: Interpret

The matrix reveals **which structural decomposition exposes the most performance heterogeneity**. A partitioning where misclassified edges concentrate in specific cells tells you exactly which structural subgroup the model struggles with.

**Critically, the edge structure adds a dimension Ma et al. cannot access:** Are misclassified nodes connected to each other? Do errors concentrate on edges between specific structural groups (e.g., edges connecting low-degree nodes to high-degree nodes)? Do misclassification patterns follow community boundaries? These are edge-level questions that bar charts cannot address.

## Concrete Example Sketch: Cora

- **Dataset:** 2,708 papers, 7 classes, citation edges
- **Model:** Train GCN → ~81% test accuracy (the original ~97.5% figure here was wrong)
- **Hive plot construction:**
  - 3-axis hive plot: axis assignment by degree bin (low / medium / high)
  - Sort nodes along each axis by prediction confidence
  - Color edges by correct (blue) vs. incorrect (red) classification
- **Hypothesis:** Low-degree nodes (few citations) cluster misclassifications because message-passing has less information to aggregate. **Additionally:** misclassification edges may concentrate between degree groups (low↔high) where message-passing crosses structural boundaries.
- **Extend to HivePlotMatrix:** sweep partition across {degree bins, community membership, local homophily ratio, geodesic distance to training set} × {GCN, GAT, GraphSAGE}

This produces a matrix where each cell shows a different view of the same predictions — revealing whether different architectures fail on *different* subgroups and whether different structural decompositions expose different failure modes.

## What This Could Reveal

### Node-Level Patterns
- **Which structural properties correlate with misclassification** — degree, community structure, local homophily, or training-set distance? The HivePlotMatrix sweep discovers which is most informative rather than pre-judging.
- **Whether different architectures fail differently** — GCN vs. GAT vs. GraphSAGE may have different blind spots
- **Non-linear relationships** that Ma et al.'s 5-bin bar charts may flatten — continuous axis sorting captures the full structural gradient

### Edge-Level Patterns (Novel)
- **Whether misclassified nodes connect to each other** — error clustering in edge space vs. isolated node-level failures
- **Which inter-group edges carry misclassification** — do errors concentrate on edges between specific structural groups?
- **Edge-type heterogeneity** — do intra-community edges vs. inter-community edges have different misclassification rates?
- **Error propagation signatures** — does the edge pattern suggest message-passing is propagating incorrect signals through specific graph regions?

### Diagnostic Outputs
- **A visual "model card" for GNN evaluation** — A single HivePlotMatrix artifact that characterizes model behavior across graph structure
- **Comparison across partitioning schemes** — which structural lens reveals the most about where this model fails?

## Candidate Datasets

| Dataset | Nodes | Classes | Why |
|---------|-------|---------|-----|
| **Cora** | 2,708 | 7 | Small, well-understood, good for prototyping. Ma et al. use this. |
| **CiteSeer** | 3,327 | 6 | Similar scale, different class structure. Ma et al. use this. |
| **PubMed** | 19,717 | 3 | Larger, tests scaling of visualization. Ma et al. use this. |
| **OGB-arxiv** | 169,343 | 40 | Large-scale stress test (datashader backend). Ma et al. test in appendix. |

## [[hiveplotlib]] Integration Path

The existing [[hiveplotlib]] infrastructure supports this workflow:

1. **Data loading:** Use NetworkX or PyTorch Geometric to load graph + GNN predictions
2. **Conversion:** `networkx_to_nodes_edges()` converter already exists in hiveplotlib
3. **Metadata:** Add prediction metadata (true label, predicted label, confidence) to `NodeCollection.data`
4. **Construction:** `HivePlotMatrix.from_variable_sweep()` is the natural construction mode — sweep partitioning variables across rows/columns
5. **Rendering:** matplotlib backend for small graphs; datashader backend for OGB-scale graphs

No new hiveplotlib features are required — this is an *application* of existing functionality.

## Key References

- **[[ma-2021-subgroup-fairness|Ma, Deng & Mei 2021]]** — PAC-Bayesian analysis proving subgroup accuracy disparity; identifies training-set distance as key driver. Node-level bar charts only — no network visualization, no edge analysis, single-variable-at-a-time.
- **[[subramonian-2024-degree-bias|Subramonian et al. 2024]]** — NeurIPS survey of 38 papers on degree bias; finds many hypotheses not rigorously validated. No visualization framework.
- **[[gnnfairviz-2025|GNNFairViz (Ye et al. 2025)]]** — Closest visual analytics competitor; targets demographic-attribute fairness with standard graph views. Different framing from structural subgroup diagnosis.
- **[[nollenburg-2023-computing-hive-plots|Nöllenburg & Wallinger 2023]]** — Parameter selection is NP-complete → systematic sweep via HivePlotMatrix is the correct approach
- **[[krzywinski-2012]]** — Hive plots reveal structural patterns that [[force-directed-layout|force-directed layouts]] miss
- **[[kipf-2017-gcn|Kipf & Welling 2017]]** — Foundational GCN paper establishing the baseline architecture and evaluation methodology
- **Zhu et al. 2020 ("Beyond Homophily in Graph Neural Networks", NeurIPS); Loveland et al. 2023 ("On Performance Discrepancies Across Local Homophily Levels in GNNs")** — Prior art for the homophily-predicts-accuracy result; any homophily finding here is confirmation of this literature, not a contribution

## Open Questions

1. **Which structural properties are most informative** for decomposing GNN performance? Ma et al. found training-set distance is the strongest monotonic predictor, but degree and centrality may show non-linear patterns that continuous hive plot axes capture better than 5-bin bar charts. The HivePlotMatrix sweep is designed to answer this empirically.
2. **What does edge-level heterogeneity look like?** Do misclassified nodes cluster in edge space? Do errors propagate along specific edge types? This is entirely unexplored in the literature and is a primary contribution of the hive plot approach.
3. **Is there a threshold of structural heterogeneity** below which hive plots add nothing over standard metrics? (i.e., if performance is truly uniform, the visualization is uninformative)
4. **Could this become a general-purpose "GNN evaluation diagnostic" tool?** A standard output alongside accuracy/F1 in any GNN paper?
5. **How does this relate to the fairness-aware GNN literature?** The fairness framing ([[gnnfairviz-2025|GNNFairViz]], FairACE) emphasizes equity across demographic groups; the hive plot framing emphasizes *understanding structural failure modes*. These are complementary — and hive plots could serve both purposes.
6. **What about edge-level and graph-level tasks?** This proposal focuses on node classification, but link prediction and graph classification may benefit from similar structural decomposition.

## See Also

- [[hive-plot-matrix]] — The visualization tool at the core of this proposal
- [[hiveplotlib]] — Implementation platform
- [[graph-neural-networks]] — The models being evaluated
- [[gnn-evaluation]] — The evaluation gap this addresses
- [[structural-heterogeneity]] — The phenomenon being visualized
- [[node-assignment]] — How hive plots map structural properties to axes
- [[ma-2021-subgroup-fairness]] — Theoretical foundation: subgroup accuracy disparity is real and predictable
- [[subramonian-2024-degree-bias]] — Survey confirming degree bias is widespread but poorly understood
- [[gnnfairviz-2025]] — Closest visual analytics work (different framing: demographic fairness)
- [[applications-cybersecurity]] — Parallel: cybersecurity uses hive plots as ML features; this proposal uses hive plots to *evaluate* ML
- [[examples-and-applications]] — catalog of hiveplotlib examples and application explorations
