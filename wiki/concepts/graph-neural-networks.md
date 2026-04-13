---
title: Graph Neural Networks
type: concept
created: 2026-04-13
updated: 2026-04-13
sources: [kipf-2017-gcn, ma-2021-subgroup-fairness, subramonian-2024-degree-bias, gnnfairviz-2025]
tags: [graph-neural-networks, machine-learning, node-classification]
---

# Graph Neural Networks

Graph neural networks (GNNs) are a family of deep learning models that operate directly on graph-structured data. They learn node, edge, or graph-level representations by iteratively aggregating information from local neighborhoods — a paradigm known as **message passing**.

## Message-Passing Paradigm

At each layer, a GNN updates every node's representation by:

1. **Aggregating** messages from neighboring nodes (sum, mean, attention-weighted, etc.)
2. **Combining** the aggregated message with the node's own representation
3. **Transforming** through a learned function (typically a neural network layer)

After *k* layers, each node's representation encodes information from its *k*-hop neighborhood. This neighborhood aggregation is both the strength and limitation of GNNs — it works well in homophilous regions (where neighbors share labels) but can struggle under heterophily.

## Key Architectures

| Architecture | Key Idea | Reference |
|-------------|----------|-----------|
| **GCN** | Spectral convolution approximated as normalized neighbor averaging | [[kipf-2017-gcn|Kipf & Welling 2017]] |
| **GAT** | Attention-weighted neighbor aggregation | Veličković et al. 2018 |
| **GraphSAGE** | Inductive learning via sampled neighborhoods | Hamilton et al. 2017 |
| **GIN** | Maximally expressive message passing (WL-equivalent) | Xu et al. 2019 |
| **Graph Transformers** | Self-attention over graph structure with positional encodings | Ying et al. 2021, Rampášek et al. 2022 |

## Standard Tasks

- **Node classification** — Predict labels for individual nodes (e.g., paper topic in a citation network). This is the primary task relevant to [[gnn-heterogeneity-hive-plots|hive plot evaluation]].
- **Link prediction** — Predict missing or future edges
- **Graph classification** — Predict properties of entire graphs (e.g., molecular activity)

## Standard Benchmarks

| Dataset | Nodes | Classes | Domain |
|---------|-------|---------|--------|
| **Cora** | 2,708 | 7 | Citation network (ML papers) |
| **CiteSeer** | 3,327 | 6 | Citation network |
| **PubMed** | 19,717 | 3 | Citation network (diabetes) |
| **OGB-arxiv** | 169,343 | 40 | arXiv papers |

These benchmarks are typically evaluated with aggregate accuracy or F1 — an approach that masks [[structural-heterogeneity|structural performance variation]], as discussed in [[gnn-evaluation]].

## Connection to Hive Plots

GNNs produce predictions over graph-structured data where nodes have rich structural properties (degree, clustering coefficient, community membership). This makes GNN outputs a natural fit for [[hive-plot|hive plot]] visualization: nodes can be assigned to axes by structural properties and sorted by prediction confidence, revealing patterns that aggregate metrics hide. See [[gnn-heterogeneity-hive-plots]] for the full research proposal.

## See Also

- [[gnn-evaluation]] — How GNN performance is measured (and what's missed)
- [[structural-heterogeneity]] — Why GNNs perform unevenly across graph structure
- [[gnn-heterogeneity-hive-plots]] — Research proposal: hive plot matrices for GNN diagnostics
- [[hive-plot]] — The visualization method
- [[kipf-2017-gcn]] — Foundational GCN paper
- [[subramonian-2024-degree-bias]] — Degree bias depends on graph filter type (RW vs. SYM vs. ATT)
- [[gnnfairviz-2025]] — Visual analytics for GNN fairness (demographic-attribute framing)
