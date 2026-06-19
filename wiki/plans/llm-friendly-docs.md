# Plan: LLM-friendly Sphinx docs (curated llms.txt)

<!--
Working plan, wiki submodule. Concise per mental-model rule 17.
Scope collapsed 2026-06-19 (see Plan amendments / Alignment grill Wave 1):
llms-full.txt and the sphinx-llms-txt extension are out; DIY hand-curated llms.txt only.
-->

## Goal

Make the hiveplotlib Sphinx docs agent-first friendly as table stakes (no large overhead). Ship a hand-curated `llms.txt` (a fixed-shape markdown file per the llmstxt.org standard: H1 title, blockquote summary, `## Section` headings with bulleted `[page](url): description` links) served at the docs site root on Read the Docs via `html_extra_path`. The index is curated so the conceptual entry points (`introduction_to_hive_plots`, `quick_hive_plots`, `introduction_to_p2cps`) rank first, then the API reference, with the ~40 gallery notebooks last, so an agent pointed at the site finds the load-bearing pages first instead of drowning in gallery noise. The curated index is the load-bearing deliverable; API-reference ranking is a low priority (autodoc is already present). This is docs-infrastructure only: no user-facing Python API, no library behavior change, no new dependency. (`llms-full.txt`, full-text concatenation, dropped — see Holdouts.)

## Alignment (grill)

### Maintainer shared-understanding pass (grill), Wave 1 — premise, scope, dependency (2026-06-19)

Run inline before any dispatch (branch-46 gate already merged; Gary will cut a proper issue + branch before implementation).

**Driver (Q1): part-c, "agent-first friendly as table stakes."** Not a specific observed-agent-failure trigger (a) and not JOSS optics (b). Gary wants to meet standard expectations of agent-friendly open source without large overhead. Consequence: the thin curated `llms.txt` is the load-bearing deliverable; `llms-full.txt` is "almost decorative" for this goal. API-reference ranking is not a strong priority (autodoc already present); he's "not super committed" there.

**Scope (Q2): decouple confirmed, and `llms-full.txt` dropped.** The two artifacts were treated as one deliverable behind one gate, but the entire WS-A feasibility risk (figure-noise from forced `figure_formats={'svg','pdf'}`) lives only on `llms-full.txt`. Gary leans against bothering with the full file. Decision: ship `llms.txt`; `llms-full.txt` → out of scope.

**Dependency (Q3): DIY, no extension.** Verified the leading extension is `jdillard/sphinx-llms-txt` (PyPI v0.7.1 Dec 2025, 18 releases, ~31 stars, solo maintainer; active but small). Its value-add is exactly the two things now being dropped: `llms-full.txt` concatenation and auto-sync of the index. The llms.txt standard (llmstxt.org) is just a fixed-shape markdown file served at site root; in Sphinx that is one file plus an `html_extra_path` line. Auto-derivation would likely follow toctree order, which is *worse* than the curated ordering Gary wants. Decision: **hand-curated `llms.txt`, no new dependency, served via `html_extra_path`.** Claude drafts the file (does not count as "by hand" for Gary's purposes).

**Net effect on the plan (routes to amend-plan):**
- Drop `llms-full.txt` from the goal; record as out-of-scope with a revival trigger (revive if agents need full-text ingestion *and* the figure-noise problem becomes cheap to solve).
- Drop the `sphinx-llms-txt` dependency and the `conf.py` `extensions` edit. No `pyproject.toml` `docs`-extra change.
- **WS-A (feasibility spike) is no longer needed** — its only crux (figure-noise) was `llms-full.txt`-only. Remove it.
- Collapses to roughly two workstreams: (1) author the curated `llms.txt` + wire `html_extra_path` to serve it at root; (2) verify it serves at the site root on RTD and links resolve under `make linkcheck`.
- Curation intent unchanged: conceptual entry points first (`introduction_to_hive_plots`, `quick_hive_plots`, `introduction_to_p2cps`), then API reference, gallery last; one-line descriptions in hive-plot vocabulary. The "pin load-bearing pages, don't over-maintain" instinct from old WS-C carries forward; with a hand file the upkeep is a one-line add per major page.

Grill stopped after Wave 1: the premise/scope/dependency forks all resolved the same direction (collapse), no remaining divergence surfaced. Routing the collapse to orchestrator `amend-plan`.

## Prior ADRs / design docs

No prior ADR or plan touches llms.txt / machine-readable docs; this is net-new design space and a likely future docs-infrastructure ADR candidate (only ADR 0001, networkx-integration, exists, and is unrelated). Adjacent in-flight plans that touch the same files, with coordination notes:

- `wiki/wiki/plans/docs-cheat-sheet-and-readme.md` (ready-for-execution, gated on branch-46 merge) — **edits the `conf.py` `extensions` list and `index.rst` toctree** (adds `sphinx_design` and a "Quick Reference" toctree caption). This plan no longer touches the `extensions` list (it adds a new `html_extra_path` line instead), so the rebase is additive, not a clobber. Whichever lands second simply keeps both edits.
- `wiki/wiki/plans/edge-coverage-and-gotchas-docs.md` (ready-for-execution, gated on branch-46 merge) — adds `examples/hive_plot_gotchas.ipynb`, a conceptual entry point that should rank **high** in the curated index, and grows the gallery corpus the index must down-rank against. WS-1's pinned-pages rule keeps it out of the hand-list until it exists (see WS-1 done-when); add it in a one-line follow-up when that plan lands.
- `wiki/wiki/plans/joss-submission.md` — records the binding **README mirror constraint**: `docs/source/README.md` mirrors root `README.md` and must stay identical. `conf.py:113-119` excludes `README.md` from the build, but `index.rst:37-38` `.. include::`s it into the landing page. Any per-page raw-text or README handling must not edit `docs/source/README.md` independently of root.

## Patterns this replaces

None, this is a net new addition. No existing machine-readable docs artifact, no `llms.txt`, no `html_extra_path` entry in `conf.py` (confirmed absent; `html_static_path` is the only `html_*_path` set, at `conf.py:197`). The replace-and-sweep audit has nothing to sweep; QA confirms no stray `llms` references predate this work.

## Default justifications

- Hand-curated `llms.txt`, not auto-derived: auto-derivation would follow toctree order, which ranks the ~40 gallery notebooks alongside the entry points and defeats the whole point of the file (surface the load-bearing pages first). A small fixed file curated by hand is the cheapest way to get the ordering that matters.
- `llms.txt` curation = conceptual entry points first, API reference second, gallery notebooks last: matches how a human or agent onboards (concepts → reference → examples); the gallery is breadth, not the load-bearing path, and is the bulk that must not crowd out the rest.
- Links point at the published RTD `stable` URLs, not build-relative paths: the file is served raw (not parsed as an in-page link), so a relative path wouldn't resolve to a real page for a consumer fetching `llms.txt` directly. Absolute `https://hiveplotlib.readthedocs.io/stable/...` URLs are the only form that resolves for the file's actual consumer.
- Served via `html_extra_path` (one `conf.py` line), not a builder or extension: the llmstxt.org standard is just a fixed-shape markdown file at the site root; `html_extra_path` copies it there verbatim with zero new dependency or build path.

## Naming audit

Mostly infra/config; the only reader-facing name is the artifact filename.

- Artifact filename: `llms.txt`. This is the de-facto community convention (llmstxt.org); do not rename. `ok`.
- No new make target: the file is copied into the build by `html_extra_path` during the normal `make docs` run, so there is no separate build path to name. `ok`.

The curated index *contents* (the one-line page descriptions) are reader-facing prose and must use hive-plot vocabulary (hive plot, P2CP, axis, edge, node), enforced in WS-1's done-when. The source-file name and its in-tree location are internal/config and out of scope (WS-1 picks the cleanest location).

## API usage examples

No API surface change. This is docs-infrastructure: one `conf.py` line (`html_extra_path`) plus a hand-curated markdown file. There is no user-facing Python API, no new function, class, parameter, or method, and no `pyproject.toml` change. **api-critic involvement is N/A** for every workstream. Skipping the Proposed / Critic's-take / post-impl subsections per the template's "No API surface change" instruction.

## Notebook review

No notebook change. This plan names notebook pages in the curated index (by their published RTD URL) but adds, removes, and restructures no `examples/` notebook. editorial-critic involvement is N/A.

## Workstreams

Two workstreams, no gating spike (the figure-noise crux that drove the old WS-A was `llms-full.txt`-only and is now out of scope). WS-1 authors and wires the file; WS-2 verifies the served result. WS-1 must land before WS-2 (WS-2 inspects WS-1's build output), but there is no feasibility gate ahead of either.

### Workstream 1: Author the curated `llms.txt` and serve it at site root

**Status:** not started
**Files:**
- A new source file `docs/source/_llms/llms.txt` (the curated markdown index; Claude drafts it). Rationale for the location: a dedicated `_llms/` directory under `docs/source/` keeps the raw served file out of the toctree and away from `nbsphinx`/`myst` parsing (only `.rst`/`.md` files Sphinx treats as documents get parsed; an isolated dir avoids any accidental `index.md`-style pickup), and gives `html_extra_path` a single clean directory to copy from. **Not** `docs/source/README.md` (binding mirror constraint, see Prior ADRs).
- One line in `docs/source/conf.py`: `html_extra_path = ["_llms"]` (added near `html_static_path` at `conf.py:197`; no other config touched). `html_extra_path` copies the directory's *contents* to the build root, so `_llms/llms.txt` lands as `<root>/llms.txt`. Note `exclude_patterns` (conf.py:113-119) "also affects html_extra_path" per the comment at conf.py:112; `_llms/` is not excluded, so no interaction.

**Content shape** (llmstxt.org standard): H1 title, blockquote one-line summary of hiveplotlib, then `## Section` headings each holding a bulleted list of `[page title](url): one-line description` links. Ordering:
1. Conceptual entry points first: `introduction_to_hive_plots`, `quick_hive_plots`, `introduction_to_p2cps`.
2. API reference second (the autodoc index).
3. Gallery notebooks last (an `## Optional` section is the natural home, so an agent can skip the breadth).

**Link form:** absolute published RTD URLs, because the file is served raw (a consumer fetches `llms.txt` directly; build-relative links won't resolve the way an in-page Sphinx xref would). Verified forms from the published site: notebook pages are `https://hiveplotlib.readthedocs.io/stable/notebooks/<page>.html`, gallery pages `https://hiveplotlib.readthedocs.io/stable/gallery_examples/...html`, API reference under the same `/stable/` base. Use `/stable/` (the maintained alias), matching how README.md already links (README.md:43-48).

**Done when:**
- `docs/source/_llms/llms.txt` exists, valid llmstxt.org shape (H1, blockquote, `## Section`s of `[text](url): desc` bullets).
- Ordering is entry points → API reference → gallery (gallery under `## Optional`).
- Each entry's description is one line in hive-plot vocabulary (hive plot, P2CP, axis, edge, node), not a raw filename.
- All links are absolute `https://hiveplotlib.readthedocs.io/stable/...` URLs.
- **Pinned-load-bearing-pages rule:** only the conceptual entry points and the API-reference root are hand-listed and pinned at the top. Gallery upkeep is a one-line add. Do **not** hand-list `examples/hive_plot_gotchas.ipynb` — it does not exist on this branch yet (it ships with the edge-coverage-and-gotchas plan); listing it now would point at a not-yet-published page (linkcheck/broken-link risk). Add it in a one-line follow-up when that plan lands, or note it in a code comment as the pinned slot to fill on arrival.
- `html_extra_path = ["_llms"]` added to `conf.py`; a local `make docs` places `llms.txt` at the build root (`public/llms.txt`). (Functional placement is asserted here; the served-shape + linkcheck verification is WS-2.)
- Coordinate the `conf.py` edit with whichever branch-46-gated docs plan (`docs-cheat-sheet-and-readme`, `edge-coverage-and-gotchas-docs`) lands first; this adds a new line (`html_extra_path`) rather than touching the `extensions` list those plans edit, so the rebase is additive, not a clobber.

### Workstream 2: Verify served-at-root, raw-serving, and linkcheck

**Status:** not started (WS-2 inspects WS-1's build output; runs after WS-1)
**Files:** possibly `docs/source/conf.py` (a `linkcheck_ignore` entry only if a curated link legitimately trips the checker); verification-only otherwise.
**Done when:**
- A local `make docs` (full extras) confirms `llms.txt` lands at the build root (`public/llms.txt`), served as raw text/markdown (not wrapped in the HTML theme — `html_extra_path` copies verbatim, so confirm the file is byte-for-byte the source).
- **Linkcheck scope decision (record it):** Sphinx `linkcheck` scans links inside *parsed documents*; a raw `html_extra_path` file is copied, not parsed, so its links are **not** in linkcheck's scope by default. Confirm this against a local `make linkcheck` run. If confirmed, record "curated `llms.txt` links are out of linkcheck scope (raw-copied, not parsed); validate them manually instead" and do a one-time manual spot-check that each pinned URL resolves on the published `/stable/` site. If linkcheck *does* pick them up and one trips, add a justified `linkcheck_ignore` entry with a comment matching the existing `conf.py:181-185` style.
- `make linkcheck` passes (no new failures attributable to this work).
- RTD's `fail_on_warning: true` (`.readthedocs.yaml:24`) is not tripped: confirm the `html_extra_path` addition emits no new build warning (a stray-document or duplicate-file warning would fail RTD). Surface via a `make docs-strict` check used *only* to flag would-be RTD failures (Gary prefers `make docs` for normal runs per memory).

## Plan amendments

### Amendment 1 — Scope collapse to a single curated `llms.txt` (2026-06-19)

**Trigger:** grill-me alignment pass, Wave 1 (recorded in `## Alignment (grill)`). Premise/scope/dependency forks all resolved toward collapse; maintainer confirmed. Routed here under rule 14 (user ask modifying the workstream set and a load-bearing decision).

- **In-scope tweak — driver clarified.** Goal reframed to "agent-first friendly as table stakes (part-c)." The curated `llms.txt` is the load-bearing deliverable; API-reference ranking demoted to low priority (autodoc already present).
- **Deferred follow-up — `llms-full.txt` dropped to out of scope.** Full-text concatenation removed from Goal and recorded in Holdouts with a revival trigger (do not silently delete, per Gary's deferred-work memory). The entire old WS-A feasibility risk (figure-noise from forced `figure_formats={'svg','pdf'}` at conf.py:121-124) lived only on this artifact.
- **In-scope tweak — `sphinx-llms-txt` dependency and extension approach dropped.** The verified extension (`jdillard/sphinx-llms-txt`, PyPI v0.7.1) added value only via `llms-full.txt` concatenation + index auto-sync, both now unwanted. No `pyproject.toml` `docs`-extra change, no `conf.py` `extensions` edit. Approach is DIY: a hand-curated markdown file served via a one-line `html_extra_path` addition.
- **Removed workstream — old WS-A (feasibility spike) deleted.** Its only crux (figure-noise) was `llms-full.txt`-only. Strict gating language removed; no spike gates the work now.
- **Workstream set collapsed from four (A/B/C/D) to two.** New WS-1 (author + wire the curated file) and WS-2 (verify served-at-root + linkcheck scope). Old WS-B (extension wiring), WS-C (curation surface), WS-D (artifact verification) are superseded by WS-1/WS-2; their content folds in (curation ordering and the pin-load-bearing-pages rule into WS-1; linkcheck and RTD `fail_on_warning` checks into WS-2).
- **Feasibility (this amendment):** no new Python entry point and no new attribute reads of user data; the only entry point is a Sphinx config knob. `html_extra_path` confirmed absent from conf.py (only `html_static_path` at conf.py:197); the new file is net-new. RTD URL form verified against README.md:43-48 (`https://hiveplotlib.readthedocs.io/stable/notebooks/<page>.html`).

## Holdouts

- **`llms-full.txt` (full-text concatenation of the docs).** Dropped 2026-06-19 (Amendment 1). Clearly decided against for now, not forgotten. **Revival trigger:** revive if agents start needing full-text ingestion AND the figure-noise problem (forced `figure_formats={'svg','pdf'}` at conf.py:121-124, which would dump SVG/PDF markup into the concatenated text) becomes cheap to solve. Reviving it likely re-introduces the `sphinx-llms-txt` extension evaluation (its concatenation feature was the value-add) and a fresh feasibility spike on the figure-noise lever.

## Implementation log

Append-only. One line per workstream on completion.

- **WS-1 (2026-06-19):** Authored `docs/source/_llms/llms.txt` (curated llmstxt.org-shape index: H1, blockquote summary, Getting started → API reference → Optional gallery, descriptions in hive-plot vocabulary) and wired `html_extra_path = ["_llms"]` in `docs/source/conf.py` near `html_static_path`. URLs use the published `/stable/...` RTD form (verified against README.md). `make docs` builds clean (no new warnings); `llms.txt` lands byte-identical at `public/llms.txt` (site root, not nested under `_llms/`). `hive_plot_gotchas` deliberately not hand-listed (not on this branch); left as a pinned-slot HTML comment. WS-2 (served-shape/linkcheck verification) and qa-engineer pass remain.
- **WS-1 refinement (2026-06-19):** Tightened `docs/source/_llms/llms.txt` header prose. (1) Removed the `<!-- PINNED SLOT ... -->` HTML comment: the file is served raw as text/plain via `html_extra_path`, so the comment rendered as visible text and leaked the internal `hive_plot_gotchas` plan name; the "add gotchas on ship" reminder is already covered by the CLAUDE.md trip-wire, notebook-author agent, and qa-engineer drift check. (2) Expanded the orientation prose to two short paragraphs giving a standalone mental model: core model (nodes on radial axes by partition, sorted along axes, edges as curves; P2CP = same layout for tabular data), the two-or-three-axes framing with `HivePlotMatrix.from_partition` as the canonical >3-groups path, and how `NodeCollection`/`Edges`/`HivePlot`/`HivePlotMatrix`/`P2CP` + swappable backends fit together; kept the entry-points/API-reference/gallery closing guidance. Link lists and descriptions untouched. `make docs` builds clean (no WARNING/ERROR lines); `public/llms.txt` byte-identical to source, comment gone (`grep PINNED SLOT` → 0).
- **WS-1 Installation section (2026-06-19):** Added a compact `## Installation` section to `docs/source/_llms/llms.txt`, placed directly after the header orientation prose and before `## Getting started`. Three index-voice bullets: base install `pip install hiveplotlib` (linked to the PyPI project page, since no dedicated install/getting-started rST page exists in `docs/source/` — only `index.rst`/`changelog.rst`/`roadmap.rst`/`404.rst`, plus the mirror-constrained `README.md`), matplotlib as the default backend working on the base install (matplotlib/numpy/pandas are base deps in `pyproject.toml`), and the optional extras with real pip syntax verified against `[project.optional-dependencies]`: `[bokeh]`, `[plotly]`, `[holoviews]`, `[datashader]`, `[networkx]`, noting there is no umbrella `[all]`/`[viz]` extra (only a `dev` developer bundle). Header prose, link lists, and descriptions untouched. `make docs` builds clean (no warnings); `public/llms.txt` byte-identical to source with the new section served at root.
- **WS-2 (2026-06-19):** Verified the served result and ran the release-readiness pass (docs-infrastructure only; no Python source/tests/notebooks touched, so the unit-test/coverage gate is N/A). `make docs` and `make docs-strict` (RTD `fail_on_warning` `-W` probe) both exit 0 with zero warnings; the `html_extra_path` addition emits no new warning, so RTD will not fail. `public/llms.txt` re-confirmed at the build root (no nested `public/_llms/`) and byte-identical to source (matching sha256). `make linkcheck` passes with zero broken links; the raw-copied `llms.txt` is out of linkcheck scope (copied verbatim, not parsed into the doctree) — zero `llms` references in the linkcheck run, no `linkcheck_ignore` entry needed. linkcheck independently validates `https://hiveplotlib.readthedocs.io/stable/` as `ok`. Direct `curl` spot-checks of the pinned `/stable/...` entry-point and autodoc URLs returned HTTP 429 (RTD CDN rate-limiting automated requests from this network), not 404 — a manual browser check of those four URLs is the residual follow-up. Replace-and-sweep clean (no `llms-full`/`sphinx_llms_txt` leakage); `docs/source/README.md` and `pyproject.toml` untouched (only `conf.py` modified, +4 lines). No `linkcheck_ignore` change made.
