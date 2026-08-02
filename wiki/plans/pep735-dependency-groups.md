# Plan: PEP 735 dependency groups for developer-only extras

## Goal

Move the six developer-only extras (`dev`, `docs`, `testing`, `perf`, `ruff`, `ty`) from `[project.optional-dependencies]` to a PEP 735 `[dependency-groups]` table in pyproject.toml, and sweep every consumer of those extras (install.sh, Makefile, .gitlab-ci.yml, .readthedocs.yaml, CONTRIBUTING.md, runners docs/tooling) to the `--group` install forms. Dev-only extras currently ship in published PyPI metadata; dependency groups live only in the source checkout, which is the correct home. User-facing extras (`bokeh`, `datashader`, `holoviews`, `networkx`, `plotly`, `polars`) stay extras, PyPI-installable, untouched. `narwhals` stays in base `dependencies`, untouched (settled; do not relitigate — see repo CLAUDE.md frames.py note). End users are unaffected; the one user-visible consequence is that `pip install hiveplotlib[dev]` (and the other five dev names) from PyPI stops existing, which earns a CHANGELOG entry.

Tooling feasibility (verified at plan time): pip >= 25.1 supports `pip install --group` (`[path:]group`, default `./pyproject.toml`); the local uv is 0.11.21 and its `uv pip install --help` lists `--group <GROUP>` ("install the specified dependency group from a pylock.toml or pyproject.toml"). CI uses `ghcr.io/astral-sh/uv:python3.12-trixie-slim` images (tracks current uv, well past `--group` support). RTD and the GPU-venv path use plain pip and need a pip upgrade step to clear the 25.1 floor.

Brief-mode gate: knowingly skipped — decisions extracted conversationally with the maintainer before planning; the post-plan grill still runs.

## Alignment (grill)

### Maintainer shared-understanding pass (grill), Wave 1 — premise, scope, open questions (2026-07-22)

- **Premise driver (open question 4):** confirmed — cleaner, more modern tooling (metadata hygiene plus ecosystem alignment) is the driver; no blocked workflow claimed, costs accepted as priced. Goal stands as written.
- **Smaller move (open question 5):** rejected — full migration including `docs`; Workstream D stays. A half-migration leaves one dev extra published and muddies the story; the RTD `build.jobs.install` pattern is officially documented.
- **lint_check / format_check (open question 1):** maintainer indifferent, delegated to the dispatching session on maintainability grounds; call is **leave as holdouts** (group entry and CI install are both unpinned `ruff`, so no drift is possible; aligning adds a checkout-parse dependency and pip-floor step for zero behavior change).
- **`make format` (open question 2):** confirmed — keep `-U` (`uv pip install -U --group ruff`), matching CI's latest-ruff behavior.
- **CHANGELOG phrasing (open question 3):** confirmed — entry explicitly names the lost `pip install hiveplotlib[dev]`-style paths and the replacement.
- **New maintainer ask (emergent, routed to amend-plan):** CONTRIBUTING gains a general note that dev installs need pip >= 25.1 or uv (beyond the GPU-venv-block one-liner already in Workstream B).

Wave 2 (correctness) surfaced no divergence (inventory adversary-verified; timing risks priced in Wave 1). The failure-mode wave ran 2026-07-22; the four named modes are recorded under `## Failure modes`.

## Failure modes

Named by the maintainer at the grill's failure-mode wave (2026-07-22):

- The published wheel still advertises dev extras somehow — the metadata-hygiene claim, the entire point, silently fails while all CI is green.
- A contributor follows CONTRIBUTING verbatim on a fresh machine and hits a wall the docs don't explain (old pip, missing group, wrong directory) — the "modern tooling" story costs the contributor experience it was supposed to improve.
- RTD silently builds the docs with the wrong deps — the build goes green but the environment isn't what the config claims.
- Any user is blocked from installing the code as they need to, from what's supposed to work from the user perspective.

## Prior ADRs / design docs

- `wiki/wiki/adr/0002-performance-regression-harness.md` — the perf CI lane installs perf+testing fail-loud with no dependency skipifs, and `perf` self-references `hiveplotlib[datashader]` so the two can't drift. The migration must preserve both: the `perf` group keeps the `hiveplotlib[datashader]` package-spec self-reference, and `performance_gates` keeps installing both perf and testing.
- `wiki/wiki/adr/0001-networkx-integration.md` — `networkx` stays a user-facing extra (pulls scipy); nx-parallel is deliberately dev-only (it rides in the `dev` group). The user-extras vs dev-groups boundary this plan draws matches that ADR; don't blur it.
- `wiki/wiki/plans/archived/i-want-to-plan-optimized-hoare.md` (~914-918, 1311) — prior pyproject restructure: `dev` chains all backend extras via `hiveplotlib[<extra>]` self-references as a structural completeness guarantee, which justified deleting the meta-test `tests/pyproject_toml_test.py`. This plan re-affirms that design: the self-references move into the `dev` group verbatim (see composition decision under Workstream A), so the deleted-meta-test rationale still holds and the meta-test stays deleted.
- `wiki/wiki/plans/conda-forge-distribution.md` — its per-extra closures cite `pyproject.toml:26-105` line ranges, which this migration will drift. Its extras-mapping decisions concern only the kept backend extras, so no substantive conflict; the line-anchor drift is flagged for that plan's next touch, not fixed here.
- `wiki/wiki/plans/joss-submission.md` — plans a CONTRIBUTING.md rewrite and a pyproject SPDX license change. This plan's CONTRIBUTING edits are surgical (install command lines only) to minimize churn against that rewrite; the pyproject `[project]` table is untouched here.
- `wiki/wiki/plans/scaling-large-networks.md` (~2391) — shipped must-fix: CONTRIBUTING install lines must be runnable verbatim (pytest.ini addopts need testing deps present). Carried as a done-when on Workstream B.
- `wiki/wiki/plans/llm-friendly-docs.md` (~105) — .readthedocs.yaml runs with `fail_on_warning: true`; the Workstream D restructure must keep that behavior and a warning-clean build.
- `wiki/wiki/plans/archived/gitlab-group-migration.md` (~448-477) — release topology: publishing runs through GitLab CI (`release_gitlab`, `readthedocs_*` via rtd_deploy.py). None of those jobs install dev extras, but the weekly released-code jobs the sweep does touch sit in the same pipeline; the sweep must not disturb the release-critical jobs.
- Gap noted by research-liaison: no wiki page records the weekly released-code matrix / `tests-<tag>` convention. The `testing`-group design decision for those jobs is therefore recorded explicitly under Workstream C rather than treated as a mechanical rename.

## Patterns this replaces

Old pattern: dev-only names in `[project.optional-dependencies]` and `.[...]` / `hiveplotlib[...]` install forms naming them. Replace with a `[dependency-groups]` table and `--group` install forms.

- `pyproject.toml:30-119` — `dev`, `docs`, `perf`, `ruff`, `testing`, `ty` entries under `[project.optional-dependencies]` → `[dependency-groups]`.
- `install.sh:7` — `uv pip install -e .[dev,docs,perf,ruff,testing,ty]` → editable install plus six `--group` flags.
- `Makefile:73` — `uv pip install -U ruff` in the `format` target → `uv pip install -U --group ruff` (group becomes the single naming source; `-U` keeps the refresh-to-latest behavior matching CI's unpinned ruff).
- `Makefile:155` — commented-out `# uv pip install -U ty` in the `ty` target → comment updated to the `--group ty` form (a `-U` here would break the `ty==0.0.20` pin; the group form can't).
- `.gitlab-ci.yml:37,67` — `uv pip install -e .[dev,testing]` (unit_tests, examples) → `-e . --group dev --group testing`.
- `.gitlab-ci.yml:53` — `uv pip install -e .[perf,testing]` (performance_gates) → `-e . --group perf --group testing`.
- `.gitlab-ci.yml:116` — `uv pip install -e .[dev,testing,ty]` (ty_check) → `-e . --group dev --group testing --group ty`.
- `.gitlab-ci.yml:150,164` — `uv pip install -e .[dev,docs]` (check_pages_external_links, .pages) → `-e . --group dev --group docs`.
- `.gitlab-ci.yml:209` — `uv pip install -e .[testing]` (base_installs) → `-e . --group testing`.
- `.gitlab-ci.yml:235` — `uv pip install -e .[$ADDON,testing]` (add_ons_installs) → `-e .[$ADDON] --group testing` ($ADDON is always a kept extra).
- `.gitlab-ci.yml:270` — `uv pip install hiveplotlib[testing]==${VER:1}` (.test_released_code_base) → `uv pip install hiveplotlib==${VER:1}` plus `uv pip install --group testing` read from the checked-out tag's pyproject.toml (the job checks out the tag or its `tests-<tag>` branch before installing, so the groups table is present in-tree even though groups are not PyPI-installable).
- `.gitlab-ci.yml:323` — `uv pip install hiveplotlib[$ADDON,testing]==${VER:1}` (.test_released_code_add_ons) → `uv pip install hiveplotlib[$ADDON]==${VER:1}` plus `uv pip install --group testing`, same checkout reasoning.
- `.readthedocs.yaml:29-39` — `python.install` block with `docs` in `extra_requirements` → `build.jobs.install` steps (pip upgrade, then one `pip install .[<backend extras>] --group docs` invocation), per RTD's documented "install dependencies from dependency groups" pattern. Single-invocation form authorized by the maintainer in Amendment 8.
- `CONTRIBUTING.md:126` — GPU venv `pip install -e "<path/to/repository>[datashader,testing]"` → editable install with the `datashader` extra plus `pip install --group testing` run from the repo directory, with a preceding `pip install --upgrade pip` (a fresh `python -m venv` pip predates the 25.1 `--group` floor).
- `runners/performance/README.md:73` — `pip install -e ".[perf,testing]"` → `pip install -e . --group perf --group testing`.
- `runners/performance/README.md:161` — `uv pip install -e .[perf]` → `uv pip install -e . --group perf`.
- `runners/performance/profiling_utils.py:294` — error message "install the 'perf' extra (pip install 'hiveplotlib[perf]')" → name the `perf` dependency group and the source-checkout `--group` install (this is repo tooling under `runners/`, not shipped package code; the message must not imply a PyPI-installable form, since groups aren't).
- `tests/performance_measurement_test.py:83` — docstring citing `pip install -e ".[perf,testing]"` as the documented install → the `--group` form, matching the updated README.
- `CHANGELOG.rst:81` — unreleased 0.29.0 entry saying "The ``hiveplotlib[perf]`` extra now pulls everything the harness needs" → reworded to the `perf` dependency group (added by Amendment 6; unreleased entries are living text, unlike the historical holdout lines, now at 280/283).

QA greps post-execution for `[\[,](dev|docs|perf|ruff|testing|ty)[,\]]` and `optional-dependencies` survivors; anything outside `## Holdouts` fails. (Pattern broadened per adversary must-fix, Amendment 1: the comma-preceded forms — `[datashader,testing]` at CONTRIBUTING.md:126, `[$ADDON,testing]` at .gitlab-ci.yml:235,323 — must match too.)

## Default justifications

No new user-facing defaults. The migration introduces no API parameters; group names and contents are carried over 1:1.

## Naming audit

- Group names: `dev`, `docs`, `testing`, `perf`, `ruff`, `ty` — kept identical to the extras they replace. `dev` is also the PEP 735 / uv ecosystem convention for the default developer group, so the dominant-ecosystem vocabulary check passes with no renames. Renaming any group would multiply the sweep for zero user value.
- New prose-only term: "dependency group" (in CONTRIBUTING, CHANGELOG, and the profiling_utils error message) — the PEP 735 term, matching what pip/uv docs call it.
- Internal composition: package-spec self-references (`hiveplotlib[datashader]` etc.) keep their existing names since the referenced extras are unchanged.

## API usage examples

No Python API surface change. The user-facing "API" here is the install command per persona; each mapping below is exact and runnable, and each has a verification witness named in the owning workstream's done-when.

```bash
# Example 1: end user installing from PyPI (UNCHANGED — the point of the migration)
pip install hiveplotlib
pip install hiveplotlib[datashader]

# Example 2: contributor dev setup (install.sh, run via `make install`)
uv venv --python 3.12
source .venv/bin/activate
uv pip install -e . --group dev --group docs --group perf --group ruff --group testing --group ty

# Example 3: CI unit-test job (.gitlab-ci.yml unit_tests)
uv pip install -e . --group dev --group testing

# Example 4: CI weekly released-code job (after checking out the release tag)
uv pip install hiveplotlib==${VER:1}
uv pip install --group testing   # reads the checked-out tag's pyproject.toml

# Example 5: Read the Docs (build.jobs.install steps)
# single install invocation (maintainer-authorized, Amendment 8): extras and the
# docs group resolve together, so a second invocation can't up/downgrade the first's
# resolution
pip install --upgrade pip
pip install .[bokeh,datashader,holoviews,networkx,plotly] --group docs

# Example 6: GPU verification venv (CONTRIBUTING.md; plain pip, run from the repo)
python -m venv ~/venvs/hiveplotlib-gpu
~/venvs/hiveplotlib-gpu/bin/pip install --upgrade pip
~/venvs/hiveplotlib-gpu/bin/pip install cupy-cuda12x cudf-cu12 dask-cudf-cu12
cd <path/to/repository>
~/venvs/hiveplotlib-gpu/bin/pip install -e ".[datashader]" --group testing
```

### API Critic's take (planning mode)

Not applicable — no Python API surface change; api-critic is not invoked on this plan (packaging/CI/docs only). The adversary planning pass and post-impl critics still apply.

### API Critic — post-implementation review

Not applicable — no user-facing Python API change (see above).

## Notebook review

No notebook change. Notebook install-instruction cells reference only the kept user-facing extras (verified by grep; e.g. `pip install hiveplotlib[networkx]`).

## Viz review

No figure change.

## Adversary review

### Adversary's challenge (planning mode)

Status: challenge (5 items)
Plan reviewed: wiki/wiki/plans/pep735-dependency-groups.md (cold)
Items:
  - [must-fix] The plan's own QA survivor-grep (`\[(dev|docs|perf|ruff|testing|ty)[,\]]`) cannot match a dev name preceded by a comma, so the two mixed-form sites the sweep itself names — `CONTRIBUTING.md:126` (`[datashader,testing]`) and `.gitlab-ci.yml:235,323` (`[$ADDON,testing]`) — are invisible to it. If an implementer misses one of those exact sites, the enforcement gate stays green. — at "Patterns this replaces" (QA grep line)
    Rubric: no rubric yet — pre-grill
    Push: broaden the pattern to `[\[,](dev|docs|perf|ruff|testing|ty)[,\]]` (or run a second comma-anchored pass) so the gate covers every form the sweep touches. Verified against the working tree: my broader grep found no sites beyond the plan's inventory, so the inventory itself is complete — only the gate is holed.
  - [must-fix] The `uv_audit` CI job (`.gitlab-ci.yml:124-135`, bare `uv audit` at repo root, pinned uv 0.11.21 image) is a consumer of pyproject dependency structure the sweep never inventories. Moving six extras into `[dependency-groups]` may silently change what that security audit covers (whether uv audit includes groups by default, and which ones, is unverified in the plan). A dependency-configuration migration that narrows a security-audit surface without a recorded decision is exactly what the qa security trip-wire on Workstreams A/C exists to catch. — at Workstream C Files list / "Patterns this replaces"
    Rubric: no rubric yet — pre-grill
    Push: verify `uv audit`'s group handling on the pinned image and add a Workstream A or C done-when bullet recording the outcome (groups covered, or deliberately excluded with the reason); the job line itself may need no edit, but the coverage decision must be on the record.
  - [must-fix] Workstream A's done-when proves the self-referencing groups "resolve and install cleanly" but never checks *what* satisfied the `hiveplotlib[bokeh]`/`hiveplotlib[datashader]` package-spec self-references. As a group entry this is an ordinary requirement, not an intra-dist extra: if the resolver ever satisfies it from PyPI instead of the co-installed editable, unit_tests/examples/ty_check silently run released code under a green pipeline — the highest-cost failure shape available here, and none of those jobs carries the `uv pip freeze | grep hiveplotlib` check the weekly jobs have. — at Workstream A done-when
    Rubric: no rubric yet — pre-grill
    Push: add one done-when line: after the scratch-venv `--group dev` install, `uv pip freeze | grep hiveplotlib` (or `pip show`) must show the local editable path, not a PyPI version. Note the plan's "resolves against the co-installed editable/local project under both uv and pip" claim is only ever exercised under uv (no plain-pip consumer installs a self-referencing group), so the pip half of that sentence is untested prose; scope the claim to what the witness covers.
  - [worth-discussing] Premise: no user workflow blocked today is named. The stated benefit is metadata hygiene ("dev extras ship in published metadata"), which is real but aesthetic on its own; the priced costs are a pip >= 25.1 floor in three places, an RTD restructure whose verification only happens on the first live build under `fail_on_warning: true`, an accepted red-weekly-job window if the release stalls, and the loss of `pip install hiveplotlib[dev]` for anyone using it. Likely live drivers exist (cleaner extras surface for the conda-forge mapping work; a leaner CONTRIBUTING story before JOSS) but the plan doesn't claim them. — at Goal
    Rubric: no rubric yet — pre-grill
    Push: name the concrete driver in the Goal (or confirm in the grill that hygiene-plus-ecosystem-alignment alone is worth this cost set). The size-and-maintenance angle otherwise favors the plan — it deletes six names of published surface — so this is a make-the-tradeoff-explicit push, not a kill.
  - [worth-discussing] The smaller move is unmentioned: migrate five groups and keep `docs` as an extra, leaving `.readthedocs.yaml` untouched. Workstream D is the plan's riskiest piece (only verifiable post-push, on a fail-on-warning build) and exists solely because `docs` moves; RTD is a machine consumer that extras serve fine. The counterargument (a half-migration muddies the "dev names gone from PyPI metadata" story and leaves one stray dev extra published) may well win, but the plan never has it. — at Workstream D / Goal
    Rubric: no rubric yet — pre-grill
    Push: record the docs-stays-an-extra alternative as considered-and-rejected with the reason, or adopt it and drop Workstream D.

Verified in this pass (no challenge): the call-site inventory is complete against the working tree (a broadened grep across the repo, excluding generated docs and submodules, surfaced no site outside "Patterns this replaces" + Holdouts); the cited line numbers are all accurate; the Holdouts entries check out (lint_check/format_check at `.gitlab-ci.yml:85,100` genuinely never referenced the `ruff` extra; the CHANGELOG lines are historical; `asv.conf.json:31` names a kept extra); the ADR 0002 perf/testing fail-loud contract and the hoare-plan self-reference design are correctly carried; Example 5's extras set matches the current RTD file exactly (`polars` excluded today too); the weekly-job checkout-then-`--group` design is coherent with the `tests-<tag>` convention and its pre-migration-tag failure window is explicitly accepted by the maintainer, not smuggled.

**Addendum (post-grill rubric-check, 2026-07-22):**

Status: challenge (1 new item)
Plan reviewed: wiki/wiki/plans/pep735-dependency-groups.md (cold, post-grill rubric-check)
Delta against the four newly named failure modes only; the cold pass already priced mode 4's accepted loss, so this is a teeth check on the rest. Mode 1 (wheel metadata): covered — Workstream A's done-when carries the direct `Provides-Extra` witness plus the QA survivor-grep. Mode 2 (contributor hits a wall): covered — initially flagged here as a gap (the grill's emergent CONTRIBUTING ask was recorded in "Alignment (grill)" but unlanded), but Amendment 4 landed it into Workstream B's Files and done-when concurrently with this check; verified present, no item stands. Mode 4 (user blocked from installing): covered — extras untouched, Example 1 unchanged, notebook-cell grep, CHANGELOG names the one lost path, and the weekly released-code jobs exercise the PyPI install. Mode 3 (RTD silently wrong deps) is the one gap:
  - [worth-discussing] The RTD mode as named is the *silent-green* shape, but Workstream D's teeth (fail_on_warning, "watch the first latest build") only catch loud failures; the watch names no witness for a green build with the wrong environment. The preserved `pre_build` pip-freeze step is the natural witness and the plan never says to read it. — at Workstream D done-when
    Rubric: "RTD silently builds the docs with the wrong deps"
    Push: extend D's residual-risk flag with one line: on the first post-migration RTD build, check the pip-freeze output in the build log lists the docs-group packages (sphinx et al. at expected versions), not just that the build went green.

### Adversary post-impl

```
Pending — invoke the adversary in post-impl mode after each workstream ships.
```

### Adversary post-impl — Workstream A

Status: propose
Artifact reviewed: Workstream A — pyproject.toml migration (working tree, uncommitted)
Dispositions held: yes — Amendment 3's editable-origin witness landed in the done-when and the Implementation log records the freeze output (`-e file:///home/garyk/repos/hiveplotlib`); the composition decision held exactly (six extras in `[project.optional-dependencies]`, six groups in `[dependency-groups]`, self-references only in `dev`/`perf`, no `include-group` edges, no umbrella group, `ty==0.0.20` and the narwhals pin intact, explanatory comments carried); no scope balloon — the diff is confined to the one declared file.
Concerns:
  - [low-confidence] The four runtime done-when witnesses (six-group scratch install, `--group testing` alone, editable-origin freeze, wheel `Provides-Extra` inspection) are recorded as executed in the Implementation log but leave no in-tree artifact, and this reviewer ran no shell (no diff vs HEAD, no witness re-run); the static file passes every checkable bullet, but the load-bearing claims rest on the log entry. Acceptable for a migration gate; noted so qa knows the verification chain. — at pyproject.toml:30-121
    Rubric: "The published wheel still advertises dev extras somehow"
  - [low-confidence] The no-dev-`Provides-Extra` invariant is one-shot verified at migration time; nothing in CI guards it against future drift (a later edit re-adding a dev name under `[project.optional-dependencies]` goes green). The plan deliberately kept the `tests/pyproject_toml_test.py` meta-test deleted via the structural-completeness argument, which covers group topology, not extras-table re-growth, so this is an observation for the maintainer, not a re-litigation: the rubric's mode 1 is phrased as a silent-while-CI-green failure, and post-migration that shape remains possible. — at pyproject.toml:30
    Rubric: "The published wheel still advertises dev extras somehow"

### Adversary post-impl — Workstream B

Status: propose
Artifact reviewed: Workstream B — local dev tooling sweep (install.sh, Makefile, CONTRIBUTING.md; working tree, uncommitted)
Dispositions held: yes — Amendment 4's general note landed verbatim in the dev-setup section (CONTRIBUTING.md:56-58: PEP 735 groups, pip >= 25.1 or uv, `make install`-handles-it framing, no process references); Amendment 3's rescope claim ("the GPU venv installs only `testing`") verified true against the artifact — the GPU block's `--group testing` pulls a group with no self-referencing `hiveplotlib[...]` entries, so the plain-pip path never exercises the uv-only resolution claim; Amendment 1's broadened survivor-grep run over all three files surfaces only the kept `[datashader]` user extra. No scope balloon: the diff surface is exactly the declared files/lines (install.sh whole, Makefile:73 and the :155 comment, CONTRIBUTING dev-setup note + GPU block). Every checkable done-when bullet passes: install.sh matches Example 2 line-for-line (same group order); the GPU block (CONTRIBUTING.md:128-132) matches Example 6 exactly, with the pip-upgrade reason at :135-136; the walked-verbatim fresh-shell check holds for the GPU block (venv → pip upgrade → CUDA wheels → cd repo → editable + `--group testing`, with `--group` reading pyproject.toml from the cwd the block just established, and all six groups confirmed present in pyproject.toml).
Concerns:
  - [worth-discussing] The happy path still has an undocumented prerequisite: `make install` → install.sh runs `uv venv`, and the new note names uv as the tool that "handles this," but CONTRIBUTING never says how to get uv — a fresh machine hits `uv: command not found` with no pointer. Pre-existing gap, not introduced by this diff, but the migration raises its price (pre-migration a manual `pip install -e ".[dev]"` worked on any pip; now the only routes are uv or pip >= 25.1), and the section was in-scope this workstream. One link to uv's install docs closes it. No bearing on downstream workstreams C/D/E. — at CONTRIBUTING.md:48-58
    Rubric: "A contributor follows CONTRIBUTING verbatim on a fresh machine and hits a wall the docs don't explain"
  - [low-confidence] The two execution gates ("`make format` still runs clean"; GPU block runnable on a real CUDA machine) were not run in this read-only pass. Statically sound: the standalone-group form `uv pip install -U --group ruff` is valid uv syntax, the `ruff`/`testing` groups exist, and make runs from the repo root the `--group` default path needs — but the runs-clean witness rests with qa, not this review. — at Makefile:73
    Rubric: "A contributor follows CONTRIBUTING verbatim on a fresh machine and hits a wall the docs don't explain"

### Adversary post-impl — Workstream C

Status: propose
Artifact reviewed: Workstream C — .gitlab-ci.yml sweep (working tree, uncommitted)
Dispositions held: yes — Amendment 1's broadened grep has real teeth here (the two `[$ADDON,testing]` sites at old lines 235/323 are gone; shipped forms are `-e .[$ADDON] --group testing` and `hiveplotlib[$ADDON]==${VER:1}` + separate `--group testing`, no comma-form survivors); Amendment 2 held exactly (uv_audit job untouched at lines 124-135, pinned 0.11.21 image intact, coverage decision recorded in the done-when with the first-post-migration-pipeline re-confirm as a named forward watch); the grill's leave-the-holdouts call held (lint_check/format_check keep bare `pip install ruff` at lines 85/100); no scope balloon — release-critical jobs (readthedocs_latest, the versioning include) and every non-install line are untouched, and variable expansion survived the rewrite everywhere it matters (`$ADDON` at 235/237 and 325/331, `${VER:1}` at 270/325, `$PYVERSION` template inheritance unchanged). All ten mapped lines carry the exact forms from "Patterns this replaces"; the split-install shape in the released-code templates matches the recorded testing-group design decision (checkout-then-`--group`, groups table read from the tag's tree, `rm -rf src` leaves pyproject.toml in place); `performance_gates` still installs `--group perf --group testing` fail-loud per ADR 0002. Install order in the split jobs is the safe direction: package pin first, group second, and `testing` carries no `hiveplotlib[...]` self-reference (self-refs live only in `dev` and `perf`), so the `--group testing` step cannot re-source or re-pin the PyPI install.
Concerns:
  - [low-confidence] The "should see a remote install" guard (`uv pip freeze | grep hiveplotlib`) is presence-only: it cannot distinguish the PyPI wheel from a locally-built origin, and the sweep just added a second install step after the pin that the guard implicitly vouches for. Today that step is inert w.r.t. hiveplotlib (no self-reference in `testing`), so this is a latent-guard observation, not a live defect; it goes live only if `testing` ever gains a self-reference. — at .gitlab-ci.yml:274,329
    Rubric: "Any user is blocked from installing the code as they need to" (nearest mode; the weekly jobs are its sentinel), thin fit — no entry squarely covers guard fidelity.
  - [low-confidence] The editable-origin witness that disposed planning must-fix 3 ran once, locally, on the maintainer's uv; the CI jobs consuming self-referencing groups (unit_tests, examples, ty_check, docs jobs at lines 37/67/116/150/164) run unpinned latest-uv images and carry no freeze guard of their own, consistent with the accepted disposition (a one-time scratch witness, not a CI check). The residual: a future uv resolution change could satisfy `hiveplotlib[bokeh]` from PyPI in CI with everything green. Accepted shape; recorded so the decision's boundary is visible. — at .gitlab-ci.yml:37
    Rubric: no entry (dev-CI-only shape; the named modes are user/contributor/RTD-facing).
Forward watches carried, not closeable from the working tree: the uv_audit findings-or-clean re-confirm and the pre-migration-tag red-weekly window both resolve on the first post-migration pipeline/release, per the done-when and the accepted release sequencing. Yml validity and the two local-verification blind spots are recorded as checked in the Implementation log (yaml parse, repo-wide ruff, CI-only diff); this reviewer ran no shell and verified the static file only.

### Adversary post-impl — Workstream D

Status: propose
Artifact reviewed: Workstream D — .readthedocs.yaml restructure (working tree, uncommitted)
Dispositions held: yes — the grill's rejection of the docs-stays-an-extra smaller move (planning item 5) held: Workstream D shipped exactly as scoped, one file, no scope balloon. Amendment 5's freeze-log witness (from the post-grill rubric-check item on failure mode 3) is structurally honored in the artifact: the `pre_build` `pip freeze` step is preserved verbatim and ordered after the `build.jobs.install` steps, and the Implementation log records the residual-risk flag per Amendment 5.
Blind-pass verification (RTD v2 semantics, reasoned not executed): the three install steps match Example 5 verbatim; `pre_build` (ipython install, pip freeze, notebook copy) and `sphinx.fail_on_warning: true` are preserved unchanged; the key structure is valid v2 (`build.jobs.install` as a command list is the documented override form). Semantics checks against failure mode 3: (a) overriding `build.jobs.install` fully replaces RTD's default install job, including RTD's own default sphinx install — covered, since the `docs` group carries sphinx and the rest of the witness list; (b) overridden and `pre_*` jobs run with the build venv activated, so `pip` in every step resolves to the same environment and the `pre_build` freeze genuinely observes the install job's result — the Amendment 5 witness mechanism is sound; (c) `pip install --group docs` resolves `./pyproject.toml` at the checkout root, the same cwd assumption the pre-existing `cp examples/*.ipynb` step has always made; (d) the pip >= 25.1 floor fails loud, not silent (an old pip errors on the unknown `--group` option), and the upgrade step precedes it. No CRLF introduced; the in-file comment is self-contained with no process references.
Concerns:
  - [low-confidence] The extras install and the group install are two separate pip invocations resolved independently; the second can upgrade or downgrade a dependency the first resolved, with pip emitting only a resolver-conflict warning in a build log nobody gates on — the one silent sliver of failure mode 3 the freeze witness catches only if the reader compares versions, not just presence. A single joint invocation (`pip install ".[bokeh,datashader,holoviews,networkx,plotly]" --group docs`) would resolve both in one pass. The shipped file matches Example 5 as written, so this targets the plan's contract, not the implementation. — at .readthedocs.yaml:18-19
    Rubric: "RTD silently builds the docs with the wrong deps"
  - [low-confidence] Bookkeeping: Workstream D's `**Status:**` field still reads "not started" while the Implementation log records it complete (the other shipped workstreams' fields are equally stale); a stale status field can misroute a future dispatch or qa pass that trusts the field over the log. — at plan Workstream D block (Status line)
    Rubric: no entry
Forward watch carried, not closeable from the working tree: real RTD verification happens only on the first post-migration `latest` build; per Amendment 5 the maintainer reads the build log's `pip freeze` output for the docs-group packages (sphinx, nbsphinx, myst-parser, pydata-sphinx-theme, sphinx-copybutton, ipywidgets, toml), not just the green check. This reviewer ran no shell and verified the static file only.

### Adversary post-impl — Workstream E

Status: propose
Artifact reviewed: Workstream E — prose/tooling stragglers + CHANGELOG (working tree, uncommitted, including the Amendment 6 reopen)
Dispositions held: yes — the broadened QA grep (Amendment 1) runs clean on `runners/performance/` and the test file; the CHANGELOG entry names the lost `hiveplotlib[dev]`-style path per grill question 3's disposition; Amendment 6's reword at CHANGELOG.rst:81 shipped and is accurate against pyproject (the `perf` group does pull `hiveplotlib[datashader]` plus psutil, so both the reworded claim and the profiling_utils remedy are true); the holdout set is exactly as recorded (89 migration-entry mention, 282/285 historical, nothing else in the four files); the profiling_utils message gives a working source-checkout form (`pip install -e . --group perf`) with no PyPI implication; the CHANGELOG entry names all six group names, the correct pip >= 25.1 floor, and the uv alternative; no process references, no em-dashes, no scope balloon beyond the four declared files.
Concerns:
  - [worth-discussing] Two legitimate survivors of the broadened QA grep sit outside `## Holdouts`: `CHANGELOG.rst:88` (the migration entry's own ``[project.optional-dependencies]`` mention matches the grep's `optional-dependencies` clause) and pyproject.toml's kept `[project.optional-dependencies]` table heading for the six user-facing extras. Per the gate's letter ("anything outside `## Holdouts` fails") the plan-end qa sweep will either false-fail or ad-hoc waive them; bears on the not-yet-run qa close, so record both as holdouts before qa runs. — at CHANGELOG.rst:88
    Rubric: no entry (enforcement bookkeeping, not a shipped-text defect)
  - [worth-discussing] `runners/performance/README.md:73` presents plain-pip `pip install -e . --group perf --group testing` as "the sufficient install" with no pip >= 25.1 floor anywhere in the file and no pointer to CONTRIBUTING's note (Amendment 4 centralized the floor there); a contributor on default system pip hits "no such option: --group" with nothing in this doc explaining it. In-contract (the mapped form from "Patterns this replaces" shipped verbatim) but squarely failure mode 2's shape; a half-line "(pip >= 25.1, or uv)" or a CONTRIBUTING pointer closes it. No downstream workstream bearing (all five shipped); batchable to plan-end qa. — at runners/performance/README.md:73
    Rubric: "A contributor follows CONTRIBUTING verbatim on a fresh machine and hits a wall the docs don't explain"
  - [low-confidence] The CHANGELOG replacement recipe (`pip install --group <name>` from a source checkout) installs only the group's dependencies, not hiveplotlib itself, whereas the removed `pip install hiveplotlib[dev]` installed both; the fully equivalent replacement is `pip install -e . --group <name>` (the form the perf README and profiling_utils correctly use). The entry shipped the done-when's phrasing verbatim, so this is a completeness nit against the migrating user's mental model, not a contract miss. — at CHANGELOG.rst:90
    Rubric: "Any user is blocked from installing the code as they need to" (marginal)
Not re-verifiable from the static tree: `make test` (1591 passed / 100% coverage) and the repo-wide ruff run rest on the Implementation log entry; this reviewer ran no shell.

## Workstreams

Sequencing: A first (everything else consumes the groups table). B, C, D, E can then run in any order or concurrently (disjoint files). Release sequencing (settled with the maintainer; clarified in Amendment 8): this branch merges to master only when the maintainer is essentially ready to cut the release, and the release follows immediately, so the first post-migration tag carries the groups table before the next weekly scheduled run. The red-weekly window is therefore near zero rather than merely accepted — the risk is smaller than originally priced, not just tolerated. No fallback shim is wanted. The `base_just_released` / `add_ons_just_released` jobs are self-consistent (same commit supplies the CI yml and the tag).

Process notes (binding):

- Agents never commit or push; the maintainer reviews everything before any commit (explicitly re-confirmed for this plan).
- No process references in shipped code/docs: CONTRIBUTING, CHANGELOG, comments, and error messages carry no plan/workstream/grill citations.
- Verification installs run in a scratch venv (e.g. under /tmp), never by re-running install.sh against the working `.venv` (install.sh starts by uninstalling the env; do not destroy the maintainer's live environment mid-flight).
- qa-engineer's security audit applies to Workstreams A, C, and D (dependency configuration and CI/publishing surfaces); its checklist component must not report `n/a` for them. The performance check is `n/a (no executable change)` territory only if the diff stays config/prose-only — Workstream E touches `runners/performance/profiling_utils.py` (an error-message string) and a test docstring, both non-executable-path changes, but say so explicitly rather than defaulting.

### Workstream A: pyproject.toml migration

**Status:** complete
**Files:** `pyproject.toml`
**Composition decision (deliberate, not mechanical):** the six groups move to `[dependency-groups]` with contents verbatim. Backend-extra coupling stays via package-spec self-references (`hiveplotlib[bokeh]`, ..., in `dev`; `hiveplotlib[datashader]` in `perf`): PEP 735's `include-group` can only reference other groups, not extras, so the self-reference is the only mechanism that preserves the coupling, and it resolves against the co-installed editable/local project under uv (the only tool that installs the self-referencing groups: every `dev`/`perf` consumer is a uv call site; the plain-pip consumers — RTD and the GPU venv — install only `docs` and `testing`, which carry no self-references, so no pip-resolution claim is made or needed). No `include-group` edges are introduced: no group currently includes another, composition happens at call sites (matching current usage), and no umbrella group is invented. This preserves the hoare-plan structural-completeness guarantee, so the deleted `tests/pyproject_toml_test.py` meta-test stays deleted.
**Done when:** `[project.optional-dependencies]` contains exactly the six user-facing extras (`bokeh`, `datashader`, `holoviews`, `networkx`, `plotly`, `polars`); `[dependency-groups]` contains the six dev groups with contents, pins (`ty==0.0.20`, `narwhals` untouched in base deps), and explanatory comments carried over; a scratch-venv `uv pip install -e . --group dev --group docs --group perf --group ruff --group testing --group ty` resolves and installs cleanly (this is the runnability witness for Examples 2-3); `uv pip install --group testing` alone resolves in a scratch venv (witness for Example 4's second line); after the scratch-venv `--group dev` install, `uv pip freeze | grep hiveplotlib` (or `uv pip show hiveplotlib`) shows the local editable path, not a PyPI release — the editable-origin witness proving the `hiveplotlib[bokeh]`/`hiveplotlib[datashader]` self-references were satisfied by the co-installed local project, since a PyPI-satisfied resolution would silently run released code under a green pipeline; `python -m build`-produced metadata (or `uv build` + wheel METADATA inspection) shows no `Provides-Extra` for the six dev names.

### Workstream B: local dev tooling sweep (install.sh, Makefile, CONTRIBUTING.md)

**Status:** complete
**Files:** `install.sh`, `Makefile` (format target line ~73, ty target comment line ~155), `CONTRIBUTING.md` (GPU verification section, line ~126; "Install the Developer Environment" section, ~line 46)
**Done when:** install.sh uses the Example 2 form; CONTRIBUTING's "Install the Developer Environment" section carries a general one-or-two-sentence note that developer installs use PEP 735 dependency groups, which need pip >= 25.1 (`pip install --group`) or uv, linking uv's official installation docs at the uv mention so a fresh machine doesn't dead-end on `uv: command not found` (Amendment 7 item 1; `make install` handles the floor via uv, so the note is context for anyone installing manually, not a new step in the happy path — placement there rather than a new section keeps the JOSS-plan CONTRIBUTING churn minimal), with no process references; `make format` installs ruff via `uv pip install -U --group ruff` and still runs clean; the ty target's commented install line reads the `--group ty` form; CONTRIBUTING's GPU venv instructions match Example 6 exactly, runnable verbatim from a fresh shell (the scaling-plan must-fix bar: no command in the block assumes un-listed prior state beyond a CUDA machine), including the `pip install --upgrade pip` step with a one-line reason (pip >= 25.1 needed for `--group`); no other CONTRIBUTING line references a dev extra (verified by grep — `make install` prose needs no change since it delegates to install.sh).

### Workstream C: .gitlab-ci.yml sweep

**Status:** complete
**Files:** `.gitlab-ci.yml` (lines 37, 53, 67, 116, 150, 164, 209, 235, 270, 323)
**Testing-group design decision (recorded, not mechanical):** the weekly/just-released PyPI jobs split "install the released package from PyPI" from "install the test toolchain from the checked-out tag's groups table". This is correct because the jobs check out the tag (or its `tests-<tag>` branch) before installing, so the source-only groups table is present; the trade-off (a pre-migration tag has no groups table, so the first scheduled run before the post-migration release would fail) is accepted per the release sequencing note above. There is no wiki page for the weekly matrix; this paragraph is the durable record of the decision until one exists.
**Done when:** all ten lines use the mapped `--group` forms from "Patterns this replaces"; the release-critical jobs (`release_gitlab`, `pypi_upload`, `readthedocs_*`, versioning include) are untouched; `performance_gates` still installs both `perf` and `testing` groups fail-loud (ADR 0002 contract); yml validates (e.g. GitLab CI lint API or a yaml parse); the two local-verification blind spots from prior churn are re-checked for this diff shape (repo-wide bare ruff run; `performance_gates` minimal-env collection sim is unaffected by a CI-only diff but confirm no source/test file changed in this workstream); the `uv_audit` job (lines 124-135) stays untouched with its coverage decision on the record: bare `uv audit` on the pinned uv 0.11.21 audits dependency groups by default, including non-default groups, alongside extras (verified empirically 2026-07-22: a known-vulnerable pin placed in a non-default group of a scratch project was reported with no flags), so moving the six dev extras into `[dependency-groups]` does not narrow the security-audit surface — re-confirm the job still emits findings-or-clean output (not a group-skipping no-op) when the post-migration pipeline first runs.

### Workstream D: .readthedocs.yaml restructure

**Status:** complete (first-post-push RTD build watch pending, with the `pip freeze` docs-group witness)
**Files:** `.readthedocs.yaml`
**Done when:** the `python.install` block is replaced by `build.jobs.install` steps per Example 5 (pip upgrade, then a single `pip install .[bokeh,datashader,holoviews,networkx,plotly] --group docs` — same extras set as today, `polars` stays excluded as it is now; the single-invocation form is maintainer-authorized in Amendment 8 and supersedes the grilled two-step form, closing the independent-resolution sliver); the existing `pre_build` steps (ipython install, pip freeze, notebook copy) and `sphinx.fail_on_warning: true` are preserved unchanged; the file parses as valid RTD v2 config. Note: full RTD verification only happens on a real RTD build after push; flag this residual risk in the workstream report so the maintainer watches the first `latest` build. That watch has a concrete witness, not just a green build: the preserved `pre_build` `pip freeze` step runs after the `build.jobs.install` steps, so its output in the RTD build log must list the docs-group packages (sphinx, nbsphinx, myst-parser, pydata-sphinx-theme, sphinx-copybutton, ipywidgets, toml) — a green build whose freeze output lacks them means the `--group docs` step silently did not take effect (failure mode 3).

### Workstream E: prose/tooling stragglers + CHANGELOG

**Status:** complete
**Files:** `runners/performance/README.md` (lines ~73, ~161), `runners/performance/profiling_utils.py` (line ~294), `tests/performance_measurement_test.py` (line ~83), `CHANGELOG.rst`
**Done when:** the three straggler references use the mapped forms from "Patterns this replaces"; the profiling_utils message names the `perf` dependency group with a source-checkout install form (no PyPI-form implication); CHANGELOG gains one entry under the unreleased version stating that the developer-only extras (`dev`, `docs`, `testing`, `perf`, `ruff`, `ty`) moved to PEP 735 dependency groups, that `pip install hiveplotlib[<those names>]` from PyPI no longer exists, and the replacement (source checkout + `pip install --group <name>`, pip >= 25.1 or uv) — this is a released-behavior change (published metadata), so it clears the CHANGELOG-only-for-released-behavior bar; historical CHANGELOG entries are untouched (see Holdouts); `make test` passes (the touched test file's docstring edit changes no behavior); the message/docstring edits carry no process references; `runners/performance/README.md` carries a half-line note that its plain-pip `--group` form needs pip >= 25.1 (or a pointer to CONTRIBUTING's floor note); the CHANGELOG replacement recipe uses the accurate `pip install -e . --group <name>` form (bare `pip install --group <name>` installs only the group's deps, not hiveplotlib itself).

## Plan amendments

### In-scope tweak: broaden the QA survivor-grep to catch comma-preceded dev-extra names

**Date:** 2026-07-22
**Trigger:** adversary planning-pass must-fix 1 (gate could not match `[datashader,testing]` / `[$ADDON,testing]` forms the sweep itself names)
**Workstream affected:** cross-cutting (QA enforcement line in "Patterns this replaces")
**Change:** grep pattern `\[(dev|docs|perf|ruff|testing|ty)[,\]]` → `[\[,](dev|docs|perf|ruff|testing|ty)[,\]]`; applied directly in "Patterns this replaces" in this pass.

### In-scope tweak: record the `uv_audit` coverage decision

**Date:** 2026-07-22
**Trigger:** adversary planning-pass must-fix 2 (uv_audit is a consumer of pyproject dependency structure the sweep never inventoried; migration could silently narrow the security-audit surface)
**Workstream affected:** C (.gitlab-ci.yml sweep)
**Change:** verified empirically on uv 0.11.21 (the pinned image version): bare `uv audit` audits dependency groups by default, including non-default groups, alongside extras — a known-vulnerable pin in a non-default group of a scratch project was reported with no flags. The job line needs no edit; Workstream C's done-when now records the decision and requires re-confirming the job's output on the first post-migration pipeline.

### In-scope tweak: editable-origin witness for self-referencing groups; scope the resolution claim to uv

**Date:** 2026-07-22
**Trigger:** adversary planning-pass must-fix 3 (nothing checked *what* satisfied `hiveplotlib[bokeh]`/`hiveplotlib[datashader]` self-references; PyPI-satisfied resolution would silently test released code; the "under both uv and pip" claim was untested prose on the pip half)
**Workstream affected:** A (pyproject.toml migration)
**Change:** Workstream A's done-when adds a `uv pip freeze | grep hiveplotlib` editable-origin witness after the `--group dev` scratch install; the composition-decision claim is rescoped to uv only, with an explicit note that no plain-pip consumer installs a self-referencing group (RTD and the GPU venv install only `docs` and `testing`).

Adversary items 4-5 (worth-discussing: unnamed premise driver; docs-stays-an-extra smaller move) are deliberately **not** disposed here — planning-mode challenges at the premise level are the maintainer's to fight in the grill. They are queued verbatim as items 4-5 under "Open questions for the grill." *(Post-grill note, 2026-07-22: both were fought and resolved at the grill — driver confirmed, full migration confirmed; see `## Alignment (grill)`.)*

### In-scope tweak: general pip >= 25.1-or-uv note in CONTRIBUTING's dev-setup section

**Date:** 2026-07-22
**Trigger:** maintainer ask at the grill (emergent; beyond the GPU-venv-block one-liner already in Workstream B)
**Workstream affected:** B (local dev tooling sweep)
**Change:** Workstream B's files gain CONTRIBUTING's "Install the Developer Environment" section (~line 46), and its done-when now requires a general note there that developer installs use PEP 735 dependency groups needing pip >= 25.1 or uv (`make install` already satisfies this via uv; the note is context for manual installs). Placed in the existing dev-setup section rather than a new one to minimize churn against the JOSS plan's CONTRIBUTING rewrite.

### In-scope tweak: concrete witness for the first-post-migration RTD build watch

**Date:** 2026-07-22
**Trigger:** adversary post-grill rubric-check, worth-discussing on failure mode 3 (RTD silently builds with wrong deps): Workstream D's residual-risk flag only caught loud failures
**Workstream affected:** D (.readthedocs.yaml restructure)
**Change:** Workstream D's done-when now names the witness for the first-build watch: the preserved `pre_build` `pip freeze` output in the RTD build log (which runs after `build.jobs.install`) must list the docs-group packages, not just show a green build. Verification-only tweak directly serving a maintainer-named failure mode at no cost, so it is applied without a maintainer halt per the autonomous must-fix-halt policy's in-contract carve-out.

### In-scope tweak: reclassify the unreleased `hiveplotlib[perf]` CHANGELOG line from Holdout to Workstream E scope

**Date:** 2026-07-22
**Trigger:** Workstream E completion flag — the Holdouts entry pinned `CHANGELOG.rst:81` as historical, but that bullet sits in the UNRELEASED 0.29.0 block; at release it would describe `perf` as an extra in the same version that ships it as a dependency group
**Workstream affected:** E (prose/tooling stragglers + CHANGELOG; reopens for this one-line wording fix)
**Change:** `CHANGELOG.rst:81` reworded to say the `perf` dependency group (matching the maintainer's standing convention that unreleased entries are living text — refinements to unreleased features fold in rather than accumulate). The Holdouts entry now covers only the truly historical lines, with anchors corrected to 282/285 after Workstream E's insertion shifted the old 276/279, plus a new holdout for E's own entry at ~89, which deliberately names the lost `pip install hiveplotlib[dev]` path and would otherwise fail the broadened QA grep. In-contract consistency fix (serves the shipped Workstream E done-when's "released-behavior" bar), applied without maintainer halt per the standing policy.

### In-scope tweak (batch, Amendment 7): adversary post-impl dispositions across Workstreams B, D, E

**Date:** 2026-07-22
**Trigger:** five adversary post-impl passes complete, zero must-fix; batch disposition of the worth-discussing / low-confidence / bookkeeping items
**Workstreams affected:** B and E (each reopened for one fix dispatch), D (bookkeeping only), Holdouts (cross-cutting)
**Changes:**
1. (WS B, worth-discussing) CONTRIBUTING's dev-setup note now must link uv's official installation docs at its uv mention; a fresh machine otherwise dead-ends on `uv: command not found`. In-contract fix serving failure mode 2 (contributor hits a wall the docs don't explain); applied without maintainer halt per the standing policy.
2. (WS E, worth-discussing) Two legitimate `optional-dependencies` survivors recorded as Holdouts so the qa close doesn't false-fail: `CHANGELOG.rst:~88` (the migration entry's own mention) and pyproject.toml's kept `[project.optional-dependencies]` heading (still holds the six user extras). Record-only; no code change.
3. (WS E, worth-discussing) `runners/performance/README.md` gains a half-line pip >= 25.1 floor note (or CONTRIBUTING pointer) for its plain-pip `--group` line; failure mode 2 again. Applied without halt, same rationale.
4. (WS E, low-confidence, adopted) CHANGELOG's replacement recipe corrected to `pip install -e . --group <name>` — the bare `--group` form installs only the group's deps, not hiveplotlib. Folded into the same reopen since the file is being touched anyway; accuracy fix on the one user-facing loss the entry documents.
5. (bookkeeping) All five workstream Status fields updated from "not started" to complete; B and E marked reopened for the items above until the fix dispatch lands; D's status carries the pending first-build watch.

Not dispositioned here (queued for the maintainer at plan close, no action taken): WS A's observation that no durable CI guard prevents `Provides-Extra` regrowth in future edits; WS D's nit that the two pip invocations resolve independently (shipped file matches the grilled Example 5, so this is a design note, not a defect). *(Both resolved at maintainer review — see Amendment 8: the CI guard is declined; the RTD invocations are merged into one.)*

### In-scope tweak (batch, Amendment 8): maintainer review decisions

**Date:** 2026-07-22
**Trigger:** maintainer review of the shipped work
**Workstreams affected:** D and A (each a small authorized file edit, applied concurrently by a code-engineer), plus plan-level records
**Changes:**
1. (WS D, authorized simplification) The two RTD pip steps merge into one: `pip install .[bokeh,datashader,holoviews,networkx,plotly] --group docs`, still preceded by `pip install --upgrade pip`. One dependency resolution instead of two independent ones, which closes the residual sliver the WS D adversary flagged (a second invocation can up/downgrade what the first resolved). This deviates from the grilled Example 5, so Example 5, the `.readthedocs.yaml` line in "Patterns this replaces", and Workstream D's done-when are all updated to the single-invocation form, which supersedes the grilled two-step form. The `pip freeze` docs-group witness for the first-build watch is unchanged and still applies.
2. (WS A, authorized) The `hygeine` typo in the pyproject `ruff` / `ty` group comments is fixed (previously deliberately left out of scope).
3. (WS A observation, **declined**) A CI guard against `Provides-Extra` regrowth is considered-and-rejected by the maintainer: regrowth would be a deliberate act rather than drift, and the guard would need updating whenever a real user-facing extra is intentionally added (e.g. the igraph extras deferred in ADR 0001).
4. (Sequencing, clarified) Branch 53 will not merge to master until the maintainer is essentially ready to cut the release, shrinking the accepted red-weekly window to near zero; the sequencing note now states the risk is smaller than originally priced, not merely accepted.
5. (ADR promotion, timing) The maintainer green-lights ADR promotion **after the release ships**, not before. Do not action the promotion flag early; the same applies to the plan's move to `plans/archived/`.

## Holdouts

- `CHANGELOG.rst:280,283` (were 276,279 before Workstream E's insertion, then 282,285 until qa's rule-13 compression of the migration entry re-shifted them): kept as `[dev]` extra language — historical entries describing released versions where those were extras; changelogs are append-only history. (`CHANGELOG.rst:81` was misclassified here — it sits in the unreleased 0.29.0 block, so it is in-scope living text, reclassified to Workstream E by Amendment 6.)
- `CHANGELOG.rst:~89`: kept as `pip install hiveplotlib[dev]` — Workstream E's own migration entry deliberately names the lost install path; the QA survivor-grep must not flag it.
- `CHANGELOG.rst:~88`: kept as `optional-dependencies` — the migration entry's own description of what moved; a legitimate survivor of the QA grep's `optional-dependencies` clause.
- `pyproject.toml` `[project.optional-dependencies]` heading: kept — it still holds the six user-facing extras; the migration removes dev names from under it, not the table itself.
- `asv.conf.json:31`: kept as `{wheel_file}[datashader]` — `datashader` is a kept user-facing extra.
- `.gitlab-ci.yml:85,100` (lint_check, format_check): kept as bare `pip install ruff` — they never referenced the `ruff` extra, and switching to `--group` would add a pip-upgrade step to both jobs for no behavior change (ruff is unpinned in the group too). Queued as a grill question; default is leave.
- All docs prose, notebooks, `docs/source/README.md`, and `docs/source/_llms/llms*.txt` mentions of `hiveplotlib[bokeh|datashader|holoviews|networkx|plotly|polars]`: kept — user-facing extras are unchanged, so no llms.txt edit is warranted (no consequential change to how anyone uses the library from PyPI).
- `Makefile:128-136` (test-gpu target): kept — it points at the GPU venv whose setup CONTRIBUTING owns; the target itself installs nothing.

## Open questions for the grill

**All five resolved at the grill (2026-07-22); dispositions live in `## Alignment (grill)`: driver confirmed (cleaner/modern tooling), full migration confirmed (Workstream D stays), items 1 (leave as holdouts, delegated call), 2 (keep `-U`), and 3 (confirmed phrasing) closed as recommended. The list below is kept as the record of what was asked.**

1. lint_check / format_check CI jobs: align their bare `pip install ruff` to the `ruff` group for single-source naming, or leave (recommended: leave; see Holdouts)?
2. `make format` ruff refresh: the plan changes it to `uv pip install -U --group ruff` to keep "latest ruff" semantics while sourcing the name from the group — confirm, or prefer dropping `-U` so local ruff only tracks the group's (unpinned) resolution at install time?
3. CHANGELOG phrasing: confirm the entry should explicitly name the lost `pip install hiveplotlib[dev]`-style paths (recommended: yes; that is the one user-visible loss).
4. (Adversary worth-discussing, premise angle — maintainer's to fight, not disposed by the orchestrator.) The Goal's stated benefit is metadata hygiene plus ecosystem alignment; no blocked user workflow is named, while the priced costs are the pip >= 25.1 floor in three places, the post-push-only RTD verification, the accepted red-weekly window, and the loss of `pip install hiveplotlib[dev]`. Name the concrete driver (candidates the adversary suggests: cleaner extras surface for the conda-forge mapping; leaner CONTRIBUTING before JOSS) or confirm hygiene-plus-alignment alone is worth this cost set.
5. (Adversary worth-discussing, smaller-move angle — maintainer's to fight, not disposed by the orchestrator.) Keep `docs` as an extra and migrate only five groups, leaving `.readthedocs.yaml` untouched and deleting Workstream D (the plan's riskiest piece, verifiable only on a live fail-on-warning build)? The counterargument: a half-migration muddies the "dev names gone from PyPI metadata" story and leaves one stray dev extra published. Adopt the smaller move, or record docs-stays-an-extra as considered-and-rejected with the reason.

## Implementation log

(append-only; one line per completed workstream)

2026-07-22: Workstream A complete. Moved `dev`/`docs`/`perf`/`ruff`/`testing`/`ty` verbatim (pins, comments, `hiveplotlib[<extra>]` self-references) from `[project.optional-dependencies]` to `[dependency-groups]` in pyproject.toml; all four done-when witnesses executed and passed (six-group scratch install clean, `--group testing` alone clean, freeze shows `-e file:///home/garyk/repos/hiveplotlib`, wheel METADATA `Provides-Extra` = exactly the six user-facing extras).
2026-07-22: Workstream A touch-up (maintainer-authorized). Reworded the two `perf` comments that still called the group an extra ("performance profiling dependency group"; the `hiveplotlib[datashader]` note now says it references the datashader extra); swept the other five group comments, no other stale "extra(s)" wording found; "hygeine" typo left untouched per instruction.
2026-07-22: Workstream E complete. Swept the three stragglers to `--group` forms (runners/performance/README.md install lines + adjacent extras prose, profiling_utils.py RuntimeError message + psutil docstring/comment mentions, performance_measurement_test.py docstring) and added the unreleased-version CHANGELOG Tooling Changes entry naming the lost `pip install hiveplotlib[dev]`-style paths and the source-checkout `--group` replacement (pip >= 25.1 or uv); historical CHANGELOG holdouts untouched; ruff clean repo-wide, `make test` 1591 passed / 100% coverage.

2026-07-22: Workstream C complete. Swept all ten .gitlab-ci.yml install lines to the mapped `--group` forms (editable jobs `-e . --group ...`, add-ons `-e .[$ADDON] --group testing`, released-code jobs split into PyPI package install + `uv pip install --group testing` from the checked-out tag); release-critical jobs, lint/format holdouts, and uv_audit untouched; yaml parses, survivor grep clean, repo-wide ruff check/format clean, no CRLF, no src/tests file changed (CI-only diff).
2026-07-22: Workstream B complete. install.sh line 7 → Example 2 six-`--group` form; Makefile `format` → `uv pip install -U --group ruff`, ty target's commented install → `--group ty` (no `-U`); CONTRIBUTING GPU venv block → Example 6 verbatim (pip upgrade step + pip >= 25.1 reason line) and the dev-setup section gained the general PEP 735 / pip >= 25.1-or-uv note; survivor-grep on CONTRIBUTING clean (only `.[datashader]`, a kept extra, remains).
2026-07-22: Workstream D complete. Replaced .readthedocs.yaml's `python.install` block with `build.jobs.install` steps per Example 5 (pip upgrade, extras install minus `docs`, `pip install --group docs`); `pre_build` steps and `sphinx.fail_on_warning` preserved unchanged; YAML parses with the expected RTD v2 key structure; residual risk (first-build freeze-log witness for the docs-group packages) flagged in the report per Amendment 5.
2026-07-22: Workstream E reopened per Amendment 6 and re-closed. Reworded CHANGELOG.rst:81 (unreleased 0.29.0 Tooling Changes) from "The ``hiveplotlib[perf]`` extra" to "The ``perf`` dependency group", rest of the bullet unchanged; survivor grep on CHANGELOG.rst now matches only the recorded holdouts (89 migration-entry mention, 282/285 historical).
2026-07-22: Amendment 7 fix dispatch complete (Workstreams B and E re-closed). CONTRIBUTING dev-setup note links uv's official installation docs at the uv mention; runners/performance/README.md's plain-pip `--group` line gained a half-line "needs pip >= 25.1" note; CHANGELOG's replacement recipe corrected to `pip install -e . --group <name>`; broadened survivor grep + `optional-dependencies` grep over tracked files match exactly the recorded Holdouts (CHANGELOG 89/282/285; CHANGELOG 88 + pyproject `[project.optional-dependencies]` heading).
2026-07-22: QA close passed; plan complete pending maintainer commit. 1591 tests / 100% coverage, zero-warning docs build, independent wheel-metadata witness re-confirmed (Provides-Extra = exactly the six user-facing extras); qa auto-fix compressed the CHANGELOG migration entry 6 → 4 lines per the rule-13 cap, shifting the historical Holdouts anchors 282/285 → 280/283 (Holdouts updated; earlier log lines above keep the anchors as they were when written). Remaining post-commit items for the maintainer: first-post-push RTD build watch (freeze-log docs-group witness), release-before-next-weekly sequencing, and the two queued observations from Amendment 7 (Provides-Extra-regrowth CI guard question; RTD two-invocation resolution note).

2026-08-02: Maintainer-authorized touch-ups. Merged the two `.readthedocs.yaml` install steps into one `pip install .[bokeh,datashader,holoviews,networkx,plotly] --group docs` invocation (extras and docs group now resolve together, closing the queued two-invocation resolution observation) and rewrote the adjacent comment to match while keeping the pip >= 25.1 rationale; fixed the pre-existing `hygeine` → `hygiene` typo in the `ruff` and `ty` dependency-group comments in pyproject.toml. Verified both files parse (RTD v2 structure intact, `pre_build` / `sphinx.fail_on_warning` / os+tools unchanged; six dependency groups and six user-facing extras unchanged), no `hygeine` survivors in tracked files, no CRLF flip, and `git diff --stat` shows only these two files by my hand.
