---
title: "GNN Hive Plots: Broadened Research Directions"
type: analysis
created: 2026-07-06
updated: 2026-07-06
sources: [kipf-2017-gcn, ma-2021-subgroup-fairness, hsu-2022-gnn-calibration, jin-2022-gnnlens, huang-2021-correct-and-smooth]
tags: [gnn-evaluation, hive-plot-matrix, research-directions, machine-learning, over-smoothing, link-prediction, training-dynamics]
---

# GNN Hive Plots: Broadened Research Directions

The first GNN exploration boxed itself into the most-studied corner of the field, and the honest [[gnn-heterogeneity-findings|findings scorecard]] shows the cost: a refuter or an owner for every candidate discovery. This page reopens the scope. It is a menu of directions chosen for one goal, a **strong visual artifact**, and it is honest about the tradeoff up front.

> **Register (read first).** These directions are ranked for *artifact strength* and for *speeding a viewer to a known conclusion*, not for research novelty. The maintainer's own framing: they are "more likely to give us something visually appealing, but probably nothing research-profound." That is a legitimate win for a visualization library, and it is a different bar than the findings page held. Each direction still carries a **prior art to check** line so a downstream run applies the same counterfactual discipline the findings page did, rather than re-claiming a discovery.

## Why the old scope was narrow

The first pass changed none of the field's defaults. It stayed on **node classification**, framed as **error diagnosis**, on **homophilous citation graphs**, with **one static encoding** (structural axis, confidence sort, correctness color). That is the single most-saturated cell in GNN research, which is why the prior-art wall was immediate. There is also a structural mismatch: a hive plot's first-class object is the *edge*, but node classification treats edges as a derived attribute (color by endpoint correctness). The work was fighting the medium.

Broadening means changing at least one of four levers:

1. **The task.** What the GNN predicts (node label vs. edge vs. graph vs. next snapshot).
2. **What you plot.** Outcomes (correct/wrong) vs. the model's internal *mechanism* (embeddings, message flow).
3. **The datasets.** Beyond the three citation graphs.
4. **The encoding grammar.** Static overlay vs. [[differential-hive-plot|differential]] vs. edge-native vs. per-frame movie.

## Directions, ranked

| # | Direction | Lever | Artifact bet | Lift |
|---|---|---|---|---|
| 1 | [[gnn-over-smoothing|Over-smoothing]] as a per-layer matrix | what you plot (mechanism) | watch class separation collapse with depth | low |
| 2 | GNN training dynamics as a per-epoch movie | what you plot + grammar | watch classes split out as the model learns | low–med |
| 3 | Link prediction, edge as the prediction | task + grammar | stop fighting the medium; hallucinated vs missed links | med |
| 4 | Differential hive plot (architecture / seed / layer) | grammar | shared failure cancels, only the delta shows | low (data exists) |
| 5 | KG / heterogeneous link prediction (Hetionet) | task + data | typed, edge-native, "one plot per question" | high |
| 6 | Mechanism: over-squashing, influence, embedding-vs-structure | what you plot | unclaimed visual territory | high |

### 1. Over-smoothing as a per-layer hive plot matrix (top pick)

Over-smoothing is a marquee, well-understood GNN phenomenon: as depth grows, node representations converge toward a common vector and class separation dissolves (Li, Han & Wu 2018; Oono & Suzuki 2020). It is communicated today with scalar curves (Dirichlet energy; the MAD metric of Chen et al. 2020) or a t-SNE blob, neither of which shows the *shape* of the collapse. A [[hive-plot-matrix|HivePlotMatrix]] with one cell per layer, axes = classes (or communities), within-axis position = a 1-D projection of that layer's embedding, edges = adjacency, renders the collapse literally: well-separated axis distributions at layer 1 tighten into a hub tangle by layer 8. This is the cleanest "known concept, shown so the conclusion lands instantly" candidate. The pipeline already records what it needs (cache per-layer embeddings). See [[gnn-over-smoothing]] for the mechanism and the build recipe.

- **Prior art to check:** embedding-visualization tools ([[jin-2022-gnnlens|GNNLens]], the [[nn-training-dynamics-p2cp-exploration|nn-viz]] work, t-SNE-over-depth figures in over-smoothing papers). The phenomenon is textbook; the claim is a cleaner rendering, and even that must be positioned against those.
- **Data:** any node-classification graph; Cora is fine for the prototype.

### 2. GNN training dynamics as a per-epoch movie

The [[nn-training-dynamics-p2cp-exploration|nn-viz exploration]] watches a tiny MLP learn MNIST and Fashion-MNIST as a hive-plot/P2CP movie over training. On a GNN this is *less contrived*, because there is a real graph to watch: as training proceeds, the classes visibly split out across the actual network structure, not a contrived feature space. Encode axes by class or community, sort within-axis by prediction confidence or embedding coordinate, and play one frame per checkpoint. The [[differential-hive-plot|differential]] grammar sharpens it (frame-to-frame diff = what just changed). This also subsumes the **learning-order axis** idea from `DIRECTIONS.md`: record the first epoch each node becomes permanently correct and use it as a sort variable, then check whether late-learned regions coincide with the residual-screen pockets.

- **Prior art to check:** the nn-viz page's own prior-art map (softmax-P2CP as a Grand Tour complement; the encoding is prior art). GNN-specific training-dynamics viz is thinner than the depth story, so scope the search to GNN-training animations specifically.
- **Data:** any node-classification graph with cheap per-epoch checkpoints.
- **Already exists:** the `training-gnn-hiveplot` repo is this direction, with a working first pass. It runs the full node-task pipeline (prepare data, train, enrich per checkpoint, render frames, stitch movie) and renders per-epoch correct/incorrect `HivePlotMatrix` panels with a shared training-curve and confusion-matrix. This is polish-and-extract, not build-from-scratch.

### 3. Link prediction, so the edge becomes the prediction

This fixes the medium-mismatch head-on. Instead of coloring edges by a node outcome, plot the model's edge decisions: true positive, false positive (hallucinated), false negative (missed), as two or three flat tags over the true-edge context, nodes sorted by a structural property. "What kind of links does this model invent, and what kind does it miss" is genuinely edge-native, which node classification never was. Multi-tag `Edges` coloring already supports the encoding.

- **Prior art to check:** link-prediction evaluation is mature (Hits@K, MRR); the contribution is the *structural decomposition of the error edges*, not the metric. Confirm nobody already lays predicted-link errors on a rational layout.
- **Data:** the citation graphs for a prototype, then OGB link-prediction sets (`ogbl-collab`, `ogbl-ddi`, `ogbl-citation2`) for standardized splits.
- **Already exists:** substantially built in `training-gnn-hiveplot`'s link task, which renders a TP/FN/FP/TN hive grid (fixed node positions, only the drawn edge subset changes per cell), animated over training via an inner-product decoder on shared conv encoders. The remaining work is polishing it and pulling out the static hero frame (final-epoch FP-vs-FN over the true-edge context).

### 4. The differential hive plot, given its ML incarnation

`DIRECTIONS.md` already flags the architecture diff as the most promising unbuilt build, and [[differential-hive-plot]] is documented but unimplemented. This is the reusable grammar under directions 1, 2, and 6: tag edges by GCN-fails-but-GraphSAGE-does-not vs. the reverse (shared failure cancels, only architecture-specific blind spots survive in color); or seed-vs-seed stability (consistently wrong = structural, unstably wrong = training noise); or clean-vs-perturbed for robustness. The cancellation *is* the visual work: the plot is mostly empty except where the models genuinely disagree. Notebook 02 reports errors are highly correlated across architectures (kappa ~0.6–0.7), so the disagreement set is small and plottable without overplotting.

- **Prior art to check:** [[krzywinski-2017-differential|Krzywinski 2017]] is the visual grammar; the ML application is the delta. [[jin-2022-gnnlens|GNNLens]] compares models but not on a rational layout.
- **Data:** already in `outputs/nodes.csv` (three models) and `outputs/seeds/` (five seeds). Nearly free.

### 5. KG / heterogeneous link prediction (Hetionet)

This unifies two live wiki threads that currently sit apart: [[hive-plots-for-knowledge-graphs|hive plots for knowledge graphs]] and the GNN work. KG completion is edge-native *and* typed: a [[metapath]]-scoped hive plot (e.g. Compound–treats–Disease) showing which predicted drug–disease links a GNN gets, misses, and hallucinates. Partial groundwork already exists: the [[knowledge-graph-hetionet-spike|Hetionet spike]] shipped a vendored CbGaD slice (1,272 nodes, 5,942 edges, CC0) and a reader on a hiveplotlib worktree branch. This is the highest-lift direction (heterogeneous data, KG semantics, GNN link-prediction training) but the one where "one hive plot per question" and "the edge is the prediction" reinforce each other.

- **Prior art to check:** KG-completion viz and drug-repurposing dashboards; position against them, not as an empty space.
- **Data:** the vendored Hetionet CbGaD slice (already local); Himmelstein et al. 2017, *eLife*, CC0.
- **Already exists:** the `kg-hetionet-spike` worktree carries the `hetionet_cbgad_data()` reader, the vendored CbGaD slice, and `examples/hetionet_kg_views.ipynb` (metapath axes on `kind`, edge filtering to a metapath's relations, per-relation edge tags). The only new build is the GNN link-prediction layer: predict held-out `associates` (Gene–Disease) or `binds` (Compound–Gene) edges and tag them TP / FN / FP into that existing layout. Note the slice has no direct Compound–treats–Disease edges, so the drug-repurposing framing needs a re-subset via the runner, later.
- **Outcome (2026-07-06):** built in `hiveplotlib-hetionet`. A 2-layer GraphSAGE/GCN link predictor (learnable embeddings, inner-product decoder) on held-out `associates` edges reaches test AUC around 0.83, and the TP / FN / FP outcomes render as small multiples on the metapath layout. Honest read: `associates` is bipartite (Disease-Gene), so the third (Compound) axis is inert and the hive plot is under-motivated for this task; the compelling three-axis version is the CtD drug-repurposing prediction, which needs a re-subset. Errors concentrate on high-degree hub genes, which the degree-sorted axis surfaces positionally (ties to [[subramonian-2024-degree-bias]]). The reusable takeaways are package-fit and are logged in [[hive-plots-for-knowledge-graphs]] (gap #7: edge styling by column and fixed-layout edge layers; released-API portability; per-edge directedness). The prior-art check reconfirmed [[jin-2022-gnnlens|GNNLens]] and drug-repurposing dashboards; the claim is a positional error view, not a method.

### 6. Mechanism instead of outcome (higher risk, higher reward)

Three named, currently-metric-only phenomena besides over-smoothing:

- **Over-squashing** (Alon & Yahav 2021; Topping et al. 2022 tie it to negative Ricci curvature on bottleneck edges). A curvature-sorted axis could show *where* the graph strangles information flow.
- **Influence / receptive field** (which training nodes actually move a test node's prediction; Xu et al. 2018 Jacobian-influence). A training-set axis with influence-weighted edges shows under-reaching directly.
- **Embedding-vs-structure mismatch** (`DIRECTIONS.md` Direction 4): partition one matrix row by Louvain community, another by k-means of the GNN's hidden embeddings; cells where they disagree are where the model's internal geometry has detached from the graph.

These are less turnkey and need a novelty check, but they are unclaimed visual territory.

## Datasets, directly

The current set is dated for the questions being asked.

- **Replace the heterophily benchmarks.** The Chameleon/Squirrel degree-inversion note in `DIRECTIONS.md` rests on datasets that Platonov et al. 2023 ("A critical look at the evaluation of GNNs under heterophily", ICLR) showed have train/test leakage from duplicate nodes; results flip when the leakage is fixed. Any claim on them is fragile. Their replacements (roman-empire, amazon-ratings, minesweeper, tolokers, questions) are the modern standard and should be adopted before leaning on a heterophily story.
- **Add scale and standardized splits via OGB.** `ogbn-arxiv` / `ogbn-products` for node tasks stress the datashader backend and give leaderboard context; the `ogbl-*` sets feed direction 3.
- **Temporal.** The Temporal Graph Benchmark (`tgbl-*`) or classic dynamic sets (Bitcoin-OTC, UC-Irvine messages) feed the differential grammar and a snapshot-to-snapshot movie.
- **Cross-domain.** Hetionet (direction 5) to leave citation graphs entirely; molecular graph classification (ZINC, `ogbg-molhiv`) is a weak hive fit (each graph is tiny) and is not recommended as a lead.

## Dispatch decomposition

These are independent work items and should be tasked separately rather than as one bundle. Independence is good practice on its own (each has its own data, model, and figure), and it keeps a stall on one from blocking the others.

**Existing homes (most directions already have a repo).** The scope is less green-field than it looks:

| Direction | Repo | State |
|---|---|---|
| 2 training movie, 3 link prediction | `training-gnn-hiveplot` | working first pass; polish + extract |
| 4 differential, 6 mechanism | `gnn-hiveplots` | post-processing / extension of existing outputs |
| 1 over-smoothing | `training-gnn-hiveplot` (reuse frame/movie infra) or a small new repo | new build (needs deep models + per-layer embeddings) |
| 5 Hetionet | **`hiveplotlib-hetionet`** (new standalone repo); dual-homed with `hiveplotlib` | loader, CbGaD slice, and metapath views already built on the `kg-hetionet-spike` worktree; only the GNN link-prediction layer is new |

Only direction 5 clearly warrants a new repo; direction 1 is the one genuinely new build and can reuse `training-gnn-hiveplot`'s rendering. Everything else extends a repo that already exists.

1. **Pure-ML cluster (lowest friction):** over-smoothing (1), training dynamics (2), link prediction (3), differential (4). All run on public ML benchmarks (Cora, OGB) with no domain-sensitive content. These are the natural first tasks.
2. **KG / biomedical cluster (task separately):** Hetionet link prediction (5). It is the most distinct technically (heterogeneous graph, different loader, KG semantics) and has its own local groundwork, so it earns its own brief regardless. Keeping it separate also means a benign-but-flaggable biomedical dataset does not gate the pure-ML work. When tasking it, state the actual purpose plainly: network *visualization* of a public, CC0, peer-reviewed knowledge graph (Himmelstein et al. 2017), no wet-lab or protocol content, purely a rendering/exploration exercise.
3. **Research-track cluster (highest uncertainty):** the mechanism directions (6). Task only after a novelty check, since these are the ones with a real chance of either an unclaimed result or a quick refutation.

## Honest framing for any writeup

- Lead with the *artifact* and the *exploration-acceleration* claim, not a discovery. The findings page already spent the novelty budget on this problem.
- Every one of these has textbook antecedents for the underlying phenomenon (over-smoothing, over-squashing, calibration, error correlation). The contribution is the layout, and even the layout must be positioned against [[jin-2022-gnnlens|GNNLens]] and embedding-viz tools.
- The statistics do the finding; the hive plot contributes positional layout. That ceiling from the findings page still holds.

## See Also

- [[gnn-heterogeneity-findings]] — the honest scorecard that motivated reopening the scope
- [[gnn-heterogeneity-hive-plots]] — the original (superseded) proposal
- [[gnn-over-smoothing]] — the flagship direction's mechanism and build recipe
- [[nn-training-dynamics-p2cp-exploration]] — the training-movie analog on an MLP
- [[differential-hive-plot]] — the comparison grammar under several directions
- [[hive-plots-for-knowledge-graphs]] / [[knowledge-graph-hetionet-spike]] — the KG thread direction 5 joins
- [[gnn-evaluation]] — the evaluation gap, with its honest ceiling
- [[hive-plot-matrix]] — the construction mode for the per-layer and per-epoch sweeps
- [[examples-and-applications]] — the catalog this direction set feeds
