---
title: Metapath
type: concept
created: 2026-06-19
updated: 2026-06-19
sources: [krzywinski-2012]
tags: [knowledge-graph, heterogeneous-network, metapath, hive-plot]
---

# Metapath

A **metapath** is a sequence of node types connected by relation types, defined over a [[knowledge-graph]] schema. It names a composite relationship without committing to specific instances. Example, from an academic graph:

```
Author --writes--> Paper --publishedIn--> Venue
```

This metapath (often abbreviated `Author-Paper-Venue` or `APV`) stands for "an author who wrote a paper published in a venue," and matches every concrete chain of that shape in the data. In the biomedical KG Hetionet, the metapath `Compound--binds-->Gene--associates-->Disease` (CbGaD) encodes "a drug binds a protein whose gene is associated with a disease," one of the patterns used for drug-repurposing inference.

Metapaths come from the Heterogeneous Information Network literature, where they are the unit of meaningful traversal, sampling, and embedding in a typed graph. A heterogeneous graph has too many type-to-type connection patterns to reason about at once; a metapath picks one short, semantically coherent path through that thicket.

## Why metapaths matter for hive plots

This is the key bridge between knowledge graphs and hive plots.

A [[hive-plot]] wants a small number of axes (two or three) where each axis connects only to its neighbors. [[krzywinski-2012|Krzywinski's]] original constraint is explicit: more than three axes stay readable *only* when nodes can be ordered so that edges run between neighboring axes; otherwise axes must be duplicated or edges routed across the figure, and interpretability collapses.

A metapath satisfies that constraint by construction. Lay each node type in the metapath on its own axis, in order:

- A length-2 metapath (`A-r-B`) is a bipartite, two-axis hive plot.
- A length-3 metapath (`A-r1-B-r2-C`) is the canonical three-axis hive plot, and its connectivity is path-shaped, so every edge runs between neighboring axes with no crossings forced.
- A cyclic metapath (`A-B-C-A`) wraps cleanly around a three-axis plot.

In other words, the slices of a KG that hive plots render without compromise are exactly its metapaths. Choosing a metapath *is* choosing the axes.

## Relation to prior work

MetapathVis (2025) visualizes the *effect* of metapaths on learned embeddings, via projection and visual analytics, rather than the structural slice itself. Rendering the actual instance subgraph of a metapath as a rational, sorted layout is a different and complementary use, and the one hive plots are suited for.

## See Also

- [[knowledge-graph]] — the heterogeneous graphs metapaths are defined over
- [[hive-plots-for-knowledge-graphs]] — metapath views as one pattern in a larger taxonomy
- [[hive-plot]] — the layout a metapath maps onto
- [[node-assignment]] — node type as partition, one axis per metapath step
- [[krzywinski-2012]] — the "edges only between neighboring axes" constraint
