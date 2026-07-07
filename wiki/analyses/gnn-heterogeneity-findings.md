---
title: "GNN Heterogeneity Hive Plots: Empirical Findings and Prior-Art Positioning"
type: analysis
created: 2026-07-06
updated: 2026-07-06
sources: [ma-2021-subgroup-fairness, subramonian-2024-degree-bias, kipf-2017-gcn, hsu-2022-gnn-calibration, huang-2021-correct-and-smooth, jia-benson-2020-residual-correlation, congalton-1988-error-autocorrelation, ma-2022-homophily-necessity, jin-2022-gnnlens, gnnfairviz-2025]
tags: [gnn-evaluation, hive-plot-matrix, heterogeneity, machine-learning, findings, counterfactual]
---

# GNN Heterogeneity Hive Plots: Empirical Findings and Prior-Art Positioning

Results from executing the [[cora-prototype-plan|Cora prototype plan]] (and extending it to CiteSeer), then running an adversarial literature check on every finding. This page is the honest scorecard: it supersedes the optimistic novelty claims in the [[gnn-heterogeneity-hive-plots|original proposal]]. The short version: of the three candidate findings, one is a rediscovery, one is refuted, and the third is real but its mechanism is already owned by prior theory. The durable contribution is a method (a covariate-adjusted residual screen) and a set of honest negative results, not a headline discovery.

Code and figures live in the `gnn-hiveplots` repo (`scripts/`, `notebooks/`, `outputs/`). Numbers below were recomputed directly from those outputs.

## What was built

- **Models:** GCN, GAT, GraphSAGE (2-layer, standard hyperparameters) on Cora and CiteSeer node classification. GCN test accuracy: Cora 80.7%, CiteSeer 68.2% (both in the expected literature range; GAT and GraphSAGE landed a couple of points low on CiteSeer, so cross-architecture claims there are held back).
- **Per-node structural properties:** degree, clustering, betweenness, Louvain community, local homophily, geodesic and aggregated-feature distance to the training set.
- **Per-edge error categories:** `both_correct` / `one_wrong` / `both_wrong` from endpoint correctness.
- **Visualizations:** confidence-sorted 3-axis overlay hive plots (both_wrong edges in red over gray context), `from_partition` matrices for multi-group variables, and a residual-screen view. The design went through two corrections: raw edge-density plots showed volume rather than heterogeneity, and multi-axis plots silently dropped most group pairs. Both are documented in the repo's `DIRECTIONS.md`.

## Findings scorecard

### 1. Homophily is the strongest simple failure predictor. Confirmed, and known.

Accuracy by local homophily bin, Cora GCN: heterophilous 31%, mixed 64%, homophilous 91% (Cramér's V 0.46; stable on the test split alone and across five seeds). CiteSeer repeats it: 30% / 58% / 83%. Degree is the weakest partition (Cora V 0.08), consistent with [[ma-2021-subgroup-fairness|Ma et al.]] finding no clear centrality trend.

This is a confirmation of established results, not a discovery. Local homophily predicting GNN accuracy is documented in Zhu et al. 2020 ("Beyond Homophily in GNNs"), [[ma-2022-homophily-necessity|Ma et al. 2022]] ("Is Homophily a Necessity?"), Loveland et al. 2023 (local homophily discrepancies), and Mao et al. 2023 ("Demystifying Structural Disparity in GNNs"). It is also close to mechanical: local homophily is computed from true neighbor labels, and a smoothing classifier votes each node toward its neighbors' classes. Any writeup must cite this literature and frame the result as a sanity confirmation on these benchmarks.

### 2. "Errors cluster on edges" as contagion. Refuted.

The proposal's headline novelty was edge-level heterogeneity: misclassified nodes connect to each other, so errors might propagate. The raw signal looked strong. `both_wrong` edges are enriched 2.22x over a node-independence null (Cora GCN). But that null is weak, and the enrichment collapses under progressively stronger ones:

| Null (shuffle `correct` within) | Cora GCN `both_wrong` enrichment |
|---|---|
| node independence | 2.22x |
| degree bin (A.1) | 2.81x |
| homophily bin (A.2) | 1.49x |
| Louvain community x homophily bin (A.3) | **1.16x** (seed-stable 1.20 +/- 0.05) |

Under the community-x-homophily null the excess is essentially gone. The apparent edge clustering is a consequence of errors concentrating in particular communities and homophily bands, not pairwise error propagation. There is a mechanical reason to expect even the residual: a 2-layer GCN gives adjacent nodes largely overlapping receptive fields, so correlated errors follow from shared inputs, not contagion.

Worse for the novelty claim, the *premise* is textbook. That GNN classification errors are positively autocorrelated along edges is the explicit operating assumption of [[huang-2021-correct-and-smooth|Correct & Smooth]] (Huang et al., ICLR 2021), which propagates training-node errors across edges to *improve* predictions, and it is modeled directly by [[jia-benson-2020-residual-correlation|Jia & Benson]] (KDD 2020) for regression. So there was never an edge-contagion discovery to be had. This is recorded as a negative result so nobody re-claims it.

### 3. Miscalibration structured by distance-to-training. Real, but prior art.

Among misclassified nodes, those close to the training set fail with high confidence while distant and disconnected nodes fail with low confidence. Spearman rho of geodesic-distance vs wrong-node confidence is -0.48 on Cora and -0.59 on CiteSeer. It survives a homophily control (Cora -0.48 to -0.35 after removing homophily-bin means; CiteSeer barely moves, -0.59 to -0.57).

This was the "keeper" until the literature check. It is prior art. [[hsu-2022-gnn-calibration|Hsu et al. 2022]] ("What Makes GNNs Miscalibrated?", NeurIPS) lists distance-to-training nodes as one of five named factors governing per-node GNN calibration, measured on GCN, on Cora and CiteSeer, with the same shortest-path distance metric. SimCalib (AAAI 2024) states the ECE-distance correlation outright and feeds min-distance-to-training into its calibrator. [[jin-2022-gnnlens|GNNLens]] already visualized the accuracy half (misclassified nodes tend to be far from labeled nodes). So "calibration depends on distance to training" cannot be claimed.

What narrowly survives, and it is a refinement rather than a finding:

- **The error-conditional slice.** Hsu report aggregate nodewise calibration error over all nodes and conclude near-training nodes are *better* calibrated. That aggregate hides a complementary fact: the errors that *do* occur near training are confident errors. On Cora, 57% of adjacent misclassifications carry confidence above 0.5 versus 5% of far ones; on CiteSeer, 85% versus 14%. The correct-minus-wrong confidence gap stays positive at every distance (roughly 0.19 to 0.23 on Cora), so near-training errors sit genuinely above the reliability diagonal, not merely on a globally decaying confidence curve. That rules out the obvious artifact (GNNs grow under-confident far from labels, so wrong-node confidence would fall with distance mechanically).
- **Homophily-partialled distance.** Hsu treat distance (their factor 3) and neighborhood agreement (their factor 5) as separate and never deconfound them. The residualized rho is a small step they did not take.
- **The disconnected-component endpoint** (no-path nodes as the low-confidence limit) is undocumented in this literature.
- **The accuracy-to-calibration bridge.** [[ma-2021-subgroup-fairness|Ma, Deng & Mei 2021]] establish accuracy dropping with distance-to-training; no found paper explicitly carries that to calibration, though Hsu effectively closes the gap.

Honest verdict: confirmation of a known calibration factor with a narrow error-conditional refinement. Not a headline.

### 4. Residual screen finds intra-class failure pockets. Novel as a composition; mechanism owned; label noise not ruled out.

The residual screen fits a logistic baseline predicting per-node correctness from known risk factors (homophily, degree, distance-to-train, clustering, feature distance, class; Cora accuracy 0.85 with homophily the dominant coefficient at +1.24), converts endpoint failure probabilities to an edge-level independence null q = (1 - p_i)(1 - p_j), and flags `both_wrong` edges the baseline rated safe (q below 0.15). These "unexpected failures" concentrate on intra-class edges inside otherwise-homophilous regions: Cora 255 unexpected-fail edges, 112 inside Probabilistic_Methods; CiteSeer 360, led by IR (95), DB (87), ML (60).

**Method novelty: the composition only.** Every ingredient is prior art:
- Errors correlate on edges: [[huang-2021-correct-and-smooth|Correct & Smooth]], [[jia-benson-2020-residual-correlation|Jia & Benson]].
- Per-node error is predictable from these exact covariates: [[ma-2021-subgroup-fairness|Ma et al. 2021]], [[subramonian-2024-degree-bias|degree-bias theory]], Loveland et al. 2023, Mao et al. 2023.
- An independence null from multiplied marginals is textbook (modularity's expected-edge term, configuration models, network-backbone "surprise", epidemiology observed/expected ratios).
- Testing whether classification errors cluster beyond chance on an adjacency is [[congalton-1988-error-autocorrelation|Congalton 1988]]: the spatial-statistics join-count "both-endpoints-wrong" statistic is structurally identical to the `both_wrong` count, with a spatial-permutation null.
- Fitting an interpretable meta-model to separate expected-hard from model-weakness errors exists (arXiv:2302.09952), as does residual-vs-model bias scanning (Zhang & Neill 2017).
- Slice discovery (SliceFinder, SliceLine, DOMINO, Spotlight, DivExplorer, Exceptional Model Mining) is a mature field, but always marginal (raw error vs overall), tabular, never over graph topology.
- The nearest visual-analytics cousin, [[jin-2022-gnnlens|GNNLens]], computes almost the same covariate list for GNN error diagnosis but externalizes correlation-finding to the human eye, with no fitted baseline, no residual, and no edge-level unit.

What no source does, and the only defensible novelty: a covariate-fitted per-node error baseline, an edge-level independence-null joint-failure target, over graph topology, used purely as a diagnostic screen (not to improve predictions), with a positional layout. Congalton 1988 is the sharpest antecedent to cite and distinguish (permutation null vs our covariate-conditioned null); Correct & Smooth is the mandatory citation (we test what it assumes).

**The finding (clustered same-class failures): mechanism owned, localization is ours, label noise open.** [[ma-2022-homophily-necessity|Ma et al. 2022]] and Luan et al. 2023 (the "mid-homophily pitfall" and intra-class node distinguishability) already explain *why* homophilous same-class nodes can fail: their neighborhood distributions are indistinguishable from another class. A two-sentence empirical antecedent exists on these exact benchmarks (Zorro, Funke et al. 2022, Section 8.2: whole same-class high-homophily groups assigned the wrong label). So the phenomenon's existence is not new; the reproducible covariate-adjusted *localization* is.

The live threat is label noise. GraphCleaner (ICML 2023) documents real mislabels in Cora and CiteSeer, and a 2025 position paper reports heavy feature-label leakage and duplicate nodes in both. Critically, mislabel detectors define a mislabel as a node that *disagrees* with its neighborhood, so a coherent pocket of consistently mislabeled or genuinely ambiguous same-topic papers would be invisible to them and would surface as exactly our "unexpected intra-class joint failures." Since intra-class status is defined by the possibly-wrong labels, the pockets could be data artifacts. A partial triage (the pocket nodes fail at low confidence, 0.38 to 0.49, with diffuse confusion rather than one dominant confuser) is more consistent with genuinely ambiguous subpopulations than with clean label swaps, but does not rule out coherent noise. The highest-value next check is a manual audit of the pocket papers against GraphCleaner's released mislabel lists.

## What the hive plot actually contributes

Every validated result here came from a groupby table, a chi-squared test, or a permutation test. The hive plot did not generate a finding; in one case an early misread of a swapped column led the narrative astray until the statistics corrected it. What the layout does contribute is positional: it shows whether a hot pocket is edges between groups, within a group, or at one end of a confidence-sorted axis, which a heatmap cell cannot. That is a real but modest contribution, and it is the honest ceiling on any "hive plots for GNN diagnosis" claim. [[jin-2022-gnnlens|GNNLens]] already occupies the "visual GNN error diagnosis" space with richer interactivity; the hive-plot angle is a static, layout-first complement, not a replacement.

## Prior-art map (must cite and distinguish)

| Our element | Closest prior art | How ours differs |
|---|---|---|
| Homophily predicts failure | Zhu 2020; [[ma-2022-homophily-necessity|Ma 2022]]; Loveland 2023; Mao 2023 | Confirmation only |
| Errors cluster on edges | [[huang-2021-correct-and-smooth|Correct & Smooth]]; [[jia-benson-2020-residual-correlation|Jia & Benson]] | We test (and largely dissolve) what they assume |
| Calibration by distance-to-train | [[hsu-2022-gnn-calibration|Hsu 2022]] (factor 3); SimCalib 2024 | Error-conditional slice; homophily-partialled |
| Both-wrong-edge screen | [[congalton-1988-error-autocorrelation|Congalton 1988]] join-count | Covariate-conditioned null, arbitrary GNN graph |
| Fitted error baseline | arXiv:2302.09952; Zhang & Neill 2017 bias scan | Graph covariates; edge-level target |
| Error slice discovery | SliceFinder, DOMINO, Spotlight, EMM | Residual-vs-baseline, over graph topology |
| Visual GNN error diagnosis | [[jin-2022-gnnlens|GNNLens]] | Fitted baseline + residual + edge unit; static layout |
| Clustered same-class failure | Ma 2022; Luan 2023; Zorro 8.2 | Reproducible covariate-adjusted localization |

## Open threads and next steps

- **Rule out label noise** in the residual pockets (manual audit vs GraphCleaner lists). This gates any "model blind spot" claim.
- **Strengthen the calibration slice** by reporting the correct-vs-wrong reliability gap by distance as the primary statistic (done above; fold into any writeup so the finding is not read as the known aggregate effect).
- **Differential model screen** (the most promising unbuilt direction): tag edges by GCN-fails-but-GraphSAGE-does-not vs the reverse. Shared failure structure cancels; only architecture-specific blind spots remain. Data already exists in `outputs/`. This is the ML incarnation of the [[differential-hive-plot]].
- **CiteSeer is done** and amplifies the (demoted) calibration effect and the residual pockets; it also has 1,052 unreachable nodes across 438 components, a much larger disconnected-structure story than Cora's 158.
- **Honest framing for any paper or post:** lead with the method (residual screen) and the negative results (aggregate metrics mislead, but so does naive edge-clustering), not with a discovery. The counterfactual pass found a refuter or an owner for every candidate discovery.

## See Also

- [[gnn-heterogeneity-hive-plots]] — The original proposal (optimistic novelty claims; superseded by this scorecard)
- [[cora-prototype-plan]] — The implementation plan that produced these results
- [[gnn-evaluation]] — The evaluation gap, updated with calibration and the residual screen
- [[hsu-2022-gnn-calibration]] — The prior art for finding 3
- [[huang-2021-correct-and-smooth]] / [[jia-benson-2020-residual-correlation]] — The prior art for finding 2 and the residual screen's skeleton
- [[congalton-1988-error-autocorrelation]] — The spatial-statistics antecedent for the both-wrong-edge screen
- [[ma-2022-homophily-necessity]] — The mechanism behind the intra-class pockets
- [[jin-2022-gnnlens]] — The nearest visual-analytics prior art
- [[ma-2021-subgroup-fairness]] — Accuracy-by-distance foundation
- [[subramonian-2024-degree-bias]] — Degree-bias survey
