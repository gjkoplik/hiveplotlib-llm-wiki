# Plan: GitLab group migration (geomdata/hiveplotlib → hiveplotlib/hiveplotlib)

<!--
Working scratch plan, not curated wiki content. Concise per mental-model rule 17.
This plan is half checklist/runbook, half dispatchable-workstream set: the harness
runs the automated string sweeps; Gary runs the GitLab Transfer and re-wires the
external integrations.
-->

## Goal

Move the hiveplotlib repository off the dissolved GeoMdata/GDA group at
`gitlab.com/geomdata/hiveplotlib` and onto a new top-level group Gary owns,
`gitlab.com/hiveplotlib/hiveplotlib`, with no broken links, no broken CI, and no
broken publishing. When this ships, every in-repo reference points at the new
namespace, the published docs and badges resolve, CI runs green in the new group,
and PyPI/ReadTheDocs releases still work. The move is decided to use GitLab's
built-in **Transfer project** (preserves issues, MRs, stars, git history, and
auto-creates old→new HTTP redirects), with a fork-and-archive fallback noted for
the case where Transfer permission on geomdata is unavailable.

This is not being executed now. The deliverable is a reviewable plan plus a manual
runbook so the move can be performed in one focused session with the harness
running the automated sweeps.

**Sequencing (Gary, 2026-06-13):** the move is bundled with the in-flight branch-46 code
merge and ships as the **v0.28 release**. Driver is GDA's dissolution, so this happens
**before** the JOSS paper work. Doing it as one release means the PyPI upload and the RTD
stable-docs build refresh every stable-version link at once. Concrete consequence for
ownership: this plan (not JOSS) is overwhelmingly likely to land first, so the two edits it
shares with JOSS (author email, GDA footer) are **actively owned and executed here**, with
the idempotent guards in Amendment 1 only as a safety net for the unlikely reverse order.

## Alignment (grill)

Grill run inline with Gary on 2026-06-13, before move-day. Nothing implemented; the
waves below resolved posture, ordering, and scope. Outcomes drive Amendment 2 and the
runbook tightening in sections 2/4/5/8/9.

- **Wave: redirect posture.** Q: how much does the plan lean on the Transfer redirect?
  A: **treat geomdata as if it vanishes.** The old free-tier group will probably persist
  and the redirect is a nice SEO / "I moved" breadcrumb, but nothing in the plan relies
  on it for correctness. Every link is fixed regardless; the new namespace must actually
  serve the badge images and the `countries.shp.zip` fetch, not lean on the redirect.
  Consequence: every "redirect covers it, don't bother" softening is demoted to "fix it
  anyway" (section 6 intro reframed).
- **Wave: reversibility.** Q: is there a rollback if the move breaks something? A: **no
  rollback, fix-forward only.** Gary will never reverse the Transfer; he jumps in and
  brute-forces any breakage. The section-8 fork-and-archive path is a *pre-Transfer*
  alternative, relevant only if Owner-on-geomdata were missing. It is NOT (Gary confirmed
  Owner). There is no post-Transfer reversal.
- **Wave: ordering and CI (the big one).** Q: can the pipeline be proven before the move?
  A: **no; the order is forced and security-driven.** None of the CI works right now; all
  publishing (PyPI, RTD, GitLab release) runs through GitLab CI, and the runners are being
  stood up as **self-hosted runners on Gary's own server ("delta tower")** under a SEPARATE
  plan. The runners cannot be proven before the Transfer, because exposing machine-level
  runners to the dissolving company is a security risk; they can only be wired to the new,
  sole-control namespace. Forced order **(SUPERSEDED by Amendment 3: Transfer comes FIRST,
  branch-46 review finishes after the move with working CI)**: ~~branch-46 review complete AND
  ready → Transfer → stand up delta-tower runners → full pipeline green → local verify →
  `perform_release` → v0.28 tag~~. The release is hard-gated on runners live
  and the whole pipeline working again (gate unchanged). Accepted consequence: a window of a
  moved-but-not-yet-releasable repo with some dead links (low traffic, fine).
- **Wave: safety net in the no-CI window.** Q: is the move blind until runners exist? A:
  **no; local verification is the real safety net.** The sweep done-whens (`make test`,
  `make docs`, `make linkcheck`, `make test-nb`), the badge-image 200 checks, and the
  `countries.shp.zip` fetch all run locally with no runners. Runners only gate the automated
  pipeline and the release. Section 9 splits verification into local (anytime) vs
  CI/release (needs runners).
- **Wave: break-glass.** Q: what if runner setup drags and v0.28 must ship? A: Gary CAN
  publish manually (local `twine upload`, local `rtd_deploy.py --token=$RTD_SECRET` off his
  machine). Clarification Gary emphasized: this is **emergency-only break-glass, not the
  intended route.** The intended path is release purely through the pipeline, which by
  definition requires runners live and the full pipeline green.
- **Wave: scope freeze.** Q: what is v0.28? A: **branch-46 features hardened (correctness,
  tests, docs only, no new features; everything else becomes new follow-up issues) plus the
  migration.** JOSS is explicitly OUT of v0.28 (see Amendment 2 and the scope note in Goal).

## Prior ADRs / design docs

No `wiki/wiki/adr/` directory exists yet. This is **net-new ADR space** and a strong
post-ship ADR candidate (a namespace move with named integration-rewiring steps is
exactly the kind of durable, hard-won decision an ADR captures). Flag for ADR
promotion once the move ships.

Relevant working plans (coordination, not blockers):

- `wiki/wiki/plans/joss-submission.md` — tracks this exact namespace question as
  gate **G4** (working assumption "repo stays at geomdata", lines 36, 105, 169).
  This migration **resolves G4**. The JOSS plan's Workstream A already plans a
  GDA-mention sweep, and Workstream D enumerates URL-dependent assets
  (paper / Zenodo / `CITATION.cff` / README citation / `.gitlab-ci.yml` / issue-tracker
  support channel). **Migration should sequence before JOSS submission**, and the two
  sweeps must not double-edit the same files. Coordination rule: this plan owns the
  `geomdata/hiveplotlib → hiveplotlib/hiveplotlib` URL substitution across all
  currently-tracked files; the JOSS plan owns the *new* artifacts it creates
  (`paper/`, `CITATION.cff`, README citation section) and writes them at the new
  namespace from the start. If migration ships first (recommended), JOSS Workstream A
  finds nothing left to sweep; if JOSS Workstream A runs first, it must use the new
  namespace per G4's resolution recorded here. **Two edits overlap directly (author email
  and the GDA footer block) — see Amendment 1 for single-owner-plus-idempotent-guard
  handling so neither plan breaks regardless of which lands first.**
- `wiki/wiki/plans/conda-forge-distribution.md` — owns the README badge block
  (`README.md:5-11` + the `docs/source/README.md:5-11` mirror) and a release-process
  checklist (`RELEASING.md`, planned). This migration **edits the badge image URLs**
  (they embed `geomdata/hiveplotlib`); coordinate so the conda-forge plan's later
  badge-slot addition lands on top of the migrated URLs, not the old ones. The
  release checklist this plan extends with a "verify repo URLs resolve" item is the
  same one conda-forge plans to write. Recipe sources from PyPI (namespace-agnostic),
  so the feedstock itself is migration-neutral.
- `wiki/wiki/plans/interactive-wasm-explorer.md` — migration-neutral (separate GitHub
  repo, consumes PyPI). Relevant only as a reminder that the `.gitmodules` submodule
  pattern is GitHub-hosted and unaffected (verified below).

## Patterns this replaces

Canonical substitution: `geomdata/hiveplotlib` → `hiveplotlib/hiveplotlib`.
Full audit of the working tree (excluding `.venv/`, generated `docs/source/notebooks/`
and `docs/source/gallery_examples/`, and the `wiki/` submodule's own plan text). Every
hit classified and assigned to a workstream (WS) below.

### Group A — `gitlab.com/geomdata/hiveplotlib` URLs (the canonical substitution)

- `pyproject.toml:17` — `urls = {... Source="https://gitlab.com/geomdata/hiveplotlib"}` → WS-1
- `docs/source/conf.py:206` — GitLab icon-link `"url": "https://gitlab.com/geomdata/hiveplotlib"` → WS-2
- `README.md:5,7,8,9,10,11` — six badge **image** URLs (`/-/raw/master/.gitlab/*_badge.svg`) → WS-2
- `README.md:46` — examples-directory tree link → WS-2
- `README.md:72` — CONTRIBUTING.md blob link → WS-2
- `docs/source/README.md:5,7,8,9,10,11,46,72` — exact mirror of the above six badges + two links → WS-2
- `CONTRIBUTING.md:8,14` — Issue Tracker `/-/work_items` links → WS-3
- `docs/source/roadmap.rst:55` — `/-/work_items` link → WS-2
- `docs/source/404.rst:14` — `/-/work_items` issue link → WS-2
- `src/hiveplotlib/hiveplot.py:2815` — `/-/work_items/new` URL in a user-facing message → WS-1
- `src/hiveplotlib/graph_features/__init__.py:642` — `/-/issues` URL in a docstring → WS-1
- `src/hiveplotlib/datasets/international_trade.py:24` — `/-/blob/master/runners/...` repo link in a docstring → WS-1
- `runners/make_trade_network_dataset.py:248` — `/-/blob/master/runners/...` self-referential URL string → WS-1
- `examples/hive_plots_more_than_three_groups.ipynb:~326` — prose markdown link to the runner (`/-/blob/master/runners/...`) → WS-4
- `examples/hive_plots_more_than_three_groups.ipynb:~554` — **live `gpd.read_file()`** of
  `/-/raw/master/data/countries.shp.zip`; load-bearing, executes in `make test-nb` → WS-4
  (see Holdouts note: depends on the `data/` dir surviving on the new master; redirect
  must be verified before this URL flips, or the notebook breaks in CI)
- `docs/source/blog/v0.27.0_speedups.ipynb:349` — prose `/-/work_items` issue link (blog
  notebook, editable per CLAUDE.md) → WS-4

### Group B — bare `geomdata` / GDA-entity references (separate from the namespace, swept here)

- `pyproject.toml:12` — author email `gary.koplik@geomdata.com`. **RESOLVED 2026-06-13:**
  → `gary.koplik@gmail.com` (Gary confirmed; already the public commit-author email per
  the JOSS plan). → WS-1
- `docs/source/_templates/footer.html:1,2` — GDA company link and `GDA-logo-2025.svg`.
  **RESOLVED 2026-06-13:** remove the GDA footer block entirely (Gary confirmed). → WS-2
- `LICENSE:1,14` — `Copyright (c) 2020 - 2024, Geometric Data Analytics, Inc.` and the
  GDA non-endorsement clause. **PULLED INTO v0.28 (Amendment 6, 2026-06-16):** both →
  "Gary Koplik" (copyright line → `Copyright (c) 2020 - 2026, Gary Koplik`; clause →
  `Neither the name of Gary Koplik`), matching the verified JOSS-worktree `LICENSE`. No
  longer deferred to JOSS. → WS-5
- `CONTRIBUTING.md:5` — "we will restrict developers to employees of Geometric Data
  Analytics". **PULLED INTO v0.28 (Amendment 6, 2026-06-16):** the whole file is replaced
  wholesale with the JOSS-worktree single-maintainer-governance version (this prose line is
  removed). No longer deferred to JOSS. → WS-3 (now a full replacement; see Amendment 6)
- `.gitlab/gitlab-ci/releaser.py:2` — comment "adapted from GDA-Cookiecutter". Internal
  provenance comment, harmless. → Holdout

### Group C — `.gitlab/` directory and GitLab-CI internals (NOT namespace-coupled)

The `.gitlab/` directory name is GitLab's own convention (CI config, badge SVGs, helper
scripts), unrelated to the `geomdata` group. The many `.gitlab/...` path hits
(`Makefile:8`, `pyproject.toml:185`, `docs/source/conf.py:115`, the `.gitlab/make_*.py`
scripts, `.gitlab-ci.yml` job names, `.gitlab/gitlab-ci/versioning.yml`) are all
**Holdouts** — they reference a directory, not the group. The CI job names
(`release_gitlab`, `pages`, `readthedocs_*`) and `python-gitlab` API usage are
namespace-agnostic; the API authenticates via `CI_JOB_TOKEN` against whatever project
runs the job, so they follow the transfer automatically. See Holdouts.

## Default justifications

No new code defaults (this is a string/config sweep, not a feature). Two **config-value
decisions**, both **RESOLVED 2026-06-13** by Gary:

- `pyproject.toml` author email `gary.koplik@geomdata.com` → `gary.koplik@gmail.com`
  (confirmed; gmail is already the public commit-author email per the JOSS plan, so no
  new privacy cost).
- `docs/source/_templates/footer.html` GDA logo/link → **remove the GDA block entirely**
  (confirmed).

## Naming audit

No new code symbols. The only introduced strings are the new namespace URLs themselves,
which are fixed by Gary's GitLab group name (`hiveplotlib`) and the unchanged project
name (`hiveplotlib`). No placeholder/config keys are coined. Nothing to audit against
user vocabulary.

## API usage examples

No API surface change. This is a namespace/string migration; no signatures, parameters,
or entry points change. (The one behavioral surface that touches a URL is the live
`gpd.read_file()` in `hive_plots_more_than_three_groups.ipynb`, but its call shape is
unchanged; only the literal URL string moves.)

## Notebook review

WS-4 edits notebook prose/URLs only; it does not add, remove, or restructure any
notebook, change any dataset, or shift any notebook's genre or documented class. The
`hive_plots_more_than_three_groups.ipynb` data-fetch URL change is a literal-string
edit inside an existing cell. No editorial-critic pass required (no artifact-level
change). Flagged here so qa-engineer can confirm the no-restructure claim.

## Workstreams

The automated workstreams are dispatchable by the harness when Gary triggers the
move. WS-1..4 are **independent string sweeps over disjoint file sets** and can run
concurrently; **WS-5 (Amendment 6)** is the GDA-entity scrub of `LICENSE` + `docs/source/conf.py`
and is likewise independent (disjoint files). None should run until Gary has (a) performed or
scheduled the GitLab Transfer and (b) settled the two Group-B config decisions (email, footer).
The substitutions are mechanical; the risk is sequencing against the live data-fetch URL
(WS-4) and the redirect. WS-3 is now a wholesale CONTRIBUTING.md replacement and WS-5 is net-new,
both per Amendment 6.

### Workstream 1: Package metadata + source/docstring URLs

**Status:** done (Amendment 7 — branch-46 working tree, uncommitted)
**Files:** `pyproject.toml`, `src/hiveplotlib/hiveplot.py`, `src/hiveplotlib/graph_features/__init__.py`, `src/hiveplotlib/datasets/international_trade.py`, `runners/make_trade_network_dataset.py`
**Done when:** — ✅ **all in-repo done-whens verified at v0.28 closure (2026-06-18).** `pyproject.toml` Source URL on the new namespace and author email `gary.koplik@gmail.com`; `hiveplot.py` and the two blob URLs migrated; the `graph_features/__init__.py:642` `/-/issues` URL was already gone (no-op, Amendment 7); `make test` green (no `geomdata` test assertions). The closure grep confirms zero stray `geomdata` in these files.
- Every `gitlab.com/geomdata/hiveplotlib` in these files → `gitlab.com/hiveplotlib/hiveplotlib`.
- `pyproject.toml:12` author email → `gary.koplik@gmail.com` (resolved). **Idempotent
  (Amendment 1):** if the email is already `gary.koplik@gmail.com` (the JOSS worktree
  merged first), this item is a satisfied no-op — do not re-edit, do not search for the
  old `geomdata.com` string, do not flag a conflict.
- `make test` passes (the `hiveplot.py:2815` message and `graph_features/__init__.py:642` docstring are not string-asserted in tests — verify with a grep of `tests/` for `geomdata`; if any assertion exists, fold the test update in).
- No CHANGELOG entry (internal/metadata URL change, not released library behavior; per the "CHANGELOG only for released-behavior changes" memory).

Before / after (representative):
```
# pyproject.toml:17
- urls = {Documentation="https://hiveplotlib.readthedocs.io/stable/", Source="https://gitlab.com/geomdata/hiveplotlib" }
+ urls = {Documentation="https://hiveplotlib.readthedocs.io/stable/", Source="https://gitlab.com/hiveplotlib/hiveplotlib" }

# src/hiveplotlib/hiveplot.py:2815
- "https://gitlab.com/geomdata/hiveplotlib/-/work_items/new"
+ "https://gitlab.com/hiveplotlib/hiveplotlib/-/work_items/new"
```

### Workstream 2: Docs config, README badges, and footer

**Status:** done (Amendment 7 — branch-46 working tree, uncommitted)
**Files:** `README.md`, `docs/source/README.md`, `docs/source/conf.py`, `docs/source/roadmap.rst`, `docs/source/404.rst`, `docs/source/_templates/footer.html`
**Done when:** — ✅ **all in-repo done-whens verified at v0.28 closure (2026-06-18).** Six badge image URLs + click-through links on the new namespace in both `README.md` and its mirror; `conf.py` icon-link and the `roadmap.rst` / `404.rst` work_items links migrated; GDA footer block removed; the dead `www.hiveplot.com` → `https://hiveplot.com` fix (Amendment 5) applied in both READMEs. `make docs` clean and `make linkcheck` resolved the migrated URLs locally (the move is live, so the redirect-gate note no longer blocks; per the Implementation log Gary ran both locally); badge images return 200 from the new path.
- Every `gitlab.com/geomdata/hiveplotlib` (badge image URLs, click-through links, the
  conf.py GitLab icon-link, `/-/work_items` links) → new namespace, in both
  `README.md` and its `docs/source/README.md` mirror (keep them identical).
- `footer.html` GDA link/logo block removed entirely (resolved). **Idempotent
  (Amendment 1):** if the GDA block is already gone (the JOSS worktree merged first,
  leaving `<p>Release Version: {{release}}</p>`), this item is a satisfied no-op — confirm
  the block is absent and move on; do not re-add, do not error on the missing string.
- **Dead-link fix (Amendment 5):** `README.md:64` and `docs/source/README.md:64`
  `http://www.hiveplot.com/...` → `https://hiveplot.com/...` (the `www` subdomain no longer
  resolves; the apex is alive on HTTPS). Keep the two README copies identical.
- `make docs` builds with no new warnings (use `make docs`, not `make docs-strict`,
  per the memory; scan all warnings at once). `make linkcheck` resolves the new badge
  and repo URLs — **note linkcheck only passes once the Transfer redirect or the new
  project is live**, so this done-when is gated on the move having happened.
- Badges render in a local README preview (image URLs return 200 from the new path).
- No CHANGELOG entry (docs/metadata change).

Before / after (badge image URL, representative of all six):
```
# README.md:7 and docs/source/README.md:7
- [![Matplotlib Support](https://gitlab.com/geomdata/hiveplotlib/-/raw/master/.gitlab/matplotlib_badge.svg)](https://matplotlib.org/)
+ [![Matplotlib Support](https://gitlab.com/hiveplotlib/hiveplotlib/-/raw/master/.gitlab/matplotlib_badge.svg)](https://matplotlib.org/)
```
Note the `/-/raw/master/.gitlab/` path is unchanged — only the group segment moves;
the `.gitlab/` directory is GitLab convention (Holdouts), not the old group name.

### Workstream 3: CONTRIBUTING.md full replacement (REWRITTEN by Amendment 6)

**Status:** done — original URLs-only scope superseded by the wholesale rewrite (Amendment 7 — branch-46 working tree, uncommitted)
**Files:** `CONTRIBUTING.md`
**✅ verified at v0.28 closure (2026-06-18):** `CONTRIBUTING.md` is the JOSS-worktree single-maintainer-governance content (Gary Koplik named, GDA developer-restriction removed), then voice-tuned and given an issue-first numbered flow + fork-submission section per the Implementation log; the new-namespace URLs are carried in via the replacement, with `/-/issues` corrected to the non-deprecated `/-/work_items` form and the `Makefike` typo fixed. The closure grep confirms zero `Geometric Data Analytics` / `geomdata` / `Makefike` in the file.
**Scope change (Amendment 6, 2026-06-16):** the original WS-3 (migrate the two `/-/work_items`
Issue Tracker URLs only, leave the GDA-developer-policy prose to JOSS) is **superseded**. WS-3 now
replaces `CONTRIBUTING.md` wholesale with the verified JOSS-worktree version
(`~/repos/hiveplotlib-joss/CONTRIBUTING.md`): single-maintainer governance, Gary Koplik named, the
GDA developer-restriction (old `CONTRIBUTING.md:5`) removed. The old URL-only edit is overwritten
by the replacement, intentionally.
**Done when:**
- `CONTRIBUTING.md` is the JOSS-worktree content, with two adaptations applied on the way in:
  - its three `gitlab.com/geomdata/hiveplotlib/-/issues` URLs (JOSS-worktree lines 9, 18, 22) →
    `gitlab.com/hiveplotlib/hiveplotlib/-/issues` (new namespace);
  - the single `Makefike` typo (JOSS-worktree line 48, "via the `Makefike` runs") → `Makefile`.
- No `Geometric Data Analytics` / `geomdata` / `Makefike` remains in the file.
- No CHANGELOG entry (governance/docs change, not released library behavior).

(Kept as its own workstream because it is a single-file wholesale replacement with two
text adaptations, distinct from WS-2's docs-config sweep.)

### Workstream 4: Notebook prose + live data-fetch URLs

**Status:** done (Amendment 7 — branch-46 working tree, uncommitted)
**Files:** `examples/hive_plots_more_than_three_groups.ipynb`, `examples/introduction_to_hive_plots.ipynb` (Amendment 5), `docs/source/blog/v0.27.0_speedups.ipynb`
**Done when:** — ✅ **all in-repo done-whens verified at v0.28 closure (2026-06-18).** The runner-link prose and the live `gpd.read_file()` URL in `hive_plots_more_than_three_groups.ipynb` migrated (the new-namespace fetch verified HTTP 200, so the redirect-gate is satisfied); the `v0.27.0_speedups.ipynb` work_items link migrated; the `introduction_to_hive_plots.ipynb` `hiveplot.com` image + homepage link fixed (Amendment 5). `make test-nb` ran the notebook end-to-end against the new URL with no fetch error (per the Implementation log). Edits confined to `examples/` + `docs/source/blog/`. The two remaining `geomdata` hits in `.ipynb_checkpoints/` are regenerable autosave scratch, not shipped artifacts.
- The prose runner-link and the **live `gpd.read_file()` URL** in
  `hive_plots_more_than_three_groups.ipynb` → new namespace.
- The `/-/work_items` prose link in the `v0.27.0_speedups.ipynb` blog notebook → new namespace.
- **Dead-link fix (Amendment 5):** in `introduction_to_hive_plots.ipynb`, the image at ~:48
  and the homepage link at ~:50 `http://www.hiveplot.com/...` → `https://hiveplot.com/...`
  (the `www` subdomain no longer resolves; the apex serves HTTPS, image returns 200). Prose/URL
  edit only — no restructure, no dataset/genre change.
- **Gate:** the data-fetch URL only flips after the Transfer redirect is confirmed live
  (or the new project is up with the `data/` dir present on master). Verify by fetching
  `https://gitlab.com/hiveplotlib/hiveplotlib/-/raw/master/data/countries.shp.zip`
  returns 200 before committing the edit.
- `make test-nb` runs `hive_plots_more_than_three_groups.ipynb` end-to-end against the
  new URL with no fetch error.
- Edits confined to `examples/` and `docs/source/blog/` per CLAUDE.md (no edits to
  generated `docs/source/notebooks/` or `gallery_examples/`).
- No CHANGELOG entry.

### Workstream 5: GDA-entity scrub — LICENSE + docs copyright (Amendment 6)

**Status:** done (Amendment 7 — branch-46 working tree, uncommitted)
**Files:** `LICENSE`, `docs/source/conf.py`
**✅ verified at v0.28 closure (2026-06-18):** `LICENSE:1` = `Copyright (c) 2020 - 2026, Gary Koplik` and `LICENSE:14` clause = `Neither the name of Gary Koplik` (both byte-match the JOSS worktree); `docs/source/conf.py:23` = `copyright = f"2020 - {Timestamp.now().year}, Gary Koplik"  # noqa: A001`. The closure grep confirms zero `Geometric Data Analytics` / `geomdata` in either file; `make docs` clean.
**Done when:** see Amendment 6 for the full done-when. In brief: `LICENSE:1` →
`Copyright (c) 2020 - 2026, Gary Koplik`; `LICENSE:14` clause entity → `Gary Koplik` (both match
the verified JOSS-worktree `LICENSE`); `docs/source/conf.py:23` f-string entity → `Gary Koplik`,
keeping `{Timestamp.now().year}`; no `Geometric Data Analytics`/`geomdata` remains in either file;
`make docs` clean; no CHANGELOG entry. Net-new in Amendment 6; reverses the LICENSE Holdout.

## Manual runbook (Gary performs; the harness cannot)

Ordered. Run the harness sweeps (WS-1..4) only where each step says so.

**Progress (Amendment 7, 2026-06-16):** Phase 1 Transfer ✅, Phase 2 remote-remap ✅, Phase 3
runners + CI/CD variables ✅ (section 5), Phase 4 step 7 sweeps ✅ (WS-1..5 shipped onto the
**branch-46 working tree** as uncommitted edits; see Implementation log). **Remaining are Gary's
manual steps:** commit branch-46, green CI, finish review and merge as v0.28 (steps 8-9),
`perform_release` → v0.28 tag → PyPI + RTD publish (step 10), and the external-links pass (step
11). Earlier marker (Amendment 5): Phases 1-3 complete, now at Phase 4 sweeps.

**Forced order (Amendment 3, supersedes Amendment 2's branch-46-first order; local-move step
revised by Amendment 4):** the Transfer comes BEFORE branch-46 is merged, and the local move is
a `git remote set-url` remap (Amendment 4), not a re-clone. Section sequence on move-day: 0
(clean-slate preservation, now optional-only) → 1 (prerequisites) → 2 (Transfer) → repoint local
via `git remote set-url` → runners (separate plan) → 5 (integrations) → 4 (sweeps) → 9 local
verify → finish branch-46 review with working CI → merge → 9 CI/release (v0.28 tag) → 6 (external
links). Sections 3, 7, 8 are reference/hygiene.

### 0. Pre-transfer local preservation (OPTIONAL — only for the clean-slate path)
**Reframed by Amendment 4.** The primary local move is now a `git remote set-url` remap
(section 7), which deletes nothing, so **none of this preservation work is needed for the move
itself**. This whole section applies ONLY if Gary chooses the optional nuke-and-reclone
clean-slate path (section 7). The verified-state facts below stay as reference (they confirm the
clone holds nothing unpushed), but they are **no longer move-blocking** under the remap path. The
Transfer carries everything server-side; with remap the local clone is also untouched, so the
stash rescue, the `sandbox/`/`data/` copy-aside, and the belt-and-suspenders push are unnecessary
unless nuking. Verified 2026-06-13 (full detail in Amendment 3, Change 2):
- **Committed work** — all on remotes (`git log --branches --not --remotes` empty); transfers
  with the project, nothing to do.
- **Submodules** — both committed and pushed; the wiki (at `ea3e34d` = origin/main) carries
  this plan on its GitHub remote, safe independent of this clone. After re-clone, if the wiki
  submodule lands on an older pin, `cd wiki && git checkout main && git pull`.
- **JOSS worktree** `/home/garyk/repos/hiveplotlib-joss` — committed and pushed (0/0 vs
  `origin/joss-submission`); a worktree of this clone, so nuking the clone orphans it. Delete
  it too; re-add later if wanted. No data loss.
- **`stash@{0}` (the one real risk)** — WIP on `37-hiveplotmatrix-class` ("WIP hpm_*.ipynb
  nbs"), empty tracked diff implies notebooks stashed with `-u`. Stashes do NOT transfer and do
  NOT survive a nuke. Inspect `git stash show -u "stash@{0}"`; rescue via
  `git stash branch <name>` (or pop + commit + push) before nuking if wanted.
- **Gitignored working folders to copy aside** — `sandbox/` (~15 MB) and the untracked parts of
  `data/` (~90 MB: `datasaurus_prototypes/`, `profiling/`, `plan.md`; `data/pytest/` and
  `data/.ipynb_checkpoints/` are regenerable, skip). SKIP `public/` and `.venv/` (rebuilt by
  `make docs` / `make install`). `.claude/` regenerates via `make sync`; copy
  `settings.local.json` only if customized.
- **Belt-and-suspenders** — `git push --all && git push --tags` to the current remote before
  the move. Confirmed nothing unpushed; cheap insurance.

### 1. Pre-move prerequisites
- Confirm **Owner** permission on both the source project (`geomdata/hiveplotlib`) and
  the destination group (`hiveplotlib`). Transfer requires Owner on both. If Owner on
  geomdata is unavailable, use the **fork fallback** (section 8).
- Create the new top-level group `hiveplotlib` on gitlab.com if not already created.
- Group-B config decisions settled (2026-06-13): author email → `gary.koplik@gmail.com`,
  footer → remove the GDA block. WS-1/WS-2 run with these final values.
- Snapshot the current external-integration config (screenshot or notes): CI/CD
  variables, RTD token, any Zenodo webhook, any GitHub mirror. You will re-verify these
  after the move.

### 2. Perform the Transfer
**Ordering gate (Amendment 3, supersedes Amendment 2):** the Transfer comes **FIRST**, before
branch-46 is merged. (Amendment 2 had branch-46 review preceding the Transfer; Gary confirmed
2026-06-13 the dependency runs the other way: the runners can only live in the new namespace,
and Gary wants CI green on branch-46 before tagging a release, so the move must unblock runners
→ CI → validating branch-46.) The Transfer moves the whole project at once, **branch 46
included**, with its open MR, all issues, tags, full history, and CI config; there is no need
to merge 46 first. Everything downstream is hard-gated on the **delta-tower CI runners**
(separate plan) being live and the full pipeline green before any release (that gate is
unchanged from Amendment 2). No rollback exists after this step; fix-forward only.
- Project Settings → General → Advanced → **Transfer project** → select the new
  `hiveplotlib` group. Confirm.
- What Transfer **preserves**: all branches (**including branch 46**), all MRs, issues, stars,
  full git history, tags, CI config, labels, milestones, and crucially **auto-creates HTTP
  redirects** from `geomdata/hiveplotlib` to `hiveplotlib/hiveplotlib` (web and git remote
  paths redirect).
- What Transfer does **not** carry automatically: CI/CD variables and secrets, deploy
  tokens, scheduled pipelines may need re-enabling, any Pages custom domain, and
  external-service linkages (RTD, Zenodo, Codecov, GitHub mirror). Re-establish these in
  section 5.
- **Repoint the local clone after the Transfer (Amendment 4 — primary path):** run
  `git remote set-url origin git@gitlab.com:hiveplotlib/hiveplotlib.git`, then `git fetch origin`,
  then `git remote -v` to confirm. That one remap covers ALL worktrees of this clone (the
  `~/repos/hiveplotlib-joss` worktree and `.claude/worktrees/*` share the main clone's remote
  config); submodules are untouched (they point at GitHub). The clone keeps working through the
  move (the redirect even covers an un-repointed remote). See section 7 for full detail.
- **Optional clean-slate re-clone (not the default):** if Gary wants the incidental
  spring-cleaning a fresh clone gives (dropping stale local branches, the three `claude/*` agent
  worktrees, `.ipynb_checkpoints`), preserve the section-0 local-only artifacts first, then clone
  from the new URL, `make sync` to repopulate submodules, and `make install` to rebuild `.venv`.
  If the wiki submodule lands on an older pin, `cd wiki && git checkout main && git pull`. This
  hygiene is decoupled from the move and can be done anytime via `git worktree prune` plus
  deleting unwanted branches.

### 3. Keep the old path alive (do not shadow the redirect)
- With Transfer, the redirect keeps `geomdata/hiveplotlib` URLs working, so old badges,
  bookmarks, and `git remote`s continue to resolve. **Do not** create or recreate any
  project at the old `geomdata/hiveplotlib` path; a new project there would shadow the
  redirect and break inbound links. Leave the old path as the redirect source.

### 4. Run the automated sweeps
- Dispatch WS-1, WS-2, WS-3, WS-4 (harness) **from the repointed local tree** (section 2 /
  section 7; or the fresh re-cloned tree if Gary took the optional clean-slate path).
  WS-2's `make linkcheck` and WS-4's `make test-nb` data-fetch only pass once the **new
  namespace actually serves the URLs** (after the Transfer), so run the sweeps after the
  Transfer, not before. Per the revised order (Amendment 3), the sweeps run after the move and
  before the branch-46 merge.
- **No runners required for the sweeps.** Every sweep done-when runs locally off Gary's
  machine (`make test`, `make docs`, `make linkcheck`, `make test-nb`, badge 200 checks,
  the `countries.shp.zip` fetch). The sweeps are fully verifiable in the no-CI window
  before delta-tower runners exist; the move is not blind. Runners gate only the automated
  pipeline and the release (sections 5, 9, Amendments 2 and 3).

### 5. Re-establish namespace-coupled external integrations
**Runner gate (grill, 2026-06-13):** none of this CI runs until the **delta-tower
self-hosted runners** (separate plan) are stood up in the new namespace. Re-adding the
variables below is necessary but not sufficient; the pipeline only goes green once the
runners are live. The runner provisioning is **out of scope for this plan** (it gets its
own plan); this plan only gates on it. See Amendment 2.

All publishing (PyPI, ReadTheDocs, GitLab release) runs through GitLab CI and is
**token/credential-based, not OIDC** (verified against `.gitlab-ci.yml` and
`.gitlab/gitlab-ci/versioning.yml` on 2026-06-13). That makes the post-move work simple:
CI/CD variables do not transfer, so re-add the three user-defined variables below in the
new project; everything else either auto-follows or just needs a connected-repo re-point.

- **CI/CD variables to re-add** (new project → Settings → CI/CD → Variables). Exactly three
  user-defined variables drive publishing:
  - `PYPI_USERNAME` and `PYPI_PASSWORD` — used by the `pypi_upload` job
    (`twine upload -u $PYPI_USERNAME -p $PYPI_PASSWORD`, `versioning.yml:87`). Plain token
    upload, no PyPI-side publisher config to touch.
  - `RTD_SECRET` — used by `rtd_deploy.py` to trigger ReadTheDocs builds via the RTD API
    (`.gitlab-ci.yml:145`, `versioning.yml:107,110`).
  The GitLab release job (`releaser.py`) authenticates with `CI_JOB_TOKEN` / `CI_SERVER_URL`
  / `CI_PROJECT_ID` — all GitLab-predefined variables that auto-follow the project, so
  **nothing to re-add there**. Also re-enable the scheduled pipeline (the
  `base_released_scheduled` / `add_ons_released_scheduled` jobs run on `$CI_PIPELINE_SOURCE
  == "schedule"`); schedules do not always survive a transfer.
- **PyPI publishing** — token-based per above. Action: re-add `PYPI_USERNAME` /
  `PYPI_PASSWORD`. No change on PyPI's side (no trusted-publisher/OIDC config that names the
  GitLab path). The release also writes `https://pypi.org/project/hiveplotlib/${VER}/` and
  `https://hiveplotlib.readthedocs.io/v${VER}/` release links (`versioning.yml:70-71`) —
  both already namespace-independent, so no edit needed.
- **ReadTheDocs** — the published docs live on RTD (`.readthedocs.yaml` present; CI
  `readthedocs_*` jobs deploy via `rtd_deploy.py` with `$RTD_SECRET`, RTD project slug
  `hiveplotlib`). Action: re-add `RTD_SECRET`, and re-point the RTD project's connected Git
  repository to the new GitLab project (RTD clones the repo to build; the redirect would
  cover the clone, but re-point it cleanly).
- **GitLab Pages custom domain** — verify whether a custom domain is configured. The
  published docs are on **ReadTheDocs**, not Pages (the `pages` job is "internal" per
  `.gitlab-ci.yml:118,122`), so the Pages URL change is low-impact; still confirm no
  inbound link relies on a `geomdata.gitlab.io` Pages domain.
- **Zenodo–GitLab DOI archiving** — the JOSS plan (Workstream D) plans Zenodo wiring. If
  a Zenodo–GitLab webhook is *already* configured, a group move can break it; re-link the
  Zenodo project to the new GitLab repo. If not yet configured (likely, pre-JOSS), just
  note that JOSS Workstream D must wire Zenodo against the **new** namespace.
- **Codecov / coverage service** — verify whether coverage is reported to an external
  service (Codecov) linked by project path; re-link if so. (Current CI computes coverage
  in-pipeline with `pytest-cov`; confirm no external Codecov project linkage exists.)
- **GitHub mirror** — verify whether a push/pull mirror to GitHub exists; if so, re-create
  it under the new project (mirrors are project-scoped and do not transfer).

### 6. External links Gary controls (manual, list-and-update)
Posture (grill, 2026-06-13): treat geomdata as if it vanishes. The redirect may keep old
GitLab URLs alive, but the plan does not rely on it; every link Gary posted elsewhere is
his to update regardless. Gary's pass over where the old URL appears (2026-06-13):
- **X / Twitter** — likely no live URL that breaks (redirect covers it), but post a
  "hiveplotlib has moved to `gitlab.com/hiveplotlib/hiveplotlib`" note for the public
  record / discoverability.
- **LinkedIn bio** — double-check (Gary unsure whether a repo URL is in the bio).
- **Personal website** — **definitely** needs updates in several places.
- **Resume / CV** — **definitely** needs updating wherever the repo URL appears.
- **Talk slides** — none (confirmed, nothing to do).
- **Affiliated local repos** — check each for the old URL in READMEs/configs:
  `hiveplotlib-javascript`, the bioinformatics-examples repo, the WASM-explorer repo.
- **PyPI project description / long-description** — rebuilt from the repo on the next
  release once WS-1 lands, so the 0.28 release (which bundles this move; see Goal) refreshes
  it automatically. No manual PyPI edit needed.

### 7. Local move: repoint the existing clone (Amendment 4 — PRIMARY path)
**This is the local move, not just contributor hygiene.** The existing clone keeps working after
the Transfer (the redirect even covers an un-repointed remote), so the entire local move is one
remap:
```
git remote set-url origin git@gitlab.com:hiveplotlib/hiveplotlib.git
git fetch origin
git remote -v
```
- **Worktree coverage:** this single `set-url` on the main clone covers ALL its worktrees (the
  `~/repos/hiveplotlib-joss` worktree and `.claude/worktrees/*`) because git worktrees share the
  main clone's remote config. No per-worktree repoint.
- **Submodules:** `.gitmodules` points both submodules at GitHub
  (`github.com/gjkoplik/hiveplotlib-agent-harness`,
  `github.com/gjkoplik/hiveplotlib-llm-wiki`) — **verified unaffected** by a GitLab group
  move. No submodule remote/pin change needed.
- **Nothing is deleted** under this path, so the section-0 preservation work is not required.

**Optional clean-slate alternative (nuke-and-reclone — NOT the default).** A fresh clone is the
only thing that gives incidental spring-cleaning (dropping stale local branches, the three
`claude/*` agent worktrees, `.ipynb_checkpoints`). If Gary wants that: preserve section-0
local-only artifacts first, then clone from the new URL, `make sync`, `make install`, and restore
`sandbox/`/`data/`. That hygiene is decoupled from the migration — Gary can get it anytime via
`git worktree prune` plus deleting unwanted branches, independent of the move.

### 8. Fork-and-archive fallback (PRE-Transfer alternative, only if Owner access were unavailable)
**Not a rollback.** This is a *pre-Transfer* path used only if Owner-on-geomdata were
missing, which it is NOT (Gary confirmed Owner on 2026-06-13). There is no post-Transfer
reversal; once the Transfer happens the plan is fix-forward only (Amendment 2). Kept here
for completeness, not as an expected step.
- Create the new project under `hiveplotlib`, push all branches and tags
  (`git push --mirror`), and re-create issues/MRs manually or via export/import (note:
  no auto-redirect in this path).
- On the old `geomdata/hiveplotlib`: add a top-of-README "**This repository has moved**
  → `https://gitlab.com/hiveplotlib/hiveplotlib`" banner and **archive** the project
  (Settings → General → Advanced → Archive) so it is read-only and clearly superseded.
- All external integrations (section 5) must be wired fresh against the new project.

### 9. Post-move verification pass
Split by what needs runners (grill, 2026-06-13). Local is the real safety net; runners gate
only the automated pipeline and the release.

**Local (works anytime, no runners — verify the move itself):**
- `make test` green locally.
- `make docs` builds clean; `make linkcheck` resolves all migrated URLs.
- `make test-nb` runs `hive_plots_more_than_three_groups.ipynb` (the live data fetch).
- Badges render on the README (new image URLs return 200 from the new path, not via redirect).
- The `countries.shp.zip` fetch returns 200 from the new namespace.

**CI / release (needs delta-tower runners live — separate plan):**
- A pipeline runs green end-to-end in the new project (pages/check jobs at minimum).
- Per the revised order (Amendment 3): with CI green, **finish the branch-46 review and merge
  46 + the sweeps to master**, then trigger `perform_release` (manual) to publish PyPI + RTD
  and tag v0.28. Branch-46 review now happens here, after the move, not before it. This is the
  **intended release path** and is hard-gated on the runners.
- **Break-glass only (emergency, not the plan):** if runner setup drags and v0.28 must
  ship, Gary can publish manually off his machine: local `twine upload` and local
  `rtd_deploy.py --token=$RTD_SECRET`. Use only if the pipeline cannot be made green in
  time; the intended route is release purely through the pipeline.

## Plan amendments

### Amendment 1 (In-scope tweak): cross-plan ownership + idempotent guards for overlapping edits — 2026-06-13

Trigger: Gary flagged that this plan owns edits that also appear in the JOSS plan, and
the execution order across plans is unknown ("I don't know which is gonna get done
first"). The risk: whichever plan ships an edit first leaves the other plan's done-when
checking for a string that is already gone, so the second plan either re-adds it,
conflicts, or fails its check. Fix: name one owner per overlapping edit and write every
overlapping done-when as idempotent (satisfied no-op if the string is already handled).

Cross-plan state read 2026-06-13:
- `joss-submission.md` Workstream A is already **executed** in the `~/repos/hiveplotlib-joss`
  worktree (Implementation log 2026-06-12), but those edits are **not yet on branch 46 or
  master**. A's GDA sweep (its line 121) deliberately **left every `gitlab.com/geomdata`
  URL in place per gate G4** — the namespace question this plan resolves. So the only
  edits JOSS A and this plan both touch are the email and the footer.
- `conda-forge-distribution.md` touches only the README badge block (Workstream C adds a
  conda badge) and source error-message wording (Workstream E). It does **not** touch the
  email, footer, or any GDA-entity edit. The badge-URL coordination is already covered in
  "Prior ADRs / design docs" (lines 60-67). No email/GDA/footer overlap — verified.

Overlapping edits, owners, and guards:

1. **`pyproject.toml:12` author email `gary.koplik@geomdata.com` → `gary.koplik@gmail.com`.**
   Owner: **either plan, whichever lands first** (both target the identical final value;
   JOSS A already made this edit in its worktree). Guard added to WS-1 done-when below:
   if the email is already `gary.koplik@gmail.com` (handled by the JOSS worktree merging
   first), WS-1's email item is a **satisfied no-op** — do not re-edit, do not flag a
   conflict, do not look for the old `geomdata.com` string. WS-1's namespace-URL sweep in
   the same file (`pyproject.toml:17` Source URL) is unaffected and still owned here.

2. **`docs/source/_templates/footer.html` GDA logo/link block removal.**
   Owner: **either plan, whichever lands first** (both remove the same block; JOSS A
   already removed it in its worktree, leaving `<p>Release Version: {{release}}</p>`).
   Guard added to WS-2 done-when below: if the GDA block is already gone, WS-2's footer
   item is a **satisfied no-op** — confirm the block is absent and move on; do not re-add,
   do not error on the missing string.

3. **GDA-entity edits owned by the JOSS plan, NOT this plan (no overlap, recorded for
   clarity).** **SUPERSEDED BY AMENDMENT 6 (2026-06-16):** these edits are now pulled forward
   into v0.28 and owned HERE (LICENSE → WS-5, CONTRIBUTING → WS-3 wholesale replacement). JOSS no
   longer owns them; when its worktree lands, its LICENSE is byte-identical (no-op) and its
   CONTRIBUTING is already on master (drop the edit). The original text below is kept as
   historical record. `LICENSE:1,14` copyright + clause-3 entity (JOSS Amendment 1, already done
   in the JOSS worktree → "Copyright (c) 2020 - 2026, Gary Koplik") and `CONTRIBUTING.md:5`
   GDA-developer-policy prose (JOSS authorized CONTRIBUTING rewrite, already done). This
   plan's Holdouts and WS-3 already defer these to JOSS; that boundary is correct and
   unchanged. This plan never edits them.

Ownership principle: this plan owns purely-mechanical namespace/URL substitution
(`geomdata/hiveplotlib` → `hiveplotlib/hiveplotlib`) across all currently-tracked files,
plus the two settled config-value edits (email, footer) where it shares ownership with
JOSS on a no-op-if-done basis. The JOSS plan owns the legally/governance-flavored
GDA-entity edits (LICENSE copyright, CONTRIBUTING governance prose). Sequencing across the
two plans is **unknown and safe either way**: the namespace URLs are JOSS-untouched (G4
left them for this plan), and the two shared edits are idempotent by the guards above.
Whichever plan a contributor runs first, neither plan breaks. Per the Goal's sequencing
decision (2026-06-13), this plan is expected to land first as the v0.28 release, so in
practice it is the **active executor** of the email and footer edits; the guards above are
the safety net for the unlikely reverse order, not a sign these edits are deferred.

Reciprocal note for the dispatching session: the JOSS plan's Workstream A is already
complete in its worktree, so a reciprocal guard there is informational rather than
load-bearing — its already-done edits cannot "fail" against this plan. This amendment is
the single source of truth for the overlap; a separate amend to JOSS is optional (worth
it only if Gary wants the symmetry recorded in JOSS too). conda-forge needs no reciprocal
note (no overlap on these edits).

### Amendment 2 (In-scope tweak): structural decisions from the move-day grill — 2026-06-13

Trigger: a grill-me alignment pass run inline with Gary on 2026-06-13 (before move-day;
nothing implemented). It settled posture, ordering, scope, and the safety model. Recorded
here; reflected in the Alignment section and runbook sections 2/4/5/6/8/9.

1. **Posture — treat geomdata as if it vanishes.** The old free-tier group will probably
   persist and the Transfer redirect is a useful breadcrumb, but **nothing relies on the
   redirect for correctness.** Every link is fixed regardless, and the new namespace must
   actually serve the badge images and the `countries.shp.zip` fetch. Every prior "redirect
   covers it" softening is demoted to "fix it anyway" (section 6 intro flipped).

2. **No rollback — fix-forward only.** Gary will never reverse the Transfer; he brute-forces
   any breakage. The section-8 fork-and-archive path is a **pre-Transfer alternative**,
   relevant only if Owner-on-geomdata were missing (it is NOT; Owner confirmed). There is no
   post-Transfer reversal. Section 8 reframed accordingly.

3. **Forced ordering, security-driven (external dependency).** **ORDER SUPERSEDED BY
   AMENDMENT 3 (Change 1):** the order below put branch-46 review before the Transfer; Gary
   confirmed (2026-06-13) the dependency runs the other way, so the Transfer now comes FIRST
   and branch-46 review finishes after the move with working CI. The **gate** in this item
   (release hard-gated on runners live + pipeline green) and the security reasoning are
   unchanged; only the branch-46-vs-Transfer position flips. See Amendment 3 for the revised
   forced order. The original text is kept below as historical record.

   None of the CI works today;
   all publishing (PyPI, RTD, GitLab release) runs through GitLab CI, and the runners are
   **self-hosted on Gary's own server ("delta tower")** under a SEPARATE plan. They cannot
   be proven before the Transfer, because exposing machine-level runners to the dissolving
   company is a security risk; they can only be wired to the new, sole-control namespace. So
   the order is FORCED:

   > ~~branch-46 review complete AND ready → **Transfer** → stand up delta-tower runners in the
   > new namespace → full pipeline green → local verify → trigger `perform_release` (manual)
   > → v0.28 tag.~~ (superseded; see Amendment 3 for the Transfer-first order)

   The Transfer can physically precede the runners (it is just a project move), but the
   **release is hard-gated on runners live and the whole pipeline working again.** Recorded
   in runbook sections 2 (Transfer), 5 (integrations), and 9 (verification). Accepted
   consequence: a window of a moved-but-not-yet-releasable repo with some dead links (low
   traffic, acceptable).

   **External-dependency gate:** this plan **gates on** the separate **delta-tower CI
   runners** plan. That runner provisioning is **out of scope for this plan**; it gets its
   own plan, and no migration workstream folds in runner setup.

4. **Local verification is the real safety net; runners gate only the pipeline + release.**
   The sweep done-whens (`make test`, `make docs`, `make linkcheck`, `make test-nb`), the
   badge 200 checks, and the `countries.shp.zip` fetch all run locally with no runners, so
   the sweeps are fully verifiable in the no-CI window. The move is not blind. Section 9
   split into "local (anytime)" vs "CI / release (needs runners)".

5. **Break-glass escape hatch (fallback, NOT the plan).** If runner setup drags, Gary can
   publish v0.28 manually: local `twine upload` and local `rtd_deploy.py --token=$RTD_SECRET`
   off his machine. The **intended** path is release purely through the pipeline (runners
   live, full pipeline green). Local publish is **emergency-only break-glass.** Recorded in
   section 9.

6. **Scope freeze — what v0.28 is.** **PARTIALLY SUPERSEDED BY AMENDMENT 6 (2026-06-16):** the
   LICENSE-copyright and CONTRIBUTING-governance edits called out below as shipping LATER with JOSS
   are now pulled forward into v0.28 (WS-5 + WS-3). The rest of this item stands: v0.28 is still
   branch-46-hardening + migration with no new features, and the JOSS paper/Zenodo work still ships
   later. Only the "GDA LICENSE/CONTRIBUTING wait for JOSS" claim is reversed. Original text kept
   below as historical record. v0.28 = **branch-46 features hardened** (correctness,
   tests, docs only; **no new features** — everything else becomes new follow-up issues) +
   **the migration**. **JOSS is explicitly OUT of v0.28.** The JOSS Workstream-A worktree
   edits (LICENSE copyright, CONTRIBUTING governance rewrite) and the paper/Zenodo work ship
   LATER as their own release, because the JOSS data-source/paper work is inherently slow and
   Gary will not block the large branch-46 network-feature release on it. Accepted
   consequence: master temporarily carries the **new** email + de-GDA'd footer (this plan
   ships these in v0.28) alongside the **old** GDA LICENSE/CONTRIBUTING until the JOSS release
   lands. This is intended and consistent with Amendment 1: migration actively ships the email
   and footer in v0.28; JOSS ships the LICENSE copyright and CONTRIBUTING governance prose
   later. Amendment 1's idempotent guards remain the safety net for the unlikely reverse
   order; they do not imply these edits are deferred here.

### Amendment 3 (In-scope tweak): Transfer-first ordering (supersedes Amendment 2's order) + pre-transfer local-preservation checklist — 2026-06-13

Trigger: Gary confirmed (2026-06-13, before move-day, nothing implemented) two things. First,
the ordering dependency runs the **opposite** way Amendment 2 stated: the Transfer must come
**before** branch-46 is merged, not after. Second, he plans to nuke his local clone and
re-clone fresh after the move, so a pre-transfer local-preservation checklist is needed for
the LOCAL-ONLY artifacts the server-side Transfer does not carry.

**Change 1 — ordering flips to TRANSFER-FIRST.** This **supersedes** Amendment 2 item 3's
forced order (`branch-46 review complete AND ready → Transfer → ...`). Amendment 2's *content*
(posture, no rollback, local-verification safety net, break-glass, scope freeze) is unchanged;
only the Transfer-vs-branch-46 sequencing flips.

Reasoning (Gary, 2026-06-13): CI is dead everywhere now (the GDA runners are gone); the
self-hosted delta-tower runners can only live in the new sole-control namespace (exposing
machine-level runners to the dissolving company is the security risk); and Gary wants CI green
on the 27k-line branch-46 **before** tagging a release. So the Transfer must come first, to
unblock runners → CI → validating branch-46. GitLab Transfer moves the whole project at once
(all branches **including 46**, all MRs, all issues, tags, full git history, CI config, labels,
milestones), so there is no need to merge 46 before the move; branch 46 transfers as a branch
with its open MR intact. Branch-46 review now finishes **after** the move, with working CI.

Revised forced order (replaces Amendment 2 item 3's order; **step 1 and step 4 superseded by
Amendment 4** — local move is now a `git remote set-url` remap, and pre-transfer preservation is
optional-only):

> 1. ~~Pre-transfer local preservation (Change 2 below)~~ — Amendment 4: optional, clean-slate only
> 2. Create the new top-level group `gitlab.com/hiveplotlib` (Gary owns it)
> 3. Transfer the project into it (Settings → General → Advanced → Transfer project)
> 4. ~~Re-clone fresh from the new URL; `make sync`; `make install`~~ — Amendment 4: repoint via
>    `git remote set-url origin git@gitlab.com:hiveplotlib/hiveplotlib.git; git fetch; git remote -v`
>    (covers worktrees; submodules untouched). Clean-slate re-clone is optional.
> 5. Stand up delta-tower runners in the new namespace (separate plan); get the pipeline green
> 6. Re-add the 3 CI/CD variables (`PYPI_USERNAME`, `PYPI_PASSWORD`, `RTD_SECRET`); re-point
>    the RTD connected repo; re-enable the scheduled pipeline
> 7. Dispatch link sweeps WS-1..4 + local verify
> 8. Finish branch-46 review **with working CI**; merge 46 + sweeps to master
> 9. Trigger `perform_release` (manual) → v0.28 tag → PyPI + RTD publish through the pipeline
> 10. External links (CV, website, X "moved" note, affiliated repos) + verification pass

Still true under the new order: the release is hard-gated on runners live and the full pipeline
green (Amendment 2 item 3's gate survives, only its position moves); there is no rollback
(Amendment 2 item 2); local verification remains the no-runner safety net (Amendment 2 item 4);
break-glass stays emergency-only (Amendment 2 item 5); scope freeze is unchanged (Amendment 2
item 6). Runbook sections 2, 4, and 9 are updated to point at this revised order, and Amendment
2 item 3's order statement is annotated as superseded by this amendment.

**Change 2 — pre-transfer local-preservation checklist** (new runbook section 0, since it runs
before everything). The Transfer moves everything server-side, so only LOCAL-ONLY artifacts
need hand-preserving before the clone is nuked. Verified state, 2026-06-13:

- **Committed branch work** — all on remotes already (`git log --branches --not --remotes`
  empty). Transfers with the project; nothing to do.
- **Submodules** — both committed and pushed (wiki at `ea3e34d` = origin/main and **contains
  this plan**; agent-harness clean). The plan is safe on the wiki's GitHub remote independent
  of the hiveplotlib clone. Caveat to record: after re-clone, if the wiki submodule lands on an
  older pinned commit, `cd wiki && git checkout main && git pull` to get the latest plan (it is
  on wiki origin/main regardless).
- **JOSS worktree** `/home/garyk/repos/hiveplotlib-joss` — committed and pushed (0/0 vs
  `origin/joss-submission`). It is a worktree of the main clone, so nuking the clone orphans
  the dir; just delete it too and re-add the worktree later if wanted. No data loss.
- **The one real risk: `stash@{0}`** (WIP on `37-hiveplotmatrix-class`, "WIP hpm_*.ipynb nbs").
  Empty tracked diff implies untracked notebooks stashed with `-u`. Stashes do **not** transfer
  and do **not** survive a nuke. Action: inspect `git stash show -u "stash@{0}"`; rescue via
  `git stash branch <name>` (or pop + commit + push) if wanted, before nuking.
- **Gitignored working folders to copy aside** before nuking: `sandbox/` (~15 MB scratch
  notebooks) and the untracked parts of `data/` (~90 MB: `datasaurus_prototypes/`, `profiling/`,
  `plan.md`). The `data/pytest/` and `data/.ipynb_checkpoints/` parts are regenerable; skip
  them. SKIP `public/` and `.venv/` (rebuilt by `make docs` / `make install`). `.claude/` is
  regenerable via `make sync`; copy `settings.local.json` only if customized.
- **Belt-and-suspenders:** `git push --all && git push --tags` to the current remote before the
  move. Confirmed nothing is unpushed, but cheap insurance.

Recorded in runbook: new section 0 (pre-transfer local preservation), and section 2's
re-clone-fresh note. Sequenced first in the revised forced order above.

### Amendment 4 (In-scope tweak): remote-remap is the primary local move (supersedes Amendment 3's re-clone framing for the local-move step) — 2026-06-13

Trigger: Gary confirmed (2026-06-13, before move-day, nothing implemented) he will move his local
clone with `git remote set-url` (remote-remap), **not** nuke-and-reclone. Remap is strictly
simpler and lower-risk. This **supersedes Amendment 3's re-clone framing for the local-move step
only**; Amendment 3's Transfer-FIRST ordering and everything else (Change 1's reasoning, the gate,
no-rollback, local-verification safety net, break-glass, scope freeze) stand. Amendments 1, 2, and
3 are otherwise intact.

**Change 1 — remote-remap is the primary local move.** After the GitLab Transfer, the existing
local clone keeps working (the redirect even covers an un-repointed remote). The whole local move
is:
```
git remote set-url origin git@gitlab.com:hiveplotlib/hiveplotlib.git
git fetch origin
git remote -v
```
- **One `set-url` on the main clone covers ALL its worktrees** (the `~/repos/hiveplotlib-joss`
  worktree and `.claude/worktrees/*`) because git worktrees share the main clone's remote config.
  No per-worktree repoint.
- **Submodules are untouched** (they point at GitHub).

**Change 2 — section 0 reframed to optional-only.** With remap, **nothing is deleted**, so the
stash rescue, the `sandbox/`/`data/` copy-aside, and the belt-and-suspenders push are
**unnecessary for the move itself**. Section 0 now applies ONLY if Gary chooses the optional
nuke-and-reclone clean-slate path. The verified-state facts (nothing unpushed, the wiki carries
this plan on GitHub, `stash@{0}` exists) stay as reference but are **no longer move-blocking**.

**Change 3 — nuke-and-reclone demoted to an optional "clean slate" note.** The fresh-clone path
(clone from new URL + `make sync` + `make install` + restore `sandbox/`/`data/`) is the ONLY thing
that gives incidental spring-cleaning (dropping stale local branches, the three `claude/*` agent
worktrees, `.ipynb_checkpoints`). Gary can get that hygiene anytime via `git worktree prune` plus
deleting unwanted branches, **decoupled from the migration**.

**Revised runbook order (local-move step supersedes Amendment 3's; everything else of Amendment 3
stands):**

> 1. (optional) pre-transfer hygiene, only if choosing clean-slate
> 2. Create new top-level group `gitlab.com/hiveplotlib`
> 3. Transfer the project (all branches incl 46, MRs, issues, tags, history, CI config)
> 4. Verify redirect + contents in the new location
> 5. **Repoint local:** `git remote set-url origin ...; git fetch; git remote -v` (covers
>    worktrees; submodules untouched)
> 6. Stand up delta-tower runners in the new namespace; pipeline green
> 7. Re-add CI/CD vars (`PYPI_USERNAME`, `PYPI_PASSWORD`, `RTD_SECRET`); re-point RTD connected
>    repo; re-enable the scheduled pipeline
> 8. Link sweeps WS-1..4 + local verify
> 9. Finish branch-46 review with working CI; merge 46 + sweeps
> 10. `perform_release` → v0.28 tag → PyPI + RTD publish
> 11. External links + verification pass

Recorded in runbook: section 0 reframed (optional-only), section 2's local-move note leads with
remap (clean-slate re-clone demoted to an explicit option), section 4's "fresh re-cloned tree"
reconciled to "repointed local tree", section 7 retitled the local move and leads with remap,
and the move-day header order updated. Amendment 3's revised-forced-order list has steps 1 and 4
annotated as superseded here.

### Amendment 5 (Added scope + progress markers): dead `www.hiveplot.com` link folded into WS-2/WS-4; Phases 1-3 done; sweeps run onto branch-46 working tree — 2026-06-16

Trigger: move-day in progress. CI linkcheck went red in the new namespace, surfacing a dead
upstream link; and Phases 1-3 of the runbook completed, so the plan needs its progress markers
and an execution-context note for the WS-1..4 sweeps now running.

**Change 1 — added scope: fix the dead `www.hiveplot.com` link (folded into WS-2 and WS-4).**
Surfaced 2026-06-16 when the new-namespace CI linkcheck went red. Diagnosis (verified by live
check): `www.hiveplot.com` no longer resolves (DNS for the `www` subdomain is gone); the apex
`hiveplot.com` is alive and serves HTTPS (`http://hiveplot.com/` → 301 → `https://hiveplot.com/`
→ 200; the image `https://hiveplot.com/img/networklayouts.png` → 200). This is an **upstream
site change** (Krzywinski's reference site), **not** a geomdata namespace issue, but it lands in
the same files as WS-2 and WS-4, so it is folded in rather than run as a separate pass.

Edits (`http://www.hiveplot.com/...` → `https://hiveplot.com/...`):
- `README.md:64` and `docs/source/README.md:64` (keep the mirror identical) — **WS-2**.
- `examples/introduction_to_hive_plots.ipynb` — the image at ~:48 and the homepage link at
  ~:50 — **WS-4**.

The `wiki/` mentions are research notes, not shipped artifacts, and are **left alone** (consistent
with the existing Holdouts policy on wiki text). No CHANGELOG entry (docs link fix, not released
library behavior; per the "CHANGELOG only for released-behavior changes" memory). Note this widens
WS-4's file set to include `examples/introduction_to_hive_plots.ipynb`, which the original WS-4
did not touch; that notebook gets a prose/URL edit only (no restructure, no dataset/genre change),
so the Notebook-review section's no-restructure claim still holds.

**Change 2 — runbook progress markers (Phases 1-3 complete; now at Phase 4).**
- **Phase 1 — Transfer:** ✅ done.
- **Phase 2 — remote-remap (Amendment 4 local move):** ✅ done.
- **Phase 3 — runners + CI/CD variables:** step 5 (delta-tower runner online) ✅; step 6 (CI/CD
  variables) ✅, rode the Transfer, no rewire needed. Runner root cause for the record: a
  `config.toml` in `~/.gitlab-runner/` was not read by the systemd service; fix was move to
  `/etc/gitlab-runner/config.toml` + restart. Captured in the gitlab-runners repo's
  `self-hosted-runner-setup.md` troubleshooting section.
- **Now at Phase 4 — the sweeps (WS-1..4).**

**Change 3 — execution note: sweeps run onto the branch-46 working tree as uncommitted edits.**
Decision 2026-06-16: do **not** merge branch-46 first. The WS-1..4 sweeps (including the
Amendment-5 Change-1 link fix) are run onto the **branch-46 working tree** as uncommitted edits
for Gary to review and commit as part of the v0.28 release, so branch-46's pipeline goes green
before the big review/merge. This is consistent with the Amendment-3 Transfer-first order (CI is
live, branch-46 review finishes after the move); it pins the sweeps specifically to the branch-46
tree rather than master.

Also recorded — Gary's release-hardening mitigations on branch-46 (context, not sweep work):
- Lowered the notebook job's pytest `-n 7` → `-n 5` to stop kernel deaths from resource
  exhaustion on the Dell tower.
- A lint failure already fixed by Gary.

### Amendment 6 (Added workstream WS-5 + supersedes WS-3): GDA-entity scrub pulled forward into v0.28 (reverses the LICENSE/CONTRIBUTING Holdouts deferral) — 2026-06-16

Trigger: Gary noticed (2026-06-16) the GDA copyright still on branch-46 and decided to pull the
GDA-entity scrub forward into v0.28 rather than leave it to the JOSS plan. This **reverses** the
earlier deferral (Holdouts + Amendment 1 item 3) of `LICENSE` copyright and `CONTRIBUTING.md`
governance prose to JOSS. The driver is the same as the whole migration: GDA dissolved, so the
shipped v0.28 tree should carry zero GDA-entity references, not just zero `geomdata` namespace
URLs. The JOSS-worktree versions of both files are the verified source of truth (already written,
content confirmed 2026-06-16); they are pulled forward onto the branch-46 tree.

After this amendment the shipped tree should have **zero** `geomdata` / `Geometric Data Analytics`
references except the one intentional `geomdata` mention in the `CHANGELOG.rst` repo-move note.

**New Workstream 5 — GDA-entity scrub (LICENSE + docs copyright).**
- **Files:** `LICENSE`, `docs/source/conf.py`.
- **Done when:**
  - `LICENSE:1` `Copyright (c) 2020 - 2024, Geometric Data Analytics, Inc.` →
    `Copyright (c) 2020 - 2026, Gary Koplik` (matches the verified JOSS-worktree `LICENSE`).
  - `LICENSE:14` non-endorsement clause `Neither the name of Geometric Data Analytics` →
    `Neither the name of Gary Koplik` (matches the JOSS-worktree `LICENSE`).
  - `docs/source/conf.py:23` copyright f-string entity `Geometric Data Analytics` → `Gary Koplik`,
    keeping the dynamic `{Timestamp.now().year}` (final form:
    `copyright = f"2020 - {Timestamp.now().year}, Gary Koplik"  # noqa: A001`).
  - No `Geometric Data Analytics` / `geomdata` remains in either file.
  - `make docs` builds with no new warnings (the docs-site footer copyright now reads
    "Gary Koplik"). `make test` unaffected (LICENSE/conf.py copyright are not string-asserted;
    verify with a `tests/` grep for `Geometric Data Analytics` if any doubt).
  - No CHANGELOG entry (copyright-holder/license change is not released library behavior; per the
    "CHANGELOG only for released-behavior changes" memory). See item 4 below for the optional note.

**WS-3 superseded — full CONTRIBUTING.md replacement (was: URLs-only).** WS-3's original scope
(migrate the old CONTRIBUTING's issue-tracker URLs, leave the GDA-developer-policy prose to JOSS)
is **superseded**. The whole file is now replaced wholesale with the verified JOSS-worktree
version (`~/repos/hiveplotlib-joss/CONTRIBUTING.md`): single-maintainer governance, Gary Koplik
named, the GDA developer-restriction removed. WS-3's URL edit is overwritten by the replacement,
and that is intended. Two adaptations applied to the JOSS-worktree text on the way in:
- Its three `gitlab.com/geomdata/hiveplotlib/-/issues` URLs (JOSS-worktree lines 9, 18, 22) →
  `gitlab.com/hiveplotlib/hiveplotlib/-/issues` (the new namespace). (Note: branch-46's current
  CONTRIBUTING already carries the new namespace at lines 8/14, but the wholesale replacement
  drops that file entirely, so these adaptations are what carry the new namespace forward.)
- The `Makefike` typo (JOSS-worktree line 48, in the Testing section: "via the `Makefike` runs")
  → `Makefile`. There is exactly one `Makefike` in the JOSS-worktree source; fix that one.
WS-3's done-when is rewritten in place below to reflect the replacement.

**Cross-plan coordination (updates Amendment 1 item 3 + the JOSS note).** The JOSS Workstream-A
`LICENSE` and `CONTRIBUTING.md` edits now **ship in v0.28** via WS-5 and the WS-3 replacement, so
JOSS no longer owns them. When the JOSS worktree later lands: its `LICENSE` is byte-identical to
what v0.28 ships (clean merge / no-op), and its `CONTRIBUTING.md` is already on master, so **JOSS
should drop both edits**. After this amendment there is **no GDA-entity item left to JOSS** on
these two files; the email and footer were already owned here (Amendment 1 items 1-2), and the
namespace URLs by WS-1..4. Amendment 1 item 3 (which deferred LICENSE + CONTRIBUTING prose to
JOSS) is **superseded by this amendment**; its text is left intact as historical record.

**CHANGELOG (item 4).** The repo-move note is already in `CHANGELOG.rst` Tooling Changes. The
copyright-holder/license change is not released library behavior, so **no separate CHANGELOG
entry** per the "CHANGELOG only for released-behavior changes" memory. Flag for Gary: if he wants
a one-line "copyright updated to Gary Koplik following GDA's dissolution" note he can ask for it,
but it is not auto-added.

**Sections updated by this amendment:** Holdouts (LICENSE and CONTRIBUTING-GDA-prose removed as
deferred items; the `releaser.py` "GDA-Cookiecutter" comment Holdout is unchanged), WS-3 done-when
(rewritten for the wholesale replacement), the Group-B audit rows for `LICENSE:1,14` and
`CONTRIBUTING.md:5` (re-pointed from deferred-to-JOSS to in-scope here), and the Amendment 1 item-3
cross-reference (annotated superseded). WS-1, WS-2, WS-4, and the runbook are unaffected.

### Amendment 8 (Record-keeping / closure): in-repo done-whens ticked, runbook tail is PENDING-RELEASE-ACTION, plan CLOSEABLE-AFTER-RELEASE — 2026-06-18

**Closure reconciliation (record-keeping, no scope change).** A v0.28 closure pass (read-only
audit of the branch-46 working tree). No workstream added or removed, no done-when changed in
substance, no API surface. It records that **every in-repo done-when has shipped and is verified**,
and isolates the only remaining work as Gary's release actions, not a code gap. Triggered by a
user closure ask (rule 14: closing out a done-when set ahead of archiving), not a critic finding.

**In-repo work verified shipped (✅ ticks added to WS-1..5 above).** The closure grep over the
shipped tracked tree (excluding `.venv/`, generated `docs/source/notebooks/` + `gallery_examples/`,
the `public/` build dir, `.ipynb_checkpoints/` autosave, and the `wiki/` submodule) confirms:
- Every in-repo `geomdata/hiveplotlib` URL / badge / click-through link migrated to the new
  namespace (WS-1, WS-2, WS-4); `LICENSE` + `docs/source/conf.py` copyright now read "Gary Koplik"
  (WS-5); `CONTRIBUTING.md` is the single-maintainer governance rewrite (WS-3, per Amendment 6).
- **Zero stray `geomdata` / `Geometric Data Analytics` references remain except the two intentional
  `CHANGELOG.rst` migration-note lines** (`CHANGELOG.rst:193-194`: the `geomdata` group-name mention
  and the `geomdata/hiveplotlib` old-links mention inside the repo-move note). Those two lines are
  **intentional and left as-is** (the migration note's whole point is to name where the repo moved
  *from*). The `public/changelog.html` and `.ipynb_checkpoints/` hits are generated/autosave
  artifacts, not shipped source.

**Runbook tail is PENDING-RELEASE-ACTION (Gary's, not a code gap).** Runbook steps 8–11 (Amendment
4's order) remain, and they are **release actions the maintainer performs**, not missing
implementation:
- **Step 8** — commit branch-46 and get **green CI in the new `hiveplotlib/hiveplotlib` namespace**
  (delta-tower runners are live per Amendment 5).
- **Step 9** — finish the branch-46 review and **merge 46 + the sweeps to master as v0.28**.
- **Step 10** — trigger `perform_release` (manual) → **v0.28 tag → PyPI + RTD publish** through the
  pipeline (intended path; local `twine` / `rtd_deploy.py` is emergency-only break-glass per
  Amendment 2 item 5).
- **Step 11** — the **external-links pass** Gary controls (CV, personal website, X "moved" note,
  affiliated repos — runbook section 6).

The release stays **hard-gated on the full pipeline green** (Amendment 2 item 3's gate, position
per Amendment 3); there is no rollback (fix-forward only). None of this is harness-dispatchable
work — the sweeps are done; what's left is the maintainer's commit/CI/merge/release/links sequence.

**CLOSEABLE-AFTER-RELEASE.** The plan's automated, in-repo deliverable (WS-1..5) is **complete and
verified**. The plan is **closeable once the runbook tail (steps 8–11) is performed** — i.e. once
v0.28 is merged, tagged, and published and the external links are updated. Archive (move to
`wiki/wiki/plans/archived/`) after release, not before, since the runbook tail is still live; until
then it stays in the active plans dir as in-flight-on-the-release-action. (Research Liaison proposes
the archive move; the maintainer confirms and performs it. This plan is also flagged a strong ADR
candidate — see "Prior ADRs / design docs".)

**Done-whens touched:** none added/removed/changed in substance; ✅ verification ticks added to
WS-1..5's done-when blocks (recording the closure audit), and this entry isolates the runbook tail
as the only remaining (release) work.

*Sections edited:* WS-1..5 "Done when" blocks (✅ closure-verification ticks); "Plan amendments"
(this entry).

### Amendment 7 (Record-keeping): WS-1..5 shipped onto the branch-46 working tree; log resolved, statuses flipped — 2026-06-16

Trigger: Gary confirmed (2026-06-16) the in-repo migration work is complete on the branch-46
working tree (uncommitted, pending his commit + green CI + merge as v0.28). This amendment is a
record-keeping pass: no scope change. It flips WS-1..5 Status fields to done (WS-3's original
URLs-only scope is recorded superseded by the Amendment-6 wholesale rewrite), advances the runbook
progress marker (Phases 1-3 + Phase 4 step 7 sweeps complete; remaining steps are Gary's manual
commit / CI / merge / release / external-links), and populates the Implementation log below.

Two reconciliations to record (shipped reality vs. the plan as written; neither is a scope change):
- **WS-1's expected `graph_features/__init__.py:642` `/-/issues` URL did not exist.** Gary had
  already removed that docstring commentary on branch-46 before the sweep, so there was no live URL
  to migrate; no other live `geomdata` `/-/issues` URL exists in the tree. The Group-A audit row
  for that line is moot (the edit was a no-op because the target was already gone).
- **The `releaser.py` "adapted from GDA-Cookiecutter" comment was scrubbed by Gary**, even though
  the Holdouts section and Amendment 6 kept it as an intentional internal-provenance Holdout. The
  shipped tree therefore carries it gone, which over-satisfies (does not violate) Amendment 6's
  "zero GDA references except the CHANGELOG migration note" goal. The Holdouts text below is left
  intact as historical record; the post-sweep grep audit confirmed the only remaining intentional
  `geomdata` reference is the `CHANGELOG.rst` migration note.

## Holdouts

The replace-and-sweep audit should leave these alone:

- `.gitlab/` directory references everywhere (`Makefile:8`, `pyproject.toml:185`,
  `docs/source/conf.py:115`, `.gitlab/make_*.py`, `.gitlab-ci.yml` job config,
  `.gitlab/gitlab-ci/versioning.yml`, `.gitlab/gitlab-ci/releaser.py`): GitLab's own
  directory convention, not the `geomdata` group name.
- `.gitlab-ci.yml` job names (`release_gitlab`, `pages`, `readthedocs_*`) and
  `python-gitlab` API usage: namespace-agnostic; authenticate via `CI_JOB_TOKEN` against
  the running project and follow the Transfer automatically.
- `.gitlab/gitlab-ci/releaser.py:2` "adapted from GDA-Cookiecutter" comment: internal
  provenance, harmless. (Still a Holdout — internal comment, not a shipped GDA-entity
  reference, so the Amendment 6 "zero GDA references" goal explicitly excludes it.)
- `.gitmodules` submodule URLs: GitHub-hosted, unaffected by the GitLab move.
- `wiki/wiki/plans/*.md` and other wiki-submodule text mentioning `geomdata`: working
  scratch in a separate repo, not shipped artifacts; not a sweep target.

## Implementation log

Append-only. After each workstream completes, one line in the same turn.

**2026-06-16 — WS-1..5 shipped onto the branch-46 working tree (uncommitted, pending Gary's commit + green CI + merge as v0.28). Recorded per Amendment 7.**

- **WS-1 (package metadata + source URLs): done.** `pyproject.toml` Source URL → new namespace and author email → `gary.koplik@gmail.com`; `hiveplot.py` work_items message URL; `international_trade.py` and `runners/make_trade_network_dataset.py` blob URLs migrated. The expected `graph_features/__init__.py:642` `/-/issues` URL did not exist (Gary had already removed that commentary on branch-46); no other live `geomdata` `/-/issues` URL exists.
- **WS-2 (docs, badges, footer): done.** Six badge image URLs across `README.md` + `docs/source/README.md`, the `conf.py` GitLab icon link, `roadmap.rst` and `404.rst` work_items links, GDA footer block removed, and the dead `www.hiveplot.com` link fixed to `https://hiveplot.com` (Amendment 5).
- **WS-3 (CONTRIBUTING URLs): superseded.** Replaced by the wholesale CONTRIBUTING rewrite (Amendment 6); the URLs are carried in that rewrite.
- **WS-4 (notebooks): done.** Runner-link prose and the live `gpd.read_file()` data fetch in `hive_plots_more_than_three_groups.ipynb` (new URL verified HTTP 200), the `v0.27.0_speedups.ipynb` work_items link, and the `introduction_to_hive_plots.ipynb` hiveplot.com image + link (Amendment 5).
- **WS-5 (GDA-entity scrub): done.** `LICENSE:1,14` and `docs/source/conf.py:23` copyright → `Gary Koplik` (LICENSE matches the JOSS-worktree LICENSE for a clean future merge); `CONTRIBUTING.md` replaced with the JOSS governance rewrite pulled forward, then voice-tuned (neutral/passive, single-maintainer framing kept, LinkedIn line kept), given an issue-first numbered flow + fork-submission section, `/-/issues` corrected to `/-/work_items` (deprecated-form regression fixed), and de-emphasized to remove "self-hosted infrastructure" language.
- **Post-sweep verification: grep audit clean.** Gary ran docs / linkcheck / unit tests locally; a `geomdata` / `Geometric Data Analytics` / `GDA` grep audit returned zero shipped references except the intentional `CHANGELOG.rst` migration note. The `releaser.py` GDA-Cookiecutter comment was scrubbed by Gary (over-satisfies Amendment 6's goal; see Amendment 7).
- **CHANGELOG:** repo-move note added to the v0.28.0 Tooling Changes section.
- **Cross-repo:** final CONTRIBUTING mirrored byte-identical into the `hiveplotlib-joss` worktree (Amendment 6 coordination satisfied); runner / CI security posture verified and documented in the `gitlab-runners` repo's `self-hosted-runner-setup.md` (default fork-pipeline-in-fork, protected + masked secrets, MR-protected-resources off, fork flag at default, stale GDA runners unassigned/harmless).

**Remaining (Gary's manual steps, runbook steps 8-11):** commit branch-46, green CI, finish the review and merge as v0.28, `perform_release` → v0.28 tag → PyPI + RTD publish, then the external-links pass.
