---
title: "Bioinformatics Hive Plot Examples (exploration)"
type: analysis
created: 2026-07-04
updated: 2026-07-04
sources: [hiveplotlib-python-repo, tabacof-2013-connectome-hive-plots, cook-2019-both-sexes-connectome, krzywinski-2012, yu-gerstein-2006-regulatory-hierarchy, kobourov-2018-same-stats-different-graphs, krzywinski-2017-differential]
tags: [bioinformatics, connectome, c-elegans, regulatory-networks, hiveplotlib-satellite, exploration]
---

# Bioinformatics Hive Plot Examples (exploration)

> **Public example repo, not a hiveplotlib feature.** This documents
> `hiveplotlib-bioinformatics-examples` (GitHub `gjkoplik/hiveplotlib-bioinformatics-examples`),
> a public companion repo of real-biological-network hive plots built on [[hiveplotlib]]. Nothing
> here ships into the library; the canonical detail (per-dataset provenance, modeling decisions,
> open questions) lives in that repo's `README.md`, `DATASETS.md`, and `notes/`. Filed here for the
> reusable [[hiveplotlib]] takeaways and, above all, the honest disposition of what the exploration
> confirmed, refuted, and left open. A sibling to [[nn-training-dynamics-p2cp-exploration]] and
> [[soccer-passing-hive-plots]], but with a stronger adoption thesis behind it.

> **Prior-art reconciliation (2026-07-04).** A bounded literature panel (six lenses, three verification
> vouchers, all findings verified against primary sources) validated and *corrected* the earlier
> backbone. The headline holds: neither built example is the instant-comparison hero, and the
> engineered [[same-stats-different-graphs|Datasaurus-for-networks]] work is where that hero lives.
> What changed: both of our built "stories" are **re-tellings** of published figures, not firsts;
> the E. coli role-partition attribution and the GRN license read were both wrong and are corrected
> below; and two claimed dead ends (C. elegans position metadata, the structural dimorphism figure)
> turned out to be **buildable**. The corrections are stated plainly in place.

## Executive summary

Bioinformatics is hiveplotlib's strongest adoption beachhead (see [[applications-bioinformatics]]),
so this exploration built real biological-network hive plots to hunt for a "hot damn" figure: one
that shows something you can *only* see with a hive plot, the kind of figure a JOSS or methods paper
could lead with. Two datasets were built to publishable quality: the **C. elegans connectome** and
the **E. coli gene regulatory network**. Both are credible real-data demonstrations for an audience
that already knows the format. **Neither is the hero**, and, as the literature panel established,
neither framing is *ours* to claim as first: both are hive-plot re-tellings of already-published
biology figures (details below). Even the best versions collapse to a *density* difference or reward
careful study rather than delivering an instant gut-punch, because a real biological network controls
for nothing. The exploration's durable strategic conclusion, now grounded rather than asserted: the
instant-comparison hero is the **Datasaurus-for-networks** work (engineered graphs with matched
statistics and pixel-identical node positions, see [[same-stats-different-graphs]]), where the
control is built in. The bioinformatics repo's role is confirmed as the credible **real-data** half
of the story, delta positioned as *execution and medium* (a deterministic, reproducible hiveplotlib
re-encoding), not as primacy.

## Why bioinformatics

Hive plots were invented in genomics by [[martin-krzywinski|Martin Krzywinski]], the audience already
knows the format, and the tools that carried it there (HiveR in R, hive_networkx in Python) went
dormant, leaving the niche orphaned. The core pitch is sharper than in any other domain: building a
hive plot means choosing what goes on each axis, and axis assignment is one of three provably
NP-complete subproblems ([[nollenburg-2023-computing-hive-plots|Nöllenburg & Wallinger 2023]]). In
biology that choice is often *a defensible partition of the system*, not a design decision to invent
from scratch: transcription-factor-vs-target role for a regulatory network, neuron class for a
connectome.

**Correction (softened from "biology hands you the axes for free").** For connectomes that "free
ground truth" claim is contested and the run refuted the strong form. Most C. elegans neurons are
polymodal, and more than half of interneuron classes have unknown function, so a
sensory / interneuron / motor split is a *defensible simplification*, not a fact the biology hands you
untouched. The honest pitch is "biology gives you **a defensible partition**," not "biology gives you
the axes for free." Where the partition is genuinely categorical and near-complete (a directed GRN's
regulator/manager/workhorse roles, below), the stronger claim survives; for connectomes it does not.

## C. elegans connectome (built; a re-telling of OpenWorm's figure, not the hero)

The nematode connectome: 302 hermaphrodite neurons, each with a settled functional class (sensory /
interneuron / motor), with synaptic signaling running sensory -> interneuron -> motor. Those three
biology-given, mutually exclusive Cook classes are the axes (300 of 302 neurons covered; only CANL
and CANR are unclassified, having no assigned synaptic function). Restricted to the chemical
neuron-only subgraph, 236 neurons participate.

**Prior art (confirmed re-telling, cite it).** The C. elegans connectome as a hive plot with
sensory / inter / motor axes, and the "beyond the hairball" framing, was published by Tabacof,
Busbice & Larson (OpenWorm) as a Frontiers in Neuroinformatics / Neuroinformatics 2013 conference
abstract, DOI 10.3389/conf.fninf.2013.09.00032 ([[tabacof-2013-connectome-hive-plots]]). Their own
abstract claims, verbatim, that "to the best of our knowledge this is the first application of the
hive plot visualization technique to any connectome data set." Our `hairball_vs_hiveplot.py` figure
is squarely their territory. The delta is **execution and medium**, a reproducible hiveplotlib
re-encoding, not primacy; cite them, do not frame the hairball-vs-hiveplot idea as ours.

**Sources (all MIT, all OpenWorm; science is Cook et al. 2019).** Connectivity from
`herm_full_edgelist.csv` in openworm/c302; functional classification parsed (via `ast`, never
imported) from the Cook category lists in openworm/ConnectomeToolbox `cect/Cells.py`. The
male-vs-hermaphrodite differential reads *both* sexes from the same Cook 2019 SI5 adjacency workbook,
so the comparison is not confounded by mixing processing pipelines. No data is committed; loaders
fetch and cache under a gitignored `data/`.

Four figures were built, in order of attempt:

1. **`c_elegans_flow.py`**: a single hive plot, chemical synapses colored by flow direction
   (feedforward / feedback / lateral, computed from endpoint classes, not plot geometry). Muddy: the
   worm is highly recurrent, so a single static plot does not read cleanly. Demoted to a loader
   sanity check.
2. **`hairball_vs_hiveplot.py`**: the same 236 neurons as a force-directed hairball versus a hive
   plot. Proves the format is reproducible and *organized*, but "organized" was judged not the same
   as "informative"; a supporting argument, not a headline. Prior-art note: this is the
   [[tabacof-2013-connectome-hive-plots|Tabacof 2013]] figure re-told.
3. **`sex_differential.py`**: the male-vs-hermaphrodite differential as one overlaid plot, emulating
   Krzywinski's [[differential-hive-plot|differential hive plot]] technique with edge tags. Among 227
   shared classified neurons: 1,073 shared connections, 1,774 hermaphrodite-only, 596 male-only.
   Real signal, but busy: the colored masses overlap.
4. **`sex_panels.py`**: the maintainer's own design, and the best of the four. A two-panel
   [[hive-plot-matrix|HivePlotMatrix]] (base constructor, `unify_axes=True`): hermaphrodite and male
   side by side, **identical node positions** (both sorted by degree over the *union* of both sexes),
   each panel drawing the wiring shared by both sexes in faint grey (an identical scaffold across
   panels) and that sex's unique wiring in color. Coloring only the differences is what surfaces the
   heterogeneity.

**Prior art for the differential (confirmed re-telling, benchmark against it).** The
male-vs-hermaphrodite comparison figure already exists as the canonical dimorphism visualization:
Cook et al. 2019 (*Nature*, "Whole-animal connectomes of both C. elegans sexes",
[[cook-2019-both-sexes-connectome]]) produced the **summed** herm+male network with edges color-coded
by which sex dominates (green = hermaphrodite-stronger, blue = male-stronger) on a **shared** layout
using an anatomical anteroposterior horizontal axis and a sensory -> inter -> motor vertical flow.
Our `sex_differential` / `sex_panels` figures are a **radial hive-plot re-encoding of that validated
Cartesian layout**. Benchmark against Cook's figure, not against a naive two-panel density plot; the
honest delta is the hive-plot medium and hiveplotlib's reproducible absolute-coordinate model, not a
new scientific reading.

**Verdict: confirmed as a credible real-data re-encoding, refuted as the hero.** Even the two-panel
version collapses to a **density** difference, not readable *structure*. Datashader would not rescue
it: at ~1-2k edges this is not an overplotting problem, so scale tooling has nothing to do. The root
cause is structural, not cosmetic: restricting to neurons shared by both sexes and sorting by a
scalar (degree) leaves only density free to vary, and that same restriction *deletes* the actual
dimorphism, since the real difference between the sexes is the male-specific tail and mating
circuitry, which lives in neurons the shared-only view throws away. An honest caveat compounds the
read: the two sexes were reconstructed from different animals with different methods, so the
1,774-vs-596 asymmetry is partly a detection-threshold artifact; any claim should lean on *where*
sex-specific wiring concentrates, not on gained-vs-lost totals.

**Correction (the position-metadata dead end was wrong; the structural figure is buildable).** The
earlier backbone concluded that showing the tail circuitry as structure "would need anatomical
position metadata that isn't cleanly sourceable." That is **refuted**. Anteroposterior soma position
*is* cleanly sourceable: the WormAtlas NeuronType table gives a numeric AP position (0 = nose,
1 = tail) plus 10-way ganglion codes, and Skuhersky et al. 2022 (BMC Bioinformatics 23:195,
github.com/bluevex/elegans-atlas) gives roughly 300 neuron positions in microns. So a
**hermaphrodite** position-sorted structural hive plot (sort each axis by AP body position, so a
localized block appears where the tail circuitry lives) is buildable today. The one residual gap: the
**male-specific** tail-neuron positions are not confirmed in a single tidy file (WormWiring is the
likely home). This is a **newly-opened opportunity**, not a closed door; see "What's open" below.

## E. coli RegulonDB regulatory network (built; Krzywinski's own worked example, re-told)

A gene regulatory network is directed and **signed**: each transcription factor activates or
represses each target. Signed, directed edges are exactly what a hive plot reads and a hairball
cannot. This example follows Krzywinski's *original* hive plot construction: partition nodes by
**role** in the directed graph and color edges by sign. This is not a novel application: the E. coli
role-partition GRN hive plot is [[krzywinski-2012|Krzywinski et al. 2012]]'s own worked
directed-network example ("Hive plots, rational approach to visualizing networks," *Briefings in
Bioinformatics* 13(5):627-644) and a HiveR staple. We re-told his canonical figure.

Source is RegulonDB's `network_tf_gene.txt`, ~4,544 unique TF->target pairs (2,295 activation,
2,028 repression, 216 dual). The dominant flow is dominant-role -> workhorse (3,115 edges); the
middle-role axis is repeated to draw the intra-core regulatory edges (330 edges). Citation for the
data release: Santos-Zavaleta et al., "RegulonDB v10.5," *Nucleic Acids Research*, 2019.

**Correction (role-partition attribution: the vocabulary is not Krzywinski's).** The earlier backbone
implied the regulator / manager / workhorse role partition was Krzywinski's invention. It is not, and
the run refuted that. Krzywinski **imports** the scheme from his ref [63] = Yu & Gerstein 2006
(*PNAS*, "Genomic analysis of the hierarchical structure of regulatory networks",
[[yu-gerstein-2006-regulatory-hierarchy]]), who coined "master regulators / middle managers /
workhorses." Krzywinski *applied* that scheme to RegulonDB as a hive plot. His own text, verbatim:
"We use the terminology from ref. [63] to refer to sources (out edges only) as 'regulators', sinks
(in edges only) as 'workhorses' and the other nodes as 'managers'." So the honest attribution is:
partition vocabulary = Yu & Gerstein 2006; hive-plot application of it = Krzywinski 2012.

**Correction (the repo's role label "dual" is wrong on two counts).** The repo currently labels the
middle role "dual," which is doubly incorrect: (1) the middle role is **"managers"** in the
Yu-Gerstein / Krzywinski vocabulary, not "dual"; and (2) "dual" *collides* with RegulonDB's own,
separate **sign** classification, where a regulatory interaction is activator / repressor / **dual**
(dual = context-dependent). Using "dual" for a *role* invites confusion with "dual" as a *sign*. The
repo has an actionable rename pending (regulator/dual/target -> regulator/manager/workhorse);
reference it as a follow-up, **not** as done.

Two figures (`regulatory_signs.py`):

- **Full hierarchy**: shows the "few control many" hierarchy *shape* that a hairball hides (a small
  regulator set and manager core fanning out to a huge workhorse axis). But at that density the sign
  of control collapses to a dense mix.
- **Regulatory core** (roles = regulator + manager, ~216 nodes): few enough edges that
  activation-vs-repression actually reads per transcription factor. This is a genuine **categorical**
  reading, not a density one: you can see dedicated repressors versus activators as bands of color.

**Licensing (corrected: restrictive EULA, not CC-BY; there is no clean permissive swap).** The
earlier backbone treated the licensing as an academic/non-commercial-with-citation nuance to confirm.
The run found the read was worse than that and corrects it: RegulonDB's *database* terms are a
**restrictive academic / noncommercial EULA** (v2.0, 2006) that forbids **distribution, derivative
works, and integration into other databases** (verified verbatim from
regulondb.ccg.unam.mx/manual/aboutUs/terms-conditions), *not* CC-BY. Two consequences the repo must
act on: (1) the repo cannot bundle the data, and the current **student-repo mirror fetch is itself an
EULA violation** (redistribution by the mirror); users must self-download from the canonical source
under the EULA. (2) There is **no clean permissive signed E. coli GRN** to swap to. Abasy Atlas is
CC-BY-NC (verified live), but being RegulonDB-derived it stays EULA-encumbered, so it is **not** a
clean permissive replacement. (Note: the run specifically refuted a sub-claim that Abasy carries a
third-party-rights caveat, no such text is on the live site, so do not assert that.)

**Verdict:** the core view delivers a real categorical reading (a step up from the connectome's
density-only story), but still "rewards study more than it grabs." A solid real-data example, and a
faithful re-telling of Krzywinski's own construction, not the hero either.

## The Datasaurus reframe (the key strategic conclusion, now grounded)

The exploration's most durable finding is a *counterfactual about where the hero lives*. The
instant-"hot damn" comparison artifact is **not** a single real biological network. It is the
**Datasaurus-for-networks** work (see the plan [[same-stats-different-graphs]], the grounding run
[[same-stats-different-graphs-grounding]], and the `hiveplotlib-datasaurus` scratch repo): engineered
graphs with matched summary statistics, identical degree sequences, **pixel-identical hive-plot node
positions**, and unmistakably different *readable* structure. That is the Anscombe / Datasaurus move,
where the control is **built in** by construction, so the only thing that can differ between two
panels is the structure the hive plot exposes.

**Prior art for the device on networks (confirmed; position our delta honestly).** The
"matched statistics, different structure" device applied *to networks* already exists: Chen, Soni,
Lu, Huroyan, Maciejewski, Kobourov, "Same Stats, Different Graphs," Graph Drawing GD'18
([[kobourov-2018-same-stats-different-graphs]], arXiv:1808.09913, with a 2019 extension
arXiv:1911.01527). The run verified it is **Anscombe-motivated** and uses a **spring / node-link**
reveal, no hive or radial layout. The device lineage is celebrated, not a gimmick:
Anscombe 1973 -> the Datasaurus (Matejka & Fitzmaurice, CHI 2017) -> Gelman's causal quartets 2023;
the counter-evidence lens searched for and found **no credible "gimmick" dismissal** of it. So
hiveplotlib's honest, *narrow* delta is not the device (Kobourov did it on networks first) but the
**deterministic hive-plot reveal medium**: the layout is a **function of the data** (a degree-rank
ordering applied cleanly and reproducibly), not a random seed, versus Kobourov's **seed-dependent**
node-link drawings. Claim the reproducible-deterministic-medium delta; do not claim the
matched-stats-on-networks idea.

Real biological networks reward study but cannot deliver that instant gut-punch, because nothing
controls for confounds: density, reconstruction method, and missing metadata all vary at once, and
the structural signal is entangled with them (exactly what sank the C. elegans density read above).
So the division of labor is settled: **Datasaurus is the hero** (engineered, handled separately by
the maintainer), and the **bioinformatics examples are the credible real-data demonstrations** for an
audience that already knows hive plots. Framing the connectome or the GRN as "hero examples" would
oversell them; framing them as real-world corroboration of the Datasaurus point is honest and holds.

## What was tried and dropped

Captured so it is not re-litigated:

- **A single undifferentiated hive plot as the headline**: the weakest rhetorical case *when the axis
  choice is not load-bearing*. Note the softening: a single hive plot does **not** always collapse to
  density. Celebrated single-panel hive plots (the EMBO 2011 cover, the crucifer synteny figure) read
  as more than density precisely because the axis assignment carries meaning. The honest claim is that
  an **undifferentiated single plot without a load-bearing axis choice** collapses to density; the
  format's superpower is being a *fixed coordinate system*, which pays off most in **comparison**.
  (Both `c_elegans_flow.py` and the GRN full-hierarchy view sit at the undifferentiated end.) The
  general "comparing two networks by density is hard" limitation is a recognized field-wide problem
  (e.g. NodeTrix); treat that as **related reading**, not as primary grounding for our specific claim.
- **C. elegans flow-direction coloring**: muddy, because the worm is highly recurrent.
- **hairball-vs-hiveplot as the hero**: proves tidiness and reproducibility, not insight (and it is
  [[tabacof-2013-connectome-hive-plots|Tabacof 2013]]'s figure, not ours).
- **The overlaid sex differential**: real signal but busy; the two-panel unified-axes version is
  cleaner but, as above, still density-only, and it re-encodes [[cook-2019-both-sexes-connectome|Cook
  2019]]'s validated Cartesian dimorphism layout.
- **Datashader to rescue the density story**: rejected. Datashader is for *scale* (millions of
  edges, genuine overplotting), not for manufacturing structural nuance at ~1k edges. Reaching for it
  here would be a category error.
- **The GRN full-hierarchy view for the sign story**: sign collapses to a mix at that density; the
  regulatory-core subset is where sign reads.

## Reusable hiveplotlib patterns

The durable *library* value of the exploration, the API moves that carried these figures:

- **`HivePlot(graph=...)` and `node_graph_metrics=[...]`** (per [[0001-networkx-integration|ADR 0001]],
  see [[graph-features]]) to build straight from a network and attach structural sort variables
  (degree) in one step, replacing manual extract-and-merge.
- **Multi-tag `Edges` plus per-tag `update_edge_viz_kwargs(tag=...)`** for semantic edge coloring:
  synapse flow direction, activation/repression sign, or shared/sex-only membership, each a tag styled
  independently. This is the mechanism that carries every colored-edge reading in the repo.
- **The base `HivePlotMatrix` constructor with `unify_axes=True`** for a two-network comparison with
  **identical node positions**. The load-bearing trick: sort *both* networks by a **shared** metric
  (degree over the union), not by each network's own metric, or the panels stop being comparable. This
  is the pattern behind `sex_panels.py`.
- **`repeat_axes=[...]`** to draw intra-partition edges (manager->manager in the GRN core, same-class
  connections), which are otherwise undrawable when both endpoints share an axis.
- **A differential hive plot can be emulated today with edge tags** (shared / A-only / B-only on one
  fixed layout). This is a working spec for hiveplotlib's roadmapped native
  [[differential-hive-plot|differential-hive-plot]] feature
  ([[krzywinski-2017-differential|Krzywinski et al. 2017]]): the tag-based emulation shows the
  ergonomics a first-class feature would replace.

## Status and open questions

Exploration, with a settled disposition: two credible real-data examples confirmed (both re-tellings
of published figures), the hero-figure question answered *negative* for real biological networks, and
the hero relocated to the Datasaurus work. The literature panel corrected two attributions (the GRN
role vocabulary is Yu & Gerstein's, not Krzywinski's; the GRN data is under a restrictive EULA, not
CC-BY) and re-opened two "dead ends" as buildable opportunities. This exploration ran with a
counterfactuals-first disposition in the spirit of [[gnn-heterogeneity-hive-plots]]: the negative
result (real networks are not the hero) is a first-class outcome, not a failed run.

**What's open (counterfactuals the run surfaced, each grounded honestly):**

- **The buildable-but-unbuilt structural C. elegans figure.** Sort each axis by AP body position so
  the male's tail circuitry appears as a localized block. Hermaphrodite positions are clean
  (WormAtlas NeuronType, Skuhersky 2022); the one gap is confirmed male-specific tail-neuron
  positions in a tidy file (WormWiring likely). This is the run's clearest *newly-opened* opportunity.
- **Native differential hive plots as a hiveplotlib feature.** Krzywinski, Nip, Birol, Marra,
  "Differential Hive Plots: Seeing Networks Change," *Leonardo* 50(5):504 (2017,
  [[krzywinski-2017-differential]]) is the technique; it needs exactly hiveplotlib's
  absolute-coordinate model. The run found **no confirmed published differential-connectome or
  differential-GRN figure** (weak-evidence absence, searched but unindexed literature could hide one),
  so "the comparative angle done reproducibly" may be genuinely under-trodden. Framed as
  weak-evidence, not a firm first.
- **Hemisphere-axis brain connectome (the cleanest untried AXIS story).** The Budapest Reference
  Connectome (via Netzschleuder) offers a literal hemisphere node attribute, the tidiest
  "biology-hands-you-the-axis" case found, and it also motivates the **scaling** story. Blocker: its
  **redistribution license is unconfirmed / unstated** (no CC statement on the page; the page's AGPL
  is graph-tool's, not the data's). Present this as "license needs resolving," **not** as
  redistributable.
- **Lower-priority framings (each with its licensing / fit caveat):** metabolic / pathway networks
  (Reactome is CC0 and usable; KEGG is license-blocked); PPI by GO cellular-compartment axes (BioGRID
  is MIT, STRING is CC-BY, but the prior tooling is crowded and localization is many-to-one);
  cell-cell communication (fits [[p2cp|P2CP]] better than a 2-3 axis hive plot).
- **GRN data source and licensing (must resolve before public ship).** Replace the EULA-violating
  student-mirror fetch with a self-download from the canonical RegulonDB source under its EULA;
  there is no clean permissive signed E. coli GRN to swap to (Abasy Atlas is CC-BY-NC but
  RegulonDB-derived and EULA-encumbered).
- **The repo's role-label rename** (regulator/dual/target -> regulator/manager/workhorse) to fix the
  wrong middle-role name and the "dual" role/sign collision. Actionable follow-up, not yet done.

## See Also

- [[examples-and-applications]] — Catalog of hiveplotlib example/application explorations (this is one)
- [[applications-bioinformatics]] — Why bioinformatics is the strongest adoption domain (this fills the connectome/neuroscience gap)
- [[same-stats-different-graphs]] — The Datasaurus-for-networks plan: the engineered hero this real-data work corroborates rather than replaces
- [[same-stats-different-graphs-grounding]] — The grounding run behind the Datasaurus hero (network-science foundations confirmed; the degree-rank-ordering caveat)
- [[tabacof-2013-connectome-hive-plots]] — OpenWorm's first C. elegans connectome hive plot; the hairball-vs-hiveplot figure we re-told
- [[cook-2019-both-sexes-connectome]] — The canonical both-sexes dimorphism figure our sex-panels re-encode radially; benchmark against it
- [[yu-gerstein-2006-regulatory-hierarchy]] — Coined the master-regulator / middle-manager / workhorse role vocabulary Krzywinski imported
- [[kobourov-2018-same-stats-different-graphs]] — "Same Stats, Different Graphs" (GD'18): the matched-stats device on networks, with a spring reveal, that predates our deterministic hive-plot medium
- [[hiveplotlib]] — The library exercised
- [[hive-plot-matrix]] — The two-panel unified-axes comparison behind `sex_panels.py`
- [[differential-hive-plot]] — Roadmapped native feature; emulated here with edge tags
- [[krzywinski-2017-differential]] — The differential-hive-plot technique (Leonardo 2017)
- [[graph-features]] — Structural metrics (degree) attached at construction
- [[martin-krzywinski]] — Invented hive plots in genomics; the E. coli role-partition GRN worked example and the differential technique are his
- [[krzywinski-2012]] — The founding paper, whose E. coli / RegulonDB role-partition hive plot we re-told
- [[nollenburg-2023-computing-hive-plots]] — Axis assignment is NP-complete, which is exactly the part a defensible biology partition helps with
- [[gnn-heterogeneity-hive-plots]] — Precedent for a counterfactuals-first, honest-scorecard exploration
- [[nn-training-dynamics-p2cp-exploration]] — Sibling exploration (NN-viz)
- [[soccer-passing-hive-plots]] — Sibling exploration (soccer passing)
- [[force-directed-layout]] — The hairball this work argues against
