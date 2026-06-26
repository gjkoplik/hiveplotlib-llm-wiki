---
title: "Walkthrough: Zachary's Karate Club Hive Plot"
type: analysis
created: 2026-04-06
updated: 2026-06-25
sources: [hiveplotlib-python-repo]
tags: [walkthrough, karate-club, hiveplotlib, social-network]
---

# Walkthrough: Zachary's Karate Club Hive Plot

A step-by-step walkthrough of the `karate_club.ipynb` example from [[hiveplotlib]], demonstrating how to build a [[hive-plot]] from a classic social network dataset.

## The Dataset

**Zachary's Karate Club** (1977) — Wayne Zachary observed a university karate club from 1970–1972 that split into two factions:
- **"Mr. Hi" faction** — Supported the instructor (node 0)
- **"Officer" faction** — Supported the club officers (node 33)

34 members, edges represent who socialized outside of class. The data was recorded right before the formal split.

Available as `nx.karate_club_graph()` in NetworkX.

## Construction Steps

### 1. Load and Convert Data

```python
import networkx as nx
from hiveplotlib.converters import networkx_to_nodes_edges

G = nx.karate_club_graph()
nodes, edges = networkx_to_nodes_edges(G)
```

The converter extracts node IDs and edge pairs from the NetworkX graph into [[hiveplotlib]]'s `NodeCollection` and `Edges` objects.

### 2. Add Degree Information

```python
import pandas as pd

degrees = pd.DataFrame(G.degree, columns=[nodes.unique_id_column, "degree"])
nodes.data = nodes.data.merge(degrees, on=nodes.unique_id_column)
```

Node degree (number of connections) is computed from NetworkX and added to the node data. This will be the sorting variable.

### 3. Create the HivePlot

```python
from hiveplotlib import HivePlot

hp = HivePlot(
    nodes=nodes,
    edges=edges,
    partition_variable="club",        # Axis assignment: faction membership
    sorting_variables="degree",       # Node position: degree (more connected → farther from center)
    repeat_axes=True,                 # Duplicate axes to show intra-faction edges
    non_repeat_edge_kwargs={"color": "darkgray"}
)
```

Key decisions:
- **[[node-assignment|Partition variable]]** = `"club"` → nodes assigned to axes by faction
- **Sorting variable** = `"degree"` → high-degree nodes at axis tips
- **`repeat_axes=True`** → each faction axis is duplicated, enabling visualization of within-faction connections

### 4. Style Edges

```python
hp.update_edges("Mr. Hi", "Mr. Hi", color="royalblue")      # Intra-faction: blue
hp.update_edges("Officer", "Officer", color="darkorange")     # Intra-faction: orange
# Inter-faction edges remain darkgray (set in constructor)
```

Three visual categories emerge from the [[edge-rendering|edge kwarg system]]:
- Blue = Mr. Hi internal connections
- Orange = Officer internal connections
- Gray = cross-faction connections

### 5. Clean Up Repeat Edges

```python
hp.reset_edges(axis_id_1="Mr. Hi_repeat", axis_id_2="Officer")
```

Removes redundant edges between repeat axes to avoid visual clutter.

### 6. Plot and Annotate

```python
fig, ax = hp.plot()
# Highlight the two leaders with scatter markers
```

## What the Hive Plot Reveals

1. **Faction separation is clear.** Blue and orange edges dominate; gray cross-faction edges are sparse. The two factions are socially distinct.

2. **Leaders are visually prominent.** Mr. Hi (node 0) and Officer (node 33) sit at the tips of their axes because they have the highest degree. The sorting variable makes the social hierarchy explicit.

3. **Cross-faction connections are NOT driven by sociability.** If the most social people were the bridge-builders, gray edges would cluster at axis tips (where high-degree nodes are). Instead, gray edges are distributed across all positions — cross-faction engagement is independent of within-faction popularity.

4. **The repeat axis trick works.** Without axis duplication, intra-faction edges would be invisible (same-axis self-connections). The `repeat_axes=True` parameter makes the dominant blue/orange edge bundles visible.

## Design Takeaways

- **Partition variable choice matters most.** Faction membership is the obvious choice here, but alternative partitions (e.g., by betweenness centrality) would reveal different structure.
- **Degree as sorting variable** creates an intuitive "more connected = further out" reading.
- **Color-coding by edge category** turns a complex network into an immediately readable visualization.
- **The hive plot directly answers research questions** that [[force-directed-layout|force-directed layouts]] leave ambiguous.

## Comparison with Other Examples

| Example | Data | Partition | Sorting | Backend | Insight Type |
|---------|------|-----------|---------|---------|-------------|
| **Karate Club** | 34 nodes, social | Faction | Degree | matplotlib | Group separation |
| **Election 96** | 944 voters, survey | Vote × party (P2CP) | Education/age/income | matplotlib | Demographic patterns |
| **Bitcoin OTC** | 1000s, trust ratings | Trust level received | Trust given | datashader (HivePlotMatrix) | Temporal behavior change |

## See Also

- [[hiveplotlib]] — The library
- [[hive-plot]] — The visualization method
- [[node-assignment]] — Partitioning strategy
- [[edge-rendering]] — Edge styling
- [[examples-and-applications]] — catalog of hiveplotlib examples and application explorations
