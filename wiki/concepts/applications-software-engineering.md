---
title: "Applications: Software Engineering"
type: concept
created: 2026-04-06
updated: 2026-04-06
sources: [krzywinski-2012, bostock-2012-d3-hive-plots]
tags: [applications, software-engineering, code-dependencies]
---

# Applications: Software Engineering

Software dependency graphs are a natural fit for [[hive-plot|hive plots]] because code modules have clear categorical roles (sources, sinks, connectors) and quantitative metrics (dependency count, import depth).

## Key Applications

### Linux Kernel Call Graph
[[krzywinski-2012]] used the Linux kernel function call graph as a key demonstration, comparing it with the E. coli gene regulatory network. The hive plot revealed that the Linux call graph has a top-heavy hierarchy (>80% of functions in upper levels), while E. coli has a conventional pyramid — a structural difference invisible in [[force-directed-layout|force-directed layouts]].

### Flare Visualization Toolkit
[[bostock-2012-d3-hive-plots|Bostock (2012)]] visualized the Flare toolkit's class dependency graph with D3. Three axes: sources (only outgoing deps, top), targets (only incoming deps, bottom-left), and both (duplicated, bottom-right). Nodes sorted by package to reveal macro structure.

Key insight from Bostock's example: "The highest-level implementations (such as layouts and controls) are arranged in the top axis, while interfaces and internal abstractions are in the bottom-right."

### hiveplotlib Examples
[[hiveplotlib]]'s example notebooks include NetworkX graph visualizations that demonstrate dependency-style analysis patterns.

## Why Hive Plots Work Well Here

- Code modules have natural categories: libraries, interfaces, implementations, utilities
- Dependency count and import depth are meaningful sorting metrics
- Reproducibility matters for tracking how dependencies evolve across versions
- [[krzywinski-2017-differential|Differential hive plots]] could show how dependency graphs change between releases

## See Also

- [[hive-plot]] — The visualization method
- [[bostock-2012-d3-hive-plots]] — Flare example
- [[applications-bioinformatics]] — Strongest adoption domain
- [[applications-cybersecurity]] — Most innovative domain
