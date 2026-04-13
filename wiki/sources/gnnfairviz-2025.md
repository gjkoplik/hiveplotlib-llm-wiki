---
title: "GNNFairViz (Ye et al. 2025) — Visual Analytics for GNN Fairness"
type: source
created: 2026-04-13
updated: 2026-04-13
sources: []
tags: [gnn-evaluation, fairness, visual-analytics, demographic-fairness]
---

# GNNFairViz (Ye et al. 2025) — Visual Analytics for GNN Fairness

## Citation

Xinwu Ye, Jielin Feng, Erasmo Purificato, Ludovico Boratto, Michael Kamp, Zengfeng Huang, Siming Chen. "GNNFairViz: Visual Analysis for Graph Neural Network Fairness." *IEEE Transactions on Visualization and Computer Graphics*, vol. 31, no. 10, pp. 7153–7170, October 2025. DOI: 10.1109/TVCG.2025.3542419.

**Affiliations:** Fudan University (visualization), University of Cagliari (fair ML/RecSys).

**Code:** `github.com/xinwuye/GNNFairViz` — also on PyPI (`pip install gnnfairviz`).

**Preprint:** TechRxiv (techrxiv.org/users/800054/articles/1180757).

## Summary

A multi-view interactive visual analytics tool for diagnosing fairness in [[graph-neural-networks|GNN]] predictions. Model-agnostic, supports multiple sensitive attributes with intersectional analysis. The core contribution is a **bias taxonomy** that separates model bias from data bias (attribute bias + structural bias) using counterfactual comparisons.

## System Design: Three-Phase Workflow

### Phase 1: Node Selection
- Embedding projection views (likely t-SNE/UMAP of GNN representations)
- Neighborhood influence views — how large and repetitive k-hop computation graphs become
- Structural concentration views — dense subgraph identification
- Interactive brush/select/filter to identify node subsets driving disparities

### Phase 2: Fairness Inspection
- Group-level output distributions across sensitive attribute groups
- **Intersectional analysis** — check whether coarse groupings (e.g., gender alone) mask disparities in finer subpopulations (e.g., age × nationality)
- Extended fairness metrics for multi-class predictions and multinary sensitive attributes (beyond binary)

### Phase 3: Bias Diagnosis
- **Counterfactual "what-if" comparisons:**
  - Obscure node attributes → if disparity changes, attribute bias is implicated
  - Remove edges → if disparity changes, structural bias is implicated
  - Both → interaction effects
- Change in group-level output distributions serves as evidence for each bias component
- Data-centric approach: probes how different data components drive model bias without modifying the model

## Bias Taxonomy (Core Methodological Contribution)

| Bias Type | Definition | Diagnostic Method |
|-----------|-----------|------------------|
| **Model bias** | Disparities in model outputs across sensitive groups | Measured by extended fairness metrics |
| **Attribute bias** | Group-conditional differences in feature distributions | Counterfactual: obscure attributes |
| **Structural bias** | Group-conditional differences in connectivity and neighborhood influence | Counterfactual: remove edges |

**Key finding: "Overwhelming Effect"** — when minority groups are small, their nodes connect largely into majority neighborhoods, and message passing dominates their representations with majority-group information. This demonstrates that graph structure mediates GNN outputs in ways that aggregate metrics miss.

## Models and Datasets

- **Model-agnostic** framework; tested with GCN and GAT
- Standard GNN fairness datasets: Pokec-z/Pokec-n (Slovak social network), NBA, German Credit, Bail/Recidivism
- Evaluated via two usage scenarios + expert interviews

## Graph Layouts Used

- Embedding projection views (dimensionality reduction of GNN representations)
- Connectivity summaries (group-to-group at different hop distances)
- **No hive plots, no axis-based rational layouts, no force-directed node-link diagrams confirmed**
- Structural analysis is always **group-conditional** — "how does structure mediate fairness between demographic groups?"

## Limitations

- **Scalability** to larger graphs remains challenging
- **Single-model analysis** — no cross-model comparison (future work)
- **Node classification only** — link prediction and recommendation planned for future
- **Learning curve** for users unfamiliar with coordinated interactive views
- **No automated guidance** — requires human judgment to interpret counterfactual results

## What GNNFairViz Does NOT Do

This is the critical distinction for the [[gnn-heterogeneity-hive-plots|hive plot proposal]]:

- **Does NOT partition by structural properties** (degree, centrality, community) as independent analysis axes
- **Does NOT ask "do high-degree nodes classify better than low-degree nodes?"** — only "do demographic group A nodes have different outcomes than group B?"
- **Does NOT analyze edge-level heterogeneity** — no examination of which edge types carry misclassification
- **Does NOT provide a systematic sweep** across candidate structural decompositions
- **Does NOT produce a static "visual model card"** — designed for interactive iterative exploration

## Relationship to Hive Plot Proposal

| Dimension | GNNFairViz | [[gnn-heterogeneity-hive-plots|HivePlotMatrix Proposal]] |
|-----------|-----------|------------------------|
| **Fairness framing** | Demographic attributes (gender, race, region) | Structural subgroups (degree, centrality, training distance) |
| **Graph layout** | Embedding projections, connectivity summaries | Rational axis-based layout ([[hive-plot]]) |
| **Decomposition** | Fixed by sensitive attribute | Swept across candidate properties via matrix |
| **Edge analysis** | Not a focus | Primary contribution — edge-level heterogeneity |
| **Structural analysis** | Group-conditional ("how does structure mediate demographic fairness?") | Structure-primary ("how does structure predict model failure?") |
| **Output format** | Interactive multi-view tool | Static visual model card (HivePlotMatrix) |
| **Target user** | Fairness auditor | ML practitioner diagnosing model behavior |

The two approaches are **complementary**. GNNFairViz's "Overwhelming Effect" finding is directly relevant — it confirms that graph structure mediates GNN outputs in ways aggregate metrics miss. The hive plot approach could extend this insight from demographic groups to structural subgroups. GNNFairViz's counterfactual diagnostic methodology (obscure attributes / remove edges / both) could also potentially be combined with hive-plot-based structural decomposition.

## See Also

- [[gnn-heterogeneity-hive-plots]] — Research proposal for structural (not demographic) fairness diagnosis
- [[ma-2021-subgroup-fairness]] — Theoretical foundation for structural subgroup analysis
- [[gnn-evaluation]] — The evaluation gap both approaches address
- [[graph-neural-networks]] — The models analyzed
- [[hive-plot]] — The layout method distinguishing the HivePlotMatrix approach
- [[subramonian-2024-degree-bias]] — Degree bias survey (also no visualization framework)
