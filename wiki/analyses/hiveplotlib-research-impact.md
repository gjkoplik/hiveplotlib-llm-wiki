---
title: hiveplotlib Research Impact — Evidence
type: analysis
created: 2026-06-12
updated: 2026-06-12
sources: [research-impact-web-sweep-2026-06-12]
tags: [hiveplotlib, joss, research-impact, citations, downloads]
---

# hiveplotlib Research Impact — Evidence

Evidence for the JOSS paper's **Research impact statement**. Honest summary: direct citations are **thin but real and improving**. JOSS's January-2026 criteria accept "credible near-term significance" / design-and-impact framing over raw citation counts, so the paper should lean on the capability-gap argument (P2CPs, datashader-scale, fine edge control) backed by the genuine downstream uses below. All findings accessed 2026-06-12.

## Direct citations / downstream uses

**Most credible genuine downstream uses** — two 2025 peer-reviewed biology papers:

- **Shi et al. (2025)** — "Sex-biased transcriptome in in vitro produced bovine early embryos," *Cell & Bioscience* (Springer), 2025.
- **Wittmer et al. (2025)** — "Rational design of induced regeneration via somatic embryogenesis...," *The Plant Cell* (Oxford UP), 2025.

> **Maintainer-confirmation flag.** Both papers are **paywalled and UNVERIFIED at full text.** Do **not** cite either as a confirmed hiveplotlib user until Gary confirms the actual usage. Flagged for maintainer confirmation before citing.

**Other mentions:**

- **Cong (IEEE)** — references hiveplotlib as "the open-source python package" for hive plots.
- **Korob (2022)** — "Text Network Analysis," a *Towards Data Science* piece, mentions hiveplotlib.

**Coverage:** the maintainer's own "Introducing Hiveplotlib" TDS/Medium article, 2020-09-30.

## Search sweep results

- **Google Scholar** "hiveplotlib": **~5 results** (the items above are the credible subset).
- **OpenAlex:** ~0 real hits.
- **Semantic Scholar:** rate-limited, **not retrieved** (open gap).
- **arXiv 2101.00863** and **2309.02273:** do **NOT** mention hiveplotlib (checked, negative).
- **NCBI PMC7887807:** verified **FALSE POSITIVE** — recorded here so it is not re-cited.

## Download stats

From pypistats.org, accessed 2026-06-12, package **v0.27.0** (trailing windows):

| Window | Downloads |
|--------|-----------|
| Last day | 79 |
| Last week | 466 |
| Last month | 1,743 |

## Open gaps (harden before submission)

- **Authenticated GitHub/GitLab code search** for `import hiveplotlib` not yet run — downstream *code* usage is unquantified and likely understates real adoption.
- **Semantic Scholar** sweep unretrieved (rate-limited).

## Assessment

Thin-but-real-and-improving. Two genuine 2025 peer-reviewed bio uses (pending Gary's full-text confirmation), an IEEE reference, real and steady PyPI downloads (~1,700/month), and an active competitor landscape ([[nxviz-comparison|nxviz]] revived June 2026) that hiveplotlib out-features on control and scale. Per JOSS's 2026 criteria, the paper should frame impact around **credible near-term significance** and the **capability gap** (polar parallel coordinates, datashader-scale rendering, fine-grained edge control) rather than raw citation count.

## See Also

- [[hiveplotlib]]
- [[nxviz-comparison]] — companion State-of-the-field evidence
- [[p2cp]] — a core capability-gap argument
- [[joss-ai-disclosure-precedents]] — companion JOSS-evidence page
