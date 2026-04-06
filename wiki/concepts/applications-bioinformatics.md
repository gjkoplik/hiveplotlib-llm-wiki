---
title: "Applications: Bioinformatics"
type: concept
created: 2026-04-06
updated: 2026-04-06
sources: [krzywinski-2012, perez-2021-hype]
tags: [applications, bioinformatics, genomics, microbiome]
---

# Applications: Bioinformatics

Bioinformatics is the **strongest adoption domain** for [[hive-plot|hive plots]], which is unsurprising given that the method originated at a genome sciences center.

## Key Applications

### Gene Regulatory Networks
[[krzywinski-2012]] demonstrated hive plots on the **RegulonDB** E. coli gene regulatory network (1,584 genes). The source/manager/sink axis assignment revealed regulatory hierarchy patterns invisible in [[force-directed-layout|force-directed layouts]].

### Gene-Disease Networks
[[krzywinski-2012]] also applied hive plots to cancer gene-disease networks (258 genes, 3,057 edges), revealing bridge genes connecting cancer to other disease systems.

### Microbial Co-occurrence Networks
[[perez-2021-hype|HyPE (2021)]] was demonstrated on a forest soil microbiome network (1,880 OTUs, 13,605 edges), revealing community modules partitioned by harvesting treatment and soil depth.

### Protein Interaction Networks
**HiveGraph** (Wodak Lab, Hospital for Sick Children, Toronto) — a web application for hive plots of protein-protein, protein-DNA, and protein-ligand interaction networks.

### Genotype-Phenotype Pleiotropy
Seoane et al. (2014, *PLOS Computational Biology*) used hive plots to visualize gene-phenotype association rules in a cohort of 4,286 women, identifying novel pleiotropic associations including NSF with triglyceride levels.

## Why Hive Plots Work Well Here

- Biological networks have natural node categories (regulators/targets, genes/diseases, etc.) that map to axes
- Structural metrics (degree, clustering coefficient) have biological interpretations
- Reproducibility is critical for scientific publication
- Networks are often large enough that force-directed layouts fail

## See Also

- [[hive-plot]] — The visualization method
- [[krzywinski-2012]] — Foundational bioinformatics examples
- [[applications-cybersecurity]] — Another strong domain
- [[applications-software-engineering]] — Another application domain
