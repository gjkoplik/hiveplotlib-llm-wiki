---
title: "Spectral Hive Plots: Cluster and Sort from One Decomposition"
type: analysis
created: 2026-07-04
updated: 2026-07-04
sources: [hiveplotlib-python-repo, krzywinski-2012, perez-2021-hype, nollenburg-2023-computing-hive-plots, koren-2005-graph-drawing, atkins-1998-spectral-seriation, nedialkova-2014-diffusion-map-md, aron-2020-gnps, scalfani-2022-chemical-space-networks, inoue-2010-diffusion-ppi]
tags: [spectral-clustering, node-assignment, hive-plot, research-proposal, network-analysis, diffusion-maps, cheminformatics]
---

# Spectral Hive Plots: Cluster and Sort from One Decomposition

A research direction (prototype in the `hiveplotlib-spectral` repo) for driving a
[[hive-plot]] entirely from a spectral decomposition of a network. The idea came
from a chemistry-minded collaborator (personal communication): spectral methods
are already how many computational and medicinal chemistry workflows cluster and
reason about networks, so a hive plot that reads directly off the spectrum could
land in an audience that already thinks in cuts and eigenvectors.

> **Status: prototype-backed, literature-grounded (2026-07).** A five-angle
> research pass (21 sources, 25 verified claims, none refuted) settled the
> novelty question and ranked the use cases. The verdict and its citations are
> below.

## The core idea

A hive plot needs three inputs. A single spectral decomposition supplies all
three, which is what makes the pairing natural rather than arbitrary:

| Hive plot input | Supplied by |
| --- | --- |
| which axis a node sits on ([[node-assignment]]) | the k-way partition (the first cuts) |
| where along the axis it sits | a *higher* eigenvector, one the partition does not use |
| edges between axes ([[edge-rendering]]) | the affinity graph itself |

The collaborator's framing: cut the data in two places (three clusters); the
first eigenvectors define those two cuts and become the three axes; the third
and later eigenvectors, which are not needed to define the partition, become the
within-axis sort. See [[spectral-clustering]] for the underlying method and the
eigenvector bookkeeping.

## Novelty verdict: a novel recombination

The idea is **new in its assembly, not in its parts, and not something already
done in this exact form.** Each of the three ingredients is established prior
art, and no source was found that assembles all three into a hive plot:

- **Hive-plot axis assignment is never spectral.** In [[krzywinski-2012]], nodes
  sit on axes "according to a well-defined coordinate system" set by Boolean
  tests ("is the node a sink?") and rank-ordered node parameters such as
  connectivity. The Hive Panel Explorer ([[perez-2021-hype|Perez et al. 2021]])
  and the combinatorial formalization
  ([[nollenburg-2023-computing-hive-plots|Nöllenburg & Wallinger 2023]]) keep
  assignment and within-axis order tied to user-chosen vertex attributes; the
  2023 contribution is optimization of edge length and crossings, not spectral.
  The 2012 paper explicitly contrasts hive plots *against* spectral layouts.
- **Spectral graph drawing already uses eigenvectors, but as Cartesian axes.**
  [[koren-2005-graph-drawing|Spectral graph drawing]] (Hall 1970; Koren 2005)
  places node *i* at continuous coordinates (v2_i, v3_i, ...) on orthogonal
  axes, "without any quantization or discretization." That is one eigenvector
  per orthogonal axis, not a radial cluster axis defined by a quantized cut. So
  "use Laplacian eigenvectors as coordinates" cannot be claimed as new; the
  radial-cluster-axis framing is the departure.
- **Ordering by an eigenvector is classic spectral seriation.** The within-axis
  1-D ordering is exactly [[atkins-1998-spectral-seriation|Fiedler-vector
  seriation]] (Atkins, Boman & Hendrickson 1998), which provably recovers the
  correct chain ordering in the noiseless case. So the sort component is not
  novel either.

The closest spatial analog, a [[inoue-2010-diffusion-ppi|diffusion-process
spectral clustering of PPI networks]] (Inoue, Li & Kurata 2010), does lay nodes
out radially from the origin, but it is an angular-clustering *scatter*, not a
hive layout, and uses no higher-eigenvector within-axis order. It gets the
spectral-embedding half, not the recombination.

**The mechanism is independently validated in chemistry.** The one part that
felt like a leap, reading *within-cluster* structure off a *higher* eigenvector,
is standard practice in diffusion-map analysis of molecular dynamics. See below.

## Findings status (triangle prototype in `hiveplotlib-spectral`)

An honest scorecard from the toy prototype. The toy is a triangle in the plane:
three ground-truth types, each with half its mass in a tight blob on a vertex and
half decaying along the directed edge to the next vertex. The decaying tails
manufacture controllable inter-cluster overlap.

**The keeper.** The three-ingredients mapping works end to end. Because the
triangle is topologically a noisy cycle, its Laplacian eigenvectors behave like
cycle harmonics: the fundamental pair makes the cuts, and the next harmonics vary
*within* each cluster (dense vertex core at one end, sparse overlap tail at the
other). Sorting an axis by such a harmonic produces a core-to-periphery order the
partition alone does not carry, and the cross-cluster affinity edges concentrate
on the overlap boundary.

**Required correction: local scaling is not optional here.** The data is
deliberately multi-scale (tight blobs, sparse edges). A global RBF bandwidth
turns the sparse edges into near-cuts and fills the spectrum with spurious
near-zero eigenvalues; the early eigenvectors then localize onto fragments
instead of forming clean global harmonics. Per-point local scaling (Zelnik-Manor
& Perona) fixes it; ground-truth agreement went from ~0.67 to ~0.94.

**Counterfactual, rejected: a different eigenvector per axis.** The triangle's
threefold symmetry makes the first harmonic family degenerate, so any single
higher eigenvector spreads some axes and flattens others. A per-axis
"symmetry-adapted" sort was built and it did make all three axes spread at once.
It was removed: it injects a choice the collaborator never asked for, and the
degeneracy driving it is an artifact of the toy's exact symmetry. Notably, the
literature confirms this is not just a toy quirk: eigenvector swapping under
eigenvalue near-degeneracy is a recognized fragility of spectral methods, so the
within-axis order can be unreliable on real affinity graphs (see Open questions).

**Counterfactual, decided: random-walk over symmetric normalization.** For the
sort, normalization matters even though the partition does not care (sym vs rw
partitions agree 99.7%). The rw eigenvectors are the relaxed normalized-cut
indicator functions and the diffusion-map coordinates; the sym eigenvectors are
the same functions scaled by sqrt(degree), a solver convenience. On this data the
degree factor varies ~2x and reshuffles the within-axis order (Spearman ~0.95).
**The literature corroborates this choice:** the chemistry precedent below uses
diffusion / transfer-operator eigenvectors (kinetic maps; Noë & Clementi 2015),
which are the rw-family coordinates, not the sym ones. Sorting by rw gives axis
position the cut/kinetic interpretation the audience wants.

**Honest limitation.** Everything above is one symmetric toy. The clean harmonic
story and the degeneracy are both consequences of the construction. A real
network (ideally a chemistry one) is the necessary next test.

## Relevance and use cases (ranked, chemistry-weighted)

1. **MD conformational analysis (diffusion maps / Markov state models) — most
   technically credible.** This is where the mechanism is already proven. In
   [[nedialkova-2014-diffusion-map-md|diffusion-map analysis of alanine
   pentapeptide]], v1 organizes points by a folded/unfolded criterion while v2
   and v9 resolve substructure *inside* individual states, exactly the
   "low eigenvector = cut, higher eigenvector = within-cluster order" split the
   hive plot relies on. And the eigenvectors carry principled kinetic meaning:
   they are the optimal reaction coordinates, and rescaled they form a kinetic
   map where Euclidean distance equals kinetic distance (Noë & Clementi 2015).
   MSM microstates are already coarse-grained by the sign of each eigenvector
   (PCCA). A spectral hive plot here would visualize an object practitioners
   already compute.
2. **MS/MS molecular networks (GNPS) and chemical space networks — the clearest
   tooling gap.** [[aron-2020-gnps|GNPS molecular networking]] visualizes MS2
   spectra as force-directed node-link graphs whose connected components are
   "molecular families," and the standard in-browser view "can only display one
   molecular family at a time." [[scalfani-2022-chemical-space-networks|Chemical
   space networks]] likewise default to Fruchterman-Reingold force-directed
   layout with no spectral component. A combined static view of the cut, the
   within-family substructure, and the inter-family bridges plausibly fills a
   real gap. This is a design inference, not a tested claim (see caveats).
3. **PPI network partitioning — a supporting analog.** Diffusion spectral
   clustering of PPI networks ([[inoue-2010-diffusion-ppi|Inoue et al. 2010]]) is
   the closest existing radial spectral layout, but it stops at angular
   clustering.

## What would kill this idea

- **A single paper that already does the full recombination** (spectral-cluster
  axes + higher-eigenvector within-axis order + affinity edges, in any radial or
  hive layout). None was found, but the novelty claim is absence-of-evidence, so
  a niche-venue counterexample could still exist. A targeted graph-drawing /
  manifold-learning search is the way to firm this up.
- **Evidence that a t-SNE / UMAP scatter of the eigenvectors reads cut structure
  and inter-cluster overlap strictly better than an axis layout**, making the
  hive framing redundant. The research pass did *not* resolve this comparison; it
  is an open question, not a settled point in our favor.

## Open questions

- **Hive placement vs. t-SNE / UMAP of the same eigenvectors** for reading cut
  structure and overlap. No confirmed evidence either way yet.
- **Stability of the within-axis order under eigenvalue near-degeneracy.**
  Eigenvector crossing/swapping is a known spectral fragility and is exactly what
  our symmetric toy triggered. How badly does it bite on real, less-symmetric
  affinity graphs?
- **Empirical bridge legibility.** On a real GNPS / CSN / MSM dataset, does the
  spectral hive plot actually reveal inter-cluster bridge edges more legibly than
  the incumbent force-directed view, or do the radial axes obscure them?
- **Operator mismatch.** Some chemistry precedents use the Markov
  propagator / transfer-operator spectrum, not a strict graph-Laplacian
  normalized cut. Same "low = cut, high = substructure" family, not an identical
  operator, so the mapping is an analogy to firm up, not a proven equivalence.

## Method (prototype)

Pipeline in `hiveplotlib-spectral` (dataset -> spectral -> hive figures, one
`spectral.npz` as the single source of truth):

1. Build a symmetric, locally-scaled RBF k-NN affinity ([[spectral-clustering]]).
2. Random-walk Laplacian; take the smallest-eigenvalue eigenvectors.
3. k-means on the first k for the partition (the axes); one higher eigenvector as
   the within-axis sort.
4. Cross-cluster affinity edges as the hive edges; render with matplotlib, and
   with datashader-rasterized edges for the density-heavy case.
5. A per-ground-truth decomposition (one facet per type, drawn from
   `Axis.node_placements`) to defeat node occlusion.

## Next steps

- Run a real chemistry network (MD/MSM eigenvectors are the highest-credibility
  entry point since they are already computed) and retest the harmonic-sort story
  without the toy's symmetry.
- Write the practitioner guidance: which higher eigenvector to sort by, rw vs
  sym, and per-axis range clipping when a harmonic localizes on a few points.
- Firm up the novelty absence-of-evidence with a targeted graph-drawing search,
  and resolve the hive-vs-UMAP comparison empirically.

## See Also

- [[spectral-clustering]] — the method and eigenvector bookkeeping
- [[node-assignment]] — spectral clustering as a "which axis / where on axis" rule
- [[hive-plot]] — the visualization
- [[hiveplotlib]] — the library the prototype builds on
- [[nedialkova-2014-diffusion-map-md]] — the validating chemistry precedent
- [[gnn-heterogeneity-hive-plots]] — sibling research direction, same page shape
