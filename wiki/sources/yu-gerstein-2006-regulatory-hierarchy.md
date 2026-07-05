---
title: "Yu & Gerstein 2006 — Hierarchical Structure of Regulatory Networks"
type: source
created: 2026-07-04
updated: 2026-07-04
sources: [yu-gerstein-2006-regulatory-hierarchy]
tags: [regulatory-networks, network-hierarchy, transcription-factors, node-roles]
---

# Yu & Gerstein 2006 — Hierarchical Structure of Regulatory Networks

**Citation:** Yu, H. and Gerstein, M., 2006. Genomic analysis of the hierarchical structure of regulatory networks. *Proceedings of the National Academy of Sciences (PNAS)*, 103(40), pp.14724–14731. DOI: 10.1073/pnas.0508637103.

## Summary

Analyzes directed transcriptional regulatory networks as hierarchies and coins the three-tier role vocabulary: **master regulators** (top, out-edges only), **middle managers** (intermediate), and **workhorses** (bottom, in-edges only). The trichotomy is defined by position in the directed graph (a node's in-/out-edge profile), giving a categorical, near-complete partition of a regulatory network's nodes by their control role.

## Bearing on the bioinformatics-examples record

This is the **origin** of the regulator / manager / workhorse vocabulary that the hiveplotlib E. coli GRN example uses, and it corrects a misattribution: the scheme is Yu & Gerstein's, not [[martin-krzywinski|Krzywinski]]'s. Krzywinski imports it (his ref [63]) and *applies* it to RegulonDB as a hive plot, writing verbatim: "We use the terminology from ref. [63] to refer to sources (out edges only) as 'regulators', sinks (in edges only) as 'workhorses' and the other nodes as 'managers'." Honest attribution: partition vocabulary = Yu & Gerstein 2006; hive-plot application = [[krzywinski-2012|Krzywinski 2012]]. This source also grounds the repo's pending role-label fix (the middle role is "managers," not "dual"; "dual" collides with RegulonDB's separate sign classification). See [[hiveplotlib-bioinformatics-examples]].

## See Also

- [[hiveplotlib-bioinformatics-examples]] — The exploration whose GRN role vocabulary this source originates
- [[krzywinski-2012]] — Imported this scheme and applied it to RegulonDB as a hive plot
- [[martin-krzywinski]] — Applied, but did not coin, the role vocabulary
- [[hive-plot]] — The visualization the role partition feeds
