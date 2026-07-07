---
title: "Cora Prototype Implementation Plan"
type: analysis
created: 2026-04-13
updated: 2026-07-06
sources: [hiveplotlib-python-repo, ma-2021-subgroup-fairness, gnn-heterogeneity-hive-plots]
tags: [implementation-plan, cora, gnn-evaluation, hive-plot-matrix, prototype]
---

# Cora Prototype Implementation Plan

> **Status (2026-07-06): shipped.** This plan was executed, and the work extended to CiteSeer. The empirical results, with an honest findings scorecard and prior-art positioning, are in [[gnn-heterogeneity-findings]]. Read that page for what actually happened; this page is kept as the original implementation record. Note that the plan below still frames edge-level heterogeneity as the primary contribution, which the results did not bear out (see the findings page).

Concrete implementation plan for the [[gnn-heterogeneity-hive-plots|GNN Heterogeneity Diagnosis via Hive Plot Matrices]] research proposal. This plan is self-contained and designed to be executed in a fresh session.

**Goal:** Train GCN/GAT/GraphSAGE on the Cora citation network, then build HivePlotMatrix visualizations that expose where each model fails and what edge-level patterns emerge.

**Key reference notebooks in hiveplotlib:**
- `/home/garyk/repos/hiveplotlib/examples/hpm_from_variable_sweep.ipynb` — HivePlotMatrix variable sweep (the core workflow)
- `/home/garyk/repos/hiveplotlib/examples/creating_hive_plots_from_networkx.ipynb` — NetworkX conversion
- `/home/garyk/repos/hiveplotlib/examples/visualizing_edge_metadata.ipynb` — Edge coloring by property
- `/home/garyk/repos/hiveplotlib/examples/visualizing_node_metadata.ipynb` — Node coloring by property

---

## Phase 1: Environment and Data Setup

### 1.1 Install dependencies

```bash
pip install torch torch-geometric networkx pandas numpy matplotlib
pip install -e /home/garyk/repos/hiveplotlib  # or pip install hiveplotlib
```

PyTorch Geometric provides Cora via `torch_geometric.datasets.Planetoid`.

### 1.2 Load Cora and extract structural properties

```python
import torch
import networkx as nx
import numpy as np
import pandas as pd
from torch_geometric.datasets import Planetoid

# Load Cora
dataset = Planetoid(root='/tmp/Cora', name='Cora')
data = dataset[0]

# Convert to NetworkX for hiveplotlib
G = nx.Graph()
for i in range(data.num_nodes):
    G.add_node(i)
for src, dst in data.edge_index.t().tolist():
    if src < dst:  # avoid duplicate edges in undirected graph
        G.add_edge(src, dst)

# Compute structural properties for each node
degree = dict(G.degree())
clustering = nx.clustering(G)
betweenness = nx.betweenness_centrality(G)
# Community detection (Louvain or similar)
import community as community_louvain  # pip install python-louvain
partition = community_louvain.best_partition(G)

# Store as node attributes
for node in G.nodes():
    G.nodes[node]['degree'] = degree[node]
    G.nodes[node]['clustering_coefficient'] = clustering[node]
    G.nodes[node]['betweenness_centrality'] = betweenness[node]
    G.nodes[node]['community'] = partition[node]
    G.nodes[node]['true_label'] = data.y[node].item()
```

---

## Phase 2: Train GNNs and Collect Predictions

### 2.1 Train GCN, GAT, GraphSAGE

Use standard PyTorch Geometric training loops. For each model:
- 2-layer architecture, 16 hidden dimensions
- Train on Cora's standard train/val/test split (140/500/1000 nodes)
- Use the standard split: `data.train_mask`, `data.val_mask`, `data.test_mask`

```python
import torch.nn.functional as F
from torch_geometric.nn import GCNConv, GATConv, SAGEConv

class GCN(torch.nn.Module):
    def __init__(self, in_channels, hidden_channels, out_channels):
        super().__init__()
        self.conv1 = GCNConv(in_channels, hidden_channels)
        self.conv2 = GCNConv(hidden_channels, out_channels)

    def forward(self, x, edge_index):
        x = self.conv1(x, edge_index).relu()
        x = F.dropout(x, p=0.5, training=self.training)
        x = self.conv2(x, edge_index)
        return x

# Similarly for GAT (GATConv) and GraphSAGE (SAGEConv)
# Train each for ~200 epochs with Adam optimizer, lr=0.01
```

### 2.2 Collect predictions and metadata

For each trained model, record per-node:

```python
model.eval()
with torch.no_grad():
    logits = model(data.x, data.edge_index)
    probs = F.softmax(logits, dim=1)
    predicted = logits.argmax(dim=1)
    confidence = probs.max(dim=1).values  # confidence in predicted class

for node in G.nodes():
    G.nodes[node]['gcn_predicted'] = predicted[node].item()
    G.nodes[node]['gcn_confidence'] = confidence[node].item()
    G.nodes[node]['gcn_correct'] = int(predicted[node].item() == data.y[node].item())
    # Repeat for GAT, GraphSAGE with different prefixes
```

### 2.3 Compute training-set distance variables

Following [[ma-2021-subgroup-fairness|Ma et al.]]'s key finding:

```python
# Geodesic distance to nearest training node
train_nodes = data.train_mask.nonzero(as_tuple=True)[0].tolist()
geodesic_to_train = {}
for node in G.nodes():
    if node in train_nodes:
        geodesic_to_train[node] = 0
    else:
        min_dist = float('inf')
        for t in train_nodes:
            try:
                d = nx.shortest_path_length(G, source=node, target=t)
                min_dist = min(min_dist, d)
            except nx.NetworkXNoPath:
                pass
        geodesic_to_train[node] = min_dist

# NOTE: For efficiency on larger graphs, use BFS from all train nodes simultaneously:
# distances = nx.multi_source_dijkstra_path_length(G, train_nodes)
# or BFS-based shortest path from all train nodes at once.

for node in G.nodes():
    G.nodes[node]['geodesic_to_train'] = geodesic_to_train[node]

# Aggregated-feature distance to training set (Ma et al.'s theoretical variable)
# Z = (D+I)^{-1}(A+I)(D+I)^{-1}(A+I)X  (two-step aggregation)
A = nx.adjacency_matrix(G).toarray()
D = np.diag(A.sum(axis=1))
I = np.eye(A.shape[0])
D_inv = np.linalg.inv(D + I)
Z = D_inv @ (A + I) @ D_inv @ (A + I) @ data.x.numpy()

train_Z = Z[train_nodes]
for node in G.nodes():
    dists = np.linalg.norm(train_Z - Z[node], axis=1)
    G.nodes[node]['agg_feature_dist_to_train'] = dists.min()
```

### 2.4 Compute local homophily ratio

```python
for node in G.nodes():
    neighbors = list(G.neighbors(node))
    if len(neighbors) == 0:
        G.nodes[node]['local_homophily'] = 0.0
    else:
        same_label = sum(1 for n in neighbors if G.nodes[n]['true_label'] == G.nodes[node]['true_label'])
        G.nodes[node]['local_homophily'] = same_label / len(neighbors)
```

---

## Phase 3: Convert to Hiveplotlib and Build Visualizations

### 3.1 Convert to hiveplotlib structures

```python
from hiveplotlib.converters import networkx_to_nodes_edges

nodes, edges = networkx_to_nodes_edges(graph=G)
# nodes.data is a DataFrame with all the node attributes we added
# edges.data has columns: from, to (+ any edge attributes)
```

### 3.2 Add edge-level metadata (the novel part)

This is where edge-level heterogeneity analysis begins. For each edge, compute:

```python
edge_df = edges.data.copy()

# For each model, tag edges by correctness of endpoint nodes
for model_name in ['gcn', 'gat', 'sage']:
    correct_col = f'{model_name}_correct'
    pred_col = f'{model_name}_predicted'

    # Edge correctness categories:
    # both_correct: both endpoints classified correctly
    # one_wrong: exactly one endpoint misclassified
    # both_wrong: both endpoints misclassified
    from_correct = nodes.data.set_index('unique_id').loc[edge_df['from'], correct_col].values
    to_correct = nodes.data.set_index('unique_id').loc[edge_df['to'], correct_col].values

    edge_df[f'{model_name}_both_correct'] = (from_correct == 1) & (to_correct == 1)
    edge_df[f'{model_name}_one_wrong'] = (from_correct != to_correct)
    edge_df[f'{model_name}_both_wrong'] = (from_correct == 0) & (to_correct == 0)

    # Edge error score: 0 = both correct, 1 = one wrong, 2 = both wrong
    edge_df[f'{model_name}_error_score'] = 2 - (from_correct.astype(int) + to_correct.astype(int))

    # Cross-group edges: do endpoints have different predicted labels?
    from_pred = nodes.data.set_index('unique_id').loc[edge_df['from'], pred_col].values
    to_pred = nodes.data.set_index('unique_id').loc[edge_df['to'], pred_col].values
    edge_df[f'{model_name}_cross_predicted'] = (from_pred != to_pred)

    # Degree disparity across edge
    from_deg = nodes.data.set_index('unique_id').loc[edge_df['from'], 'degree'].values
    to_deg = nodes.data.set_index('unique_id').loc[edge_df['to'], 'degree'].values
    edge_df[f'{model_name}_degree_ratio'] = np.maximum(from_deg, to_deg) / np.maximum(np.minimum(from_deg, to_deg), 1)

edges.data = edge_df
```

### 3.3 Create partition variables for the sweep

```python
# Bin continuous variables for axis assignment
nodes.create_partition_variable('degree', cutoffs=[3, 8], labels=['low', 'medium', 'high'],
                                partition_variable_name='degree_bin')
nodes.create_partition_variable('geodesic_to_train', cutoffs=[1, 2, 3],
                                labels=['adjacent', 'near', 'moderate', 'far'],
                                partition_variable_name='train_distance_bin')
nodes.create_partition_variable('local_homophily', cutoffs=[0.33, 0.67],
                                labels=['heterophilous', 'mixed', 'homophilous'],
                                partition_variable_name='homophily_bin')
nodes.create_partition_variable('betweenness_centrality', cutoffs=3,
                                partition_variable_name='betweenness_bin')
nodes.create_partition_variable('agg_feature_dist_to_train', cutoffs=3,
                                partition_variable_name='agg_dist_bin')
# Community is already categorical
```

### 3.4 Build initial single hive plots (sanity check)

Start with a single hive plot to verify the pipeline works:

```python
from hiveplotlib import HivePlot

hp = HivePlot(
    nodes=nodes,
    edges=edges,
    partition_variable='degree_bin',
    sorting_variables='gcn_confidence',
    repeat_axes=True,
    backend='matplotlib',
)

fig, ax = hp.plot(
    node_kwargs={'c': 'gcn_correct', 'cmap': 'RdYlGn', 's': 20, 'vmin': 0, 'vmax': 1},
    all_edge_kwargs={'alpha': 0.1, 'linewidth': 0.3},
)
ax.set_title("GCN on Cora: partitioned by degree, sorted by confidence")
fig.savefig("cora_gcn_degree_confidence.png", dpi=150, bbox_inches='tight')
```

### 3.5 Build HivePlotMatrix: partition sweep (key visualization)

Sweep across different structural decompositions to find which reveals the most heterogeneity:

```python
from hiveplotlib import HivePlotMatrix

# Sweep partition variables (rows) x sorting variables (columns)
hpm = HivePlotMatrix.from_variable_sweep(
    nodes=nodes,
    edges=edges,
    partition_variables_list=['degree_bin', 'community', 'homophily_bin',
                              'train_distance_bin', 'agg_dist_bin'],
    sorting_variables='gcn_confidence',
    repeat_axes=True,
    backend='matplotlib',
    all_edge_kwargs={'alpha': 0.05, 'linewidth': 0.2},
    node_kwargs={'c': 'gcn_correct', 'cmap': 'RdYlGn', 's': 10, 'vmin': 0, 'vmax': 1},
)

fig, axes = hpm.plot()
fig.suptitle("GCN on Cora: Which structural decomposition reveals the most misclassification heterogeneity?")
fig.savefig("cora_gcn_partition_sweep.png", dpi=200, bbox_inches='tight')
```

### 3.6 Build HivePlotMatrix: model comparison

Compare architectures using the same partitioning:

```python
# For this, build separate HivePlots for each model and arrange manually,
# OR run the sweep for each model and compare side-by-side.
# The from_variable_sweep can sweep sorting_variables across models' confidence scores:

hpm_models = HivePlotMatrix.from_variable_sweep(
    nodes=nodes,
    edges=edges,
    partition_variable='degree_bin',
    sorting_variables_list=['gcn_confidence', 'gat_confidence', 'sage_confidence'],
    repeat_axes=True,
    backend='matplotlib',
    all_edge_kwargs={'alpha': 0.05, 'linewidth': 0.2},
    node_kwargs={'c': 'gcn_correct', 'cmap': 'RdYlGn', 's': 10, 'vmin': 0, 'vmax': 1},
)
# NOTE: node color should match the model being shown in each column.
# May need to build each column separately and compose manually if
# the node color column differs per cell.
```

**Important:** If each column needs a different node color column (e.g., `gcn_correct` vs. `gat_correct`), you may need to build individual HivePlots and arrange them in a matplotlib grid manually rather than using `from_variable_sweep`. Check the hpm_from_variable_sweep.ipynb notebook for examples of how to handle per-cell customization.

### 3.7 Edge-focused visualization

Build hive plots specifically highlighting edge-level patterns:

```python
from hiveplotlib.edges import Edges

# Split edges into tagged groups by error pattern
correct_mask = edges.data['gcn_error_score'] == 0
one_wrong_mask = edges.data['gcn_error_score'] == 1
both_wrong_mask = edges.data['gcn_error_score'] == 2

tagged_edges = Edges(data={
    'both_correct': edges.data[correct_mask],
    'one_wrong': edges.data[one_wrong_mask],
    'both_wrong': edges.data[both_wrong_mask],
})

hp_edges = HivePlot(
    nodes=nodes,
    edges=tagged_edges,
    partition_variable='degree_bin',
    sorting_variables='gcn_confidence',
    repeat_axes=True,
    backend='matplotlib',
)

# Style each edge group differently
fig, ax = hp_edges.plot(
    node_kwargs={'c': 'gcn_correct', 'cmap': 'RdYlGn', 's': 15, 'vmin': 0, 'vmax': 1},
    # Edge styling per tag:
    repeat_edge_kwargs={'alpha': 0.02, 'linewidth': 0.1, 'color': 'lightgray'},
    non_repeat_edge_kwargs={'alpha': 0.02, 'linewidth': 0.1, 'color': 'lightgray'},
)
# Then overlay the error edges more prominently — check hiveplotlib API for
# per-tag edge styling, or plot in layers

ax.set_title("GCN on Cora: edge error patterns by degree")
```

---

## Phase 4: Analysis and Interpretation

### 4.1 Questions to answer from the visualizations

**Node-level (extending Ma et al.):**
- Which partition variable concentrates misclassified nodes most? (The cell with the most uneven red/green distribution wins.)
- Does training-set distance dominate degree as a predictor, or do they reveal different failure modes?
- Are there non-linear patterns in centrality that Ma et al.'s 5-bin bar charts missed?

**Edge-level (novel contribution):**
- Do "both wrong" edges cluster in specific cells? (If misclassified nodes connect to each other, errors may propagate through message-passing.)
- Are "one wrong" edges concentrated between specific structural groups? (e.g., many one-wrong edges between low-degree and high-degree axes would suggest errors at structural boundaries.)
- Do cross-community edges carry more misclassification than intra-community edges?
- Is there a degree-ratio signature for misclassification edges?

### 4.2 Quantitative summary statistics

Alongside the hive plots, compute:

```python
# Per-cell misclassification rate
# (to confirm what the visualization shows numerically)
for partition_var in ['degree_bin', 'community', 'homophily_bin', 'train_distance_bin']:
    print(f"\n--- {partition_var} ---")
    for group, group_df in nodes.data.groupby(partition_var):
        acc = group_df['gcn_correct'].mean()
        n = len(group_df)
        print(f"  {group}: {acc:.3f} accuracy ({n} nodes)")

# Edge-level: misclassification rate by edge type
for partition_var in ['degree_bin']:
    from_groups = nodes.data.set_index('unique_id').loc[edges.data['from'], partition_var].values
    to_groups = nodes.data.set_index('unique_id').loc[edges.data['to'], partition_var].values
    edge_groups = pd.DataFrame({
        'from_group': from_groups,
        'to_group': to_groups,
        'error_score': edges.data['gcn_error_score'].values,
    })
    cross_tab = edge_groups.groupby(['from_group', 'to_group'])['error_score'].mean()
    print(f"\nMean edge error score by {partition_var} groups:")
    print(cross_tab.unstack().round(3))
```

---

## Phase 5: Iterate and Scale

### 5.1 Refine based on initial results

- If degree reveals interesting patterns, try finer bins (5 instead of 3)
- If training-set distance dominates, try combining it with degree in a 2D sweep
- If edge patterns are striking, build dedicated edge-focused hive plots with more prominent error edge styling

### 5.2 Scale to larger datasets

If Cora results are promising:

```python
# CiteSeer and PubMed are also available via Planetoid
dataset = Planetoid(root='/tmp/CiteSeer', name='CiteSeer')
dataset = Planetoid(root='/tmp/PubMed', name='PubMed')

# For OGB-arxiv (169K nodes), switch to datashader backend:
from ogb.nodeproppred import PygNodePropPredDataset
dataset = PygNodePropPredDataset(name='ogbn-arxiv')

hpm = HivePlotMatrix.from_variable_sweep(
    ...,
    backend='datashader',  # Required for 169K nodes
)
fig, axes, im_nodes, im_edges = hpm.plot(dpi=150, pixel_spread_nodes=3)
```

### 5.3 Write up findings

If results are interesting enough for a paper or blog post, the key artifacts are:
1. The HivePlotMatrix partition sweep (which structural decomposition is most informative?)
2. The model comparison matrix (do GCN/GAT/GraphSAGE fail differently?)
3. The edge-level analysis (do misclassification errors cluster in edge space?)
4. Quantitative tables backing up the visual patterns

---

## Dependencies Summary

```
torch
torch-geometric
networkx
pandas
numpy
matplotlib
python-louvain  # for community detection
hiveplotlib     # from /home/garyk/repos/hiveplotlib or pip
```

## See Also

- [[gnn-heterogeneity-hive-plots]] — The research proposal this plan implements
- [[ma-2021-subgroup-fairness]] — The theoretical foundation (training-set distance as key variable)
- [[hiveplotlib]] — The visualization library
- [[hive-plot-matrix]] — The core visualization construct
