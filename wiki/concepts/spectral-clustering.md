---
title: Spectral Clustering
type: concept
created: 2026-07-04
updated: 2026-07-04
sources: [koren-2005-graph-drawing, atkins-1998-spectral-seriation, nedialkova-2014-diffusion-map-md]
tags: [spectral-clustering, graph-partitioning, laplacian, diffusion-maps, network-analysis]
---

# Spectral Clustering

Partition a graph (or a similarity graph built from data) using the eigenvectors
of a graph Laplacian, rather than by optimizing a combinatorial cut directly. It
is the machinery [[spectral-hive-plots]] runs on, and it happens to supply both
halves of [[node-assignment]] at once: a partition (which axis) and a continuous
coordinate (where on the axis).

## The pipeline

1. **Affinity.** Build a symmetric similarity matrix W (for example an
   RBF-weighted k-nearest-neighbor graph). Large W_ij means i and j are similar.
2. **Laplacian.** Form a graph Laplacian from W and the degree matrix D
   (D_ii = sum_j W_ij).
3. **Embedding.** Take the eigenvectors of the smallest eigenvalues. Stacking the
   first few as columns gives each node a low-dimensional "spectral embedding."
4. **Cluster.** Run k-means on the rows of that embedding for the hard partition.

Von Luxburg's "A Tutorial on Spectral Clustering" (2007) is the standard
reference for the whole construction.

## Which Laplacian: the normalization matters

Three standard choices, differing by a density factor:

- **Unnormalized**: L = D - W.
- **Symmetric** (Ng, Jordan & Weiss 2002): L_sym = I - D^-1/2 W D^-1/2.
- **Random walk** (Shi & Malik 2000, normalized cut): L_rw = I - D^-1 W.

The rw eigenvectors are the *relaxed normalized-cut indicator functions* (entry i
reads "which side of the cut is node i on") and coincide with the
[[nedialkova-2014-diffusion-map-md|diffusion-map]] coordinates (Coifman & Lafon
2006). The sym eigenvectors are the same functions scaled by sqrt(degree), a
numerical convenience with no cut meaning. So a *partition* is nearly identical
under sym vs rw, but a *within-cluster ordering* read off the eigenvectors is
not, which is why [[spectral-hive-plots]] sorts by rw.

## Eigenvector bookkeeping

For a connected graph, in ascending eigenvalue order:

- **Column 0**: the trivial near-constant vector (eigenvalue ~ 0). No information.
- **Columns 1 .. k-1**: the cuts that carve k clusters. These define the
  partition (the "which axis" decision).
- **Columns k, k+1, ...**: higher harmonics the partition does not use. They vary
  *within* clusters and are the candidate within-axis sort. In
  [[nedialkova-2014-diffusion-map-md|diffusion-map MD analysis]] exactly this
  appears: a low eigenvector splits the top-level states while higher ones
  resolve substructure inside a single state.

## Local scaling for multi-scale density

A single global affinity bandwidth fails when density varies sharply (tight
clusters plus sparse bridges): the sparse regions become near-cuts, the spectrum
fills with spurious near-zero eigenvalues, and the early eigenvectors localize
onto fragments. **Local scaling** (Zelnik-Manor & Perona 2004) gives each point
its own bandwidth (the distance to its k-th neighbor), restoring clean global
structure. The `hiveplotlib-spectral` prototype needs this for its multi-scale
toy.

## Relatives that use the same eigenvectors differently

- [[koren-2005-graph-drawing|Spectral graph drawing]]: eigenvectors as continuous
  Cartesian coordinates.
- [[atkins-1998-spectral-seriation|Spectral seriation]]: the Fiedler vector as a
  1-D ordering.
- Diffusion maps: eigenvectors rescaled by their eigenvalues, giving a geometry
  where Euclidean distance equals diffusion distance.

## A known fragility

When two eigenvalues are near-degenerate, their eigenvectors can swap or mix
arbitrarily within the shared eigenspace, so an ordering read from an individual
eigenvector can be unstable. The `hiveplotlib-spectral` toy hit exactly this
through its threefold symmetry, which is worth remembering before trusting a
within-axis order on a near-symmetric real graph.

## See Also

- [[spectral-hive-plots]] — driving a hive plot from this decomposition
- [[node-assignment]] — partition = which axis, higher eigenvector = where on axis
- [[hive-plot]] — the visualization
- [[nedialkova-2014-diffusion-map-md]] — the chemistry precedent for the higher-eigenvector sort
