# Plan: JOSS submission (direct route)

## Goal

Hiveplotlib gets a peer-reviewed, citable publication: a JOSS paper (Journal of Open Source Software) with a DOI, a Zenodo-archived tagged release, and the repo-hygiene artifacts reviewers check (open contribution policy, code of conduct, citation file). Route is JOSS direct; netgraph (10.21105/joss.05372, July 2023, single author) is the precedent. pyOpenSci is explicitly rejected (the ~2-year maintenance pledge and community framing fit a solo maintainer poorly; GitLab hosting there is unconfirmed). Realistic end-to-end timeline once submitted: 2-6 months.

## Settled decisions (do not re-litigate)

1. **JOSS direct, no pyOpenSci.** Gary's call.
2. **Two external gates before submission** (modeled below like the WASM-explorer plan): v0.28 release, and resolution of the same-stats-different-graphs demo (expected paper figure source).
3. **Gary can rewrite CONTRIBUTING.md unilaterally.** No GDA sign-off gate.

## Alignment (grill)

```
Not yet run — recommended before dispatch for major plans. Run the grill-me skill
or knowingly skip; record each wave below. Route any resulting plan change to
amend-plan (rule 14).
```

## Prior ADRs / design docs

No ADRs exist yet; no prior publication/citation thinking in the wiki. Net-new design space. Feeder pages for the paper:

- `wiki/wiki/entities/hiveplotlib.md` — canonical framing ("most comprehensive open-source implementation of Krzywinski's hive plot method"); Statement of need.
- `wiki/wiki/sources/krzywinski-2012.md` — foundational citation (DOI 10.1093/bib/bbr069); "If the layout shows a pattern, you can be sure it is due to structure in the underlying data..." quote.
- `wiki/wiki/concepts/force-directed-layout.md` — problem-statement table + Bostock quote; Statement of need.
- `wiki/wiki/concepts/hive-plot.md` — five key properties + competing-implementations table (needs a maintenance-status column added; see Workstream B).
- `wiki/wiki/sources/perez-2021-hype.md` — HyPE (Bioinformatics 2021), closest published comparator; HyPE-vs-HivePlotMatrix table ready; State of the field.
- `wiki/wiki/sources/nollenburg-2023-computing-hive-plots.md` — NP-completeness result validates the exploration-over-optimization design philosophy; Software design section.
- `wiki/wiki/overview.md` — best single-page synthesis.
- Applications concept pages (bioinformatics, cybersecurity, software-engineering) — Research impact statement.
- `wiki/wiki/plans/same-stats-different-graphs.md` — Datasaurus-style demo; **the work behind this plan's G2 and the expected hero figure — it must complete first** (its own sequencing: story standardization in hiveplotlib-datasaurus, then generator + notebook in hiveplotlib on a fresh branch post-46-merge). Prior art: Chen et al. GD 2018, arXiv:1808.09913.

## JOSS requirements (verified June 2026)

- **Hosting/license:** GitLab hosting is fine (precedents: cRacklet, petitRADTRANS). BSD-3-Clause is fine. Minor polish: `LICENSE` isn't labeled BSD-3-Clause and `pyproject.toml:16` uses `license = {file = "LICENSE"}`; an SPDX identifier helps reviewers.
- **Substantiality:** easily met (first commit 2020-09-02, 326 commits, ~17.7k source lines, tags through v0.27.0, 100% coverage CI, RTD docs). Single author acceptable per the netgraph precedent, but 2026 guidance wants stated support/governance expectations for single-author projects.
- **Paper artifacts:** `paper.md` (750-1750 words, YAML front matter) + `paper.bib` in the repo. Required sections per 2026 criteria: Summary (non-specialist), Statement of need, State of the field (why alternatives are insufficient), Software design (architecture trade-offs), Research impact statement (evidence of research use or credible near-term significance), AI usage disclosure (mandatory: tools, scope, confirmation of human review — significant here given the agent-harness workflow; wording drafted deliberately and reviewed by Gary). Figures via Markdown image syntax with captions. Compile via the openjournals/inara Docker image or the Open Journals GitHub Action; during review, `@editorialbot generate pdf`.
- **At acceptance:** tagged release + Zenodo archive with DOI; archive metadata must match the paper.
- **Review mechanics:** pre-review issue on openjournals/joss-reviews (GitHub), editor assigned; reviewer checklist covers installation, functionality, statement of need, example usage, API docs, automated tests, community guidelines, license, version tags.

## Repo blockers (verified in working tree)

- `CONTRIBUTING.md:3-8` restricts developers to GDA employees — a desk-reject trigger under the community-guidelines checklist item. Rewrite authorized.
- No `CODE_OF_CONDUCT.md`.
- No `CITATION.cff` (recommended; pairs with the Zenodo step). README has no citation section either.
- No nxviz comparison anywhere in repo or wiki; reviewers will expect one in State of the field (nxviz is the only living alternative; its old OO `HivePlot` class is deprecated, the functional API does hive plots among other layouts). Net-new research.
- Research impact statement needs evidence gathering: citations of hiveplotlib in papers, downstream users, download stats (~1,787/month PyPI per pypistats, June 2026).
- No author/affiliation/ORCID/funding facts recorded anywhere; must come from Gary directly.

## Patterns this replaces

- GDA-employees-only contribution policy at `CONTRIBUTING.md:5-8`. Replace with an open external-contribution policy. Sweep: `README.md:72` and `docs/source/README.md:72` link to CONTRIBUTING.md (keep the two READMEs in sync; `docs/source/README.md` mirrors the root README); confirm the rewritten file still matches how those links describe it. No other repo or wiki content states the GDA restriction (verified by grep).

## Default justifications

No new defaults (no code-level API change).

## Naming audit

- **Paper location:** JOSS accepts `paper.md` at the repo root or in a subdirectory. Use `paper/paper.md` + `paper/paper.bib` (+ `paper/figures/`) to keep the root clean; this is the common community layout and what the inara action defaults handle.
- **Community files:** `CODE_OF_CONDUCT.md` and `CITATION.cff` at repo root — ecosystem-standard names GitHub/GitLab and JOSS tooling auto-detect. Do not rename.
- Prose-only terms: "Statement of need," "State of the field," "Research impact statement" — JOSS's own section vocabulary; use verbatim as paper headings.

## API usage examples

No API surface change. (Skipping all three subsections.)

## Notebook review

No notebook change.

## External gates

- **G1: v0.28 released.** Currently `0.28.0a0` (NetworkX first-class I/O + graph_features metrics). The paper describes and the Zenodo archive snapshots the released library; submission waits for the release tag. Blocks Workstream D (and final figure/code claims in C).
- **G2: same-stats-different-graphs demo resolved.** Gated on `wiki/wiki/plans/same-stats-different-graphs.md` completing (per its Amendment 4: WS-0 story standardization in hiveplotlib-datasaurus, then generator + tutorial notebook in hiveplotlib); that work must finish before this gate clears, and its outcome supplies the expected hero figure. Blocks figure selection in Workstream C. If that plan dies, fall back to an alternative figure (route through amend-plan).
- **G3: author metadata from Gary.** Legal name, affiliation(s), ORCID, funding acknowledgments. Nothing in repo or wiki records these. Blocks paper front matter in Workstream C; collect via Workstream B's checklist.

## Workstreams

### Workstream A: Community and governance artifacts

**Status:** not started
**Files:** `CONTRIBUTING.md`, `CODE_OF_CONDUCT.md` (new), `CITATION.cff` (new), `LICENSE`, `pyproject.toml`, `README.md`, `docs/source/README.md`, `CHANGELOG`
**Sequencing (2026-06-10):** the README edits here land *after* `docs-cheat-sheet-and-readme.md` WS-B (that plan gates only on branch 46 and ships first; this workstream builds on, and may reuse as statement-of-need prose, its leading text — see that plan's Amendment 3).
**Done when:**
- CONTRIBUTING.md welcomes external contributions (GDA restriction gone), keeps the existing dev-workflow content (Makefile targets, testing, lint/type/docs), and adds issue-first guidance for external MRs.
- CODE_OF_CONDUCT.md exists (Contributor Covenant is the standard pick; Gary as contact).
- A short support/governance statement exists (in CONTRIBUTING.md or README): single-maintainer project, how to get help, response expectations. Satisfies the 2026 single-author guidance.
- CITATION.cff exists and validates (`cffconvert --validate` or equivalent); README gains a "Citing" section (mirrored in `docs/source/README.md`). Placeholder DOI noted as pending until Zenodo wiring in Workstream D.
- LICENSE labeled BSD-3-Clause; `pyproject.toml` carries `license = "BSD-3-Clause"` (SPDX string) with the classifier consistent. `make test`, `make docs` still pass.
- CHANGELOG: one entry for the community-artifact additions (repo-visible change for released users); no entries for paper drafts.
- Prose in all shipped files passes the human-voice rules (no em-dashes, no AI filler).

### Workstream B: Evidence gathering

**Status:** not started
**Files:** wiki: new `wiki/wiki/analyses/nxviz-comparison.md`, new `wiki/wiki/analyses/hiveplotlib-research-impact.md`, update `wiki/wiki/concepts/hive-plot.md` (maintenance-status column)
**Done when:**
- nxviz comparison filed: current API state (deprecated OO `HivePlot`, functional API scope), maintenance status, feature gaps vs hiveplotlib — enough specifics to write the State of the field paragraph.
- Research-impact evidence filed: papers citing or using hiveplotlib (Google Scholar / Semantic Scholar sweep), known downstream users, download stats with date, with honest assessment of strength (JOSS 2026 accepts "credible near-term significance" if direct citations are thin).
- Competing-implementations table in `hive-plot.md` gains a maintenance-status column.
- Author-metadata checklist (name, affiliation, ORCID, funding) presented to Gary; answers recorded for Workstream C (clears G3).
- Wiki index.md and log.md updated per wiki schema.

### Workstream C: Paper authoring

**Status:** not started (front matter gated on G3; figures gated on G2; final claims gated on G1)
**Files:** `paper/paper.md` (new), `paper/paper.bib` (new), `paper/figures/` (new)
**Done when:**
- paper.md is 750-1750 words with all required sections: Summary, Statement of need, State of the field (incl. nxviz and HyPE from Workstream B), Software design (backend-agnostic architecture, exploration-over-optimization rationale citing Nöllenburg 2023), Research impact statement, AI usage disclosure.
- paper.bib covers every citation (Krzywinski 2012, Perez 2021, Nöllenburg 2023, Chen 2018 if the Datasaurus figure lands, nxviz, backend libraries as appropriate).
- Figures: hero figure from the same-stats demo (per G2) plus, if word count allows, one architecture/backend figure; each with a real caption via Markdown image syntax.
- AI usage disclosure wording drafted as a distinct, flagged item; **gated on Gary's explicit review** before the draft is called done — it must describe the agent-harness workflow and human review accurately.
- All prose follows the human-voice rules; the full paper.md draft is **gated on Gary's review** (this is Gary-facing shipped writing under his name).
- No CHANGELOG entry (paper is not released library behavior).

### Workstream D: Submission mechanics and review loop

**Status:** not started (gated on G1; needs A-C complete)
**Files:** `paper/` (compile fixes only), possibly `.gitlab-ci.yml` or a local script for the inara compile check, `CITATION.cff` (DOI update)
**Done when:**
- paper.md compiles cleanly through the openjournals/inara Docker image locally (repo is on GitLab, so the GitHub Action isn't the default path); compiled PDF reviewed by Gary.
- Zenodo wired: v0.28 tagged release archived with DOI; archive metadata (title, authors, license, version) matches the paper; CITATION.cff and README citation section updated with the real DOI. (Zenodo's GitLab integration or manual upload; record which in the Implementation log.)
- Submission filed at joss.theoj.org; pre-review issue opens on openjournals/joss-reviews (GitHub account: Gary's).
- Review-response loop: reviewer asks route through this plan via amend-plan (new workstreams for substantive change requests; in-scope tweaks for paper edits). Workstream closes at acceptance and final DOI registration.

## Plan amendments

```
None yet. The Orchestrator will populate this section in amend-plan mode if
emergent work surfaces (rule 14 trigger).
```

## Holdouts

- `wiki/wiki/plans/conda-forge-distribution.md:192`: references the old CONTRIBUTING line numbers; historical plan record, do not edit.

## Implementation log

Append-only.
