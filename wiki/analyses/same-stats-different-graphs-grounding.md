---
title: "Same Stats, Different Graphs: Is the Hive-Plot Demo Well-Grounded?"
type: analysis
created: 2026-07-04
updated: 2026-07-04
sources: [krzywinski-2012, nollenburg-2023-computing-hive-plots, chen-2018-same-stats, newman-2002-assortative, matejka-2017-datasaurus, maslov-sneppen-2002, fosdick-2018-configuration, behrisch-2016-matrix-reordering, okoe-2017-nodelink-matrix, liljeros-2001-sexual-contacts]
tags: [hive-plot, hiveplotlib, same-stats-different-graphs, datasaurus, assortativity, degree-preserving-rewiring, network-visualization, joss, research-run]
---

# Same Stats, Different Graphs: Is the Hive-Plot Demo Well-Grounded?

A bounded literature-mode research run (2026-07-04) against the "Same Stats, Different Graphs" demo under development in `hiveplotlib-datasaurus` (working plan: `wiki/wiki/plans/same-stats-different-graphs.md`). The demo builds networks that share an identical degree sequence (so identical n, m, density, and degree histogram) but differ in higher-order structure, produced by degree-preserving double-edge swaps, and shows them as hive plots on a degree-rank axis. The run asked one question: is the demo well-grounded and honestly framed? Prior art attributed right, network-science claims correct, and "hive plots distinguish what summary statistics and force-directed layouts cannot" defensible or an overreach?

Verdict: **validated finding.** The foundations are solid and the demo is a novel synthesis, with one honest caveat about the comparison framing and two small fixes. The literature run established grounding and novelty; a follow-up data-validity run (2026-07-04, recorded below) then executed the generator and inspected the renders, confirming the matched-statistics invariants and the measured values. The one caveat is a framing point, not a data question.

## What was established

**The network-science foundations are correct.**

- Newman's sign pattern holds: social networks tend to be degree-assortative, technological and biological networks degree-disassortative. The assortativity coefficient is a Pearson correlation of the degrees at the two ends of an edge, on [-1, 1]. (Newman 2002/2003.)
- A double-edge swap provably preserves every node's degree, hence the full degree sequence, and with it n, m, density, and the degree histogram. It is the standard degree-preserving rewiring (Maslov & Sneppen 2002; networkx `double_edge_swap`), and the configuration model is the standard fixed-degree null (Fosdick et al. 2018). The "identical statistics" claim is exact for simple graphs.

**The demo is a novel synthesis, not a reinvention.**

- The cited analog, Chen et al. 2018 ("Same Stats, Different Graphs" for graphs), holds roughly five to ten *aggregate* statistics inside ranges (average path length, density, clustering, degree assortativity, connectivity), draws with a spring layout, and never mentions hive plots (verified by direct text extraction: the word "hive" appears zero times, and "degree sequence" never appears). It does not hold the degree sequence fixed.
- The Datasaurus Dozen (Matejka & Fitzmaurice 2017) is the scatter-data ancestor.
- The bare scientific fact "same degree distribution, different structure" is published as structural analysis (Grisi-Filho 2013; PNAS 2019), but never as a hive-plot teaching artifact and via different generators.
- The novel combination is one *fixed degree sequence* + degree-preserving swaps + a hive plot on a degree-rank axis. Holding the full degree sequence identical is a stronger construction than Chen et al.'s aggregate-statistic ranges. Framed as "searched, not found," not "provably first."

**The visualization framing is Krzywinski's own.**

- Hive plots were introduced as a "rational" alternative whose node coordinates are "typically derived from the absolute or rank-ordered value of a node parameter, such as connectivity" (degree). Force-based and spectral layouts are critiqued as lacking "reproducibility and perceptual uniformity because they do not use a node coordinate system," the "hairball." (Krzywinski et al. 2012.) Degree-rank on the axis is canonical, not a demo invention (Nöllenburg & Wallinger 2024).

## What was validated, and how

Every load-bearing claim was re-read by an independent voucher (the claim-maker was never its own voucher), default-refuted:

- Chen et al.'s aggregate-stats-and-spring-layout, no-hive distinction: confirmed by direct `pdftotext` + `grep` of the paper.
- The Krzywinski "rank-ordered value of a node parameter, such as connectivity" quote: confirmed verbatim (exact wording is "*typically* derived," a small but real hedge: degree is the standard axis, not the only one).
- Matrix reordering also exposing block/community structure: confirmed from the Behrisch et al. 2016 primary PDF.
- The sexual-contact correction (below): confirmed from Newman's primary tables.

A cold convergence-gate adversary then validated all four findings against a failure-mode rubric (structure-is-artifact, already-known, n-too-small, uncontrolled-comparison, grounding). The load-bearing verdict: the run **owns** the uncontrolled-comparison failure mode rather than committing it (next section).

## The one honest caveat: the comparison is partly uncontrolled

The strongest fair critique of "hive plots win where force-directed fails": the discriminating power belongs to the **degree-rank ordering**, not the hive glyph. The hive plot is *handed* the degree metadata (its axis) that the force-directed foil is never given, so it is not a like-for-like comparison. Any method that orders nodes by degree exposes assortativity, since assortativity *is* a degree-degree correlation; a reordered adjacency matrix reveals the same block/community structure (Behrisch et al. 2016).

What survives the attack:

- The mechanism is untouched, and "summary statistics can't tell them apart" is definitional (the shared degree sequence pins every degree-based statistic).
- The hive plot's advantage over force layouts in *reproducibility and perceptual uniformity* is real, and is Krzywinski's own claim.

The honest reframe the demo can stand behind: **a degree-rank ordering is what makes the differences pop, and the hive plot is a clean, reproducible, perceptually-uniform way to apply one at network scale.** The notebook already concedes the mechanism ("placement now encodes a node property; that single concession is the whole idea"); one added sentence noting that a reordered adjacency matrix would also expose it preempts the reviewer and costs nothing. No controlled hive-vs-force user study exists, so keep the comparison illustrative, never a measured win.

## Small fixes for the demo

- **`ANCHORS.md` disassortative row:** "sexual-contact networks" is a mis-anchor for degree-disassortative. Those networks are known for a scale-free *degree distribution* (Liljeros et al. 2001) and for disassortativity by *gender* (an attribute), not by degree; Newman's degree table has no such row. The shipping notebook already uses "predator-prey," so this is a one-line cleanup of the working doc, not a notebook error. (Fixed 2026-07-04.)
- **The novelty pitch for JOSS:** lead with "we hold the *full degree sequence* identical," which is stronger and cleaner than Chen et al.'s aggregate-statistic ranges, and cite Chen 2018 plus the Datasaurus as ancestors.

## What the data-validity run confirmed (2026-07-04)

The deferred data-validity mode was run: the generator was executed at the shipped scale (`seed=42`) and the story figures inspected. Every claim the literature run had to quarantine now holds.

- **The matched-statistics invariant is exact.** All six variants share n=1000, m=3068, density=0.00614, the identical sorted degree sequence and degree histogram, and the identical degree at *every individual node*, so the degree-rank axis positions are pixel-identical across variants. The baked `degree_rank` / `degree_group` / `community` attributes are identical across variants, and generation is deterministic under the seed.
- **The measured structure matches the story.** Degree assortativity: assortative +0.86, disassortative -0.50, stripes -0.26, random and community both near 0; the community variant has 93% within-community edges versus about 33% for the rest. (core_periphery measures -0.50, essentially identical to disassortative, which is exactly why it was cut from the story.)
- **The renders behave as claimed.** Spring layout collapses three of the four ladder variants into indistinguishable blobs and unrolls only the assortative one into a comet; re-ordering the same circle by degree already separates the variants (confirming the ordering, not the glyph, does the work); the hive plots give each variant a distinct signature on the shared scaffold; and the community variant is invisible under the degree partition but lights up under the community partition.

So the demo is validated end to end: grounded in the literature, novel as a synthesis, and empirically exactly what it claims. The one remaining caveat is not a data question but a framing one (the comparison hands the hive plot the degree axis; see above).

## What's open

- Whether to add the reordered adjacency matrix as an explicit "other degree-ordered method" beat in the notebook, or leave it as the one-sentence acknowledgment now drafted into Beat 3 of the draft notebook.
- Whether any variant beyond the Newman-anchored triad (assortative, disassortative, community) earns a slot in the JOSS headline figure; the working plan already leans this way, with stripes and the cut core-periphery as the swing cases.

## Sources

Compact; provenance carried by the run's per-claim quotes.

- Krzywinski, Birol, Jones, Marra 2012, "Hive plots—rational approach to visualizing networks," Brief. Bioinform. 13(5):627-644. [[krzywinski-2012]]
- Nöllenburg & Wallinger 2024, "Computing Hive Plots: A Combinatorial Framework," JGAA 28(2). [[nollenburg-2023-computing-hive-plots]]
- Chen, Soni, Lu, Maciejewski, Kobourov 2018, "Same Stats, Different Graphs," Graph Drawing 2018, arXiv:1808.09913.
- Matejka & Fitzmaurice 2017, "Same Stats, Different Graphs" (Datasaurus Dozen), CHI 2017.
- Newman 2002, "Assortative Mixing in Networks," PRL 89, 208701; 2003, "Mixing patterns in networks," PRE 67, 026126.
- Maslov & Sneppen 2002, Science 296:910-913 (degree-preserving rewiring).
- Fosdick, Larremore, Nishimura, Ugander 2018, "Configuring Random Graph Models with Fixed Degree Sequences," SIAM Review 60(2).
- Behrisch et al. 2016, "Matrix Reordering Methods for Table and Network Visualization," Comput. Graph. Forum 35(3).
- Okoe, Jianu, Kobourov 2017, "Revisited Experimental Comparison of Node-Link and Matrix Representations," GD 2017 (matrices vs node-link; a narrow cluster-counting edge, not a blanket win).
- Liljeros, Edling, Amaral, Stanley, Åberg 2001, "The web of human sexual contacts," Nature 411:907-908 (the mis-anchor correction).

## See Also

- [[hive-plot]] — the method
- [[force-directed-layout]] — the foil the demo sets up
- [[node-assignment]] — degree-rank sorting is the axis choice under discussion
- [[structural-heterogeneity]] — conceptual kin (same degree, different structure)
- [[hiveplotlib]] — the library the demo ships into
- [[Martin Krzywinski]] — "rational layout" and the hairball critique

---
*Produced by a bounded literature-mode research run (a four-lens shallow panel: prior-art, counter-evidence, network-science grounding, visualization methods; four independent verify vouchers; one cold convergence-gate adversary), followed by a data-validity run that executed the generator and inspected the renders. 2026-07-04.*
