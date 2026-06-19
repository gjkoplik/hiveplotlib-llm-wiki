---
title: Knowledge Graph
type: concept
created: 2026-06-19
updated: 2026-06-19
sources: []
tags: [knowledge-graph, heterogeneous-network, network-visualization, rdf, property-graph]
---

# Knowledge Graph

A knowledge graph (KG) is a graph whose nodes are typed entities (Person, Gene, Company, Concept, Event, ...) and whose edges are typed, directed relationships between them (`worksFor`, `bindsTo`, `locatedIn`, `causes`, ...). Unlike the homogeneous networks hive plots were first designed for, a KG is **heterogeneous**: many node types and many edge types coexist in one graph, and the edge type carries as much meaning as the nodes it connects.

This page defines the data model and the properties that make KGs distinctive for visualization. For how hive plots can render them, see [[hive-plots-for-knowledge-graphs]].

## Two data models

Two formalisms dominate, and both are heterogeneous typed directed multigraphs.

**RDF / triples (semantic web).** Data is a set of `(subject, predicate, object)` triples. The predicate is the edge label; the graph is directed and edge-labeled. RDF is built for shared ontologies, federation, and machine reasoning. Visually, each triple is a node-arc-node edge.

**Property graphs (Neo4j and similar).** Nodes carry one or more *labels* (their types) and arbitrary key/value *properties*; each relationship has a single *type* and can also carry properties. Property graphs are built for application storage and analytics, queried with Cypher/GQL. The relationship-can-hold-properties feature is the main modeling convenience over plain RDF.

The distinction matters for ingestion, not for layout. Both reduce to "typed nodes, typed directed edges, attributes on both."

## Properties that shape visualization

- **Heterogeneity.** Tens to hundreds of entity types; tens to thousands of relationship types. This is the property that breaks naive layouts.
- **Typed, first-class edges.** The predicate is the point. A view that hides relationship type discards most of the information.
- **Directedness.** `(A treats B)` differs from `(B treats A)`.
- **Multigraph.** Two entities can be linked by several different predicates at once.
- **Schema vs instances.** The schema / ontology (the TBox: which types may relate to which, via which predicates) is small. The instance data (the ABox: the actual entities and edges) is large. The two live at very different scales and are worth visualizing separately.
- **Scale.** Real KGs run to millions of edges. Hetionet, a well-known biomedical KG, has roughly 47K nodes across 11 types and 2.2M edges across 24 relationship types.

## Why KGs resist standard graph drawing

Drawing the whole KG as one node-link diagram produces a hairball: extreme edge crossing, no readable structure, and relationship types collapsed into indistinguishable lines. The KG-visualization literature consistently reports that whole-graph node-link views fail at scale and that the views people actually use are scoped (a query result, a neighborhood, a chosen slice). This is the same failure mode hive plots were invented to fix for homogeneous networks (see [[force-directed-layout]]), which is what makes the KG case worth thinking about.

## Heterogeneous Information Networks

The formal vocabulary for "KG as a typed graph" is the **Heterogeneous Information Network (HIN)**: a node set and edge set plus type-mapping functions, a **network schema** (the allowed node-type / edge-type structure), and the [[metapath]] (a sequence of node types joined by relation types) as the unit of meaningful traversal. The metapath turns out to be the natural bridge to hive plots, because a metapath is a short, path-shaped slice through an otherwise dense schema.

## See Also

- [[hive-plots-for-knowledge-graphs]] — the synthesis: how (and how well) hive plots render KGs, and what hiveplotlib would need
- [[metapath]] — the slice abstraction that maps a KG onto a small set of hive-plot axes
- [[hive-plot]] — the visualization method
- [[node-assignment]] — entity type is the most natural partition variable
- [[edge-rendering]] — edge tags as the existing hook for typed relationships
- [[applications-bioinformatics]] — where typed biomedical graphs already drive hive-plot use
