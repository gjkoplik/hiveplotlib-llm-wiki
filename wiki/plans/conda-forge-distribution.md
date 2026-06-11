# Plan: conda-forge distribution

## Goal

hiveplotlib becomes installable via `conda install -c conda-forge hiveplotlib` and stays in sync with PyPI releases with near-zero per-release effort (autotick bot handles version bumps; only dependency changes need a manual feedstock PR). Ships with: a staged-recipes submission on GitHub, updated install docs/badges in this repo, and a release-process checklist so the new per-release obligations (feedstock dep review, WASM explorer re-pin) don't get lost.

Most of the work happens **outside this repo** (conda-forge/staged-recipes on GitHub, then the auto-created `hiveplotlib-feedstock`). Each workstream names its repo. Gary-only acts: GitHub fork/PR under his account, being listed as recipe maintainer, and confirming the submission.

## Alignment (grill)

```
Not yet run — recommended before dispatch for major plans. Run the grill-me skill
or knowingly skip; record each wave below. Route any resulting plan change to
amend-plan (rule 14).
```

**Gate (Amendment 1.4, 2026-06-10; updated Amendment 2.4): the grill-me pass remains required before any workstream dispatch, deferred to a later session at Gary's request.** All three judgment calls from Amendment 1 are now resolved (extras mapping cleared per Amendment 2.1). The grill-me pass is the sole remaining pre-dispatch gate. Status stays pre-dispatch.

## Prior ADRs / design docs

No ADRs exist yet (`wiki/wiki/adr/` not created); if promoted, this becomes ADR 0001. Established decisions to respect (per research-liaison pre-task findings):

- **Minimal base deps** (matplotlib, numpy, pandas only) — `wiki/wiki/sources/hiveplotlib-python.md`. Confirmed against `pyproject.toml:20-24`. The conda run deps mirror this exactly.
- **Extras as of 0.28.0a0**: bokeh, datashader, holoviews, networkx (pulls scipy), plotly, plus tooling extras; `[dev]` chains the backend extras — `wiki/wiki/plans/i-want-to-plan-optimized-hoare.md` ~914-918. Confirmed against `pyproject.toml:26-105`.
- **Future `[igraph]` / `[igraph-leiden]` extras with GPL deliberately isolated** — `wiki/wiki/plans/networkx-metric-expansion-and-backend-refactor.md` ~996-999. The conda mapping must not foreclose that split: no conda artifact may ever bundle GPL deps into the base install (the no-metapackage decision below preserves this for free).
- **Pure Python, no compiled extensions, is policy; narwhals endorsed as future core dep** — `wiki/wiki/plans/scaling-large-networks.md` ~37, 147. So `noarch: python` is correct and stays correct.
- **Python floor 3.10** (`pyproject.toml:9`), tested through 3.14 in CI.
- **Cross-ecosystem note:** this repo is GitLab-hosted; the feedstock lives on GitHub under conda-forge. Two account ecosystems to maintain; the feedstock never mirrors into GitLab.

## Key decisions

### Extras mapping: minimal co-install, no metapackages (DECIDED)

**DECIDED (Amendment 2.1, 2026-06-10): minimal co-install approach ("dirtbag minimum").** Base package only on conda-forge with minimal run deps. No metapackage outputs, now or later-by-default. No exhaustive pip-extra-to-conda mapping tables in the docs. Gary's rationale: this is the conda-forge norm for libraries of this size; metapackages add feedstock maintenance surface outside his normal GitLab workflow; the future GPL `[igraph-leiden]` split stays docs-only. Supporting reasons from the original recommendation:

1. conda has no extras concept; multi-output metapackages are the only alternative and each one is a permanent feedstock maintenance surface (5 backends × every dep change).
2. Every extra's closure is already on conda-forge under predictable names; co-install guidance is one doc paragraph.
3. Keeps the future GPL igraph split unforeclosed: no conda artifact ever encodes an extra, so adding `[igraph]` on PyPI later requires zero feedstock changes and zero license contamination.

Workstream A is no longer gated on this call. Per-extra closures (retained here as the package-list source for Workstream E's error messages and for Workstream C's optional minimal table; closures from `pyproject.toml:26-105`):

| pip extra | conda co-install |
|---|---|
| `[bokeh]` | `bokeh>=3.0.0` |
| `[holoviews]` | `bokeh>=3.0.0 holoviews>=1.15 numba` |
| `[plotly]` | `plotly` |
| `[datashader]` | `datashader dask numba seaborn` |
| `[networkx]` | `networkx scipy` |

### Recipe targets latest stable, not alphas

Recipe pins the latest stable PyPI release at submission time (0.27.0 today, 2026-06-10; if 0.28.0 ships first, target that). The autotick bot ignores pre-releases by default (it filters non-PEP-440-final versions), so 0.28.0a0 will not trigger a bump PR — Workstream A verifies this is still current bot behavior and notes it in the recipe PR description if relevant.

### License

`LICENSE` is BSD 3-Clause text (custom header, standard clauses). Recipe uses `license: BSD-3-Clause`, `license_file: LICENSE`. Workstream A must verify the LICENSE file is present inside the PyPI sdist (conda-forge requirement; `pyproject.toml:16` declares it, modern setuptools should include it — verify, don't assume). If absent from the 0.27.0 sdist, the recipe falls back to fetching the license from the GitLab tag URL, and fixing sdist inclusion becomes a deferred follow-up for the next release.

### CHANGELOG policy call

**DECIDED (Amendment 1.2, 2026-06-10): include the entry.** Distribution availability is not a code-behavior change, so under the `feedback_changelog_only_for_released_behavior` memory the default would be no entry, but Gary confirmed conda-forge availability is tooling/distribution news worth announcing — a deliberate, maintainer-approved exception. One line in `CHANGELOG.rst` under the next release's misc section ("hiveplotlib is now available on conda-forge").

## Patterns this replaces

- pip-only install prose at `README.md:13-33` and its manual mirror `docs/source/README.md:13-33` (the docs page is included via `docs/source/index.rst:37`; `build_sphinx_docs.sh` does not regenerate it — both files must be edited in lockstep). Replace with pip + conda sections.
- No-conda-badge in `README.md:5-11` badge block → add conda-forge version badge once the feedstock exists.

Everything else that says `pip install hiveplotlib` survives deliberately — see Holdouts.

## Default justifications

No library defaults. Recipe-level "defaults":

- `noarch: python`: pure Python, no compiled extensions (policy), single build for all platforms/Python versions.
- Run deps = `matplotlib-base, numpy, pandas` + `python >=3.10`: exact mirror of the minimal-base-deps design decision; nothing more.
- `matplotlib-base` not `matplotlib`: conda-forge convention for library run-deps (avoids forcing Qt onto headless users; end users who want GUI backends install `matplotlib` themselves).

## Naming audit

- Package name: `hiveplotlib` — matches PyPI, available on conda-forge (anaconda.org/conda-forge/hiveplotlib 404s as of 2026-06-10).
- `matplotlib-base` — conda-ecosystem vocabulary, see above.
- Prose-only terms: "conda-forge" (hyphenated, lowercase), "feedstock", "autotick bot" — standard conda-forge vocabulary; use as-is in docs and the release checklist.
- No new Python API names.

## API usage examples

The "API" here is the install command surface.

### Proposed (from planner / Orchestrator)

```bash
# Example 1: base install (new, the headline addition)
conda install -c conda-forge hiveplotlib

# Example 2: with a viz backend (extras have no conda equivalent; co-install)
conda install -c conda-forge hiveplotlib bokeh

# Example 3: networkx workflows (the pip extra also pulls scipy; mirror that)
conda install -c conda-forge hiveplotlib networkx scipy

# Example 4: existing pip path, unchanged and still documented first
pip install hiveplotlib
pip install hiveplotlib[networkx]
```

### API Critic's take (planning mode)

Pending — invoke api-critic in planning mode if the install-command surface warrants it; for a docs-only command surface the maintainer may skip this knowingly.

### API Critic — post-implementation review

```
Pending — invoke api-critic in post-implementation mode after Workstream C ships
(install instructions are user-facing surface).
```

## Notebook review

No notebook change (notebook pip-extras prose is a holdout; see Holdouts).

## Workstreams

### Workstream A: Author and validate the conda recipe

**Status:** not started
**Repo:** none (scratch only — work in `/tmp`, nothing lands in this repo's tree)
**Files:** `/tmp/staged-recipes/recipes/hiveplotlib/meta.yaml` (or `recipe.yaml` if staged-recipes has moved to the v1 format — check the current staged-recipes example recipe and use whichever format it shows)
**Done when:**

- Recipe generated with grayskull from the latest stable PyPI release as a starting point, then hand-corrected: `noarch: python`, `python >=3.10` in host and run, run deps exactly `matplotlib-base, numpy, pandas`, no extras encoded.
- `license: BSD-3-Clause`, `license_file: LICENSE`, with LICENSE presence in the sdist verified by downloading and listing the actual PyPI sdist.
- Test section: `pip check` plus imports of `hiveplotlib` and `hiveplotlib.viz` (matplotlib-only surface; optional-backend modules must NOT be imported in tests since their deps aren't run deps).
- Maintainer field carries Gary's GitHub handle (placeholder until he confirms it — Gary-only fact).
- Autotick pre-release behavior verified against current conda-forge docs and noted in the recipe draft.
- Recipe text and a submission checklist handed back in the report; local conda-build is optional (staged-recipes CI is the real validation gate).

### Workstream B: staged-recipes submission and review shepherding

**Status:** not started (blocked on A)
**Repo:** `conda-forge/staged-recipes` on GitHub — **Gary executes**; agents prepare text only
**Files:** `recipes/hiveplotlib/` in Gary's fork
**Done when:**

- Gary has forked staged-recipes, added the Workstream A recipe with himself as maintainer, opened the PR, and ticked the staged-recipes checklist.
- Review feedback is addressed (agents can draft responses/diffs; Gary pushes them).
- PR merged, `hiveplotlib-feedstock` auto-created with Gary as maintainer, and `conda install -c conda-forge hiveplotlib` succeeds in a fresh env (allow CDN lag, up to a few hours post-merge).

### Workstream C: Install-docs sweep in this repo

**Status:** not started (blocked on B — don't advertise a package that isn't live)
**Repo:** hiveplotlib (GitLab)
**Files:** `README.md`, `docs/source/README.md` (lockstep mirror), `CHANGELOG.rst`
**Done when** (scope shrunk per Amendment 2.3):

- Both READMEs gain one short conda paragraph (pip stays first): the `conda install -c conda-forge hiveplotlib` one-liner plus a note that optional backends install as separate packages. No mapping table required; a minimal table is acceptable only if trivially cheap.
- conda-forge version badge added to both READMEs' badge blocks.
- CHANGELOG entry added (confirmed per Amendment 1.2): one line under the next release's misc section.
- `make docs` builds clean and the rendered installation page shows both paths.
- Holdouts untouched (verified by grep: no conda mentions added to notebooks, CONTRIBUTING, or CI; source error messages are now Workstream E's surface, not a holdout).

### Workstream D: Release-process checklist

**Status:** not started (can draft in parallel with B; finalize after B so feedstock steps are real, not hypothetical)
**Repo:** hiveplotlib (GitLab) — location `RELEASING.md` at repo root, **confirmed by Gary (Amendment 1.1)**: standard practice, and the release/deploy process is currently undocumented
**Files:** `RELEASING.md` (new)
**Done when:** a brief checklist exists covering, grounded in `.gitlab/gitlab-ci/versioning.yml` and the Makefile:

- Pre-release: version bump in `pyproject.toml`, `CHANGELOG.rst` finalized, MR to `master`, protected-branch matrix jobs green.
- Release: manual `perform_release` CI job → GitLab tag + release, PyPI upload, ReadTheDocs tag build, post-release PyPI install tests.
- Post-release obligations (the new content this plan exists to capture):
  - conda-forge: autotick bot opens a feedstock version-bump PR within hours; review and merge it. **If the release changed base dependencies or the Python floor, the bot will not fix the recipe — open a manual feedstock PR updating run deps.** Pre-releases never trigger the bot.
  - WASM explorer: re-pin hiveplotlib from PyPI per the interactive-WASM-explorer plan's per-release obligation.

### Workstream E: Reword optional-backend import-guard error messages (added per Amendment 2.2)

**Status:** not started (not blocked on A/B — the wording is distribution-agnostic; gated only by the plan-wide grill-me pass)
**Repo:** hiveplotlib (GitLab)
**Files:** `src/hiveplotlib/converters.py:16`, `src/hiveplotlib/graph_features/__init__.py:92`, `src/hiveplotlib/graph_features/networkx/node_metrics.py:15`, `src/hiveplotlib/graph_features/networkx/edge_metrics.py:15`, `src/hiveplotlib/viz/bokeh.py:11`, `src/hiveplotlib/viz/holoviews.py:12`, `src/hiveplotlib/viz/plotly.py:10`, `src/hiveplotlib/viz/datashader.py:24-27`
**Done when:**

- Every optional-dep guard message follows the pattern: "Requires the \<backend\> backend dependencies (\<package list\>). Install via `pip install hiveplotlib[<extra>]`, or install those packages with your environment manager of choice." Applied consistently to all five extras (bokeh, holoviews, plotly, datashader, networkx), not just datashader (whose hidden seaborn dep motivated the change). For the networkx guards, where "backend" reads oddly, adapt minimally (e.g. "Requires the networkx dependencies (networkx, scipy)") while keeping the two-path install sentence verbatim.
- Each message names the extra's **full** package closure from the Key decisions table: bokeh → `bokeh`; holoviews → `holoviews, bokeh, numba`; plotly → `plotly`; datashader → `datashader, dask, numba, seaborn`; networkx → `networkx, scipy`.
- Planning-time audit (2026-06-10): all eight guards are module-level `try/except ImportError` blocks marked `# pragma: no cover`, and no test asserts the guard strings (grep of `tests/` for the pip-install text: zero hits). So no assertion rewrites are expected; test-side work is verification (`make test` green, coverage unchanged). If an asserting test surfaces anyway, update it in lockstep; if the strings turn out to be load-bearing elsewhere, halt per rule 9 rather than improvise.
- CHANGELOG: one line under the next release (these messages shipped in released versions, so rewording them is a released-behavior change; the `feedback_changelog_only_for_released_behavior` memory does not exempt it).
- `make format` and `make ty` clean; line lengths respected (88 code).
- Sequencing caveat for the dispatching session: `converters.py` and `graph_features/__init__.py` carry uncommitted edits on the in-flight branch `46-more-streamlined-networkx-usage-and-support`. Dispatch this workstream only when that branch's state is settled (or on top of it deliberately); an engineer finding unexpected diffs must report, not normalize.

### API Critic — post-implementation review (Workstream E)

```
Pending — invoke api-critic in post-implementation mode after Workstream E ships
(error-message text is user-facing surface).
```

## Plan amendments

### Amendment 1 (2026-06-10): maintainer decisions on the three open judgment calls + dispatch gate

Source: direct maintainer (Gary) decisions, routed via amend-plan.

1. **In-scope tweak — RELEASING.md location confirmed at repo root.** Gary's reasoning: standard practice, and the release/deploy process is currently undocumented, so a dedicated top-level file is warranted. Workstream D proceeds as drafted; its "location needs Gary's sign-off" caveat is resolved.
2. **In-scope tweak — CHANGELOG entry confirmed: include one.** Gary's reasoning: conda-forge availability is a tooling/distribution change worth announcing to users. This is a deliberate, maintainer-approved exception to the `feedback_changelog_only_for_released_behavior` default. Workstream C's done-when changes from "added or explicitly declined" to "added".
3. **Deferred follow-up — extras mapping (metapackages vs co-install docs) still open.** Gary asked for a fuller explanation of the trade-off before deciding. The "Key decisions" recommendation (no metapackages; document co-install) stands as a recommendation only, not a confirmed decision. Resolving this gates Workstream A dispatch (the recipe's output structure depends on it).
4. **Gate — grill-me alignment pass required before any workstream dispatch, deferred to a later session** at Gary's explicit request. Plan status stays ready-for-review / pre-dispatch until (a) the grill-me pass runs and (b) judgment call 3 resolves.

### Amendment 2 (2026-06-10): extras mapping decided; error-message rewording added; docs scope shrunk

Source: direct maintainer (Gary) decisions, routed via amend-plan.

1. **In-scope tweak — extras mapping DECIDED: minimal co-install ("dirtbag minimum"), no metapackage outputs, now or later-by-default.** Resolves Amendment 1.3. Base package only on conda-forge with minimal run deps; no exhaustive pip-extra-to-conda mapping tables. Rationale: conda-forge norm for libraries this size; metapackages add feedstock maintenance outside Gary's normal GitLab workflow; the future GPL `[igraph-leiden]` split stays docs-only. Key decisions section updated from STILL OPEN to DECIDED; Workstream A ungated on this call.
2. **Added workstream — Workstream E: reword ALL optional-backend import-guard error messages** so each names the full set of missing packages, not only the pip-extra command. Pattern: "Requires the \<backend\> backend dependencies (\<package list\>). Install via `pip install hiveplotlib[<extra>]`, or install those packages with your environment manager of choice." Applies to every guard (bokeh, holoviews, plotly, datashader, networkx), not just datashader (whose hidden seaborn dep motivated it). This removes the source-error-message entry from Holdouts. Feasibility audit: all eight guard sites located and read; all are `# pragma: no cover` module-level blocks; no test-asserted strings found in `tests/` (so the brief's expected test rewrites shrink to verification-only). Sequencing caveat recorded: two guard files carry uncommitted edits on the in-flight branch 46.
3. **In-scope tweak — Workstream C docs scope shrinks.** One short conda paragraph on the installation page (one-liner install plus a note that optional backends install as separate packages), conda badge, and the already-approved CHANGELOG entry. No mapping table required (minimal table acceptable only if trivially cheap). Done-when rewritten accordingly.
4. **Gate update — extras-mapping gate CLEARED; grill-me gate remains.** Gary reaffirmed: do not execute yet. The grill-me alignment pass is now the sole pre-dispatch gate; status stays pre-dispatch.

## Holdouts

The replace-and-sweep audit leaves these pip-form instructions alone:

- ~~Source optional-backend error messages~~ — **removed from holdouts per Amendment 2.2**; now in scope as Workstream E (messages name the full missing-package set plus the pip command).
- All `examples/*.ipynb` pip-extras prose (~18 notebooks, e.g. `examples/quick_hive_plots.ipynb:21`): extras syntax is pip-specific; conda users are routed from the installation page. Sweeping 18 notebooks for an install footnote isn't worth the churn.
- `CONTRIBUTING.md:16-23`: dev environment is uv/pip via `make install`; conda is a user-install path, not a dev path.
- `.gitlab-ci.yml` install commands: CI tests the pip distribution; conda coverage is the feedstock CI's job.
- `wiki/` plan/entity mentions of `pip install`: historical plan text, not user docs.

## Implementation log

(append-only)
