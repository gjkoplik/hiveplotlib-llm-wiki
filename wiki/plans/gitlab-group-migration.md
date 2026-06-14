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
  sole-control namespace. Forced order: branch-46 review complete AND ready → Transfer →
  stand up delta-tower runners in the new namespace → full pipeline green → local verify →
  trigger `perform_release` (manual) → v0.28 tag. The release is hard-gated on runners live
  and the whole pipeline working again. Accepted consequence: a window of a moved-but-not-
  yet-releasable repo with some dead links (low traffic, fine).
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
  GDA non-endorsement clause. **Decision needed and legally flavored**: the JOSS plan
  (line 19) flags the LICENSE copyright line as an OPEN item. Out of scope for an
  automated sweep; left to Gary. → manual runbook, not a sweep target
- `CONTRIBUTING.md:5` — "we will restrict developers to employees of Geometric Data
  Analytics". The JOSS plan authorizes a full CONTRIBUTING.md rewrite (single-maintainer
  governance). **Coordinate**: let the JOSS CONTRIBUTING rewrite own this line rather
  than spot-editing it here; this plan only updates the issue-tracker URLs in the same
  file. → WS-3 (URLs only), GDA-developer-policy prose deferred to JOSS Workstream A
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

The four automated workstreams are dispatchable by the harness when Gary triggers the
move. They are **independent string sweeps over disjoint file sets** and can run
concurrently. None of them should run until Gary has (a) performed or scheduled the
GitLab Transfer and (b) settled the two Group-B config decisions (email, footer). The
substitution is mechanical; the risk is sequencing against the live data-fetch URL
(WS-4) and the redirect.

### Workstream 1: Package metadata + source/docstring URLs

**Status:** not started
**Files:** `pyproject.toml`, `src/hiveplotlib/hiveplot.py`, `src/hiveplotlib/graph_features/__init__.py`, `src/hiveplotlib/datasets/international_trade.py`, `runners/make_trade_network_dataset.py`
**Done when:**
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

**Status:** not started
**Files:** `README.md`, `docs/source/README.md`, `docs/source/conf.py`, `docs/source/roadmap.rst`, `docs/source/404.rst`, `docs/source/_templates/footer.html`
**Done when:**
- Every `gitlab.com/geomdata/hiveplotlib` (badge image URLs, click-through links, the
  conf.py GitLab icon-link, `/-/work_items` links) → new namespace, in both
  `README.md` and its `docs/source/README.md` mirror (keep them identical).
- `footer.html` GDA link/logo block removed entirely (resolved). **Idempotent
  (Amendment 1):** if the GDA block is already gone (the JOSS worktree merged first,
  leaving `<p>Release Version: {{release}}</p>`), this item is a satisfied no-op — confirm
  the block is absent and move on; do not re-add, do not error on the missing string.
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

### Workstream 3: CONTRIBUTING issue-tracker URLs

**Status:** not started
**Files:** `CONTRIBUTING.md`
**Done when:**
- The two `/-/work_items` Issue Tracker URLs (`CONTRIBUTING.md:8,14`) → new namespace.
- The GDA-developer-policy prose (`CONTRIBUTING.md:5`) is **left untouched here**; it is
  owned by the JOSS plan's authorized CONTRIBUTING rewrite. Add a one-line note in the
  Implementation log so the JOSS rewrite knows the URLs are already migrated.
- No CHANGELOG entry.

(Kept separate from WS-2 because of the explicit JOSS-coordination boundary on the same
file: this plan touches only the URLs, JOSS owns the governance-prose rewrite.)

### Workstream 4: Notebook prose + live data-fetch URLs

**Status:** not started
**Files:** `examples/hive_plots_more_than_three_groups.ipynb`, `docs/source/blog/v0.27.0_speedups.ipynb`
**Done when:**
- The prose runner-link and the **live `gpd.read_file()` URL** in
  `hive_plots_more_than_three_groups.ipynb` → new namespace.
- The `/-/work_items` prose link in the `v0.27.0_speedups.ipynb` blog notebook → new namespace.
- **Gate:** the data-fetch URL only flips after the Transfer redirect is confirmed live
  (or the new project is up with the `data/` dir present on master). Verify by fetching
  `https://gitlab.com/hiveplotlib/hiveplotlib/-/raw/master/data/countries.shp.zip`
  returns 200 before committing the edit.
- `make test-nb` runs `hive_plots_more_than_three_groups.ipynb` end-to-end against the
  new URL with no fetch error.
- Edits confined to `examples/` and `docs/source/blog/` per CLAUDE.md (no edits to
  generated `docs/source/notebooks/` or `gallery_examples/`).
- No CHANGELOG entry.

## Manual runbook (Gary performs; the harness cannot)

Ordered. Run the harness sweeps (WS-1..4) only where each step says so.

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
**Ordering gate (grill, 2026-06-13):** the Transfer happens only once branch-46 review is
complete AND ready. The Transfer can physically precede the runners (it is just a project
move), but everything downstream of it is hard-gated on the **delta-tower CI runners**
(separate plan) being live and the full pipeline green again before any release. See
Amendment 2 for the forced order and the gate. No rollback exists after this step;
fix-forward only.
- Project Settings → General → Advanced → **Transfer project** → select the new
  `hiveplotlib` group. Confirm.
- What Transfer **preserves**: issues, MRs, stars, full git history, labels, milestones,
  and crucially **auto-creates HTTP redirects** from `geomdata/hiveplotlib` to
  `hiveplotlib/hiveplotlib` (web and git remote paths redirect).
- What Transfer does **not** carry automatically: CI/CD variables and secrets, deploy
  tokens, scheduled pipelines may need re-enabling, any Pages custom domain, and
  external-service linkages (RTD, Zenodo, Codecov, GitHub mirror). Re-establish these in
  section 5.

### 3. Keep the old path alive (do not shadow the redirect)
- With Transfer, the redirect keeps `geomdata/hiveplotlib` URLs working, so old badges,
  bookmarks, and `git remote`s continue to resolve. **Do not** create or recreate any
  project at the old `geomdata/hiveplotlib` path; a new project there would shadow the
  redirect and break inbound links. Leave the old path as the redirect source.

### 4. Run the automated sweeps
- Dispatch WS-1, WS-2, WS-3, WS-4 (harness). WS-2's `make linkcheck` and WS-4's
  `make test-nb` data-fetch only pass once the **new namespace actually serves the URLs**
  (after the Transfer), so run the sweeps after the Transfer, not before.
- **No runners required for the sweeps.** Every sweep done-when runs locally off Gary's
  machine (`make test`, `make docs`, `make linkcheck`, `make test-nb`, badge 200 checks,
  the `countries.shp.zip` fetch). The sweeps are fully verifiable in the no-CI window
  before delta-tower runners exist; the move is not blind. Runners gate only the automated
  pipeline and the release (sections 5, 9, Amendment 2).

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

### 7. Local-clone hygiene for contributors
- Update the local remote: `git remote set-url origin git@gitlab.com:hiveplotlib/hiveplotlib.git`
  (the redirect makes the old remote keep working, but updating avoids surprises).
- **Submodules:** `.gitmodules` points both submodules at GitHub
  (`github.com/gjkoplik/hiveplotlib-agent-harness`,
  `github.com/gjkoplik/hiveplotlib-llm-wiki`) — **verified unaffected** by a GitLab group
  move. No submodule remote/pin change needed.

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
- Full pipeline green, then trigger `perform_release` (manual) to publish PyPI + RTD and
  tag v0.28. This is the **intended release path** and is hard-gated on the runners.
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
   clarity).** `LICENSE:1,14` copyright + clause-3 entity (JOSS Amendment 1, already done
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

3. **Forced ordering, security-driven (external dependency).** None of the CI works today;
   all publishing (PyPI, RTD, GitLab release) runs through GitLab CI, and the runners are
   **self-hosted on Gary's own server ("delta tower")** under a SEPARATE plan. They cannot
   be proven before the Transfer, because exposing machine-level runners to the dissolving
   company is a security risk; they can only be wired to the new, sole-control namespace. So
   the order is FORCED:

   > branch-46 review complete AND ready → **Transfer** → stand up delta-tower runners in the
   > new namespace → full pipeline green → local verify → trigger `perform_release` (manual)
   > → v0.28 tag.

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

6. **Scope freeze — what v0.28 is.** v0.28 = **branch-46 features hardened** (correctness,
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
  provenance, harmless.
- `LICENSE:1,14` (GDA copyright + non-endorsement clause): legally flavored, owned by
  Gary / the JOSS plan's OPEN item, not an automated-sweep target.
- `CONTRIBUTING.md:5` (GDA-developer-policy prose): owned by the JOSS CONTRIBUTING
  rewrite; this plan touches only the URLs in that file.
- `.gitmodules` submodule URLs: GitHub-hosted, unaffected by the GitLab move.
- `wiki/wiki/plans/*.md` and other wiki-submodule text mentioning `geomdata`: working
  scratch in a separate repo, not shipped artifacts; not a sweep target.

## Implementation log

Append-only. After each workstream completes, one line in the same turn.
