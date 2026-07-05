---
title: Examples & Applications Catalog
type: index
created: 2026-06-25
updated: 2026-07-04
tags: [examples, applications, hiveplotlib, exploration, moc]
---

# Examples & Applications Catalog

A living catalog of exploratory examples and applications of [[hiveplotlib]]: cases that
exercise the library on a real problem or dataset, whether or not anything shipped into the
library itself. The goal is to track this direction and grow it over time, so add a row whenever
a new exploration lands as a wiki page.

**What belongs here.** Worked examples and application explorations of hiveplotlib on a concrete
problem or dataset (a prototype, a walkthrough, a "can the tool do X" probe, a domain study).
These range from documented shipped example notebooks to throwaway prototypes that may or may not
graduate to a writeup.

**What does not.** Theory and method pages (those are [[index|concepts]]), library-internals or
API pages, source summaries, comparisons and competitive analysis (e.g.
[[nxviz-comparison]]), and JOSS-paper evidence (e.g. [[hiveplotlib-research-impact]],
[[joss-ai-disclosure-precedents]]). Pure implementation plans live in `plans/`, not here; an
exploration earns a row once it has a curated analysis or concept page.

## Catalog

| Exploration | Domain / data | Status | What it shows |
| --- | --- | --- | --- |
| [[nn-training-dynamics-p2cp-exploration\|Watching a neural network learn via P2CP]] | ML / neural networks (MNIST, tiny MLP) | Throwaway prototype; writeup undecided | A [[p2cp\|P2CP]] movie of a tiny MLP learning MNIST over training: per-class output-probability petals (Figure A) and a novel cross-layer co-activation polar triangle (Figure B). Reusable usage notes (`BaseHivePlot`, datashade spread-then-shade) and the output-vs-hidden-space finding. |
| [[soccer-passing-hive-plots\|Soccer passing networks as hive plots]] | Soccer / passing networks (StatsBomb open data) | Throwaway prototype; graduation candidates named | Fixes the passing-network layout by pitch third (the pass-endpoint model) so two teams read apples-to-apples, the documented fix for average-position hairballs. A nine-figure catalog (flagship comparison, formation lineups, shot build-up, [[expected-threat\|xT]]-weighted arcs, spatial [[flow-motifs]], turnovers, Barcelona motifs at scale). Reusable `statsbombpy` gotchas and the fixed-bounds axis-pinning lesson. |
| [[hive-plots-for-knowledge-graphs\|Hive plots for knowledge graphs]] | Knowledge graphs / heterogeneous networks (Hetionet) | Open exploration | Can hiveplotlib render a KG? Reframes the KG as scoped [[metapath]]-shaped slices ("one hive plot per question"), a seven-pattern taxonomy, a Hetionet worked example, what the library supports today, and the ergonomic gaps (RDF/Neo4j ingestion, metapath helpers). |
| [[gnn-heterogeneity-hive-plots\|GNN heterogeneity diagnosis via hive plot matrices]] | ML / graph neural networks (Cora) | Research proposal + Cora prototype results | [[hive-plot-matrix\|HivePlotMatrix]] as a diagnostic for GNN classification heterogeneity that aggregate metrics mask, at node and edge level. Carries an honest scorecard from the Cora prototype; the build recipe is in [[cora-prototype-plan]]. |
| [[hiveplotlib-bioinformatics-examples\|Bioinformatics hive plot examples]] | Bioinformatics (C. elegans connectome, E. coli RegulonDB GRN) | Public example repo; credible-but-not-hero, hero relocated | Real biological-network hive plots in the strongest adoption domain ([[applications-bioinformatics]]): a C. elegans connectome (sensory/interneuron/motor axes, two-panel [[hive-plot-matrix\|HivePlotMatrix]] sex comparison) and a signed E. coli GRN (role-partitioned, activation/repression edges). Honest finding: both are credible real-data demos, but neither is a "hero" figure (real networks control for nothing, so comparisons collapse to density); the instant-comparison hero is the engineered [[same-stats-different-graphs\|Datasaurus-for-networks]] work these corroborate. Reusable `unify_axes` shared-metric-positions, multi-tag edge coloring, `repeat_axes`, and edge-tag [[differential-hive-plot\|differential]] emulation. |
| [[karate-club-walkthrough\|Zachary's Karate Club walkthrough]] | Social networks (Zachary 1977) | Documented shipped example | Step-by-step build of a [[hive-plot]] from a classic social-network dataset (the shipped `karate_club.ipynb`): faction partition, degree sort, intra/inter-faction edge styling, and what the layout reveals. |

## See Also

- [[hiveplotlib]] — The library these examples exercise (main hub)
- [[p2cp]] — Polar Parallel Coordinates, the encoding the NN-viz exploration leans on
- [[hive-plot]] — The core method
- [[index|Wiki Index]]
