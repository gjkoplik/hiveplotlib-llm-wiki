---
title: "Watching a Neural Network Learn via P2CP (exploration)"
type: analysis
created: 2026-06-25
updated: 2026-06-25
sources: [hiveplotlib-python-repo]
tags: [p2cp, machine-learning, neural-networks, training-dynamics, datashader, exploration, prototype]
---

# Watching a Neural Network Learn via P2CP (exploration)

> **Exploratory prototype, not a hiveplotlib feature.** This documents a throwaway repo
> (`hiveplotlib-nn-viz`, GitHub `gjkoplik/hiveplotlib-nn-viz`) that exercised hiveplotlib's
> [[p2cp|P2CP]] and radial layouts on a real problem. Nothing here shipped into the library; the
> canonical summary is that repo's `PLAN.md`. Filed here for the reusable hiveplotlib takeaways
> and the prior-art framing, in case it graduates to a writeup.

## The bet

Watch a small neural network learn MNIST as a movie over training, using [[p2cp|polar parallel
coordinates]] and radial layouts. Two reasons the tool fits:

1. **Deterministic layout.** Node positions are fixed by the data, so in a training movie every
   bit of motion is real signal, not the frame-to-frame wobble of t-SNE / UMAP.
2. **Generalizable and library-level.** These views are a few lines on top of an existing
   library (`p2cp_n_axes()` on the softmax), the everyday way to get the view on any classifier,
   not a bespoke one-off.

The model: one tiny MLP, `784 -> 64 -> 32 -> 10`, ReLU, CPU, fixed seed, trained on MNIST and
logged to mlflow over 51 log-spaced checkpoints (dense early, since MNIST converges by ~step 35).
Two animated figures, each a 2x5 grid of per-digit panels.

## Figure A — per-class output-probability P2CP

One panel per true digit. That class's test images are datashaded loops over 10 axes (one per
class probability). Early, every loop hugs a tight uncertain central ring; trained, each class
blooms into a single **petal** on its own axis (the home axis is drawn red), and the confusions
show as faint secondary lobes. This is the polished, legible figure.

Built with `p2cp_n_axes(df, axes=..., split_on=...)`. This is the generalizable encoding: a
P2CP on the softmax gives a Grand-Tour-flavored gestalt of training dynamics on *any* classifier,
with per-class confusion surfaced as secondary lobes for free.

## Figure B — cross-layer co-activation, polar vs straight

Three axes (hidden1 / hidden2 / output). For each image we keep its class-*selective* neurons (see
below) and draw observed co-activation between them across **all three** pairwise layer
relationships (h1<->h2, h2<->output, and h1<->output), a closed triangle. Datashaded, animated.
The edges are observed co-activation, **not** weights.

A straight-parallel-coordinates version is built as the comparison (axes `h1 -> h2 -> out -> h1`,
a repeated h1 axis to close the loop). The point of the pair: the straight version needs a repeated
axis to do what polar closes for free, the direct argument for the radial layout.

This is the headline figure for novelty (see below), but it reads subtler than Figure A. A tiny
MLP's hidden code is distributed and overlapping, so the discriminative structure lives in
**output** space, which is why Figure A is crisp and Figure B is muted.

## Hiveplotlib usage notes and gotchas (reusable)

These are the library-level lessons worth carrying forward; they apply to any hand-rolled radial /
datashaded plot, not just this prototype.

- **Build manual hive plots with `BaseHivePlot`.** `HivePlot` requires `partition_variable` /
  `sorting_variables`; when you are placing nodes and connecting edges by hand (as Figure B does),
  `BaseHivePlot` is the right entry point.
- **`datashade_edges_mpl` is edges-only.** Compose `axes_viz` for the axes; the datashader edge
  renderer does not draw them.
- **When hand-rolling datashader, spread the AGGREGATE, then shade:** `tf.shade(tf.spread(agg))`,
  not `tf.spread(tf.shade(agg))`. Spreading the shaded image composites with "over", so a light
  (low-count) line paints over a dark crossing, which is impossible if you are truly counting.
  This is what hiveplotlib's `datashade_edges_mpl` does internally. (See [[edge-rendering]] for how
  hiveplotlib renders edges.)
- **Standardize the density scale.** One global `vmax` across panels and frames, or per-frame
  renormalization fakes the over-time change.
- **Per-class P2CPs come from `p2cp_n_axes(df, axes=..., split_on=...)`.**

## Key finding: structure is in output space, not hidden space

The hidden code of this MLP is **distributed**, not sparse (measured: ~20-38 of 64 hidden1 neurons
effectively active per image, barely changing over training). There is no clean per-digit "pathway"
to reveal in hidden space; the per-digit panels stay similar however you threshold. The
discriminative structure lives in **output** space, which is why Figure A reads cleanly and Figure
B is subtler.

Figure B's edges are therefore keyed off **class-selectivity** (activation above the neuron's
all-image baseline, the "average image"), not raw activation, with a data-driven cutoff (the
smallest neuron set covering ~80% of an image's selectivity, capped).

## Novelty framing (from a prior-art scan)

A prior-art scan (deep-research workflow plus a focused follow-up) reached this verdict:

- **Figure A (output-probability P2CP): incremental-toward-novel.** The **Grand Tour** (Distill
  2020, distill.pub/2020/grand-tour) owns the *use case*: it animates MNIST softmax over training,
  per-class convergence to simplex corners, even naming per-class learning epochs. The difference
  is layout. The Grand Tour is a bespoke linear-projection interactive masterpiece; this is
  `p2cp_n_axes()` on the softmax, the generalizable library-level way to get a Grand-Tour-flavored
  gestalt on any classifier, with confusion as secondary lobes for free. Frame it as a
  generalizable encoding that complements and explicitly cites the Grand Tour, **not** "first to
  visualize softmax training dynamics."
- **Figure B (cross-layer co-activation, polar): defensibly novel, the headline.** No located
  prior art draws neuron-to-neuron co-activation edges in any radial or parallel-coordinates form
  over training. The activation/co-activation canon (ActiVis, CNNVis, M-PHATE, Activation Atlas,
  Grand Tour) uses matrices, t-SNE / projections, or edge-bundled DAGs, never parallel coordinates,
  let alone polar.
- **Caveat:** "novel" means none located after an adversarial search, not provably first.
  Polar-PC-for-NNs is recent (~2021+), so an unindexed thesis or workshop could exist.

**Closest prior art to cite:** Grand Tour (distill.pub/2020/grand-tour); ConfusionFlow
(arXiv 1910.00969, confusion-over-training, Cartesian small multiples); ActiVis (arXiv 1704.01942)
and CNNVis (arXiv 1604.07043) for the activation-viz canon.

## What was tried and dropped

So the next person does not re-litigate:

- **Confusion slopegraph** (true-vs-predicted): built, dropped. The "correct" panel was just
  "accuracy went up" as filling bands, and the confusion it showed is already in Figure A's
  secondary lobes.
- **Stability long-exposure**: built, shelved. A union-of-a-window long exposure looked the same
  as the plain dense view, because count density conflates per-edge *recurrence* with spatial
  *overlap*.
- **Raw-activation / top-K hidden pathways**: abandoned once the hidden code measured distributed
  (see the finding above); there is no clean per-digit pathway in hidden space.

## Status and open question

Exploratory prototype. The open question is whether it graduates to a writeup: a blog or notebook
would showcase hiveplotlib's [[p2cp|P2CP]] to the ML-viz audience, with Figure A's generalizable
softmax encoding as the accessible hook and Figure B's co-activation triangle as the novel
centerpiece. No decision yet.

## See Also

- [[examples-and-applications]] — Catalog of hiveplotlib example/application explorations (this is one)
- [[p2cp]] — The encoding both figures lean on
- [[hiveplotlib]] — The library exercised
- [[edge-rendering]] — How hiveplotlib draws edges (and the datashader spread-then-shade rule)
- [[hive-plot]] — The radial layout method P2CPs borrow
