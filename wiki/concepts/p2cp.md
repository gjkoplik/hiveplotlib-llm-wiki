---
title: Polar Parallel Coordinates (P2CP)
type: concept
created: 2026-04-06
updated: 2026-04-06
sources: [hiveplotlib-python-repo]
tags: [p2cp, visualization, tabular-data]
---

# Polar Parallel Coordinates (P2CP)

A visualization method for **tabular (non-network) data** that uses the same radial axis layout as [[hive-plot|hive plots]]. Each data feature gets its own axis arranged in a polar configuration, and each row becomes a "loop" through the axes.

## Relationship to Hive Plots

- **Hive plots** visualize **networks** (nodes + edges)
- **P2CPs** visualize **tabular data** (rows + columns)
- Both use radially distributed axes with [[bezier-curves|Bézier curves]]
- Both are implemented in [[hiveplotlib]] with the same backend system

## Implementation

The `P2CP` class in [[hiveplotlib]] supports all six visualization backends. Construction is simpler than hive plots — provide a DataFrame and specify which columns map to axes.

## See Also

- [[hive-plot]] — The network visualization counterpart
- [[hiveplotlib]] — Implementation
