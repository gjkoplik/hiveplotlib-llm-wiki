---
title: Node Assignment
type: concept
created: 2026-04-06
updated: 2026-04-06
sources: [krzywinski-2012, hiveplotlib-python-repo]
tags: [hive-plot, node-assignment, axis]
---

# Node Assignment

The process of mapping network nodes to axes in a [[hive-plot]]. This is the most critical design decision — it determines what the visualization reveals.

## Two Dimensions of Assignment

1. **Which axis** — A classification rule assigns each node to an axis
2. **Where on the axis** — A structural property determines position along the axis

## Axis Assignment Strategies

### For Directed Networks
From [[krzywinski-2012]]:
- **Sources** (regulators) → nodes with only outgoing edges
- **Managers** → nodes with both incoming and outgoing edges
- **Sinks** (workhorses) → nodes with only incoming edges

### For Undirected Networks
From [[krzywinski-2012]]:
- Clustering coefficient = 0 → one axis
- 0 < clustering coefficient < 1 → another axis
- Clustering coefficient = 1 → third axis

### In hiveplotlib
In [[hiveplotlib]], axis assignment is controlled by the **partition variable** — a column in the node data that classifies nodes into groups:

```python
hp.set_partition(partition_variable="node_type")
```

The `NodeCollection.create_partition_variable()` method can also generate partitions programmatically by binning a numeric column (unique values or quantiles).

### Spectral (data-driven)

Instead of a hand-written rule, a [[spectral-clustering|spectral clustering]] of the network can supply the partition: the first cuts of the graph Laplacian become the axes. This is the basis of [[spectral-hive-plots]]. It differs from the strategies above in that the axis definition is learned from the graph's connectivity rather than specified by a node attribute, and it pairs naturally with a spectral positioning variable (below).

## Positioning Parameters

Structural properties available for node positioning along axes (from [[krzywinski-2012]]):

**Node-level:** degree, flow (out − in), betweenness, closeness, eccentricity, PageRank, eigenvector centrality, clustering coefficient, topological overlap, cut vertex status.

**Network-level:** module membership, assortativity.

**Spectral:** a higher Laplacian eigenvector, one beyond the cuts that define the partition, orders nodes by within-cluster structure (see [[spectral-clustering]] and [[spectral-hive-plots]]). Ordering by an eigenvector is the classic [[atkins-1998-spectral-seriation|spectral seriation]] result.

In [[hiveplotlib]], positioning is set via:
```python
hp.set_sorting_variables({"axis_A": "degree", "axis_B": "betweenness"})
```

## See Also

- [[hive-plot]] — The visualization method
- [[hiveplotlib]] — Implementation
- [[edge-rendering]] — After nodes are placed, edges connect them
- [[spectral-clustering]] — A data-driven partition and a within-axis coordinate from one decomposition
- [[spectral-hive-plots]] — The research direction that assigns axes and sort spectrally
