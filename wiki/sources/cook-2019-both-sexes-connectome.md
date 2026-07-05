---
title: "Cook et al. 2019 — Whole-Animal Connectomes of Both C. elegans Sexes"
type: source
created: 2026-07-04
updated: 2026-07-04
sources: [cook-2019-both-sexes-connectome]
tags: [connectome, c-elegans, network-data, dimorphism, wormwiring]
---

# Cook et al. 2019 — Whole-Animal Connectomes of Both C. elegans Sexes

**Citation:** Cook, S.J., Jarrell, T.A., Brittin, C.A., et al., 2019. Whole-animal connectomes of both Caenorhabditis elegans sexes. *Nature*, 571, pp.63–71. DOI: 10.1038/s41586-019-1352-7. (WormWiring.)

## Summary

The definitive reconstruction of both *C. elegans* sexes as whole-animal connectomes (WormWiring), and the source of the settled functional classification (sensory / interneuron / motor) used to place neurons on axes. Its canonical dimorphism figure draws the **summed** hermaphrodite+male network on a **shared** layout, with edges color-coded by which sex dominates (green = hermaphrodite-stronger, blue = male-stronger), using an anatomical anteroposterior horizontal axis and a sensory -> interneuron -> motor vertical flow. That figure is the field's established way of showing the sex difference in one picture.

## Bearing on the bioinformatics-examples record

Two roles. First, this is the underlying connectivity science for all the hiveplotlib C. elegans work (the Cook class lists are the axis assignment). Second, its dimorphism figure is the **benchmark** the hiveplotlib `sex_differential` / `sex_panels` figures are measured against: they are a radial hive-plot re-encoding of Cook's validated Cartesian layout, not a new scientific reading and not merely a naive two-panel density plot. Benchmark against Cook, not against the naive alternative. See [[hiveplotlib-bioinformatics-examples]].

## See Also

- [[hiveplotlib-bioinformatics-examples]] — The exploration that re-encodes this figure radially
- [[tabacof-2013-connectome-hive-plots]] — The first C. elegans connectome hive plot
- [[hive-plot-matrix]] — The two-panel unified-axes construct used for the sex comparison
- [[differential-hive-plot]] — The comparison technique the sex figures emulate
