---
title: "Watching a Neural Network Learn: Co-Activation and Lock-In via Hive Plots (exploration)"
type: analysis
created: 2026-06-25
updated: 2026-07-04
sources: [hiveplotlib-python-repo]
tags: [p2cp, machine-learning, neural-networks, training-dynamics, co-activation, lock-in, datashader, exploration, prototype]
---

# Watching a Neural Network Learn: Co-Activation and Lock-In via Hive Plots (exploration)

> **Exploratory prototype, not a hiveplotlib feature.** This documents a throwaway repo
> (`hiveplotlib-nn-viz`, GitHub `gjkoplik/hiveplotlib-nn-viz`) that exercised hiveplotlib's
> [[p2cp|P2CP]] and radial layouts on a real problem. Nothing here shipped into the library; the
> canonical summary is that repo's `PLAN.md`. Filed here for the reusable hiveplotlib takeaways
> and the prior-art framing, in case it graduates to a writeup.

## What this is

A tiny MLP (`784 -> 64 -> 32 -> 10`, ReLU, CPU, fixed seed) trained on **both MNIST** (peak test
accuracy ~96.4%) **and Fashion-MNIST** (peak test accuracy ~86.4%). Fashion-MNIST brings the
garment confusions the digit task lacks: at convergence the upper-body classes blur together
(Shirt read as T-shirt/top 14%, Coat as Pullover 12%, Pullover as Shirt 11%, and so on across
the Shirt / T-shirt / Pullover / Coat cluster), while footwear stays cleaner (Sneaker as Ankle
boot at 7% is the top slip). Both runs are logged to mlflow over log-spaced checkpoints. Three animated
figures, each a 2x5 grid of per-class panels, using hiveplotlib's polar layouts:

- **Figure A**, per-class output-probability [[p2cp|P2CP]] over training (`p2cp_n_axes()` on the
  softmax): each class blooms from a tight uncertain central ring into a single petal on its own
  axis (home axis drawn red), confusions surfacing as faint secondary lobes.
- **Figure B**, cross-layer neuron **co-activation** in a closed polar triangle (hidden1 /
  hidden2 / output), drawing *observed co-activation* (not weights, not attribution) between each
  image's class-selective neurons across all three pairwise layer relationships, with a straight
  parallel-coordinates version built as the comparison.
- **Figure C (new)**, **lock-in vs bounce**: committed vs transient co-activation over a trailing
  6-checkpoint window, a committed density skeleton crystallizing out of a transient gray fog as
  each class's pathway fixes.

The deterministic layout is the whole point: node positions are fixed by the data, so every bit of
motion in a training movie is real signal, not the frame-to-frame wobble of t-SNE / UMAP.

## Honest yield of this re-run

This page rebuilds a prior scan's page; the panel that produced it was a **re-validation**, not a
fresh adjudication. Its genuine marginal yield was narrow and worth naming so the confident tone of
a rebuilt page does not launder inherited conclusions as freshly earned:

- **The panel *changed* three things.** It **demoted** the encoding-novelty claim (the polar-PC
  encoding is prior art, see below), **sharpened #2** from a blanket "no prior art draws
  co-activation in polar form" into a bounded three-property intersection with named near-misses,
  and **added #4's counter-current** (the lock-in finding had *zero* prior check before this run,
  and the literature turns out to actively contest it).
- **The panel *inherited* two verdicts.** #1 (softmax-P2CP positioning) and #3 (output-vs-hidden
  space) are near-certain re-confirmations of the prior scan, not fresh adjudications.
- **Two of the three staleness fixes were zero-agent page edits**, not panel work: adding
  Fashion-MNIST coverage and correcting the Figure B entry point (`HivePlotMatrix`, see usage
  notes). The panel earned its agents on #4 + the #2 refuter-recheck + re-grounding, not on
  re-deriving #1/#3.

## What was established

- **Figure A is a generalizable P2CP encoding of softmax training dynamics**, legible on both
  datasets: the ring-to-petal bloom with confusion as secondary lobes reads cleanly because the
  discriminative structure lives in output space (see #3).
- **Figure B renders observed cross-layer co-activation in a closed polar triangle**, keyed off
  class-selectivity rather than raw activation. The h1<->output edge (the closed side of the
  triangle) is a real observed co-activation measurement, which is what makes the radial layout
  earn its keep: a straight PCP needs the first-hidden axis repeated as an extra column to draw the
  same closed loop.
- **Figure C makes lock-in legible as a categorical committed/transient split.** A continuous
  mean-persistence ramp self-dilutes wherever arcs cross (pixels average toward mid-scale); the
  categorical split keeps the density signal so a committed skeleton reads against the transient
  fog, with commitment tracking class difficulty.

## What was validated, and how

Each finding below carries the adversary's convergence verdict (post-impl, `propose`: no finding
killed, mixed yield upheld as honest and un-laundered).

### #1 Softmax-probability P2CP, validated POSITIONING, not novelty

Frame Figure A as **a generalizable P2CP complement to the Grand Tour, explicitly not "first."**
The [Grand Tour](https://distill.pub/2020/grand-tour/) (Li, Zhao, Scheidegger, Distill 2020) owns
the *use case*: animating MNIST softmax over training, per-class convergence to simplex corners,
even naming per-class learning epochs. The differentiator is the *layout* (`p2cp_n_axes()` on the
softmax, generalizable to any classifier), not the phenomenon. **The encoding is prior art.**

Caveat named in the same breath: Figure A sits **conceptually close to RadViz on a probability
simplex**, an adjacency the search **did not fully reach**. RadViz is a named radial technique for
simplex/probability data; a RadViz-of-softmax-over-training, if one exists, would be a direct #1
refuter. Treat #1's position as scoped to a search that did not sweep this venue. HiPlot is an
NN + parallel-coordinates precedent, but for *hyperparameters*, not softmax dynamics.

### #2 Cross-layer co-activation in polar form, validated NARROW INTERSECTION (search-scope, not first-ness)

The defensible claim is a **three-property intersection**: co-activation edges (not attribution,
not weights) **and** polar / parallel-coordinates rendering (not force-directed) **and** per-image
over training. No single located work carries all three. Each near-miss fails a *different*, real
axis (the adversary checked this is a genuine intersection, not a novelty-preserving gerrymander):

| Near-miss | Has | Fails on | Faithful quote (the axis it lacks) |
|---|---|---|---|
| **Horta et al. 2021** (peer-reviewed, FGCS 120) | observed co-activation semantics, even on Fashion-MNIST | **rendering** (force-directed, not polar/PC) | section "Co-activation graph: Definition and construction"; "Fig. 1. Visualisation for MNIST-fashion co-activation graph made on Gephi"; "the visualisation algorithm, ForceAtlas 2 ... has placed some classes close to each other (e.g Sandal, Sneaker and Ankle Boot)" |
| **NeuroBreak** (arXiv:2509.03985, preprint) | radial rendering | **edge semantics** (attribution, not co-activation); a dashboard, not a per-image triangle | neurons "arranged along the outer ring" / "inner circle ... linked to their top 10% most influential W_down neurons via gradient-based attribution" |
| **TensorFlow Playground** (canonical, directly inspectable) | training-dynamics + connectivity | **geometry** (Cartesian node-link) | *paraphrase of the docs (not over-quoted, layout inspectable at playground.tensorflow.org):* weights drawn as colored lines between Cartesian layer columns |
| **VINCI 2010** (peer-reviewed, ACM 10.1145/1865841.1865852) | parallel coordinates on a backprop network | **what is drawn** (training samples, not connections); straight + static | "a parallel coordinates-based BP network visualization technique is proposed ... reveal the relations and rules hidden in the training samples in the process of the training" |

Optional context: chord diagrams give radial connectivity but for brains, static.

**Verdict phrasing (search-scope, never a grade):** *no refuter surfaced under a bounded search
that swept arXiv, distill, transformer-circuits, and VIS index pages; **ACM-complete, VISxAI
per-submission archives, PhD/MSc theses, and RadViz-on-a-probability-simplex were NOT fully
swept.*** This is a statement about search reach, not a claim of first-ness. Polar-PC-for-NNs is
recent (~2021+), so an unindexed thesis, workshop paper, or blog could exist in the unswept venues.

### #3 Output-vs-hidden space, validated DATA OBSERVATION

The MLP's hidden code is **distributed (dense), not sparse**: ~20-38 of 64 hidden1 neurons
effectively active per image, barely changing over training. This is a clean instance of a
*distributed* (not sparse, not modular) code, which is textbook, so the finding is that the
measurement *lands* here, not that it is novel. It is *why* the discriminative structure reads in
output space (Figure A crisp) and Figure B keys its edges off class-selectivity rather than raw
activation. Two flags ride with it: the measurement is **data-side, not literature-settleable**;
and the "distributed specialization" literature cautions against reading a distributed code as
crisp modular pathways, so Figure B is deliberately muted, not broken.

### Cross-cutting: the polar-vs-straight-PC method argument

Carried honestly with its headwind. Radial layouts **underperform** Cartesian on time and position
tasks in a controlled study (Diehl, Beck, Burch, IEEE TVCG, 674 participants), **but** that study
was **not** run on parallel coordinates / P2CP specifically, so reading it onto this design is an
**extrapolation** (it *suggests* a cost, it does not measure one here). Radial's one credited
advantage in that same work is *focusing on a single data dimension*, which is exactly what the red
home-axis design exploits. So the "polar closes the loop" argument is narrow to the **closure
affordance** (the h1<->output edge drawn for free), not a general-readability claim.

State plainly: **the polar-PC *encoding* is prior art**, not a novelty of this work. Precedents:
Kandogan Star Coordinates (IEEE InfoVis 2000), RadViz, and a separate (non-Koplik) "Polar Parallel
Coordinates Method for Time-Series" (IEEE doc 6300310). "P2CP" is the hiveplotlib author's own
coinage (arXiv:2109.10193 "extends an existing visualization ... the Polar Parallel Coordinates
Plot (P2CP)"), not an independent community term. **Novelty here is application-only, never the
encoding.**

## What was inconclusive, and why

### #4 Lock-in vs bounce, VALIDATED-INCONCLUSIVE

Figure C's committed-vs-transient split is a *visual* of pathway lock-in. The finding lands as a
**validated inconclusive**: pursued to a determination that the literature will not confidently
settle it, a first-class outcome, not a failed run. The visual-novelty note may be stated **only**
with all three caveats attached in the same breath, never parked in a slot a reader skims:

1. **The phenomenon is named, but only by analogy at the layer/weight level.** Achille, Rovere,
   Soatto, "Critical Learning Periods" (arXiv:1711.08856) uses Fisher Information for "effective
   connectivity between layers" and finds "the network is unable to change its effective
   connectivity: The connection strength of each layer remains substantially constant." The paper
   **never says "co-activation"**; it is an analogy to Figure C's construct, not the same object.
2. **The literature actively *contests* early lock-in.** Dynamic sparse training / ITOP (Liu et
   al., arXiv:2102.02887) finds "continuously exploring sparse connectivities during training" is
   *beneficial*, cutting directly against "early lock-in is good." A reader given (1) without (2)
   gets a one-sided phenomenon.
3. **The committed/transient-over-a-6-checkpoint-window construct is unnamed in the literature**,
   and whether the split is a real connectivity dynamic or an **artifact of the selectivity
   threshold / window length is UNSETTLEABLE in literature mode.** That check is deferred to a
   future data-validity-mode run (a threshold/window sweep).

So Figure C ships as "a possibly-novel *visual* of a phenomenon that is (i) named only by analogy,
(ii) contested, and (iii) unsettleable here as real-vs-artifact." It **may not** read as "the
phenomenon is real." This finding had **zero** prior-art check before this run.

## What's open

- **Data-validity of the lock-in split** (deferred, above): does the committed/transient split
  survive a selectivity-threshold and window-length sweep, or is it a knob artifact? A future
  data-validity-mode run, not a literature run.
- **RadViz-on-a-probability-simplex** and the ACM-complete / VISxAI-per-submission / thesis venues:
  an unswept reach that could hold a #1 or #2 refuter.
- **Whether it graduates to a writeup.** A blog or notebook would showcase hiveplotlib's
  [[p2cp|P2CP]] to the ML-viz audience: Figure A as the accessible hook, Figure B's co-activation
  triangle as the centerpiece, Figure C's lock-in as the training-dynamics payoff. No decision yet.

## Hiveplotlib usage notes and gotchas (reusable, corrected)

Library-level lessons for any hand-rolled radial / datashaded plot, not just this prototype.

- **Figure B (the per-class co-activation matrix) is built on the generic `HivePlotMatrix`, not
  `BaseHivePlot`.** `HivePlotMatrix(hive_plots=cells, backend="datashader")` takes a 2D list of
  per-cell hive plots and gives a built-in edge colorbar plus **cross-cell density unification**
  (one shared `vmax` across all panels via `.plot(...)`, returning the per-cell edge images to
  probe). `BaseHivePlot` is still the right entry point for a *single* hand-placed plot (`HivePlot`
  requires `partition_variable` / `sorting_variables`); here it survives only as the source of
  frozen node placements for the output-node marker.
- **`datashade_edges_mpl` is edges-only.** Compose `axes_viz` for the axes; the datashader edge
  renderer does not draw them.
- **When hand-rolling datashader, spread the AGGREGATE, then shade:** `tf.shade(tf.spread(agg))`,
  not `tf.spread(tf.shade(agg))`. Spreading the shaded image composites with "over", so a light
  (low-count) line paints over a dark crossing, which is impossible if you are truly counting.
  This is what hiveplotlib's `datashade_edges_mpl` does internally. (See [[edge-rendering]].)
- **Standardize the density scale.** One global `vmax` across panels and frames, or per-frame
  renormalization fakes the over-time change. Figure B gets this from `HivePlotMatrix`'s cross-cell
  unification; Figure C probes one densest window for the ceiling.
- **Per-class P2CPs come from `p2cp_n_axes(df, axes=..., split_on=...)`.**

## What was tried and dropped

So the next person does not re-litigate:

- **Confusion slopegraph** (true-vs-predicted): built, dropped. The "correct" panel was just
  "accuracy went up" as filling bands, and the confusion it showed is already in Figure A's
  secondary lobes.
- **Continuous mean-persistence ramp** (Figure C's first attempt): dropped for the categorical
  committed/transient split. The continuous ramp self-dilutes where arcs cross and draws a
  once-flickering edge as prominently as a committed bundle.
- **Stability long-exposure**: built, shelved. A union-of-a-window long exposure looked the same as
  the plain dense view, because count density conflates per-edge *recurrence* with spatial
  *overlap*.
- **Raw-activation / top-K hidden pathways**: abandoned once the hidden code measured distributed
  (finding #3); there is no clean per-digit pathway in hidden space.

## Sources

Compact; provenance trust noted where thin.

- **Grand Tour**, distill.pub/2020/grand-tour (Distill 2020, reviewed). Owns the softmax-over-training use case; #1's positioning anchor.
- **Horta et al. 2021**, "Extracting knowledge from Deep Neural Networks through graph analysis," FGCS 120:109-118 (peer-reviewed). Co-activation graph, even on Fashion-MNIST; #2 near-miss, fails on rendering.
- **NeuroBreak**, arXiv:2509.03985 (preprint, unreviewed). Radial neuron view; #2 near-miss, fails on edge semantics (attribution).
- **TensorFlow Playground**, playground.tensorflow.org (canonical tool; layout description is a docs *paraphrase*, not a quote). #2 near-miss, fails on geometry.
- **VINCI 2010**, ACM 10.1145/1865841.1865852 (peer-reviewed). PC on a backprop net but draws training samples; #2 near-miss, fails on what-is-drawn.
- **Critical Learning Periods**, Achille, Rovere, Soatto, arXiv:1711.08856. Layer/weight-level lock-in *by analogy* (never says "co-activation"); #4 phenomenon-naming.
- **ITOP / dynamic sparse training**, Liu et al., arXiv:2102.02887. Contests early lock-in (churn beneficial); #4 counter-current.
- **Kandogan Star Coordinates**, IEEE InfoVis 2000. Polar-PC encoding prior art.
- **Diehl, Beck, Burch**, IEEE TVCG, 674 participants (PubMed 20975130). Radial underperforms Cartesian on time/position; the cross-cutting headwind (extrapolated to PC, not tested on it).
- **Koplik P2CP**, arXiv:2109.10193. "P2CP" is the hiveplotlib author's own coinage extending an existing encoding, not a community term.

## See Also

- [[examples-and-applications]], Catalog of hiveplotlib example/application explorations (this is one)
- [[p2cp]], The encoding all three figures lean on
- [[hiveplotlib]], The library exercised
- [[edge-rendering]], How hiveplotlib draws edges (and the datashader spread-then-shade rule)
- [[hive-plot]], The radial layout method P2CPs borrow
