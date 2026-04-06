---
title: "Nöllenburg & Wallinger 2023 — Computing Hive Plots: A Combinatorial Framework"
type: source
created: 2026-04-06
updated: 2026-04-06
sources: [nollenburg-2023]
tags: [hive-plot, algorithm, graph-drawing, computational-complexity]
---

# Nöllenburg & Wallinger 2023 — Computing Hive Plots: A Combinatorial Framework

**Citation:** Nöllenburg, M. and Wallinger, M., 2023. Computing Hive Plots: A Combinatorial Framework. *Proceedings of the 31st International Symposium on Graph Drawing and Network Visualization (GD 2023)*, Springer LNCS. Also published in *Journal of Graph Algorithms and Applications*.

## Summary

The first paper to formalize [[hive-plot]] construction as a **combinatorial optimization problem**. Decomposes the problem into three NP-complete subproblems and provides algorithmic solutions.

## The Three Subproblems

1. **Vertex partitioning** — Assigning nodes to axis groups (see [[node-assignment]])
2. **Cyclic axis ordering** — Determining the angular order of axes around the center
3. **Vertex ordering on each axis** — Minimizing edge crossings within each axis

All three are shown to be **NP-complete** in the general case.

## Significance

This paper gives hive plots a **formal algorithmic foundation**, moving them from a heuristic visualization technique to one with provable optimization properties. Previous work (including [[krzywinski-2012]]) relied on domain-specific heuristics for axis assignment and node ordering.

## Implications for [[hiveplotlib]]

The combinatorial framework suggests that automated optimization of hive plot parameters is fundamentally hard, which validates the approach of:
- Using domain knowledge for [[node-assignment]] (rather than brute-force optimization)
- Exploring parameter space via [[hive-plot-matrix|HivePlotMatrix]] panels
- Human-in-the-loop parameter selection

## See Also

- [[hive-plot]] — The visualization method being formalized
- [[node-assignment]] — The first subproblem
- [[hive-plot-matrix]] — A practical response to the parameter selection problem
