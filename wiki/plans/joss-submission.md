# Plan: JOSS submission (direct route)

## Goal

Hiveplotlib gets a peer-reviewed, citable publication: a JOSS paper (Journal of Open Source Software) with a DOI, a Zenodo-archived tagged release, and the repo-hygiene artifacts reviewers check (open contribution policy, code of conduct, citation file). Route is JOSS direct; netgraph (10.21105/joss.05372, July 2023, single author) is the precedent. pyOpenSci is explicitly rejected (the ~2-year maintenance pledge and community framing fit a solo maintainer poorly; GitLab hosting there is unconfirmed). Realistic end-to-end timeline once submitted: 2-6 months.

## Settled decisions (do not re-litigate)

1. **JOSS direct, no pyOpenSci.** Gary's call.
2. **Hard pre-submission gates: v0.28 release (G1), author metadata (G3), GitLab namespace fate (G4).** The same-stats demo (G2) is preferred-with-fallback, not blocking (Amendment 3).
3. **Gary can rewrite CONTRIBUTING.md unilaterally.** No GDA sign-off gate.

## Alignment (grill)

### Maintainer shared-understanding pass (grill), Wave 1 — byline, AI disclosure, claims ceiling, G2 hardness, preliminary dispatch (2026-06-12)

**Confirmed positions:**

- **Sole author, no GDA.** GDA was dissolved as of 2026-06-11. Author is Gary Koplik alone. Affiliation is expected to be the derived company, Marcell Intelligence, but the paper front matter and CITATION.cff carry a **blank placeholder** until Gary settles his formal title/affiliation. GDA references should be removed from shipped artifacts where appropriate (LICENSE copyright line is an OPEN item, see below).
- **AI disclosure posture: middle.** Disclose that as of 2025 Claude is used in implementation and documentation, with a human signing off on all code. Do not detail the agent-harness by default; Gary wants to see examples of other JOSS papers' AI disclosures before settling final wording (new evidence item for Workstream B). Final wording remains gated on Gary's review.
- **Claims ceiling confirmed:** the paper describes what v0.28 ships, plus an explicit future-work framing for aspirational items (scaling, etc.). No forward-looking capability claims stated as current.
- **G2 softened: not existential.** The same-stats (Datasaurus) figure is strongly preferred as the hero figure, but if that work stalls, fallbacks are acceptable (karate club stories, Bitcoin visualizations, hairball-vs-hive-plot comparisons). Paper drafting proceeds independently with the hero figure slot held open.
- **Preliminary dispatch confirmed:** Workstream A starts now in the `joss-submission` worktree (branched off master at 162e5b1; `~/repos/hiveplotlib-joss`) excluding the README "Citing" section (sequenced behind docs-cheat-sheet WS-B); Workstream B starts now in the wiki. C/D unchanged.

**OPEN items:**

- LICENSE copyright line says Geometric Data Analytics; whether to alter it post-dissolution is a legal call for Gary (Wave 2).
- Affiliation placeholder (Marcell Intelligence?) pending Gary's formal title decision; G3 remains partially open (ORCID, funding statement also outstanding).

**Routing:** GDA-dissolution sweep, G2 softening, B's new AI-disclosure-precedents item, and the A-scope README carve-out go to orchestrator amend-plan (bundled after Wave 2).

### Maintainer shared-understanding pass (grill), Wave 2 — GDA dissolution fallout (2026-06-12)

**Confirmed positions:**

- **GitLab namespace:** assume `gitlab.com/geomdata/hiveplotlib` stays for now; Gary is inquiring with former coworkers. **Explicit resolution required before submission** — becomes a pre-submission decision item blocking Workstream D (alongside G1).
- **LICENSE holder change authorized.** Gary's ownership of the code is clear to him (vs. GDA), and the license is permissive. Correction recorded during the grill: the license is **BSD-3-Clause, not MIT** as Gary recalled (verified in file; 2020-2024 GDA copyright line plus GDA named in the clause-3 non-endorsement text). Change: holder line becomes Gary Koplik with years extended through 2026; clause-3 entity name updated to match. License *type* stays BSD-3-Clause.
- **Contact channels:** support/help routes to the GitLab issue tracker (primary, what JOSS checks), LinkedIn optionally encouraged in the README support section; CODE_OF_CONDUCT private reporting contact is gary.koplik@gmail.com (already public as the commit author email, so no new privacy cost).

**OPEN items (external facts, not misalignment):**

- Namespace fate (Gary inquiring; pre-submission gate).
- Affiliation placeholder, ORCID, funding statement (G3).

Wave 2 surfaced no new divergence beyond the items above; grill stopped here per the stopping rule.

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
- `wiki/wiki/plans/same-stats-different-graphs.md` — Datasaurus-style demo; the work behind this plan's G2 and the preferred hero figure (preferred-with-fallback per Amendment 3; its own sequencing: story standardization in hiveplotlib-datasaurus, then generator + notebook in hiveplotlib on a fresh branch post-46-merge). Prior art: Chen et al. GD 2018, arXiv:1808.09913.

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
- **G2 (preferred, not blocking): same-stats-different-graphs hero figure.** `wiki/wiki/plans/same-stats-different-graphs.md` completing supplies the strongly preferred hero figure (per its Amendment 4: WS-0 story standardization in hiveplotlib-datasaurus, then generator + tutorial notebook in hiveplotlib). Paper drafting in C proceeds with the hero-figure slot held open. Named fallbacks if it stalls: karate club, Bitcoin ratings, hairball-vs-hive-plot comparison. Fallback selection routes through amend-plan only if exercised at submission time.
- **G3: author metadata from Gary.** Legal name, affiliation(s), ORCID, funding acknowledgments. Nothing in repo or wiki records these. Author is Gary Koplik sole author (GDA dissolved 2026-06-11); affiliation is a blank placeholder (likely "Marcell Intelligence") until Gary settles it. Front matter in C proceeds with an explicit TODO marker (Amendment 7); collect remaining facts via Workstream B's checklist.
- **G4: GitLab namespace fate.** Repo lives at `gitlab.com/geomdata/hiveplotlib` (GDA's group). Working assumption: it stays. Gary must explicitly resolve before submission (inquiring with former coworkers). Blocks Workstream D alongside G1. If the repo migrates, paper/Zenodo/CITATION URLs all follow.

## Workstreams

### Workstream A: Community and governance artifacts

**Status:** ready to start (Amendment 5)
**Files:** `CONTRIBUTING.md`, `CODE_OF_CONDUCT.md` (new), `CITATION.cff` (new), `LICENSE`, `pyproject.toml`, `README.md`, `docs/source/README.md`, `CHANGELOG`
**Execution context (2026-06-12):** runs in the `joss-submission` worktree at `~/repos/hiveplotlib-joss`, branched off master at 162e5b1. That worktree has no wiki/agent-harness submodules (they only exist on branch 46) — fine, A touches only main-repo files.
**Sequencing (2026-06-10, narrowed 2026-06-12):** only the README "Citing" section is deferred behind `docs-cheat-sheet-and-readme.md` WS-B (that plan gates only on branch 46 and ships first; the Citing edits build on its leading text — see that plan's Amendment 3). Everything else in A starts now.
**Done when:**
- CONTRIBUTING.md welcomes external contributions (GDA restriction gone), keeps the existing dev-workflow content (Makefile targets, testing, lint/type/docs), and adds issue-first guidance for external MRs.
- CODE_OF_CONDUCT.md exists (Contributor Covenant is the standard pick); private reporting contact is gary.koplik@gmail.com.
- A short support/governance statement exists (in CONTRIBUTING.md or README): single-maintainer project, how to get help, response expectations. The GitLab issue tracker is the primary help channel (what JOSS checks); LinkedIn optionally mentioned. Satisfies the 2026 single-author guidance.
- CITATION.cff exists and validates (`cffconvert --validate` or equivalent), authored by Gary Koplik alone. Placeholder DOI noted as pending until Zenodo wiring in Workstream D. README "Citing" section deferred per Amendment 5 (lands later, mirrored in `docs/source/README.md`).
- LICENSE labeled BSD-3-Clause (license type unchanged); copyright holder line changes to Gary Koplik with years extended through 2026; the clause-3 non-endorsement entity name updated to match. `pyproject.toml` carries `license = "BSD-3-Clause"` (SPDX string) with the classifier consistent. `make test`, `make docs` still pass.
- GDA-mention sweep: enumerate remaining GDA references in repo prose/docs and update where appropriate (LICENSE handled above).
- CHANGELOG: one entry for the community-artifact additions (repo-visible change for released users); no entries for paper drafts.
- Prose in all shipped files passes the human-voice rules (no em-dashes, no AI filler).

### Workstream B: Evidence gathering

**Status:** not started
**Files:** wiki: new `wiki/wiki/analyses/nxviz-comparison.md`, new `wiki/wiki/analyses/hiveplotlib-research-impact.md`, new `wiki/wiki/analyses/joss-ai-disclosure-precedents.md`, update `wiki/wiki/concepts/hive-plot.md` (maintenance-status column)
**Done when:**
- nxviz comparison filed: current API state (deprecated OO `HivePlot`, functional API scope), maintenance status, feature gaps vs hiveplotlib — enough specifics to write the State of the field paragraph.
- Research-impact evidence filed: papers citing or using hiveplotlib (Google Scholar / Semantic Scholar sweep), known downstream users, download stats with date, with honest assessment of strength (JOSS 2026 accepts "credible near-term significance" if direct citations are thin).
- Competing-implementations table in `hive-plot.md` gains a maintenance-status column.
- AI-disclosure precedents filed: 3-5 examples of AI-usage disclosures from accepted JOSS papers (2025-2026), so Gary can calibrate disclosure wording. Starting posture (his, recorded): middle — "Claude used in implementation and documentation as of 2025, human signs off on all code," no deep harness detail by default. Final wording still gated on his review in C.
- Author-metadata checklist (name, affiliation, ORCID, funding) presented to Gary; answers recorded for Workstream C (clears G3).
- Wiki index.md and log.md updated per wiki schema.

### Workstream C: Paper authoring

**Status:** not started (drafting may proceed; hero-figure slot held open per G2; affiliation TODO per Amendment 7; final claims gated on G1)
**Files:** `paper/paper.md` (new), `paper/paper.bib` (new), `paper/figures/` (new)
**Done when:**
- Front matter: Gary Koplik sole author; affiliation carries an explicit `TODO` placeholder marker (likely "Marcell Intelligence") so drafting proceeds before G3 fully clears; placeholder resolved before submission.
- paper.md is 750-1750 words with all required sections: Summary, Statement of need, State of the field (incl. nxviz and HyPE from Workstream B), Software design (backend-agnostic architecture, exploration-over-optimization rationale citing Nöllenburg 2023), Research impact statement, AI usage disclosure.
- paper.bib covers every citation (Krzywinski 2012, Perez 2021, Nöllenburg 2023, Chen 2018 if the Datasaurus figure lands, nxviz, backend libraries as appropriate).
- Figures: hero figure from the same-stats demo if G2 lands (named fallbacks otherwise; see G2) plus, if word count allows, one architecture/backend figure; each with a real caption via Markdown image syntax.
- AI usage disclosure wording drafted as a distinct, flagged item, starting from the middle posture recorded in Workstream B and calibrated against B's JOSS-precedent examples; **gated on Gary's explicit review** before the draft is called done — it must describe AI usage and human review accurately.
- All prose follows the human-voice rules; the full paper.md draft is **gated on Gary's review** (this is Gary-facing shipped writing under his name).
- No CHANGELOG entry (paper is not released library behavior).

### Workstream D: Submission mechanics and review loop

**Status:** not started (gated on G1 and G4; needs A-C complete)
**Files:** `paper/` (compile fixes only), possibly `.gitlab-ci.yml` or a local script for the inara compile check, `CITATION.cff` (DOI update)
**Done when:**
- paper.md compiles cleanly through the openjournals/inara Docker image locally (repo is on GitLab, so the GitHub Action isn't the default path); compiled PDF reviewed by Gary.
- Zenodo wired: v0.28 tagged release archived with DOI; archive metadata (title, authors, license, version) matches the paper; CITATION.cff and README citation section updated with the real DOI. (Zenodo's GitLab integration or manual upload; record which in the Implementation log.)
- G4 explicitly resolved before filing; if the repo migrated, paper/Zenodo/CITATION.cff/README URLs all updated to the new namespace.
- Submission filed at joss.theoj.org; pre-review issue opens on openjournals/joss-reviews (GitHub account: Gary's).
- Review-response loop: reviewer asks route through this plan via amend-plan (new workstreams for substantive change requests; in-scope tweaks for paper edits). Workstream closes at acceptance and final DOI registration.

## Plan amendments

### Amendment 1 (In-scope tweak): GDA dissolution sweep — 2026-06-12

Trigger: grill Waves 1-2 (GDA dissolved 2026-06-11). Byline becomes Gary Koplik sole author; affiliation is a blank placeholder until settled (G3). Workstream A done-when gains: LICENSE holder line → Gary Koplik, years through 2026, clause-3 entity name updated (type stays BSD-3-Clause; grill correction: it is BSD-3-Clause, not MIT), plus a GDA-mention sweep across repo prose/docs. Touches A's LICENSE and CITATION.cff items and C's front matter.

### Amendment 2 (In-scope tweak): new gate G4, GitLab namespace fate — 2026-06-12

Trigger: grill Wave 2. Repo sits in GDA's `gitlab.com/geomdata` group; assume it stays, but Gary must explicitly resolve before submission. G4 added to External gates; blocks Workstream D alongside G1; D's done-when gains a URL-migration item. No effect on A-C content beyond URLs at submission time.

### Amendment 3 (In-scope tweak): G2 softened to preferred-with-fallback — 2026-06-12

Trigger: grill Wave 1. Same-stats Datasaurus figure is strongly preferred, not existential. G2 reworded; C drafts with the hero-figure slot held open; named fallbacks: karate club, Bitcoin ratings, hairball-vs-hive-plot comparison. Fallback selection routes through amend-plan only if exercised at submission time. Touches C's figures item.

### Amendment 4 (In-scope tweak): AI-disclosure precedents item in B — 2026-06-12

Trigger: grill Wave 1. B gains: collect 3-5 AI-usage disclosures from accepted JOSS papers (2025-2026), filed at `wiki/wiki/analyses/joss-ai-disclosure-precedents.md`. Gary's middle posture ("Claude used in implementation and documentation as of 2025, human signs off on all code"; no deep harness detail by default) recorded as the starting point; final wording stays gated on his review in C.

### Amendment 5 (Deferred follow-up): A's README "Citing" carve-out + execution context — 2026-06-12

Trigger: grill Wave 1 preliminary dispatch. README "Citing" section edits deferred behind `docs-cheat-sheet-and-readme.md` WS-B (existing sequencing note narrowed to that item only); everything else in A starts now. A executes in the `joss-submission` worktree at `~/repos/hiveplotlib-joss` (off master at 162e5b1; no wiki/agent-harness submodules there — A touches only main-repo files).

### Amendment 6 (In-scope tweak): contact channels in A's done-when — 2026-06-12

Trigger: grill Wave 2. Support statement names the GitLab issue tracker as primary help channel (LinkedIn optionally mentioned); CODE_OF_CONDUCT private reporting contact is gary.koplik@gmail.com. Touches A's support-statement and CODE_OF_CONDUCT items.

### Amendment 7 (In-scope tweak): affiliation TODO placeholder in C front matter — 2026-06-12

Trigger: grill Waves 1-2. C's front matter carries an explicit `TODO` affiliation marker (likely "Marcell Intelligence") so drafting proceeds before G3 fully clears; placeholder must resolve before submission. Touches C's new front-matter item.

## Holdouts

- `wiki/wiki/plans/conda-forge-distribution.md:192`: references the old CONTRIBUTING line numbers; historical plan record, do not edit.

## Implementation log

Append-only.

2026-06-12: Workstream A complete (in `~/repos/hiveplotlib-joss` worktree). CONTRIBUTING.md rewrite was done by the prior agent in this workstream; this session added `CODE_OF_CONDUCT.md` (canonical Contributor Covenant v2.1 download, contact set to gary.koplik@gmail.com) and `CITATION.cff` (sole author Gary Koplik, no affiliation, BSD-3-Clause, repository-code/url set, DOI comment pending Zenodo; passes `cffconvert --validate`), changed LICENSE holder to "Copyright (c) 2020 - 2026, Gary Koplik" with clause-3 entity updated (BSD-3-Clause text otherwise unchanged), set `license = "BSD-3-Clause"` SPDX string in pyproject.toml (no license classifier added: PEP 639 deprecates classifiers alongside SPDX expressions; sdist builds clean), updated `docs/source/conf.py` copyright to Gary Koplik. GDA sweep: all gitlab.com/geomdata URLs left per G4 (README x2, docs prose, source docstrings, dataset metadata JSON). Open items: `docs/source/_templates/footer.html` still shows the GDA logo linking to geomdata.com (branding call for Gary); `pyproject.toml` author email is gary.koplik@geomdata.com (pyproject was scoped license-only); `.gitlab/gitlab-ci/releaser.py:2` GDA-Cookiecutter provenance comment left as historical. README "Citing" section deferred per Amendment 5.

2026-06-12: Workstream A cleanup (in `~/repos/hiveplotlib-joss` worktree, both Gary-approved post-GDA-dissolution). Removed the GDA logo/link block from `docs/source/_templates/footer.html` (was `<p><a href="https://www.geomdata.com/"><img .../></a></p>`), leaving the valid `<p>Release Version: {{release}}</p>` line as a minimal footer. In `pyproject.toml`: author email `gary.koplik@geomdata.com` -> `gary.koplik@gmail.com`; added `Contact="https://www.linkedin.com/in/PLACEHOLDER"` to the `[project.urls]` inline table (PLACEHOLDER slug needs Gary's real LinkedIn). Verified pyproject parses via `tomllib`. Footer is a Jinja fragment; full Sphinx render not run (no `make install`). Remaining GDA references left untouched: all `gitlab.com/geomdata` URLs (README x2, pyproject Source url, docs prose, source docstrings, dataset metadata) per G4; `.gitlab/gitlab-ci/releaser.py:2` GDA-Cookiecutter provenance comment (historical).

2026-06-12: Workstream B wiki pages filed (from completed web research, all accessed 2026-06-12). Three new analysis pages: `wiki/wiki/analyses/nxviz-comparison.md` (State of the field — nxviz revived June 2026, v0.8.0 plotly backend + v0.9.0 chord diagrams, deprecated OO `HivePlot`, functional 3-group-capped API; fair per-dimension deltas incl. the now-stale "matplotlib-only" claim flagged: nxviz has 2 backends), `wiki/wiki/analyses/hiveplotlib-research-impact.md` (Research impact — thin-but-real; two 2025 bio papers Shi et al. / Wittmer et al. flagged UNVERIFIED+paywalled for maintainer confirmation; PMC7887807 recorded as false positive; PyPI ~1,743/month v0.27.0; open gaps: authenticated GitHub/GitLab code search + Semantic Scholar unretrieved), `wiki/wiki/analyses/joss-ai-disclosure-precedents.md` (mandatory AI disclosure — policy verbatim, accepted-paper examples by posture minimal/scoped/detailed, recommendation toward detailed end, final wording gated on Gary). Updated `wiki/wiki/concepts/hive-plot.md` (competing-implementations table gains a maintenance-status column; nxviz row added). Wiki `index.md` and `log.md` updated per schema. **Author-metadata checklist (G3) awaits Gary** — presented in the run report, not filed (needs his answers: legal-name byline, affiliation placeholder, ORCID, funding statement). Caveats carried forward for Workstream C: the two bio-paper citations must not be cited as confirmed users until Gary confirms full text; the nxviz "matplotlib-only" framing is stale and must say 2 backends; AI-disclosure final wording stays gated on Gary.

2026-06-12: G3 author-metadata values wired into shipped artifacts (Workstream A/C, in `~/repos/hiveplotlib-joss` worktree; main checkout untouched except this log entry). Gary confirmed: ORCID 0009-0005-6103-8816, LinkedIn https://www.linkedin.com/in/garykoplik/, email stays gary.koplik@gmail.com. Edits: (1) `pyproject.toml` `[project.urls]` Contact `https://www.linkedin.com/in/PLACEHOLDER` -> `https://www.linkedin.com/in/garykoplik/` (author email already gmail, confirmed present; no Twitter added pending Gary's handle); (2) `CITATION.cff` author entry gains `orcid: 'https://orcid.org/0009-0005-6103-8816'` (full-URL cff form); (3) `paper/paper.md` front matter: removed `# TODO: ORCID pending` marker and placeholder `0000-0000-0000-0000`, set bare `orcid: 0009-0005-6103-8816` (JOSS front-matter form). Validation: pyproject parses via tomllib (email + Contact confirmed); `uvx cffconvert --validate` -> "valid according to schema version 1.2.0"; paper.md YAML front matter parses via yaml.safe_load (author/orcid/affiliation read back correct). G3 now substantially cleared (ORCID + contact/LinkedIn + email done); the affiliation TODO is INTENTIONALLY still open (`# TODO: affiliation pending (Marcell Intelligence?)` and `name: "TODO: affiliation pending"` left in place per Amendment 7, pending Gary's formal title/affiliation decision). Funding statement also still outstanding. AI-disclosure GATED section untouched. No CHANGELOG entry (paper/metadata not released library behavior).

2026-06-12: Workstream A — added Gary-supplied X/Twitter handle to `pyproject.toml` `[project.urls]` inline table (`Twitter="https://x.com/hiveplotlib"`), appended after the LinkedIn Contact entry; Documentation/Source/Contact entries and the gmail author email unchanged. This closes the "no Twitter added pending Gary's handle" open item from the G3 metadata entry above. Verified pyproject parses via `tomllib` and `urls['Twitter']` reads back `https://x.com/hiveplotlib` (Contact + author email also confirmed intact). No CHANGELOG entry (project metadata, not released library behavior).

2026-06-12: Workstream C paper DRAFT authored (in `~/repos/hiveplotlib-joss` worktree; main checkout untouched except this log entry). Created `paper/paper.md`, `paper/paper.bib`, `paper/figures/.gitkeep`. paper.md body is ~1,280 words (Summary through Acknowledgements; within JOSS 750-1750, on the ~1100-1300 target). Sections present: Summary, Statement of need, State of the field, Software design, Research impact statement, AI usage disclosure, Acknowledgements, References. Thesis built around "layout is a function of the data, not a random seed" (Bostock no-intrinsic-positions + Krzywinski 2012 pattern-from-structure quotes). State of the field: nxviz stated CURRENTLY/fairly (2 backends matplotlib+plotly, 3-group cap, auto-clones-but-not-styleable, no datashader/P2CP); HyPE positioned as closest published comparator (Py2.7, ~13.6k-edge demo); dormant HiveR/pyveplot/jhive/D3 lineage noted. Differentiators led: 5 backends incl datashader, styleable repeat axes, edge-kwarg hierarchy, graph-features, P2CP as clean closer. Software design cites Nollenburg 2023 NP-completeness for exploration-over-optimization + HivePlotMatrix sweeps. Claims ceiling honored: datashader stated only at the existing Wikipedia article-network example scale (no invented numbers); 10M-edge scaling + GNN-heterogeneity confined to explicit future-work framing. Research impact honest-and-thin: ~1,700 dl/month + capability-gap + JOSS near-term-significance; the two 2025 bio papers NOT named (paywalled/unverified per evidence page); "early downstream use beginning to appear" only. GATES/TODOs/PLACEHOLDERS in the draft: (1) front matter affiliation `# TODO: affiliation pending (Marcell Intelligence?)` + ORCID `# TODO: ORCID pending` (placeholder 0000s) per Amendment 7; (2) AI-usage section fenced with `<!-- GATED: ... -->` HTML comment, drafted at scpviz/PAI detailed-but-concise end, no agent-harness internals; (3) hero figure `figures/PLACEHOLDER.png` with real draft caption + HTML comment explaining same-stats/Datasaurus preferred-with-fallback (G2); (4) top-of-file HTML comment block listing my framing decisions for Gary; (5) version written to v0.28 capabilities per G1 (worktree pyproject still pins 0.27.0). Bib unverified flags: Chen 2018 same-stats entry carries a `note` TODO (the canonical Datasaurus work is Matejka & Fitzmaurice CHI 2017; author list/venue for arXiv:1808.09913 to confirm once the hero figure's exact citation is chosen). Whole draft + AI-disclosure wording remain GATED on Gary's review. No CHANGELOG entry (paper not released behavior).

2026-06-12: Gary review pass applied to Workstream A/C drafts (in `~/repos/hiveplotlib-joss` worktree; main checkout untouched except this log entry). `paper/paper.md`: Bostock quote now cited `[@bostock2012hive]` and wording corrected verbatim to source ("...intrinsically-meaningful positions to nodes: the position is only approximate..."; colon, not the draft's semicolon); Summary superlative hedged ("To the author's knowledge, it is the most comprehensive..."); AI-disclosure sentence deduped (human-review claim stated once, three concrete supports kept: sign-off, 100%-coverage CI, all commits by maintainer; section stays fenced GATED); hero-figure comment updated to name both verified same-stats references and recommend chen2018samegraphs for a network figure; draft-note item 9 added flagging the title's "rational" overlap with both Krzywinski's 2012 subtitle (intended homage) and nxviz's tagline, leaving the call to Gary at final review, title itself unchanged. `paper/paper.bib`: added bostock2012hive (@misc, bost.ocks.org/mike/hive/); split the conflated chen2018samestats chimera into verified matejka2017samestats (Matejka & Fitzmaurice, CHI 2017, doi 10.1145/3025453.3025912) and chen2018samegraphs (Chen et al., GD 2018, arXiv:1808.09913) and removed the old entry + TODO note; nxviz year 2024 -> 2026; hagberg2008networkx retyped @article -> @inproceedings (dropped redundant journal field). Repo files: CONTRIBUTING.md LinkedIn sentence gains the real link (https://www.linkedin.com/in/garykoplik/) and intro "we'd love" -> "I'd love" for single-maintainer voice consistency; `docs/source/_templates/footer.html` gains its missing trailing newline. Verification: no em-dashes or AI-filler in paper.md live text (one pre-existing em-dash remains in the front-matter TODO comment, never renders); all live `[@key]` citations resolve to bib entries; matejka2017samestats and chen2018samegraphs are comment-only until the hero figure lands (expected). AI-disclosure section and full paper remain GATED on Gary's review. No make targets run (worktree has no venv); no git state touched. No CHANGELOG entry (paper/metadata not released behavior).
