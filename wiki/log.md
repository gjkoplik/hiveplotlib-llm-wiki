---
title: Wiki Log
type: log
updated: 2026-04-06
---

# Hive Plot Research Wiki — Log

## [2026-04-06] setup | Wiki Initialized
Wiki structure created. Directories: `raw/`, `wiki/sources/`, `wiki/entities/`, `wiki/concepts/`, `wiki/analyses/`. Schema defined in `CLAUDE.md`. Index and log files created. Ready for first source ingest.

## [2026-04-06] ingest | hiveplotlib Python repo
Ingested the hiveplotlib Python repository as a foundational source. Created source summary (`sources/hiveplotlib-python.md`), entity page (`entities/hiveplotlib.md` — designated as main hub), and contributed to 7 concept pages. The library has 6 visualization backends, 46 example notebooks, 422 tests at 100% coverage, and key classes: BaseHivePlot, HivePlot, HivePlotMatrix, NodeCollection, Edges, Axis, P2CP.

## [2026-04-06] ingest | hiveplotlib-javascript repo
Ingested the hiveplotlib-javascript repository. Created source summary (`sources/hiveplotlib-javascript.md`) and entity page (`entities/hiveplotlib-javascript.md`). Single-file D3.js library (~650 lines) that renders hiveplotlib JSON exports as browser SVGs. Maps matplotlib kwargs to SVG/D3 attributes. Published as `@hiveplotlib/d3` on npm.

## [2026-04-06] ingest | Krzywinski et al. 2012
Ingested the foundational hive plot paper. Created source summary (`sources/krzywinski-2012.md`) and entity page (`entities/martin-krzywinski.md`). Paper introduces hive plots as rational alternative to force-directed layouts, with case studies on E. coli regulatory network and cancer gene-disease network. ~257 citations. Created concept pages: hive-plot, node-assignment, edge-rendering, force-directed-layout, hive-plot-matrix, p2cp, bezier-curves.

Pages created: 13. Pages updated: index.md, overview.md, log.md.

## [2026-04-06] ingest | Perez et al. 2021 — HyPE (Hive Panel Explorer)
Ingested via web research. Created source summary (`sources/perez-2021-hype.md`). First interactive formalization of the Hive Panel concept. Python 2.7 + D3.js, demonstrated on forest soil microbiome network (1,880 OTUs, 13,605 edges). Krzywinski is co-author. Tool no longer maintained. Updated hive-plot-matrix.md and martin-krzywinski.md with cross-references.

## [2026-04-06] ingest | Bostock 2012 — D3 Hive Plots
Ingested via web research. Created source summary (`sources/bostock-2012-d3-hive-plots.md`) and entity page (`entities/mike-bostock.md`). Influential D3.js v2 hive plot implementation visualizing Flare toolkit dependencies. Key insight: "force layouts make poor use of the most effective visual channel (position)." Architecturally monolithic vs. hiveplotlib-javascript's rendering-only approach.

## [2026-04-06] ingest | Nöllenburg & Wallinger 2023 — Computing Hive Plots
Ingested via web research. Created source summary (`sources/nollenburg-2023-computing-hive-plots.md`). First formal algorithmic framework for hive plot construction — proves all three subproblems (vertex partitioning, axis ordering, vertex ordering) are NP-complete. Updated hive-plot.md and hive-plot-matrix.md.

## [2026-04-06] ingest | Krzywinski et al. 2017 — Differential Hive Plots
Ingested via web research. Created source summary (`sources/krzywinski-2017-differential.md`) and concept page (`concepts/differential-hive-plot.md`). Visual diff between two networks. Published in Leonardo (MIT Press art/science journal). Not yet implemented in hiveplotlib. Updated martin-krzywinski.md.

## [2026-04-06] ingest | Domain Application Survey
Created three application domain concept pages from web research:
- `concepts/applications-bioinformatics.md` — Strongest domain (HyPE, HiveGraph, pleiotropy)
- `concepts/applications-cybersecurity.md` — Most innovative (DDoS classification via hive plot images as ML features)
- `concepts/applications-software-engineering.md` — Linux kernel, Flare toolkit, code dependencies
Updated hive-plot.md with application domain links.

## [2026-04-06] analysis | Karate Club Walkthrough
Created `analyses/karate-club-walkthrough.md` — step-by-step walkthrough of the karate_club.ipynb example. Covers dataset, partitioning strategy (faction membership), sorting (degree), edge styling (color by intra/inter-faction), and key insights (faction separation visible, leaders prominent, cross-faction connections NOT driven by sociability). Also compared with election_96 and bitcoin_user_ratings examples.

## [2026-04-06] lint | First Health Check
Performed first wiki lint pass. Results below.

**Pages created this session:** 23 content pages + index/overview/log
**Pages updated this session:** hive-plot.md, hive-plot-matrix.md, martin-krzywinski.md
