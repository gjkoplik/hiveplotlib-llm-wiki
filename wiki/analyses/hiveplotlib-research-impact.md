---
title: hiveplotlib Research Impact — Evidence
type: analysis
created: 2026-06-12
updated: 2026-10-31
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

**Re-checked 2026-09-12 (still v0.28.0 on PyPI):** trailing windows 49 / 144 / 1,358. Monthly totals from pypistats' daily series (without mirrors) held steady through the summer: April 1,668, May 1,667, June 1,930, July 1,759, August 1,827. The dip is recent. The daily median was 54 from 2026-08-01 to 08-24 and 26.5 from 08-25 to 09-11, and September's first 11 days total 303. The series is spiky (single days swing from 3 to 187), so three weeks is a watch, not a trend. It matters for the JOSS statement only if it persists: the "~1,700/month" figure below is accurate through August and should be re-pulled before submission rather than quoted.

**Re-checked 2026-10-03 (still v0.28.0 on PyPI): the dip held for a full month.** September's total is 1,088 without mirrors, against 1,759 in July and 1,827 in August, roughly a 40% drop; trailing windows read 47 / 425 / 930. The daily median since 2026-09-12 is 34, a little above the 26.5 of late August but well under the 54 of early August. A month is long enough to stop calling this noise. The cause is not visible from here (no release since v0.28.0, and pypistats does not break out installers by source), but the JOSS statement's "~1,700/month" is now stale and must be re-pulled, not quoted.

**Re-checked 2026-10-10 (still v0.28.0 on PyPI): early October is running slightly above September.** October's first nine days total 371 without mirrors, a pace of roughly 1,280 for the month, against September's 1,088; trailing windows read 46 / 214 / 1,173. Nine days is too short to call a recovery. **New downstream signal: Fedora packaged hiveplotlib 0.28.0** (`python3-hiveplotlib`, Fedora 44, 45, and Rawhide, package created 2026-09-30 by an independent Fedora packager). A distribution picking up the library unprompted is citable evidence of use for the JOSS statement, and Fedora installs do not show up in pypistats, so it also slightly understates the PyPI numbers as a measure of reach.

**Re-checked 2026-10-17 (still v0.28.0 on PyPI): October is tracking back toward the summer pace, carried by one day.** The first 16 days total 889 without mirrors, a pace of roughly 1,720 for the month, but 245 of those landed on 2026-10-13 alone against a daily median in the forties; without that day the pace is about 1,330, still above September's 1,088. Trailing windows read 23 / 518 / 1,506. pypistats cannot say what the spike was (a CI matrix, a mirror that slipped the filter, or real installs all look the same), so one day does not reverse September's reading. The "~1,700/month" figure stays something to re-pull at submission rather than quote.

**Re-checked 2026-10-24 (still v0.28.0 on PyPI): with the spike diluted, October settles a little above September.** The first 23 days total 1,149 without mirrors, a pace of roughly 1,550 for the month; without the 245 on 2026-10-13 the pace is about 1,270, close to last week's spike-free estimate and still above September's 1,088. Trailing windows read 70 / 260 / 1,173. The "~1,700/month" figure stays flagged for a re-pull at submission.

**Re-checked 2026-10-31 (still v0.28.0 on PyPI; 0.28.1 prepared, unreleased): October has passed September and is back near the summer pace.** The first 30 days total 1,758 without mirrors, a pace of roughly 1,820 for the month; without the 245 on 2026-10-13 it is about 1,560, against September's 1,088, July's 1,759, and August's 1,827. The last seven days total 609 (2026-10-28 reached 158 and 2026-10-30 reached 145, against a mid-month daily median in the forties). pypistats cannot say why. The API's `last_month` field reads 1,173, which does not match the daily series, so this note uses the series. Wait for the full month before restoring the JOSS "~1,700/month" figure.

## Open gaps (harden before submission)

- **Authenticated GitHub/GitLab code search** for `import hiveplotlib` not yet run — downstream *code* usage is unquantified and likely understates real adoption.
- **Semantic Scholar** sweep unretrieved (rate-limited).

## Assessment

Thin-but-real-and-improving. Two genuine 2025 peer-reviewed bio uses (pending Gary's full-text confirmation), an IEEE reference, real PyPI downloads (~1,700/month through August, about 1,100 in September, October back near the summer pace with the month nearly complete; see the re-checks above), a Fedora package since 2026-09-30, and an active competitor landscape ([[nxviz-comparison|nxviz]] revived June 2026) that hiveplotlib out-features on control and scale. Per JOSS's 2026 criteria, the paper should frame impact around **credible near-term significance** and the **capability gap** (polar parallel coordinates, datashader-scale rendering, fine-grained edge control) rather than raw citation count.

## See Also

- [[hiveplotlib]]
- [[nxviz-comparison]] — companion State-of-the-field evidence
- [[p2cp]] — a core capability-gap argument
- [[joss-ai-disclosure-precedents]] — companion JOSS-evidence page
