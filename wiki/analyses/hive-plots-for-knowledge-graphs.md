---
title: Hive Plots for Knowledge Graphs
type: analysis
created: 2026-06-19
updated: 2026-06-25
sources: [krzywinski-2012, perez-2021-hype, nollenburg-2023-computing-hive-plots]
tags: [knowledge-graph, hive-plot, heterogeneous-network, metapath, hiveplotlib]
---

# Hive Plots for Knowledge Graphs

**Status:** open exploration (June 2026). Question: can hive plots, and [[hiveplotlib]] specifically, visualize [[knowledge-graph|knowledge graphs]], and if not, what is missing?

**Short answer.** A knowledge graph is too large and too heterogeneous to be one hive plot. But a hive plot is an excellent renderer for a *scoped, typed slice* of a KG, and [[hiveplotlib]] already has most of the primitives (arbitrary node attributes, typed edge tags, repeat axes, per-tag panels, directed edges). The gaps are ergonomic (ingestion, metapath and schema helpers), not fundamental.

## The core tension

A [[hive-plot]] is built for a small number of axes (two or three) holding meaningfully partitioned nodes, with edges between neighboring axes. A knowledge graph is the opposite shape: tens to hundreds of entity types, tens to thousands of relationship types, dense type-to-type connectivity, directed and multigraph (see [[knowledge-graph]]).

The naive mapping "one entity type per axis" breaks immediately. There are too many types for two or three axes, and the type-to-type connectivity is not a clean cycle, so most edges would have to cross or route around axes. [[krzywinski-2012|Krzywinski]] is explicit that this is where hive plots stop being readable, and [[nollenburg-2023-computing-hive-plots|Nöllenburg & Wallinger]] show the underlying axis-assignment optimization is NP-complete. Forcing the whole KG onto axes recreates the hairball hive plots exist to avoid.

## The reframing: a hive plot is a view, not the graph

The resolution is to stop trying to draw the whole KG. The KG-visualization literature converges on the same point: whole-graph node-link views fail, and useful views are scoped to a query, a neighborhood, or a chosen slice.

So the right unit is **one hive plot per question**, where the question fixes two things:

1. **The axes** — a small set of entity types drawn from the schema (most naturally, a [[metapath]]).
2. **The edges** — which relationship types to draw.

Under this framing hive plots are not a whole-graph layout for KGs; they are a *rational rendering of a typed subgraph*, and rationality is exactly their strength when the partition is meaningful. Entity type is the most meaningful partition there is.

## A taxonomy of hive-plot views over a KG

Seven patterns, ordered roughly from instance-level to schema-level. For each: what it answers, how to build it in hiveplotlib today, and what is missing.

### 1. Instance metapath view (the workhorse)
Pick a [[metapath]] (e.g. `Gene-Disease-Compound`); one axis per node type in the path; instances on the axes; edges are the metapath's relations. Sort each axis by a meaningful attribute (gene by expression, disease by prevalence, compound by trial phase).
- **Today:** partition on the entity-type column; filter edges to the metapath's relations; set sorting variables per axis. All supported.
- **Missing:** a helper that takes `(graph, metapath spec)` and builds this directly. Today it is manual assembly.

### 2. Predicate facet / hive panel
Hold the axes fixed and split by relationship type: one small-multiple per predicate, or color edges by predicate on a single plot. Answers "how do different relationship types wire these entity types together?"
- **Today:** strong support. Store each predicate as an edge **tag**, then `HivePlotMatrix.from_tags()` produces one panel per predicate (see [[edge-rendering]], [[hive-plot-matrix]]). Single-plot color-by-predicate works via an edge-DataFrame column plus viz kwargs. This is the [[perez-2021-hype|Hive Panel Explorer]] idea applied to relationship type.
- **Missing:** a one-call "facet by this predicate column" that builds the tags for you; a shared legend across panels.

### 3. Role decomposition (subject / object / both)
Ignore entity type. Partition by directional role: pure subjects (sources), pure objects (sinks), and entities that are both (intermediates). This is [[krzywinski-2012|Krzywinski's]] regulators / managers / workhorses, and it works for *any* KG regardless of type count.
- **Today:** supported as a partition pattern (see [[node-assignment]]); the in/out-degree roles are computable from [[graph-features]]. The same entity acting as both subject and object is handled with repeat (cloned) axes.
- **Missing:** a convenience that computes the role partition from edge direction.

### 4. Schema / ontology (TBox) hive plot
Visualize the *schema itself*: node types as nodes, relationship types as edges, weighted by instance counts. The schema is small enough to lay out rationally, so this becomes an overview and a navigation aid (a "map of the map") for the instance views above.
- **Today:** doable by hand if you first build the type-level graph yourself.
- **Missing:** a helper to extract the schema graph from an instance graph.

### 5. Aggregated supergraph view
Roll instance edges up to type-to-type edges with weight = count (or another statistic). Turns a million-edge KG into a schema-sized weighted hive plot; bundle width or color encodes volume.
- **Today:** doable if you pre-aggregate into a node/edge table yourself.
- **Missing:** the aggregation step as a utility; pairs naturally with #4.

### 6. P2CP for the attribute (literal) facet
KG entities carry literal attributes (dates, scores, measurements: RDF datatype properties). A [[p2cp]] renders the *attribute* facet of one entity type as polar parallel coordinates, complementary to the relational facet a hive plot shows. Together they cover both object properties (relations) and datatype properties (attributes).
- **Today:** fully supported; P2CP already takes a tidy table of one entity type.
- **Missing:** nothing structural; this is a framing, not a feature gap.

### 7. Ego / neighborhood / query-result scoping
Scope to a Cypher/SPARQL result or an entity's k-hop neighborhood first, then apply patterns 1-3. This is the "visualize the query result, not the database" stance.
- **Today:** works if you hand hiveplotlib the already-scoped subgraph.
- **Missing:** ingestion convenience from query results (see gaps).

## Worked example: Hetionet

Hetionet (~47K nodes, 11 node types, 2.2M edges, 24 relationship types) is a natural fit because bioinformatics is already hiveplotlib's strongest domain ([[applications-bioinformatics]]).

- **Metapath view:** `Compound-binds-Gene-associates-Disease`. Three axes; sort Compounds by trial phase, Genes by degree, Diseases by prevalence. Surfaces which druggable genes bridge compounds to diseases.
- **Predicate facet:** fix the Gene and Disease axes; one panel per Gene-Disease predicate (`associates`, `upregulates`, `downregulates`) via `from_tags()`. Shows whether regulation direction concentrates on different genes.
- **Schema view:** all 11 types as a schema hive plot, edges weighted by the 2.2M instances, used as a legend / overview for the focused views.

## What hiveplotlib supports today

Grounded in the source (node.py, edges.py, hiveplot.py, converters.py):

| KG need | Status | Mechanism |
|---|---|---|
| Entity type as axis | Supported | Partition on any node-attribute column |
| Arbitrary node/edge attributes | Supported | pandas-backed `NodeCollection` / `Edges` |
| Typed (predicate) edges | Supported | Edge **tags** (one DataFrame per type) or a type column |
| Per-predicate panels | Supported | `HivePlotMatrix.from_tags()` |
| Color edges by predicate | Supported | edge-column value + viz kwargs |
| Directed edges | Supported | `from` / `to`, clockwise vs counterclockwise rendering |
| Multigraph (parallel edges) | Supported | multiple rows per `(from, to)` |
| Self-loops | Supported | not filtered out |
| Same entity as subject and object | Supported | repeat (cloned) axes |
| More than 3 axes | Allowed, not advised | no hard cap, but readability degrades (see tension) |
| NetworkX in / out | Supported | `converters.py` |

The headline: **hiveplotlib is more KG-capable than it looks**, mostly because edge tags already model typed relationships and repeat axes already model subject/object duality.

## Gaps and proposed additions

Ordered by leverage.

1. **Ingestion converters.** The biggest adoption lever. Today only NetworkX is wired up, and NetworkX MultiGraph edge *keys* are dropped on conversion, so a predicate stored as the multigraph key is lost (store it as an attribute to keep it). Proposed: readers for RDF (`rdflib` graphs) and property graphs (Neo4j / Cypher results) that map node labels to a type column and relationship types to tags. Without this, every KG user starts with bespoke wrangling.
2. **Metapath constructor.** Something like `from_metapath(graph, ["Compound", "binds", "Gene", "associates", "Disease"])` returning a ready HivePlot. Encapsulates pattern #1.
3. **Schema + aggregation helpers.** Extract the type-level schema graph (#4) and the aggregated supergraph (#5) from an instance graph. Two small utilities that unlock the overview views.
4. **Predicate-faceting ergonomics.** A one-call "facet by this edge-type column" that builds the tags, plus shared legends across the resulting panels (#2).
5. **Role-partition convenience.** Compute the source / sink / intermediate partition from edge direction (#3).
6. **Edge filtering convenience.** A documented, first-class way to subset edges by attribute for a view, rather than manual DataFrame slicing.

## Explicit non-goal

Drawing an arbitrary dense schema on many axes is not worth pursuing. The theory says it cannot stay readable ([[krzywinski-2012]], [[nollenburg-2023-computing-hive-plots]]), and chasing it would reintroduce the hairball. The value is in scoped, metapath-shaped views, not whole-graph layout.

## Open questions

- Is the metapath constructor better as a hiveplotlib feature or a thin recipe in the docs/gallery?
- Should RDF / property-graph ingestion live in hiveplotlib or a companion package, given the heavy optional dependencies (`rdflib`, Neo4j drivers)?
- For the aggregated supergraph, what edge statistic reads best as bundle weight at KG scale, and does the datashader backend carry it?
- Does a worked Hetionet notebook make the case better than any API addition?

## See Also

- [[knowledge-graph]] — the domain and its data models
- [[metapath]] — the slice abstraction that selects axes
- [[hive-plot]] — the method
- [[node-assignment]] — entity type and directional role as partitions
- [[edge-rendering]] — edge tags, the typed-relationship hook
- [[hive-plot-matrix]] — per-predicate panels via `from_tags()`
- [[p2cp]] — the attribute-facet complement
- [[perez-2021-hype]] — panels of hive plots as a precedent
- [[nollenburg-2023-computing-hive-plots]] — why dense multi-axis layout is hard
- [[applications-bioinformatics]] — the natural first domain (Hetionet)
- [[examples-and-applications]] — catalog of hiveplotlib examples and application explorations
