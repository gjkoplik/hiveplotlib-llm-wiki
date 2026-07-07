---
title: GNN Over-smoothing
type: concept
created: 2026-07-06
updated: 2026-07-06
sources: [kipf-2017-gcn]
tags: [graph-neural-networks, over-smoothing, gnn-evaluation, hive-plot-matrix, mechanism]
---

# GNN Over-smoothing

Over-smoothing is the tendency of a [[graph-neural-networks|GNN]] to make node representations converge toward one another as depth increases. After enough message-passing layers, every node's embedding collapses toward a common vector (within a connected component), class separation dissolves, and accuracy drops. It is the main reason standard GCN/GAT/GraphSAGE models are shallow (2–3 layers) rather than deep.

## Why it happens

Each layer replaces a node's representation with a (normalized) aggregate of its neighbors' representations. Repeated aggregation is a smoothing operator on the graph: iterating it drives the signal toward the dominant eigenvector of the propagation matrix, which for a connected component is a near-constant vector up to degree scaling. So depth trades *reach* (a k-layer node sees its k-hop neighborhood) against *discriminability* (the representations of distinct nodes wash out). This is the flip side of the message-passing strength documented in [[structural-heterogeneity]]: the same averaging that helps in homophilous regions destroys class signal when applied too many times.

## How it is measured today

- **Dirichlet energy** of the node features on the graph (Cai & Wang 2020; Rusch et al. 2023 survey): the sum of squared differences across edges, which decays toward zero as smoothing proceeds.
- **Mean Average Distance (MAD)** (Chen et al. 2020, "Measuring and Relieving the Over-smoothing Problem"): average pairwise cosine distance among node representations; falls as depth grows.
- **A t-SNE / UMAP scatter** of the final-layer embeddings, which visibly collapses into a single blob at high depth.

The first two are scalars: they report *that* separation is lost but not *its shape*. The scatter shows shape but is non-deterministic frame to frame, so a depth sequence wobbles for reasons unrelated to smoothing. This is the gap a rational layout can fill.

## The hive plot angle

Foundational references: Li, Han & Wu 2018 ("Deeper Insights into Graph Convolutional Networks"), Oono & Suzuki 2020 ("Graph Neural Networks Exponentially Lose Expressive Power").

Render over-smoothing as a [[hive-plot-matrix|HivePlotMatrix]], one cell per layer:

- **Axes** = classes (or communities). Fixed across all cells so the panels are comparable ([[fixed-layout-comparability]]).
- **Within-axis position** = a 1-D projection of that layer's embedding (e.g. the first principal component, or distance to the class centroid). Deterministic, unlike t-SNE, so motion across the matrix is real signal.
- **Edges** = graph adjacency.

At layer 1 the nodes spread along their axes and the edges are structured. As depth grows, the within-axis distributions tighten and migrate toward the center, and the edges collapse into a dense hub tangle. The reader sees the collapse and its *shape* in one artifact rather than reading it off a decaying curve. The construction is turnkey: cache each layer's embeddings during a forward pass and feed them as sort variables through `HivePlotMatrix.from_partition(graph=...)`.

## Honest positioning

The phenomenon is textbook, so any contribution is the *rendering*, and even that must be checked against existing embedding-visualization tools ([[jin-2022-gnnlens|GNNLens]], the [[nn-training-dynamics-p2cp-exploration|nn-viz]] training movies, and the t-SNE-over-depth figures in the over-smoothing papers themselves). The claim to defend is narrow and legitimate: a deterministic, layout-first view that makes the collapse legible faster than a scalar metric or a wobbling scatter. See [[gnn-research-directions]] for where this sits among the broader menu.

## See Also

- [[gnn-research-directions]] — the direction menu this is the top pick of
- [[graph-neural-networks]] — the message-passing paradigm that causes smoothing
- [[structural-heterogeneity]] — the homophily story over-smoothing is the excess of
- [[hive-plot-matrix]] — the per-layer construction
- [[fixed-layout-comparability]] — why fixed axes make the layer panels comparable
- [[nn-training-dynamics-p2cp-exploration]] — the deterministic-layout-over-time precedent
- [[gnn-evaluation]] — the evaluation frame this extends from accuracy to geometry
