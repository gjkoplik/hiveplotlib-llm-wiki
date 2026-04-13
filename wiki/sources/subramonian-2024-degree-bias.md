---
title: "Subramonian, Kang & Sun 2024 — Origins of Degree Bias in GNNs"
type: source
created: 2026-04-13
updated: 2026-04-13
sources: []
tags: [gnn-evaluation, degree-bias, fairness, survey, generalization-theory]
---

# Subramonian, Kang & Sun 2024 — Origins of Degree Bias in GNNs

## Citation

Arjun Subramonian, Jian Kang, Yizhou Sun. "Theoretical and Empirical Insights into the Origins of Degree Bias in Graph Neural Networks." *NeurIPS 2024*. UCLA. arXiv:2404.03139.

Code: `github.com/ArjunSubramonian/degree-bias-exploration`

## Summary

A comprehensive survey and theoretical analysis of **degree bias** in GNNs — the phenomenon where model performance varies systematically with node degree. Surveys **38 papers**, catalogs **10+ competing hypotheses**, and provides the **first rigorous probabilistic bounds** connecting node degree to misclassification probability. A central finding: many existing hypotheses are "not rigorously validated and can even be contradictory," and the answer depends on which graph filter type is used.

## The Hypothesis Catalog (38 Papers Surveyed)

### Top 5 Hypotheses (Table 1)

| ID | Hypothesis | Papers Citing |
|----|-----------|---------------|
| **H1** | Low-degree neighborhoods contain insufficient or overly noisy information for effective representations | 21 |
| **H2** | High-degree nodes have greater influence on GNN training via more links, dominating message passing | 7 |
| **H3** | High-degree nodes exert more influence on representations as GNN layers increase | 5 |
| **H4** | In semi-supervised learning, test predictions for high-degree nodes are more likely to be influenced by training nodes (more links = more training neighbors) | 3 |
| **H5** | Representations of high-degree nodes cluster more strongly around class centers / more linearly separable | 3 |

### Additional Hypotheses (Table 2, Appendix)

| ID | Hypothesis |
|----|-----------|
| **H6** | Low-degree nodes have less overlap with training nodes |
| **H7** | GNN expressive power limits low-degree node performance |
| **H8** | Low-degree nodes suffer from over-smoothing |
| **H9** | Skip connections amplify degree bias |
| **H10** | High-degree node representations have larger variance |

**Critical contradiction:** H5 says high-degree representations cluster more tightly; H10 says they have larger variance. The paper shows this contradiction dissolves once you distinguish between graph filter types (RW vs. SYM).

## Theoretical Contributions

### Key Theorems

**Theorem 1 (General Misclassification Bound):** For test node *i* with true label *c*:
P(misclassification) ≤ 1/(1 + R_{i,c'}), where R_{i,c'} is the "squared inverse coefficient of variation" — a normalized dispersion measure of how tightly node *i*'s representations cluster.

**Theorem 2 (Random Walk Filter):** Decomposes R_{i,c'} into two interpretable factors:
- **α_i^l (collision probability):** Probability that two independent length-*l* random walks from node *i* end at the same node. **Low-degree nodes have higher collision probability** (less neighborhood diversity).
- **β_{i,c'}^l (l-hop prediction homogeneity):** Expected class-coherence of node *i*'s *l*-hop neighborhood. Captures how much the extended neighborhood "agrees" on class predictions.

**Theorem 3 (Symmetric Filter):** Analogous to Theorem 2 but with degree-discounted variants that suppress high-degree contributions via 1/√(D) weighting. Creates a **different geometric mechanism**: low-degree nodes aren't noisier, they're positioned closer to decision boundaries.

**Theorem 4 (Training Dynamics):** Loss change per training step scales with √(degree) for SYM-normalized filters. **Low-degree nodes' losses decrease more slowly during gradient descent.**

### The Core Insight

Degree bias decomposes into three factors:
1. **Neighborhood diversity** (inverse collision probability, correlates with degree)
2. **Neighborhood class coherence** (prediction homogeneity, relates to local [[structural-heterogeneity|homophily]])
3. **Differential training dynamics** (for SYM-normalized filters, loss updates scale with √degree)

The relative importance of each factor depends on the graph filter type.

## Validated vs. Contradicted Hypotheses

**Supported:**
- **H1** (partially) — Validated as "inverse collision probability," a precise formalization of "insufficient information"
- **H4, H6** — Training node overlap effects confirmed
- **H5** (for RW filter) — Low-degree nodes do exhibit larger representation variance under random walk normalization

**Contradicted or reframed:**
- **H3** — Degree bias persists in shallow networks (3 layers); NOT primarily a depth effect
- **H7** — GNNs achieve ~100% training accuracy across all degrees, so expressive power is NOT the bottleneck
- **H10** — For RW filter, *low*-degree nodes have larger variance (opposite of H10). For SYM filter, low-degree nodes have lower variance but are closer to decision boundaries — a different mechanism entirely
- **H5 vs. H10 contradiction** — Resolves once you distinguish filter types (RW vs. SYM create bias through fundamentally different geometric mechanisms)

## Experimental Setup

- **Datasets (8):** Cora, CiteSeer, PubMed, OGB-Arxiv (homophilic citation networks); Chameleon, Squirrel, Actor (heterophilic); Penn94 (social)
- **Graph filters:** RW (random walk, D⁻¹A), SYM (symmetric, D⁻¹/²AD⁻¹/²), ATT (attention-based, as in GAT)
- **Models:** GCN, GAT (implemented in PyTorch Geometric)
- **Metrics:** Test loss vs. degree (scatter + binned error bars), inverse collision probability, PCA of representations, variance by degree group, training loss curves by degree bin

## Visualization

All standard statistical plots — **no network visualization of any kind**:
- Scatter plots: test loss vs. degree with error bars (10 seeds)
- PCA projections of node representations colored by class, stratified by degree
- Variance comparison plots across degree groups
- Training loss curves by degree bin (showing differential learning rates)
- Inverse collision probability vs. degree plots

## Relevance to Hive Plot Proposal

This paper is highly relevant to the [[gnn-heterogeneity-hive-plots|HivePlotMatrix proposal]] in several ways:

1. **The field lacks rigorous validation of competing hypotheses.** A visual tool that *shows* degree-stratified patterns rather than hypothesizing about them could cut through the contradictions.

2. **The RW vs. SYM distinction creates different geometric mechanisms.** A HivePlotMatrix comparing models with different normalizations could visually expose whether misclassification patterns look different — something the paper demonstrates numerically but not visually in graph space.

3. **Inverse collision probability and prediction homogeneity are new sweep variables.** These could be added as partition or sorting variables in a HivePlotMatrix, alongside degree, centrality, and training-set distance.

4. **The paper's degree-stratified scatter plots are exactly what hive plots could enhance.** Mapping nodes to axes by degree and sorting by loss or confidence preserves the edge structure that scatter plots discard.

5. **Heterophilic graphs show weaker degree bias.** This suggests local homophily ratio is a key moderating variable in the sweep — worth including as a partition variable in the HivePlotMatrix.

## Recommended Roadmap (Section 6)

1. **Maximize inverse collision probability** for low-degree nodes via edge augmentation targeting neighborhood diversity
2. **Increase l-hop prediction homogeneity** for low-degree nodes — balance class representation in extended neighborhoods
3. **Address training dynamics** — adjusted learning rates or batch sampling to compensate for √degree scaling
4. **Be filter-aware** — RW and SYM create bias through different mechanisms, so mitigation must be filter-specific

## Limitations

- Analysis relies on **linearized GNN models**; nonlinear models may behave differently
- **Node classification only** — may not transfer to link prediction or graph classification
- Roadmap strategies are theoretical, need practical testing
- CSBM assumptions common in prior work are inadequate for real-world power-law graphs

## See Also

- [[ma-2021-subgroup-fairness]] — Foundational subgroup fairness paper (training-set distance perspective)
- [[gnn-evaluation]] — The evaluation gap this paper documents at the degree level
- [[structural-heterogeneity]] — Degree heterogeneity is a canonical case
- [[gnn-heterogeneity-hive-plots]] — Research proposal for visual diagnosis (could use ICP as a sweep variable)
- [[graph-neural-networks]] — The models analyzed, with filter-specific bias analysis
- [[gnnfairviz-2025]] — Visual analytics tool (different framing: demographic attributes)
- [[kipf-2017-gcn]] — GCN uses the SYM filter analyzed in this paper
