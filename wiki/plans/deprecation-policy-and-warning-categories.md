# Plan: Deprecation policy + named warning categories (issue TBD)

## Goal

Give downstream users running `filterwarnings = error` (and anyone tuning warning filters) a stable, importable set of warning categories to filter on, and give the project a stated deprecation contract so the next deprecation does not rebuild machinery from scratch. Three deliverables: (1) a documented deprecation policy with a one-minor-version window (deprecate in 0.Y with a warning, remove in 0.(Y+1)); (2) a `DeprecationWarning` subclass in `src/hiveplotlib/exceptions/`, kept and tested even while nothing is deprecated (the previous ad-hoc machinery was deleted with `hive_plot_n_axes` in v0.28); (3) migration of the viz backends' bare `UserWarning` calls to named categories that subclass `UserWarning`, so existing filters keep working while new per-category filters become possible.

Brief-mode gate: knowingly skipped per Gary's directive (2026-07-06).

**Execution context.** This plan file lives in the MAIN checkout's wiki submodule and stays uncommitted. Implementation happens in a fresh git worktree cut from **master** (branch `<iid>-deprecation-policy-and-warning-categories`; iid from a GitLab issue created after this plan is written). All `file:line` citations below are against master (the main checkout is on branch 53; branch 53 and MR !51 changes are NOT in scope's base). Code-engineer re-verifies the call-site inventory in the worktree before editing. One commit per workstream after the gate battery (`make format`, `make test`, `make ty`, `make docs`); push each. Never merge the MR.

### Non-goals

- **No actual deprecation ships in this plan.** The machinery is built and tested; nothing is deprecated with it. `dataframe_to_node_list` / legacy `Node` (node.py:430-450), twice flagged by the scaling plan as "belongs to a separate plan," is the likely first customer — deferred, see Plan amendments-style note under Holdouts. Deprecating it here would smuggle an API-removal decision into a machinery plan and collide with branch 53's in-flight node.py context.
- **No category migration outside `src/hiveplotlib/viz/`.** The 14 bare `warnings.warn` sites in `hiveplot.py` (13) and `p2cp.py` (1) stay as-is: both files are touched by two active branches (branch 53 scaling; MR !51 unify-axes), and sweeping them now buys merge conflicts for warnings users hit at build time rather than plot time. Deferred follow-up with a revival trigger: sweep them after branch 53 and MR !51 merge, reusing the taxonomy shipped here. See Holdouts.
- **No message-text changes.** This is a category-only migration; downstream `match=` filters and our own test assertions must keep matching byte-identical messages.
- **No per-site stacklevel changes.** Stacklevels are threaded per call depth (canonical record: archived graph-metric-conflict-validation plan); CLAUDE.md's `stacklevel=3` is a default, not uniform. Category added, stacklevel untouched.
- **No `warn_deprecated()` helper.** Classes only. A helper canonicalizing the deprecation call has zero call sites until the first real deprecation ships (deferred, see Holdouts), which is the "machinery that rots" failure mode wearing a different hat. The reusable asset is the message template, canonicalized in Workstream C's worked example instead. (Amendment 7.)

### Cross-plan coordination (flag only — this plan does not edit other plans)

- `wiki/wiki/plans/hiveplot-unify-axes.md` (MR !51, unexecuted) plans a new plain `UserWarning` (vmin/vmax displacement) with an "exceptions/ holds only Error classes" rationale in its done-when. Whichever plan ships second reconciles: after this plan ships, that warning should take a category from the new taxonomy.
- `wiki/wiki/plans/scaling-large-networks.md` (branch 53) adds two new bare warn sites inside `viz/input_checks.py` itself — `check_curves_exist`, branch-53 lines 168 (lazy-Dask edge frames) and 178 (unconstructed curves), both `stacklevel=3` — the very file Workstream B edits, making `viz/input_checks.py` a two-branch merge surface whose merge will re-break Workstream B's no-bare-warn invariant. Enumerated in Holdouts with a mechanical re-arm. (Corrected per Amendment 3; the earlier "plain-UserWarning helper on the datashader route" description was wrong on both the count and the file.)
- `wiki/wiki/plans/edge-coverage-and-gotchas-docs.md` line 34 codifies "plain UserWarning, stacklevel=3"; goes stale once Workstream B ships. Flag to the maintainer at plan close.

## Alignment (grill)

```
Status: knowingly skipped (2026-07-06, Gary's directive)
Open decisions: none
```

The post-plan grill was knowingly skipped per Gary's directive of 2026-07-06 (same directive that skipped brief mode). Gary explicitly authorized auto-dispatch anyway, overriding the harness default that grill-skipped plans keep per-workstream pauses. Recorded utterance: "auto-dispatch; one commit per workstream after the gate battery (make format, make test, make ty, make docs); push each."

Halts back to the maintainer: any `must-fix`, any `STATUS: BLOCKED`, any workstream-set change (routes via orchestrator amend-plan). Never merge the MR.

Because the grill is skipped, the adversary's planning challenge has no grill to be fought in; per harness convention its items route to orchestrator amend-plan for explicit disposition before dispatch.

## Failure modes

Grill knowingly skipped, so no maintainer-named modes. Orchestrator-seeded rubric (model's words, not the maintainer's standard; implementers append if they hit weeds-level modes):

- **Taxonomy nobody can use**: categories so coarse (one blanket class) or so fine (per-message classes) that a downstream user still can't separate "hover unsupported" noise from "your plot is empty" signal with one filter line.
- **Stacklevel drift**: the migration touches a warning call and nudges its stacklevel, silently re-pointing warning attribution at library internals instead of user code; tests still pass because they don't assert attribution.
- **Message churn smuggled in**: "while I'm here" rewording of warning text breaks downstream `match=` filters in a change sold as category-only.
- **Machinery that rots**: the deprecation class ships but is undiscoverable (not in docs, not in the policy page), so the next deprecation rebuilds ad hoc anyway — the exact failure this plan exists to prevent.
- **Two cadences on record**: the one-minor-version window ships in docs while ADR 0001's two-version note stands unreconciled, leaving contradictory contracts.
- **Hollow test**: the "tested while nothing is deprecated" test asserts only `issubclass`, not the actual filter behavior a downstream user relies on (catchable under an `error` baseline, silenceable by category).

## Prior ADRs / design docs

From research-liaison's pre-task findings:

- `wiki/wiki/adr/0001-networkx-integration.md` (Declined section, GraphMetricsSpec entry) — the only ADR text stating a deprecation cadence: a TWO-version window "per the hive_plot_n_axes 0.26→0.28 precedent." **Superseded by maintainer decision (Gary, 2026-07-06): the window is one minor version.** This plan ships the one-minor-version policy; the ADR-promotion step at plan close writes the durable supersession record (ADR 0001 is append-only — new record links old ↔ new; do not edit 0001). No shipped artifact cites the ADR (no process refs in shipped code).
- `wiki/wiki/plans/archived/graph-metric-conflict-validation.md` — deliberated named-Warning-subclass vs plain UserWarning (~line 1671 planning, ~line 1755 disposition) and shipped plain UserWarning on the rationale "exceptions/ holds only Error classes, no Warning-subclass precedent," api-critic concurring. **This plan supersedes that decision** by creating the Warning-subclass precedent deliberately. Also the canonical record of stacklevel discipline (per-call-depth threading); binding on Workstream B.
- `wiki/wiki/plans/hiveplot-unify-axes.md`, `wiki/wiki/plans/scaling-large-networks.md`, `wiki/wiki/plans/edge-coverage-and-gotchas-docs.md` — coordination items, see Goal.
- Docs-placement precedent: `wiki/wiki/plans/archived/gitlab-group-migration.md` (CONTRIBUTING rewritten to single-maintainer governance in v0.28; JOSS review will read it) and `wiki/wiki/plans/scaling-large-networks.md:2088` ("CONTRIBUTING.md is the home for maintainer-facing release-cadence guidance until a releases-specific doc exists"). Placement call made below (Workstream C): the policy is user-facing docs; CONTRIBUTING points to it.

## Patterns this replaces

Bare `warnings.warn(<msg>, stacklevel=N)` (implicit `UserWarning`) in `src/hiveplotlib/viz/`, replaced with an explicit `category=<class>` keyword (message and stacklevel byte-identical). Inventory against master, 26 sites:

**→ `EdgeKwargConflictWarning`** (edge-kwarg hierarchy conflict, higher-priority kwarg preserved; 3 sites, all `stacklevel=3`):

- `src/hiveplotlib/viz/base.py:144`, `base.py:152`, `base.py:164`

**→ `MissingPlotElementWarning`** (plotting before axes/nodes/edges exist; 10 sites, all `stacklevel=3`):

- `src/hiveplotlib/viz/input_checks.py:52`, `:58`, `:64`, `:73`, `:79`, `:85`, `:95`, `:102`, `:113`, `:119`

**→ `UnsupportedVizFeatureWarning`** (requested input disregarded because the viz path doesn't support it; 13 sites, all `stacklevel=2`):

- Hover-not-supported-for-P2CPs: `src/hiveplotlib/viz/bokeh.py:215`, `:404`, `:595`, `:873`; `src/hiveplotlib/viz/holoviews.py:232`, `:452`, `:723`, `:1009`; `src/hiveplotlib/viz/plotly.py:332`, `:439`, `:587`, `:1060`
- Multiple-tags-only-plotting-one: `src/hiveplotlib/viz/datashader.py:145`

The warning classes themselves are net new additions (no prior Warning classes exist in `exceptions/`; feasibility trace: they are plain importable classes exposed via the existing `exceptions/__init__.py` star-export path, same as every `*Error` class — no data-model attribute reads, no new conventions on user input).

QA sweep gate for Workstream B: no `warnings.warn(` call in `src/hiveplotlib/viz/` without a `category=` argument. `src/hiveplotlib/hiveplot.py` and `src/hiveplotlib/p2cp.py` survivors are expected — see Holdouts.

## Default justifications

No new defaults — no new parameters or configurable surface. Each class-to-site mapping is enumerated above, not defaulted.

## Naming audit

New classes (the user-facing surface: these names land in downstream `filterwarnings` ini lines and `-W` flags, e.g. `ignore::hiveplotlib.exceptions.HiveplotlibDeprecationWarning`):

- `HiveplotlibWarning(UserWarning)` — common base for every hiveplotlib warning category **except** `HiveplotlibDeprecationWarning` (deliberate exclusion; next bullet). One ini line filters (or errors on) every named *non-deprecation* warning category hiveplotlib defines — categories, not emissions (Amendment 10): the 14 bare-`UserWarning` holdout sites (see Holdouts) sit outside every category until the deferred sweep, so any "everything hiveplotlib emits" claim is false in shipping v0.29. The exclusion leads the class docstring's first sentence (Workstream A), and silencing all of hiveplotlib's named categories honestly takes two lines (base + deprecation class), shown in Workstream C's snippet. (Amendments 1, 10.) Prefixed because a printed warning shows only the class name, and the base class is the "this came from hiveplotlib" signal. Considered and rejected (Amendment 6): astropy's pattern — umbrella subclasses `Warning` (not `UserWarning`), viz categories dual-inherit `(UserWarning, umbrella)`, deprecation dual-inherits `(DeprecationWarning, umbrella)` — keeps a one-line umbrella over deprecations with no filter-order ambiguity, but sweeps actionable deprecation signal into the "silence hiveplotlib noise" line and pays a dual-inheritance complexity tax. Do not cite astropy as precedent for the shipped design; `AstropyWarning` subclasses `Warning`, so it is precedent for this rejected alternative.
- `HiveplotlibDeprecationWarning(DeprecationWarning)` — per Gary's brief, subclasses `DeprecationWarning`. Exact-match precedent: matplotlib's `MatplotlibDeprecationWarning(DeprecationWarning)`, machine-verified in the project venv 2026-07-06 (MRO: `DeprecationWarning` → `Warning`); astropy dropped from this bullet per Amendment 6 (its deprecation class sits *under* its umbrella — the rejected alternative above). Deliberately **not** under `HiveplotlibWarning`: dual inheritance from `UserWarning` and `DeprecationWarning` makes default visibility depend on stdlib filter order (`ignore::DeprecationWarning` vs `default::UserWarning`), an ambiguity no user should have to reason about. Single inheritance keeps stock `DeprecationWarning` semantics and matches matplotlib exactly. Considered and rejected (Amendment 2): subclassing `FutureWarning`, pandas-style, for its always-visible default. Rejected because the maintainer's brief names `DeprecationWarning` semantics (a recorded decision), and `FutureWarning`'s visibility lands on end users of downstream code who cannot act on it. The known cost, accepted knowingly: `DeprecationWarning` is hidden by default outside `__main__` and pytest, and this plan pairs the shortest window (one minor version) with the least visible category — the aggressive corner of the design space. Mitigations: the policy page states the default-visibility behavior and how to surface the warnings (Workstream C done-when), pytest surfaces them by default, and the warnings-as-errors audience this plan serves sees them as errors.
- `EdgeKwargConflictWarning(HiveplotlibWarning)` — matches house vocabulary: the edge-kwarg-hierarchy docs and CLAUDE.md call these "conflict warnings." Unprefixed like the existing specific exceptions (`InvalidVizBackendError` carries no library prefix; the module path disambiguates).
- `MissingPlotElementWarning(HiveplotlibWarning)` — fires when axes/nodes/edges are missing at plot time. "Element" generalizes over the `objects_to_plot` values (`"axes"` / `"nodes"` / `"edges"`) in `input_checks.py`. Alternatives considered: `NothingToPlotWarning` (wrong for the partial case: "at least one of your axes has no nodes"), `IncompleteHivePlotWarning` (wrong for P2CPs). api-critic may propose better in planning take.
- `UnsupportedVizFeatureWarning(HiveplotlibWarning)` — "viz" is established house vocabulary (`hiveplotlib.viz`, `InvalidVizBackendError`). Covers both the hover-for-P2CPs family and datashader's single-tag limitation: in both, user input is disregarded because the viz path can't honor it.

Suffix convention: `*Warning` per stdlib, alongside the module's existing `*Error` convention for exceptions.

Where the names appear: star-exported from `hiveplotlib.exceptions` (new module `src/hiveplotlib/exceptions/warning_categories.py`, named for the stdlib term — `warnings.warn`'s `category` parameter — and avoiding a `warnings.py` stdlib shadow; internal module name, out of audit scope). The star export is load-bearing: warning filter strings resolve the category by importing `hiveplotlib.exceptions` and looking up the attribute, so the export path IS the filter-string path. The API reference is hand-maintained rST, not sphinx-apidoc (one `.. automodule::` file per module under `docs/source/autodoc/`, git-tracked; no apidoc invocation in `conf.py` or either Makefile — verified 2026-07-06 in both the worktree and the main checkout), so the new module needs a manual `docs/source/autodoc/exceptions/warning_categories.rst` plus a toctree line, owned by Workstream C (Amendment 9).

## API usage examples

### Proposed (from planner / Orchestrator)

```python
# Example 1: downstream user under a warnings-as-errors baseline silences one
# hiveplotlib category (the pytest-ini equivalent:
#   filterwarnings =
#       error
#       ignore::hiveplotlib.exceptions.MissingPlotElementWarning
# )
# Example data:
import warnings

from hiveplotlib import BaseHivePlot
from hiveplotlib.exceptions import MissingPlotElementWarning
from hiveplotlib.viz import hive_plot_viz

hp = BaseHivePlot()  # nothing added yet; plotting it warns

# Call site:
with warnings.catch_warnings():
    warnings.simplefilter("error")  # the user's strict baseline
    warnings.filterwarnings("ignore", category=MissingPlotElementWarning)
    fig, ax = hive_plot_viz(hp)  # empty-plot warning no longer fatal
```

```python
# Example 2: one filter line covers every named NON-DEPRECATION hiveplotlib warning
# category via the common base (categories, not emissions: the bare-UserWarning
# holdouts in hiveplot.py / p2cp.py sit outside every category until the deferred
# sweep, so this line does not cover them pre-sweep — Amendment 10). Covering all of
# hiveplotlib's named categories takes a second line, and even then the pre-sweep
# holdouts remain bare:
#   ignore::hiveplotlib.exceptions.HiveplotlibDeprecationWarning
# Example data:
import warnings

from hiveplotlib import BaseHivePlot
from hiveplotlib.exceptions import HiveplotlibWarning
from hiveplotlib.viz import hive_plot_viz

hp = BaseHivePlot()

# Call site:
with warnings.catch_warnings():
    warnings.simplefilter("error")
    warnings.filterwarnings("ignore", category=HiveplotlibWarning)
    fig, ax = hive_plot_viz(hp)
```

```python
# Example 3: what the NEXT deprecation looks like using the shipped machinery
# (maintainer-side warn, user-side silence) — the "no rebuild from scratch" test.
# Example data:
import warnings

from hiveplotlib.exceptions import HiveplotlibDeprecationWarning


def old_helper() -> None:
    """Deprecated alias kept for one minor version per the deprecation policy."""
    warnings.warn(
        "`old_helper` is deprecated and will be removed in v0.31.0; "
        "use `new_helper` instead.",
        category=HiveplotlibDeprecationWarning,
        stacklevel=2,
    )


# Call site (downstream user keeping errors-on-warnings but tolerating deprecations):
with warnings.catch_warnings():
    warnings.simplefilter("error")
    warnings.filterwarnings("ignore", category=HiveplotlibDeprecationWarning)
    old_helper()
```

### API Critic's take (planning mode)

Walked the surface as a downstream user under `filterwarnings = error` doing three things: silencing one noisy category, silencing/erroring everything hiveplotlib emits, and surviving a future deprecation cycle. Names: **Agreed on all five.** `MissingPlotElementWarning` beats both audit alternatives (walked all 10 `input_checks.py` messages; "element" covers axes/nodes/edges including the partial "one axis has no nodes" case, and the printed warning shows the class name, so the user can copy it straight into an ignore line). `EdgeKwargConflictWarning` and `UnsupportedVizFeatureWarning` match house vocabulary; the datashader single-tag site fits the "input disregarded" framing. Filter-string ergonomics check out: `hiveplotlib.exceptions.X` is shorter than astropy's equivalent, the star export is exactly what ini-string resolution needs, and `hiveplotlib/__init__.py` exports no exceptions, so there is a single canonical filter path (keep it that way — do not also top-level-export these).

Amendments, tagged:

- **[worth-discussing] The `HiveplotlibWarning` pitch overclaims: "one ini line filters (or errors on) everything hiveplotlib emits" is false the day the first deprecation ships**, because `HiveplotlibDeprecationWarning` deliberately sits outside the umbrella. I agree with the single-inheritance decision itself (the justification stands; deprecations are actionable signal a "silence hiveplotlib noise" line arguably should NOT sweep up). But the exclusion must be load-bearing in the first sentence of `HiveplotlibWarning`'s docstring ("base class for all hiveplotlib warnings *except* `HiveplotlibDeprecationWarning`"), Example 2's comment should say "every non-deprecation hiveplotlib warning," and Workstream C's filterwarnings snippet should show the honest two-line "all of hiveplotlib" recipe (base + deprecation class). Binding on Workstreams A and C.
- **[low-confidence] The astropy precedent citation is inexact; matplotlib is the clean one.** `AstropyWarning` subclasses `Warning` (not `UserWarning`), and astropy's deprecation warning lives *under* its umbrella via that design — astropy is precedent for the *alternative* (base = `Warning`, viz categories dual-inherit `(UserWarning, HiveplotlibWarning)`, deprecation dual-inherits `(DeprecationWarning, HiveplotlibWarning)`), which preserves the one-line umbrella with none of the filter-order ambiguity the plan worries about. I still prefer the plan's simpler design (the umbrella-covers-deprecations behavior is of debatable value, per the previous bullet), but the docstring/plan text should cite matplotlib's `MatplotlibDeprecationWarning` as the exact-match precedent and record the astropy pattern as considered-and-rejected so this doesn't relitigate at the next deprecation. Verify astropy's current class hierarchy before shipping any docstring that names it.
- **[worth-discussing] Example 3 is a bare class, not machinery; the reusable asset is the message shape.** "X is deprecated and will be removed in vN.N.N; use Y instead" (deprecated name, replacement, removal version, in that order) is the thing the next deprecation would otherwise re-improvise. Workstream C's worked example should canonicalize that exact template (same text as Example 3), and the plan can honestly note a `warn_deprecated()` helper is deliberately not shipped (no customer yet; a helper with zero call sites is the "machinery that rots" failure mode wearing a different hat). Classes-only is the right scope.
- **[low-confidence] `exceptions/__init__.py`'s module docstring ("Custom exceptions for hiveplotlib.") becomes stale once the module also exports warning categories**; one-word widening ("Custom exceptions and warning categories for hiveplotlib.") in Workstream A.

Recurring pattern: umbrella-base classes get sold with an "everything" quantifier that the first deliberate exclusion silently falsifies; when a taxonomy carves out a member by design, the carve-out belongs in the base class's first docstring sentence and in every "filter it all" recipe, not in a footnote.

### API Critic — post-implementation review

**Workstream A (2026-07-06):**

```
Status: propose
API surface reviewed: [HiveplotlibWarning, HiveplotlibDeprecationWarning,
  EdgeKwargConflictWarning, MissingPlotElementWarning, UnsupportedVizFeatureWarning,
  hiveplotlib.exceptions star export, CHANGELOG v0.29.0 Added entry]
Concerns:
  - [must-fix] HiveplotlibWarning's docstring claims "a single filter on this class covers
    every non-deprecation warning hiveplotlib emits" — false in the shipping v0.29 — at
    src/hiveplotlib/exceptions/warning_categories.py:14 (worktree hiveplotlib-57).
    Amendment 1 closed the deprecation falsifier of this quantifier, but this plan's own
    Holdouts table is a second falsifier: 14 sites (hiveplot.py x13, p2cp.py x1) stay bare
    UserWarning until the deferred post-merge sweep, and UserWarning is not a subclass of
    HiveplotlibWarning, so a user's one base-class ignore line under an error baseline still
    errors on the common "No edges exist between axes ... run connect_axes()" family. Same
    defect shape the adversary escalated to must-fix in Amendment 1 (false "every" quantifier
    landing on exactly the warnings-as-errors user this plan serves).
    Suggested change: quantify over categories, not emissions — "A single filter on this class
    covers every named non-deprecation warning category hiveplotlib defines" (true both before
    and after the holdout sweep); one-clause edit in the module WS A owns.
    Downstream bearing: Workstream C's planned two-line "silence everything hiveplotlib emits"
    recipe (Amendment 1) is falsified by the same holdouts pre-sweep; its done-when wording
    should get the same categories-not-emissions honesty when the amend lands.
  - [worth-discussing] No test exercises the filter-STRING path users actually type
    (ignore::hiveplotlib.exceptions.X); all filter tests pass category objects — at
    tests/exceptions_warning_categories_test.py (filter tests, lines 69-187). The plan calls
    the export path "the filter-string path" and it IS the load-bearing surface: a future
    __all__ addition or export drop would pass every current test while breaking every
    downstream ini line. Suggested change: one test decorated
    @pytest.mark.filterwarnings("ignore::hiveplotlib.exceptions.HiveplotlibWarning") warning a
    viz category — pytest's marker resolves the string through the same import-module +
    getattr path as stdlib -W / ini filters, so misresolution fails loudly at test setup.
  - [low-confidence] The CHANGELOG Added entry's only worked filter example is the deprecation
    class, the one category with zero call sites in v0.29 (nothing is deprecated) — at
    CHANGELOG.rst:18-20 (worktree). The warnings-as-errors reader is hit today by the viz
    categories. Suggested change: use MissingPlotElementWarning as the example (or add it
    beside the deprecation line); WS B's Changed entry partially covers this, hence
    low-confidence.
Test-method-coverage audit: clean — module ships classes only (no methods); all 12 test
  functions sampled, each test_<subject>_<scenario> name exercises its named filter/subclass
  contract in the body; pytestmark unmarked matches house convention.
```

Walked clean (no concern): filter-string resolution mechanics verified — star export exposes
all five names from `hiveplotlib.exceptions` with no `__all__`, matching the sibling
`hive_plot.py` / `node.py` convention, so stdlib/pytest category resolution (import module,
getattr class) resolves; `hiveplotlib/__init__.py` exports no warning classes, preserving the
single canonical filter path from the planning take; the exclusion leads `HiveplotlibWarning`'s
first docstring sentence per Amendment 1; `HiveplotlibDeprecationWarning` states the
one-minor-version policy in one self-contained sentence; docstrings carry no third-party names
and no process refs (Amendment 6), wrap under 120, and "back end" matches CHANGELOG house
spelling; `exceptions/__init__.py` docstring widened per Amendment 8; the deliberate
repetition of "keep their standard visibility and filtering behavior" across the base and
deprecation docstrings is good (each class reads standalone).

**Workstream B (2026-07-06):**

```
Status: clean
API surface reviewed: [category migration across the 26 viz warn sites —
  EdgeKwargConflictWarning (base.py x3), MissingPlotElementWarning (input_checks.py x10),
  UnsupportedVizFeatureWarning (bokeh/holoviews/plotly hover x12 + datashader multiple-tags x1);
  the five viz test files' category assertions; CHANGELOG v0.29.0 Changed entry]
Concerns:
  - [low-confidence] The datashader multiple-tags site (viz/datashader.py:146,
    "Multiple tags detected between edges. Only plotting tag {tag}") is bucketed under
    UnsupportedVizFeatureWarning alongside the 12 hover sites. The fit is honest but the
    site is a different KIND from its bucket-mates: the 12 hover sites fire because a
    parameter the user PASSED (hover=...) can't be honored, whereas the datashader site
    fires because a DATA SHAPE the user built (multiple edge tags) can't be honored — one
    tag is silently rendered, the rest dropped. A user filtering by category to silence
    "the P2CP-hover noise I passed on purpose" would ALSO silence the multi-tag data-loss
    notice, which is arguably signal (you lost edges from the plot), not noise. Both share
    the "input disregarded by this viz path" framing the plan names, and Amendment 4 already
    reasoned this through, so this is not a re-bucket ask. Low-confidence, flagging for the
    record: if the deferred hiveplot.py/p2cp.py sweep ever splits UnsupportedVizFeatureWarning
    (it won't in 0.29), the passed-param-vs-dropped-data seam is where the split would fall.
    Suggested change: none for 0.29; note the seam in Holdouts so the sweep-time namer sees it.
Test-method-coverage audit: clean. Sampled the tightened assertions across all five viz test
  files (Implementation log 2026-07-06 WS B tests entry, machine-run): each of the three
  categories is asserted where its sites fire (MissingPlotElementWarning on no-axes/no-edges,
  EdgeKwargConflictWarning on repeat-kwarg, UnsupportedVizFeatureWarning on hover-P2CP and the
  sole datashader no-tag site). The three viz categories are proper siblings, so a wrong-category
  migration turns the matching pytest.warns red (verified in the log: pytest.warns of the wrong
  sibling raises DID NOT WARN). No migrated viz/ site lacks a category-asserting test. The four
  test_plot_empty_* left bare pytest.warns() is correct and disclosed: those cases co-emit a
  hiveplot.py holdout UserWarning that would re-error under the suite's error baseline if the
  outer context manager were tightened — a Holdout-driven limit, not a test-name-contract gap.
```

Walked the mapping as a downstream user typing three filter lines under `filterwarnings = error`,
against the actual 26 shipped sites (read in worktree hiveplotlib-57):

- **Q: any site in a category a user would not expect there (over-broad / mis-bucketed)?** No
  over-broad or cross-bucket leak. All 3 EdgeKwargConflict sites (base.py:144/153/166) are genuine
  edge-kwarg-hierarchy conflicts with the same "preserving kwargs already set" shape; all 10
  MissingPlotElement sites (input_checks.py:53-129) are plot-before-element cases (axes/nodes/edges
  not yet built) with no edge-kwarg or hover concern smuggled in; the 12 hover sites carry one
  byte-identical message. The one bucketing seam worth naming is the datashader site (concern
  above), and it is low-confidence because the "input disregarded" framing genuinely covers it.
- **Q: can a user cleanly separate the hover-on-P2CP noise they want gone from the empty-plot /
  missing-element signal they want to keep? (the "taxonomy nobody can use" bar)** Yes, cleanly.
  Hover sits under `UnsupportedVizFeatureWarning`; empty-plot / missing-element sits under the
  sibling `MissingPlotElementWarning`. They are distinct classes with no shared base other than
  `HiveplotlibWarning`, so `ignore::hiveplotlib.exceptions.UnsupportedVizFeatureWarning` silences
  the hover noise on P2CPs and leaves every empty-plot / missing-axes warning firing (or erroring)
  untouched. This is the exact separation the rubric's "taxonomy nobody can use" mode worried about,
  and the shipped mapping passes it.
- **Q: is the datashader multiple-tags site a natural fit under UnsupportedVizFeatureWarning?**
  A natural fit on the message-framing axis (input the user supplied is disregarded because the
  viz path can't honor it), a slight stretch on the input-KIND axis (passed parameter vs. built
  data shape). Net: fits, with the seam recorded above. Not a must-fix or worth-discussing.
- **Q: does the CHANGELOG Changed entry give the right copy-paste filter target?** Yes.
  CHANGELOG.rst:30-33 names `ignore::hiveplotlib.exceptions.MissingPlotElementWarning` as the
  worked example — a live-call-site category (10 sites fire in 0.29), so the warnings-as-errors
  reader gets a target that actually silences something today, and the "to silence plot-before-build
  warnings while keeping others" clause correctly promises the sibling separation verified above.
  This closes the Amendment-12 gap where WS A's only worked example was the zero-call-site
  deprecation class; the two entries now cover both the deprecation reader and the today's-viz reader.

Clean overall: the three-category mapping is honestly bucketed, the signal/noise separation the
plan exists to deliver is real against the shipped sites, and the CHANGELOG hands the reader a
filter target that fires in this release. One low-confidence datashader-seam note for the deferred
sweep; nothing routes to amend-plan.

## Notebook review

No notebook change.

## Viz review

No figure change (warning categories do not alter rendering).

## Adversary review

### Adversary's challenge (planning mode)

```
Status: challenge (5 items)
Plan reviewed: wiki/wiki/plans/deprecation-policy-and-warning-categories.md (cold; grill knowingly
  skipped, so the Failure modes rubric is orchestrator-seeded — treated as an input, not the
  maintainer's standard. Items route to orchestrator amend-plan for disposition before dispatch.)

Angles worked:
  - Premise: holds. The blocked workflow is real and specific — a downstream user under
    filterwarnings=error today must either blanket-ignore UserWarning or pin brittle match=
    strings to survive plot-time warnings; the deprecation machinery has a named near-term first
    customer (dataframe_to_node_list) and a just-lived counterexample (hive_plot_n_axes machinery
    deleted in v0.28). Not existential; no item.
  - Approach: mostly right shape (category-only migration, byte-identical messages, holdouts for
    conflict-prone files). Two gaps below (items 2, 4).
  - Size/maintenance: five public classes for an enumerated site population is defensible, but one
    shipped invariant has no mechanical guard (item 3).

Verification note: the working tree is branch 53, not master, and this pass had no git access, so
the master inventory could not be diffed directly. Branch-53 tree counts reconcile with the
claimed 26 master sites: 28 tree sites = 26 + two new branch-53 sites at viz/input_checks.py:168
and :178; datashader's multiple-tags site moved 145 -> 989; small line offsets in bokeh /
holoviews / plotly / input_checks. Nothing contradicts the inventory; Workstream B's
worktree re-verify + rule-9 halt is the right residual guard.

Items:
  - [must-fix] The base-class contract contradicts itself: the Naming audit sells
    HiveplotlibWarning as "one ini line filters (or errors on) everything hiveplotlib emits,"
    while the next bullet deliberately excludes HiveplotlibDeprecationWarning from that base;
    API usage example 2 repeats the false claim ("every hiveplotlib warning"). Found
    independently on this cold read; api-critic's planning take converged on the same defect
    (its first amendment, tagged worth-discussing there). Escalated to must-fix here because the
    failure lands on exactly the user this plan serves: someone sets
    ignore::hiveplotlib.exceptions.HiveplotlibWarning, believes deprecations are covered, and
    hits unwarned removal one minor version later. — at Naming audit + API usage examples
    (Example 2), flowing into Workstream A docstrings and Workstream C's ini guidance.
    Rubric: no entry — flagging anyway (nearest seeded mode: "Taxonomy nobody can use").
    Push: adopt api-critic's remedy and bind it in done-when text: reword the audit and Example 2
    to "every non-deprecation warning hiveplotlib emits," put the exclusion in the first sentence
    of HiveplotlibWarning's docstring (Workstream A), and show the honest two-line "all of
    hiveplotlib" recipe (base + deprecation class) in Workstream C's snippet.
  - [worth-discussing] DeprecationWarning's default invisibility compounds the one-minor-version
    window, and the FutureWarning alternative is never weighed. Not re-litigating the window (a
    recorded maintainer decision) or the subclass choice (per the brief), but: DeprecationWarning
    is hidden by default outside __main__/pytest, so a script or library-mediated user on 0.Y may
    never see the notice before removal in 0.(Y+1). pandas ships FutureWarning for exactly this
    visibility reason; matplotlib/astropy accept the tradeoff with longer effective windows. The
    combination of the shortest window and the least visible category is the aggressive corner of
    the design space, and the plan chose it without naming it. — at Naming audit
    (HiveplotlibDeprecationWarning) + Workstream C.
    Rubric: no entry — flagging anyway.
    Push: record the FutureWarning considered-and-rejected rationale in the Naming audit, and add
    to Workstream C's done-when that the policy page states the default-visibility behavior, so
    the one-minor-version promise is informed rather than silent.
  - [worth-discussing] Workstream B's shipped invariant (no bare warnings.warn in viz/) will be
    silently re-broken by the branch 53 merge, and the re-arm is prose only. Branch 53 already
    carries at least two new bare warn sites inside viz/ itself — viz/input_checks.py:168 and
    :178 (branch-53 lines; the Dask lazy-frame and construct-curves guidance warnings) — inside
    the very file Workstream B edits, so a textual merge collision is also plausible. The
    cross-plan note under-describes this as "a plain-UserWarning helper on the datashader route"
    (singular, and the sites live in input_checks.py, not datashader.py), and the Holdouts
    revival trigger ("after both merge, sweep these") re-arms nothing mechanically: no one
    re-runs the grep gate when the merge lands, so the invariant degrades with every gate green.
    — at Cross-plan coordination + Holdouts + Workstream B QA gate.
    Rubric: "Machinery that rots" (seeded).
    Push: enumerate the branch-53 viz/ sites in Holdouts by file and correct the coordination
    note; name a mechanical trigger for the deferred sweep (e.g., "the follow-up sweep's first
    step re-runs Workstream B's grep gate over all of src/, and any viz/ hit is in-scope"), and
    warn the implementer that viz/input_checks.py is a two-branch merge surface.
  - [worth-discussing] The taxonomy ships before the largest warning family is classified. The 14
    hiveplot.py/p2cp.py holdouts (plus !51's vmin/vmax warning) will later be retrofitted onto
    names chosen from viz-only evidence, and Holdouts already concedes the sweep "may need one
    new category." Public class names are forever; a poor fit later means either stretching
    UnsupportedVizFeatureWarning to build-time semantics or fragmenting the taxonomy. — at
    Naming audit + Holdouts.
    Rubric: "Taxonomy nobody can use" (seeded).
    Push: do the cheap read now — map the 14 holdout messages against the three viz categories
    (a short table in Holdouts) and record whether they fit or a fourth build-time category is
    anticipated, so 0.29's shipped names are chosen with the whole warning population in view
    even though the migration itself stays deferred.
  - [low-confidence] Gate-battery mismatch between the execution context and the done-when lists:
    the Execution context mandates `make docs` in every per-workstream commit battery, but
    Workstreams A and B list only format/test/ty. Implementers build to done-when, so the
    recorded battery quietly shrinks for two of three commits. — at Execution context vs
    Workstreams A/B done-when.
    Rubric: no entry — flagging anyway.
    Push: add `make docs` to A's and B's done-when bullets, or amend the Execution context to
    scope docs builds to Workstream C; either way, make the two statements agree.
```

### Adversary post-impl

Per-workstream stubs (blind-first two-message dispatch per harness convention):

**Workstream A (2026-07-06):**

```
Status: propose
Artifact reviewed: Workstream A — uncommitted diff in worktree hiveplotlib-57
  (exceptions/warning_categories.py new, exceptions/__init__.py,
  tests/exceptions_warning_categories_test.py new, CHANGELOG.rst)
Dispositions held: yes. Amendments binding WS A are all visible in the shipped artifact:
  the exclusion leads HiveplotlibWarning's first docstring sentence (Amendment 1); the full
  per-commit battery incl. make docs is in the done-when (Amendment 5); docstrings name no
  third-party libraries (Amendment 6); exceptions/__init__.py docstring widened (Amendment 8).
  No scope ballooning — the diff is exactly the four listed files; the emergent API-docs
  finding routed to amend-plan (Amendment 9) instead of being smuggled into the WS A commit.
Blind verification, clean: the "Hollow test" rubric mode is defeated (filter tests exercise
  real warnings machinery under an error baseline: per-category ignore with sibling escaping,
  base coverage, base non-coverage of the deprecation class, stock UserWarning /
  DeprecationWarning interactions); "Stacklevel drift" and "Message churn" cannot bite this
  diff (no library warn site touched — WS B territory); pytest.mark.unmarked matches the
  registered house convention (pytest.ini:21); the one-minor-version policy sentence and the
  exclusion sentence match the done-when text; no process refs, no em-dashes.
Concerns:
  - [worth-discussing] The documented filter-string form (ignore::hiveplotlib.exceptions.X,
    module docstring line 5 and CHANGELOG.rst:18-20) is exercised by no test; all filter
    tests pass category objects — at tests/exceptions_warning_categories_test.py:69-187.
    Found blind; converged with api-critic post-impl (same item, same remedy: one test
    decorated @pytest.mark.filterwarnings resolving the string). Route once, with the
    critic's item. Rubric: no entry.
  - [low-confidence] The three viz-category docstrings assert emission behavior ("Fires at
    plot time when...") that exists only after WS B migrates the sites — at
    warning_categories.py:39-55. Fine if WS B lands in the same release; verify at WS B
    post-impl that the migrated site-to-category mapping matches these descriptions.
    Rubric: no entry — verification item, not a defect today.
  - [concur, no separate item] api-critic post-impl's must-fix stands: "covers every
    non-deprecation warning hiveplotlib emits" (warning_categories.py:14) is falsified
    pre-sweep by the 14 bare-UserWarning holdouts. My blind pass checked that line against
    the Amendment 1 wording and passed it; the critic's second falsifier is real and the
    categories-not-emissions rewording is the right fix (same for WS C's two-line recipe
    wording). Rubric: "Taxonomy nobody can use" (adjacent).
Reconciled against the blind pass:
  - Blind must-fix (five classes invisible in API docs, no autodoc rST / toctree line;
    rubric "Machinery that rots") was already disposed by Amendment 9, which landed after
    my planning pass: the manual rST + toctree line are assigned to WS C's files list and
    done-when. Disposition is sound (WS C owns the docs surface; keeps WS A's one-commit
    boundary clean). Downgraded to a WS C post-impl verification item: the rubric mode
    stays open until that page ships.
  - Blind worth-discussing (one-minor-version cadence on record vs ADR 0001's two-version
    note; rubric "Two cadences on record") was already disposed at planning: Prior ADRs
    records the maintainer supersession decision (2026-07-06) with the durable supersession
    record written at ADR promotion at plan close. Verification item for plan close, not a
    concern now.
  - Blind subtraction nits (repeated visibility clause across two docstrings; "back end"
    spelling) withdrawn: api-critic post-impl walked both deliberately clean (standalone
    docstring reading; CHANGELOG house spelling), and the counter-rationale holds.
```

**Workstream B (2026-07-06):**

```
Status: propose
Artifact reviewed: Workstream B — uncommitted diff on branch 57 in worktree hiveplotlib-57
  (six viz/ source files: base.py, input_checks.py, bokeh.py, holoviews.py, plotly.py,
  datashader.py; five viz test files: viz_matplotlib_test.py, viz_bokeh_test.py,
  viz_holoviews_test.py, viz_plotly_test.py, datashader_test.py; CHANGELOG.rst). WS A is the
  committed baseline (7c1a317); this diff isolated from it.
Dispositions held: yes. Every WS-B-bearing amendment is visible in the shipped artifact.
  Amendment 3 (branch-53 merge surface): the two branch-53 input_checks sites are correctly
  absent from the master-cut worktree (no false rule-9 halt); the re-verified inventory is 26,
  not 28. Amendment 4 (holdout classification): the shipped three-category taxonomy matches the
  classification table — no build-time advisory holdout was wrongly pulled into a viz category.
  Amendment 12 (CHANGELOG live category): the Changed entry names
  ignore::hiveplotlib.exceptions.MissingPlotElementWarning as the copy-paste target, exactly as
  nudged, without reopening WS A's Added entry. No scope ballooning: the diff is exactly the
  twelve listed files; base.py's only non-category= change is the import widened to add
  EdgeKwargConflictWarning. No new helpers, methods, or surface.
Blind verification, clean on the contract:
  - 26/26 sites migrated, category correct on the actual warn-site walk (not just the table):
    base.py:144/153/166 -> EdgeKwargConflictWarning (kwarg-already-set); input_checks.py 10
    sites -> MissingPlotElementWarning (empty axes/edges/nodes at plot time); bokeh/holoviews/
    plotly 12 hover sites + datashader:146 (multiple-tags) -> UnsupportedVizFeatureWarning. No
    cross-wiring (no empty-plot condition mis-tagged Unsupported, no hover mis-tagged
    MissingPlotElement). The three categories are cleanly filter-separable, defeating the
    "Taxonomy nobody can use" rubric mode.
  - Stacklevel preserved exactly: all 13 EdgeKwarg/MissingPlotElement sites stacklevel=3, all
    13 Unsupported sites stacklevel=2 — the per-site 2/3 split the inventory demands, no drift
    ("Stacklevel drift" rubric mode defeated).
  - Grep gate honest: zero bare warnings.warn( without category= in viz/. Holdouts intact —
    hiveplot.py 13 bare / 0 category=, p2cp.py 1 bare / 0 category=, neither file touched.
  - Tests falsifiable per category: each of the five test files asserts the specific migrated
    category (bokeh/holoviews/plotly all three; datashader Unsupported + MissingPlotElement,
    correctly no EdgeKwarg site), and the category asserts pin message substrings ("Hover info
    not yet supported for P2CPs", "build_hive_plot()", multiple-tags) that corroborate the text
    at those sites is unchanged.
  - CHANGELOG (CHANGELOG.rst:30-33): sold category-only ("still subclass UserWarning, so
    existing filters keep matching"), one bullet / four lines, honest altitude, within cap, no
    process refs, no em-dashes.
Reconciled against the blind pass:
  - Blind worth-discussing (five test_plot_empty_* tests use bare pytest.warns() and leave
    three migrated MissingPlotElementWarning sites under-asserting; rubric "Hollow test"):
    WITHDRAWN — covered atomically, not a residual gap. Context I lacked blind (Implementation
    log 2026-07-06 + dispatch): each empty-plot test co-emits a hiveplot.py Holdout bare
    UserWarning ("intermediate changes"), which under the suite's filterwarnings=error baseline
    would re-error if the outer pytest.warns(X) context were tightened to a single category the
    holdout does not match — so the bare context is forced, not lazy. The three input_checks
    sites those tests touch ARE tightened to MissingPlotElementWarning in the atomic
    single-condition tests (test_warnings_no_axes_hive_plot / test_warnings_missing_content_*
    and per-backend siblings), where they fire in isolation; a wrong-category regression at any
    of them turns those atomic tests red. Verified in the log: pytest.warns(EdgeKwargConflict-
    Warning) raises DID NOT WARN on a MissingPlotElementWarning (proper siblings). The
    regression is caught; the empty-plot bare context is the correct handling of the
    holdout-coemission constraint, not an under-assertion to fix. Final tag: not a gap.
  - Blind low-confidence (byte-identical message text not machine-verified — the blind pass had
    no git, so text was confirmed by read + test-substring corroboration only, which cannot
    span a whitespace-only nudge inside a multi-line f-string): the dispatching session is
    routing a git diff -w message-preservation check to qa to close this mechanically. The
    Implementation log already asserts byte-identical message + stacklevel with the import line
    as the only non-category= diff, and everything else held, so this is expected clean. Final
    tag: low-confidence verification item, routed to plan-end qa (not amend-plan; no defect).
Concerns: none requiring amend-plan. Both blind findings resolved above (one withdrawn as
  covered atomically, one handed to qa's git diff -w). No must-fix, no downstream-bearing
  worth-discussing.
```
**Workstream C (2026-07-06):**

```
Status: propose
Artifact reviewed: Workstream C — uncommitted diff on branch 57 in worktree hiveplotlib-57
  (deprecation_policy.rst new, index.rst, autodoc/exceptions/warning_categories.rst new,
  autodoc/index.rst, CONTRIBUTING.md, llms.txt, CHANGELOG.rst). WS A (7c1a317) + WS B
  (65e6876) are the committed baseline; this diff isolated from them. Attacked blind first
  (only the WS C block, done-whens, Failure modes, Holdouts), then reconciled against my
  planning challenge + the amendment dispositions.
Dispositions held: yes, with one CHANGELOG structural defect the log did not catch. Every
  WS-C-bearing amendment landed in the artifact: Amendment 9 (the autodoc rST page +
  toctree line after exceptions/hive_plot ship, matching the sibling shape — closes the
  "Machinery that rots" carry-in from the Pending stub; the five classes are now
  discoverable in the API reference); Amendment 1+10 (the two-line silence recipe carries
  the categories-not-emissions honesty and names the hiveplot.py/p2cp.py bare-UserWarning
  holdouts as a known gap — verified those sites at hiveplot.py:1537-1553 are genuinely
  bare UserWarning, so the honesty claim is factually accurate, not hand-waved); Amendment 2
  (default-invisibility surfacing guidance present); Amendment 7 (message template
  byte-identical to the canonical Example-3 string). No scope ballooning: the diff is exactly
  the seven listed files; no new page, no rebalance beyond the one authorized llms.txt entry.
Blind findings, reconciled:
  - [must-fix] The CHANGELOG v0.29.0 "Documentation" heading is underlined with ^^^^ (13
    carets, line 23), which this file's own convention makes a sub-topic nested UNDER the
    preceding ~~~~ "Added" heading — not a sibling category. Confirmed against the v0.28.0
    block: "Added" (~) contains "NetworkX Compatibility" (^) as a child, so ^ is a
    within-category sub-heading here, never a top-level category. Result: the llms.txt and
    deprecation-policy bullets render as a "Documentation" sub-topic filed inside "Added"
    (i.e. under "Added API"), and the done-when wanted a Documentation category entry
    (line 663) peer to Added/Changed/Tooling Changes. — at CHANGELOG.rst:22-23 (worktree
    hiveplotlib-57). Rubric: no entry (build/structure correctness against the done-when's
    "CHANGELOG v0.29.0 Documentation entry"). The Implementation log (line 749) asserts
    "make docs clean"; docutils does not necessarily warn on this level jump (^ after ~ with
    content between is legal nesting), so a green make docs does not clear it — the defect is
    semantic mis-nesting, not a build break. Objective wrongness against the file's
    established convention, mechanical, no design content, no cascade: the fix is a one-line
    underline change (^^^^^^^^^^^^^ -> ~~~~~~~~~~~~~ at line 23) by docs-engineer. No
    amend-plan needed (not a scope change); the dispatching session halts to the maintainer
    per the standing must-fix halt rule.
  - [low-confidence -> WITHDRAWN] llms.txt added a "### Warnings and deprecations" subsection
    header rather than a bare line under ## Optional, which I flagged blind against the
    done-when's "one line, not a rebalance." Reconciled: DEFENSIBLE, withdrawn. Two reasons.
    (1) Every other ## Optional entry sits under a ### topic group, so a bare ungrouped line
    would be the inconsistent choice — the subsection matches the section's established shape.
    (2) The Implementation log (line 749) records this as a deliberate, reasoned decision:
    "one line under a small 'Warnings and deprecations' subsection so it doesn't masquerade
    as a gallery notebook." That is a considered call, not accidental surface creep, and the
    "one line, not a rebalance" done-when is satisfied in spirit (one content line, no
    reordering of existing entries). Not a defect. Final tag: withdrawn.
Concerns requiring routing: one must-fix (CHANGELOG heading level), halting to the maintainer
  per the standing route (one-char underline fix, docs-engineer, no amend-plan). No
  downstream-bearing worth-discussing (WS C is the last workstream; the CHANGELOG fix has no
  bearing on any not-yet-run workstream). Everything else — machinery-that-rots discoverability
  (autodoc page wired into the toctree), taxonomy honesty (holdouts named as a known gap),
  two-cadences (one-minor-version stated consistently, no stray two-version claim),
  message-template fidelity (byte-identical to canonical), process-refs / em-dashes / AI-filler
  (none across all seven files), toctree/build correctness (policy page in index.rst under
  Release Notes, not orphaned; :doc:/:class: cross-refs resolve; section underlines safe) —
  walked and passed.
```

**Workstream C (2026-07-06):**

```
Status: clean
API surface reviewed: [deprecation_policy.rst (the user-facing filtering GUIDANCE and the
  deprecation message TEMPLATE), the two-line silence recipe, the DeprecationWarning
  default-invisibility surfacing guidance, autodoc/exceptions/warning_categories.rst,
  index.rst + autodoc/index.rst toctree entries, CONTRIBUTING.md Deprecations pointer,
  llms.txt line, CHANGELOG v0.29.0 Documentation entry]
Concerns: none requiring amend-plan.
Test-method-coverage audit: n/a — WS C ships docs/rST/prose only (no source methods, no
  tests). The load-bearing filter-STRING resolution guard lives in WS A
  (test_base_ignore_filter_string_silences_viz_categories, Amendment 11), which fires the
  exact ignore::hiveplotlib.exceptions.HiveplotlibWarning string this page documents, so the
  copy-paste paths the page hands the user are machine-verified to resolve.
```

Walked deprecation_policy.rst as the two downstream users the page serves: (1) a warnings-as-errors
user silencing a hiveplotlib category, (2) a user trying to survive a future deprecation. Every
brief check passes.

- **Q: are the two-line silence recipe's importable paths correct and copy-pasteable, and is the
  "named categories, not everything emitted" framing honest per Amendment 10?** Yes on both.
  Every filter-string occurrence in the page uses the short star-exported path
  (`hiveplotlib.exceptions.HiveplotlibWarning` / `...HiveplotlibDeprecationWarning`, lines 54, 62,
  68, 80-81), which resolves through the same `import hiveplotlib.exceptions; getattr(...)` path the
  star export in `exceptions/__init__.py:7` provides and WS A's filter-string test exercises. (The
  one long-form path, line 14's `:class:` cross-reference to
  `hiveplotlib.exceptions.warning_categories.HiveplotlibDeprecationWarning`, is a Sphinx role
  needing the class's canonical `__module__`, a different mechanism from a filter string; it points
  at the autodoc page and is correct there, not a filter-string the user copies.) The honesty
  framing is load-bearing and correct: lines 83-87 name the `hiveplot.py`/`p2cp.py` bare-`UserWarning`
  holdouts explicitly ("the 'no edges exist between axes' notices"), state the two lines do not cover
  them, and hand the user the fallback (filter by message or by `UserWarning`). The quoted holdout
  message paraphrases the shipped text (`hiveplot.py:1538` "No edges exist between axes ...") so it is
  a recognizable pointer, not a stale verbatim quote. This closes the exact defect Amendment 10
  corrected on WS A's docstring, carried into WS C's recipe as the amendment required.
- **Q: does the page tell the user how to surface deprecation warnings they'd otherwise never see, so
  the one-minor-version promise is real? (Amendment 2)** Yes. The "Seeing deprecation warnings"
  section (lines 27-47) states the default-invisibility behavior (hidden outside `__main__`/pytest),
  explains the concrete failure (call a deprecated feature from an imported module and you may miss
  the notice while the removal clock runs), and gives two actionable surfacing recipes:
  `python -W default::DeprecationWarning your_script.py` and
  `warnings.filterwarnings("default", category=DeprecationWarning)`. Both filter on the stdlib base
  `DeprecationWarning`, which is the right call (catches the hiveplotlib subclass and is the safer
  default than pinning the subclass), and the pytest note is accurate. The one-minor-version promise
  is informed, not silent.
- **Q: does the message template give the next maintainer a correct, copy-pasteable template
  (deprecated name -> replacement -> removal version)? (Amendment 7)** Yes. Lines 18-25 state the
  fixed shape (deprecated name, then replacement, then removal version) and give a worked example with
  real version numbers: "``old_helper`` is deprecated and will be removed in v0.31.0; use
  ``new_helper`` instead." (deprecated in 0.30.0, removed in 0.31.0). Byte-identical to the plan's
  Example-3 template that the api-critic planning take asked to canonicalize, and the version
  arithmetic is internally consistent (0.Y=0.30, 0.(Y+1)=0.31, the "0.30.x window" prose on line 24
  matches). The next maintainer copies this verbatim.
- **Q: does the filter-string form a user types match what WS A/WS B ship (categories resolve at
  those import paths)?** Yes. `HiveplotlibWarning`, `HiveplotlibDeprecationWarning`,
  `MissingPlotElementWarning`, and `UnsupportedVizFeatureWarning` (all named in the page prose and/or
  recipes) are star-exported from `hiveplotlib.exceptions` per WS A, and the viz categories carry live
  call sites per WS B, so every filter target the page documents fires against a real emission in
  v0.29 except `HiveplotlibDeprecationWarning` (zero call sites by design, correctly framed as the
  future-deprecation channel). The autodoc page (`autodoc/exceptions/warning_categories.rst`,
  Amendment 9) ships with the correct `.. automodule::` shape and its toctree line
  (`autodoc/index.rst:65`), and the policy page links to it (line 90), so the five class names are
  now discoverable in the API reference — closing the "Machinery that rots" carry-in check from the
  WS C Pending stub. index.rst toctree (line 30, under Release Notes beside changelog), CONTRIBUTING
  Deprecations pointer (absolute stable link), and the llms.txt line all present and resolve.

Clean overall: the page is the honest, copy-pasteable user surface the plan set out to ship. The
silence recipe's paths resolve, the categories-not-emissions framing carries the Amendment 10 honesty
into the recipe and names the holdouts as a known gap, the DeprecationWarning surfacing guidance makes
the one-minor-version promise real, and the message template hands the next maintainer a correct
template. Nothing routes to amend-plan.

## Workstreams

Sequence: A → B → C. B needs A's classes; C names A's deprecation class and is otherwise independent.

### Workstream A: Warning-category classes in exceptions/

**Status:** not started
**Files:** `src/hiveplotlib/exceptions/warning_categories.py` (new), `src/hiveplotlib/exceptions/__init__.py`, `tests/exceptions_warning_categories_test.py` (new), `CHANGELOG.rst`
**Done when:**

- The five classes from the Naming audit exist in the new module with docstrings (120-char wrap) stating each class's purpose and, for `HiveplotlibDeprecationWarning`, the one-minor-version policy in one sentence (self-contained; no plan/process refs). The first sentence of `HiveplotlibWarning`'s docstring states the exclusion: base class for all hiveplotlib warning categories *except* `HiveplotlibDeprecationWarning` (Amendment 1). The base docstring's coverage sentence quantifies over categories, not emissions: "a single filter on this class covers every named non-deprecation warning category hiveplotlib defines" (true before and after the deferred holdout sweep; the earlier "every non-deprecation warning hiveplotlib emits" wording is falsified pre-sweep by the bare-`UserWarning` holdouts — Amendment 10). Docstrings name no third-party libraries; the precedent record lives in this plan, not shipped text (Amendment 6).
- `exceptions/__init__.py` star-imports the new module (matching the `hive_plot.py` / `node.py` lines), so all five classes are importable from `hiveplotlib.exceptions` — the path filter strings resolve. Its module docstring widens to "Custom exceptions and warning categories for hiveplotlib." (Amendment 8).
- Tests (flat `tests/` mirror convention; test name = test body contract):
  - subclass contract: each viz category `issubclass` of both `HiveplotlibWarning` and `UserWarning`; `HiveplotlibDeprecationWarning` `issubclass` of `DeprecationWarning` and deliberately NOT of `UserWarning` / `HiveplotlibWarning` (asserted, so the visibility decision can't regress silently).
  - filter behavior, not just hierarchy: under `simplefilter("error")`, a `warnings.warn(..., category=HiveplotlibDeprecationWarning)` raises; adding `filterwarnings("ignore", category=HiveplotlibDeprecationWarning)` silences it; same shape for one viz category via the `HiveplotlibWarning` base. This is the "machinery stays tested while nothing is deprecated" requirement (test-engineer owns specifics).
  - filter-STRING resolution (Amendment 11): one test decorated `@pytest.mark.filterwarnings("ignore::hiveplotlib.exceptions.HiveplotlibWarning")` warns a viz category and expects it silenced, exercising the documented `ignore::hiveplotlib.exceptions.X` path (the marker resolves the string through the same import-module + `getattr` path as stdlib `-W` / ini filters). This guards the load-bearing star-export surface the Naming audit calls "the filter-string path": a future dropped or renamed export fails loudly at test setup instead of passing every object-based test while breaking downstream ini lines.
- `make format`, `make test` (100% coverage), `make ty`, `make docs` pass — the full per-commit battery from the Execution context (Amendment 5).
- CHANGELOG.rst v0.29.0 Added entry: new warning categories in `hiveplotlib.exceptions`, named as filterable classes for users running warnings-as-errors.

### Workstream B: Migrate viz backend warnings to named categories

**Status:** not started
**Files:** `src/hiveplotlib/viz/base.py`, `viz/bokeh.py`, `viz/datashader.py`, `viz/holoviews.py`, `viz/input_checks.py`, `viz/plotly.py`; `tests/viz_matplotlib_test.py`, `tests/viz_bokeh_test.py`, `tests/viz_holoviews_test.py`, `tests/viz_plotly_test.py`, `tests/datashader_test.py`; `CHANGELOG.rst`
**Done when:**

- All 26 sites in "Patterns this replaces" pass `category=<class>` per the mapping there (code-engineer re-verifies the inventory in the worktree; if the worktree's sites differ from the inventory, halt per rule 9 rather than improvising).
- Merge-surface flag (Amendment 3): `viz/input_checks.py` is concurrently edited by branch 53, which adds two bare warn sites in `check_curves_exist` (see Holdouts). The worktree is cut from master, so those sites will NOT appear in the re-verified inventory — their absence is expected, not a rule-9 mismatch. The eventual branch-53 merge re-breaks the grep-gate invariant; that is handled by the Holdouts re-arm, not this workstream.
- The diff at each site is category-only: message text and stacklevel byte-identical (per-site stacklevels 2/3 preserved exactly; no rewording).
- Tests exercising the migrated sites tighten from bare `pytest.warns()` / `pytest.warns(UserWarning)` to the specific new category, so each of the three categories is asserted where its sites fire (test-engineer owns which parametrizations to tighten; every migrated file has at least one category-asserting test).
- QA grep gate: no `warnings.warn(` in `src/hiveplotlib/viz/` without `category=`; `hiveplot.py` / `p2cp.py` survivors expected per Holdouts.
- `make format`, `make test` (100% coverage, warnings-as-errors), `make ty`, `make docs` pass — the full per-commit battery from the Execution context (Amendment 5); backend-marked tests pass (`pytest -m bokeh`, etc. — covered by `make test`).
- CHANGELOG.rst v0.29.0 Changed entry: viz warnings now carry named categories; they still subclass `UserWarning`, so existing filters keep matching. The entry names at least one live viz category as the copy-paste filter target (e.g. `ignore::hiveplotlib.exceptions.MissingPlotElementWarning`), so the freshest-customer worked example — a category that actually fires in v0.29 — lands in this release without reopening WS A's shipped Added entry (Amendment 12). (These warnings shipped in released versions, so the entry is earned under the released-behavior rule.)

### Workstream C: Deprecation policy page + CONTRIBUTING pointer

**Status:** not started
**Files:** `docs/source/deprecation_policy.rst` (new), `docs/source/index.rst`, `docs/source/autodoc/exceptions/warning_categories.rst` (new, Amendment 9), `docs/source/autodoc/index.rst` (Amendment 9), `CONTRIBUTING.md`, `docs/source/_llms/llms.txt`, `CHANGELOG.rst`
**Done when:**

- **Placement call (orchestrator, argued):** the policy lives in the user docs as a new small page, not in CONTRIBUTING. Reasons: (1) a deprecation window is a promise to users about notice before breakage — an API contract read by people who never open a contributor guide; (2) the filterwarnings guidance belongs beside it and is pure user documentation; (3) the scaling plan scoped CONTRIBUTING to maintainer-facing release guidance "until a releases-specific doc exists" — this page is that doc for deprecations; (4) JOSS reviewers read CONTRIBUTING, and a pointer there still surfaces the policy. CONTRIBUTING gets the pointer, not the policy.
- `docs/source/deprecation_policy.rst` states: the one-minor-version window (deprecated in 0.Y with a `HiveplotlibDeprecationWarning` naming the replacement and the removal version; removed in 0.(Y+1)); a concrete worked example with real version numbers whose warning text canonicalizes the message template from Example 3 — deprecated name, replacement, removal version, in that order ("`X` is deprecated and will be removed in vN.N.N; use `Y` instead") — so the next deprecation copies it instead of re-improvising (Amendment 7); that each deprecation is listed under the CHANGELOG's Deprecated heading; that `DeprecationWarning`s are hidden by default outside `__main__` and pytest, and how to surface them, so the one-minor-version promise is informed rather than silent (Amendment 2); how to surface or silence the warnings (a `filterwarnings` ini snippet using the importable path `hiveplotlib.exceptions.HiveplotlibDeprecationWarning`, plus the honest two-line "silence hiveplotlib's named categories" recipe — `HiveplotlibWarning` and `HiveplotlibDeprecationWarning`, because the deprecation class deliberately sits outside the base; Amendment 1). The recipe is framed as covering hiveplotlib's *named warning categories* (base line covers every defined non-deprecation category, second line covers the deprecation class), not "everything hiveplotlib emits": the prose names the pre-sweep bare-`UserWarning` holdouts in `hiveplot.py`/`p2cp.py` as a known gap those two lines do not yet cover, so the recipe is not oversold to the warnings-as-errors reader (categories, not emissions — Amendment 10); a pre-1.0 note (SemVer allows breakage at 0.x; this policy is the stronger promise hiveplotlib makes anyway). No plan/ADR/process references; prose per the writing-voice rules (no em-dashes, no AI filler).
- The page is in `docs/source/index.rst`'s toctree under the "Release Notes" caption, beside the changelog.
- CONTRIBUTING.md gains a short "Deprecations" note in the Contributing Code section: changes that remove or rename public API follow the deprecation policy, with an absolute link (`https://hiveplotlib.readthedocs.io/stable/deprecation_policy.html`).
- `docs/source/_llms/llms.txt`: one line in `## Optional` pointing at the policy page (absolute URL). Judgment call, recorded: this clears the "consequential to how someone uses the library" bar because warning filtering under warnings-as-errors is a real downstream entry point; it is one line, not a rebalance. Docs-engineer may push back if it reads as bloat in context; removal is an acceptable outcome if argued. Second judgment call, recorded (Amendment 9): the new `warning_categories` API page gets NO llms.txt line of its own — llms.txt indexes the API reference once at `autodoc/index.html` ("every public class and function"), no per-module autodoc page carries an entry, and the user-facing entry point for warning filtering is the policy-page line above.
- API-reference page for the new module (Amendment 9): new `docs/source/autodoc/exceptions/warning_categories.rst` matching the sibling shape (see `exceptions/hive_plot.rst`: a title plus `.. automodule:: hiveplotlib.exceptions.warning_categories` with `:members:`), and an `exceptions/warning_categories` line in the Exceptions toctree at `docs/source/autodoc/index.rst` (currently lines 59-64, after `exceptions/hive_plot`). Docs-engineer verifies the built page renders all five class docstrings.
- `make docs` builds with no new warnings on the touched pages; `make format` clean.
- CHANGELOG.rst v0.29.0 Documentation Added entry for the policy page.

## Plan amendments

The standing route fired (2026-07-06): the grill was knowingly skipped, so both planning-mode critic sections routed here for explicit disposition before any workstream dispatch. Nine tagged items (adversary 5, api-critic 4) disposed in eight amendments; every item adopted, none ruled out. The workstream set (A/B/C) is unchanged. Full record surfaced to the maintainer at plan-end per the standing route.

WS A post-impl route (2026-07-06): the api-critic and adversary post-impl passes on the shipped-but-uncommitted WS A artifact routed here (rule 14, post-impl `must-fix`/`worth-discussing`). Four items disposed in Amendments 10-12: the umbrella-quantifier must-fix (categories-not-emissions, converged api-critic + adversary), the filter-string-path test (converged, folded into WS A pre-commit), and the CHANGELOG worked-example freshness (declined for WS A, nudged into WS B). The workstream set stays A/B/C. The must-fix halts auto-dispatch back to the maintainer — see the dispatch note surfaced with this amendment pass.

### Amendment 1 — Umbrella overclaim corrected (adversary must-fix; api-critic worth-discussing, converged)

**Delta:** "one ini line filters (or errors on) everything hiveplotlib emits" reworded to "every non-deprecation warning" in the Naming audit and Example 2; the `HiveplotlibDeprecationWarning` exclusion now leads `HiveplotlibWarning`'s docstring first sentence (Workstream A done-when); Workstream C's ini snippet shows the honest two-line "all of hiveplotlib" recipe (base + deprecation class). **Rationale:** the false quantifier lands on exactly the user this plan serves — an `ignore::...HiveplotlibWarning` line believed to cover deprecations means unwarned removal one minor version later. The single-inheritance design itself stands; only the claim about it changes. **Done-whens touched:** A, C. **Tag:** In-scope tweak.

### Amendment 2 — FutureWarning considered-and-rejected; default visibility named (adversary worth-discussing)

**Delta:** the Naming audit's `HiveplotlibDeprecationWarning` bullet records the `FutureWarning` rejection rationale (maintainer's brief names `DeprecationWarning`; `FutureWarning` lands on end users of downstream code who cannot act) and names the accepted cost: shortest window paired with the least visible category. Workstream C's done-when now requires the policy page to state the default-visibility behavior and how to surface the warnings. **Rationale:** the aggressive corner of the design space was chosen without being named; the choice stands (recorded maintainer decision), the silence does not. **Done-whens touched:** C. **Tag:** In-scope tweak.

### Amendment 3 — Branch-53 viz/ sites enumerated; invariant re-arm made mechanical (adversary worth-discussing)

**Delta:** Cross-plan coordination note corrected (two bare warn sites in `viz/input_checks.py:168/:178`, `check_curves_exist`, not "a plain-UserWarning helper on the datashader route"); Holdouts enumerates both sites with likely category mappings; the deferred sweep's revival trigger is now mechanical (its first step re-runs Workstream B's grep gate over all of `src/hiveplotlib/`, any `viz/` hit automatically in-scope); Workstream B gains a merge-surface flag telling the implementer those sites are expected to be absent from the master-cut worktree (not a rule-9 mismatch). **Rationale:** a prose-only revival trigger re-arms nothing; the invariant would degrade with every gate green the day branch 53 merges. **Done-whens touched:** B (flag only); the migration of those sites stays inside the existing Deferred follow-up. **Tag:** In-scope tweak.

### Amendment 4 — Holdouts classified against the taxonomy before names ship (adversary worth-discussing)

**Delta:** classification table added to Holdouts mapping all 14 holdout messages (plus !51's unshipped warning) against the three shipped categories; verified against the branch-53 tree 2026-07-06. Result: 11 of 14 fit cleanly (8 + 1 → `MissingPlotElementWarning`, 2 → `EdgeKwargConflictWarning`), and the 3 misfits form one coherent build-time advisory family for a single anticipated category at sweep time. **Rationale:** public class names are forever; the cheap read confirms no 0.29 name needs stretching and no rename is indicated, so the taxonomy ships safely ahead of the deferred sweep. **Done-whens touched:** none (Holdouts only). **Tag:** In-scope tweak.

### Amendment 5 — Gate battery reconciled (adversary low-confidence)

**Delta:** `make docs` added to Workstream A's and B's gate bullets, matching the Execution context's per-commit battery. **Rationale:** implementers build to done-when; two disagreeing statements quietly shrink the recorded battery for two of three commits. **Done-whens touched:** A, B. **Tag:** In-scope tweak.

### Amendment 6 — Precedent citations corrected (api-critic low-confidence)

**Delta:** matplotlib's `MatplotlibDeprecationWarning(DeprecationWarning)` cited as the exact-match precedent, machine-verified in the project venv 2026-07-06 (MRO: `DeprecationWarning` → `Warning`); astropy re-filed in the Naming audit as the considered-and-rejected alternative design (umbrella subclasses `Warning`; deprecation sits under it via dual inheritance) so it cannot relitigate at the next deprecation; Workstream A's done-when adds that shipped docstrings name no third-party libraries (astropy is not installed locally, so its hierarchy stays plan-recorded from the critic's description, never docstring-cited). **Done-whens touched:** A. **Tag:** In-scope tweak.

### Amendment 7 — Message template canonicalized; helper deliberately not shipped (api-critic worth-discussing)

**Delta:** Workstream C's worked example now locks the Example-3 message template (deprecated name, replacement, removal version, in that order); Non-goals gains an explicit "No `warn_deprecated()` helper" bullet (zero call sites until the first real deprecation; shipping it now is the "machinery that rots" failure mode). **Rationale:** the reusable asset is the message shape, not a bare class; canonicalizing it in docs costs nothing and prevents re-improvisation. Classes-only scope confirmed. **Done-whens touched:** C. **Tag:** In-scope tweak.

### Amendment 8 — exceptions/ module docstring widened (api-critic low-confidence)

**Delta:** Workstream A's done-when widens `exceptions/__init__.py`'s module docstring to "Custom exceptions and warning categories for hiveplotlib." **Done-whens touched:** A. **Tag:** In-scope tweak.

### Amendment 9 — API-docs premise corrected: hand-maintained rST, not sphinx-apidoc (WS A code-engineer emergent finding)

**Delta:** the Naming audit's "generated by sphinx-apidoc... no manual rST entry" claim and Workstream C's "no manual rST needed" done-when were false. Verified 2026-07-06 in both the worktree and the main checkout: `docs/source/autodoc/` is git-tracked, hand-maintained rST (one `.. automodule::` file per module, e.g. `docs/source/autodoc/exceptions/hive_plot.rst`; Exceptions toctree at `docs/source/autodoc/index.rst:59-64`), and no apidoc invocation exists in `docs/source/conf.py` or either Makefile. Both statements corrected in place. Workstream C gains the manual page: new `docs/source/autodoc/exceptions/warning_categories.rst` matching the sibling shape, plus the toctree line (files list and done-when updated). Assigned to WS C rather than reopening WS A: WS C already owns the docs surface and has not run, and this keeps WS A's shipped one-commit boundary clean. llms.txt call recorded in WS C: the API page gets no llms.txt line of its own — llms.txt indexes the API reference once at `autodoc/index.html`, no per-module autodoc page carries an entry, and the user-facing entry point for warning filtering is the policy-page line already planned, which stands unchanged. **Rationale:** without the manual rST, the five filterable class names never appear in the API reference — the "machinery that rots" failure mode (undiscoverable classes) arriving through a docs-build premise nobody had checked. **Done-whens touched:** C. **Tag:** In-scope tweak.

### Amendment 10 — Umbrella quantifier corrected a second time: categories, not emissions (WS A post-impl api-critic must-fix; adversary concurs)

**Delta:** the shipped `HiveplotlibWarning` docstring (worktree `warning_categories.py:14`) reads "A single filter on this class covers every non-deprecation warning hiveplotlib **emits**." False in shipping v0.29: the 14 bare-`UserWarning` holdout sites (`hiveplot.py` x13, `p2cp.py` x1; Holdouts) are not subclasses of `HiveplotlibWarning`, so an `error`-baseline user setting `ignore::...HiveplotlibWarning` still errors on the common `connect_axes()` family (`hiveplot.py:1521-1679`, 8 sites). Amendment 1 closed the deprecation falsifier of this "every" quantifier; the Holdouts population is a second, independent falsifier. Fix (chosen wording, quantifies over the taxonomy the class defines rather than the wire-level emissions the holdouts still add to): **"A single filter on this class covers every named non-deprecation warning category hiveplotlib defines."** True both before and after the deferred holdout sweep. Cascade: (a) WS A done-when — the base docstring's coverage sentence uses the categories-not-emissions wording; (b) the shipped worktree file line 14 is fixed by re-invoking code-engineer before the WS A commit (see dispatch); (c) Example 2's comment reworded to the same categories-framing ("every named non-deprecation hiveplotlib warning category"); (d) WS C's two-line "silence everything hiveplotlib emits" recipe done-when carries the same honesty (the base covers every defined non-deprecation category, the second line covers the deprecation class, and the pre-sweep holdouts are named as a known gap so the recipe is not oversold). Test note: the WS A test at `exceptions_warning_categories_test.py:110-119` docstring already reads "every non-deprecation hiveplotlib category" and is honest as written (it warns only via the named categories, never a holdout), so no test-body change is owed by this item. **Rationale:** the false quantifier lands on exactly the warnings-as-errors user this plan serves; the categories-vs-emissions distinction is the load-bearing correction (the base filter is complete over the defined taxonomy, not over everything that reaches the warning stream while the holdouts remain bare). Design unchanged; only the claim about it changes. **Done-whens touched:** A, C. **Tag:** In-scope tweak.

### Amendment 11 — Filter-STRING resolution path gets a test; folded into WS A pre-commit (WS A post-impl, converged api-critic + adversary worth-discussing)

**Delta:** the documented user-facing filter form `ignore::hiveplotlib.exceptions.X` (module docstring line 5, CHANGELOG v0.29.0 Added entry) is exercised by no test; every filter test in `exceptions_warning_categories_test.py` (lines 69-198) passes category *objects*. The Naming audit calls the star-export path "the filter-string path" and it IS the load-bearing surface: a future `__all__` addition or dropped export would pass every current test while silently breaking every downstream `filterwarnings` ini line and `-W` flag. Fix: add one test decorated `@pytest.mark.filterwarnings("ignore::hiveplotlib.exceptions.HiveplotlibWarning")` that warns a viz category under an implicit error-strictness expectation — pytest's marker resolves the string through the same import-module + `getattr` path as stdlib `-W` / ini filters, so a misresolution (renamed or unexported class) fails loudly at test setup rather than passing green. **Disposition — folded into WS A, not deferred:** WS A's commit has not happened; the test file is already WS A's and unmodified since ship, so adding one function is cheap and keeps the string-resolution guard in the same commit as the classes it guards. Assigned to test-engineer, pre-commit. **Rationale:** the plan sells the export path as the filter-string path but never fires a string through it; the single most likely future break (an export drop) is exactly the case no object-based test can catch. **Done-whens touched:** A (test bullet gains the string-resolution test). **Tag:** In-scope tweak.

### Amendment 12 — CHANGELOG worked-example freshness declined for WS A; nudged into WS B (WS A post-impl api-critic low-confidence)

**Delta:** the WS A CHANGELOG v0.29.0 Added entry's only worked filter example is `HiveplotlibDeprecationWarning`, the one category with zero call sites in v0.29 (nothing is deprecated); the api-critic suggested using `MissingPlotElementWarning` (a live-call-site category the warnings-as-errors reader actually hits) instead of or beside it. **Declined for WS A** (already shipped; the deprecation-class example is not wrong, and reopening the WS A CHANGELOG to swap a worked example breaks its clean one-commit boundary for a low-confidence polish). Instead, WS B's CHANGELOG Changed entry done-when is nudged: it already announces the viz categories now carry named classes; it should name at least one live viz category as the copy-paste filter target (e.g. `ignore::hiveplotlib.exceptions.MissingPlotElementWarning`), so the freshest-customer example lands in the same release without touching WS A's commit. **Rationale:** the reader's need (a filter example for a category that actually fires today) is real but WS B is the honest home for it, since that is the workstream that gives those categories their call sites. **Done-whens touched:** B (CHANGELOG Changed bullet). **Tag:** In-scope tweak.

## Holdouts

Expected survivors of the bare-`warnings.warn` sweep — out of scope, do not migrate in this plan:

- `src/hiveplotlib/hiveplot.py:323, 1521, 1537, 1548, 1572, 1590, 1601, 1625, 1679, 3640, 3841, 4300, 4527` and `src/hiveplotlib/p2cp.py:621` (master line numbers): kept as bare `UserWarning` because both files are actively modified by branch 53 (scaling) and MR !51 (unify-axes). **Deferred follow-up, mechanical re-arm (Amendment 3):** after both merge, the follow-up sweep's FIRST step re-runs Workstream B's grep gate (`warnings.warn(` without `category=`) over all of `src/hiveplotlib/`, not just this holdout list, and any `viz/` hit is automatically in-scope — so bare sites landed by merges (next bullet) cannot silently survive with every gate green. The sweep also picks up !51's vmin/vmax warning. Classified "clearly decided against for now," not forgotten.
- Branch-53 sites that land inside `viz/` on merge, enumerated per Amendment 3 (branch-53 line numbers): `src/hiveplotlib/viz/input_checks.py:168` (edge subset backed by a lazy Dask frame, which never persists curves; points at the datashader backend or materializing first) and `:178` (stored edge ids but no constructed curves; points at `construct_curves()`), both `stacklevel=3` inside `check_curves_exist`. These re-break Workstream B's invariant the day branch 53 merges; the grep-gate re-arm above catches them. Likely mapping, decided at the sweep: both are missing-element-at-plot-time warnings (`MissingPlotElementWarning`), the lazy variant arguably `UnsupportedVizFeatureWarning`.
- **Holdout classification table (Amendment 4):** the 14 holdout messages mapped against the shipped taxonomy now, so 0.29's public names are chosen with the whole warning population in view (the migration itself stays deferred). Verified against the branch-53 tree 2026-07-06 (message text matches master; branch-53 line numbers differ):

  | Holdout (master lines) | Message gist | Fit against shipped taxonomy |
  | --- | --- | --- |
  | hiveplot.py:1521-1679 (8 sites) | "No edges exist between axes X -> Y ... run `connect_axes()`" | `MissingPlotElementWarning` — clean fit |
  | hiveplot.py:3640 | `update_edges()` no-op: no edges between the named partition groups | `MissingPlotElementWarning` — acceptable fit |
  | hiveplot.py:4300 | repeated kwargs preserved per `edge_kwarg_hierarchy` | `EdgeKwargConflictWarning` — clean fit |
  | p2cp.py:621 | `all_edge_kwargs` overridden by `indices_list_kwargs` | `EdgeKwargConflictWarning` — clean fit |
  | hiveplot.py:323 | parallel edges collapsed under `graph_multigraph=False` before metric computation | no fit — build-time data-semantics advisory |
  | hiveplot.py:3841 | `angle_between_repeat_axes` may cross axes / overlap edges | no fit — layout-quality advisory |
  | hiveplot.py:4527 | structure changed without `build_hive_plot()`; viz likely incorrect | no fit — stale-structure advisory |
  | MR !51 vmin/vmax displacement (unshipped) | requested vmin/vmax displaced | classify when !51 ships; likely the same advisory family |

  Conclusion recorded: 11 of the 14 existing sites fit the shipped names cleanly; the three misfits are all "advisory about a suspect state you created," a coherent family for the ONE anticipated build-time category at sweep time. No shipped 0.29 name needs to stretch to build-time semantics and no rename is indicated, so the taxonomy ships safely ahead of the sweep.
- `dataframe_to_node_list` / legacy `Node` (node.py:430-450): flagged twice by the scaling plan as deserving deprecation; **deferred** — it is the likely first customer of the new machinery, in its own plan. Actually deprecating anything is outside this plan's brief.

## Implementation log

Append-only. After each workstream completes, one line in the same turn:

- 2026-07-06: Workstream A complete (code side; tests are test-engineer's next). Added `exceptions/warning_categories.py` with the five classes (exclusion leads `HiveplotlibWarning`'s docstring; no third-party names), star-exported from `exceptions/__init__.py` with its docstring widened per Amendment 8, CHANGELOG v0.29.0 Added entry (3 lines); `make format` clean, import + hierarchy verified. Surfaced plan hole: `docs/source/autodoc/` is hand-maintained git-tracked rST (no sphinx-apidoc anywhere in conf.py/Makefiles), so the new module needs a manual `autodoc/exceptions/warning_categories.rst` + toctree line, contradicting the Naming audit's "no manual rST entry" and WS C's "no manual rST needed" done-when.
- 2026-07-06: Workstream A tests complete. Added `tests/exceptions_warning_categories_test.py` (12 test functions, 30 parametrized cases): subclass contract (viz categories under both `HiveplotlibWarning` and `UserWarning`; deprecation class under `DeprecationWarning`, outside the base hierarchy) plus filter behavior under a `simplefilter("error")` baseline (each category escapes as error; per-category ignore silences exactly that category; base ignore covers viz categories but not the deprecation class; plain `UserWarning` ignore and `pytest.warns(UserWarning)` both still catch viz categories; stock `ignore::DeprecationWarning` covers the deprecation class). `warning_categories.py` coverage 100% (5/5); 30 passed under `-n 7`; `make format` clean.
- 2026-07-06: Workstream B complete (source side; test tightening is test-engineer's next). Re-verified the master-cut worktree inventory: 26 bare `warnings.warn` sites in `src/hiveplotlib/viz/`, matching "Patterns this replaces" exactly (branch-53's `input_checks.py:168/:178` correctly absent per Amendment 3). Added `category=` to all 26, category-only: `base.py:144/152/164` -> `EdgeKwargConflictWarning`; `input_checks.py:52/58/64/73/79/85/95/102/113/119` -> `MissingPlotElementWarning`; `bokeh.py:215/404/595/873`, `holoviews.py:232/452/723/1009`, `plotly.py:332/439/587/1060`, `datashader.py:145` -> `UnsupportedVizFeatureWarning`. Message text and per-site stacklevel byte-identical (base/input_checks all `stacklevel=3`, all others `stacklevel=2`); the only non-`category=` diff line is `base.py`'s import widened to add `EdgeKwargConflictWarning`. Imported each class from `hiveplotlib.exceptions` per each module's convention. CHANGELOG v0.29.0 Changed entry added naming `ignore::hiveplotlib.exceptions.MissingPlotElementWarning` as the live copy-paste target (Amendment 12). Paren-aware grep gate green (26 warn calls, 0 without `category=`); `make format` clean; no CRLF flip.
- 2026-07-06: Workstream B tests complete (category tightening). Tightened bare `pytest.warns()` / `pytest.warns(UserWarning)` to the specific migrated category across the five viz test files (imports added to each from `hiveplotlib.exceptions`): `viz_matplotlib_test.py` (7 tests: 4 -> `MissingPlotElementWarning` [no-axes/missing-content/node-viz/no-axes-p2cp/edge-viz-p2cp], 3 -> `EdgeKwargConflictWarning` [repeat-edge-kwarg, repeat-edge-kwarg-p2cp, plot-linewidth-hierarchy]); `viz_bokeh_test.py` and `viz_plotly_test.py` (8 tests each: `MissingPlotElementWarning` x4, `EdgeKwargConflictWarning` x3, `UnsupportedVizFeatureWarning` x1 [hover-P2CP]); `viz_holoviews_test.py` (same 8, hover tightened only on the bokeh-backend branch); `datashader_test.py` (3 tightenings: no-tag -> `UnsupportedVizFeatureWarning` [the sole datashader.py:145 site], no-edges + no-axes-p2cp -> `MissingPlotElementWarning`). Each of the three categories now asserted where its sites fire; the three viz categories are proper siblings, so a wrong-category migration turns the matching test red (verified: `pytest.warns(EdgeKwargConflictWarning)` raises `DID NOT WARN` on a `MissingPlotElementWarning`). The four `test_plot_empty_hive_plot` (mpl/bokeh/holoviews/datashader) left as bare `pytest.warns()`: each raises a mix of the hiveplot.py "intermediate changes" holdout `UserWarning` alongside `MissingPlotElementWarning`s, and tightening the outer context manager re-emits the holdout as an error under the suite's `filterwarnings = error` baseline (holdout out of scope per Holdouts). No migrated viz/ site lacks a category-asserting test. viz coverage across the five files: base/bokeh/datashader/holoviews/plotly 100%; input_checks.py:122 (HivePlot "one axis has no nodes" branch) covered outside these five files, unchanged by this pass. 249 passed across the five files; `make format` clean; no CRLF flip (`git status`, per-file `\r` probe).
- 2026-07-06: Workstream C complete. New `docs/source/deprecation_policy.rst` (one-minor-version window; worked example canonicalizing the Example-3 template `X` is deprecated and will be removed in vN.N.N; use `Y` instead; CHANGELOG Deprecated-heading note; DeprecationWarning default-invisibility + `-W`/`filterwarnings` surfacing per Amendment 2; the two-line named-category recipe framed as categories-not-emissions with the `hiveplot.py`/`p2cp.py` bare-`UserWarning` holdouts named as a known gap per Amendments 1+10; pre-1.0 SemVer note). Added to `docs/source/index.rst` toctree under Release Notes beside the changelog. New `docs/source/autodoc/exceptions/warning_categories.rst` mirroring `hive_plot.rst` + toctree line after `exceptions/hive_plot` in `docs/source/autodoc/index.rst` (Amendment 9); built page renders all five class docstrings. CONTRIBUTING.md gained a "Deprecations" note in Contributing Code with the absolute stable link. `llms.txt` gained one `## Optional` line (judgment: kept — a real warnings-as-errors entry point, one line under a small "Warnings and deprecations" subsection so it doesn't masquerade as a gallery notebook; the new API page gets NO llms.txt line per Amendment 9). CHANGELOG v0.29.0 Documentation entry for the policy page (2 lines). All doc-internal cross-refs are `:doc:`/`:class:` roles (linkcheck-safe); `make docs` clean (zero new warnings, all cross-refs resolved to links), `make format` clean, no CRLF flip.
- 2026-07-06: Workstream A post-impl pre-commit touch-up (Amendments 10 + 11, combined pre-commit pass). Amendment 10 (docstring, dispatched to code-engineer in parallel): `HiveplotlibWarning`'s coverage sentence reworded to categories-not-emissions ("every named non-deprecation warning category hiveplotlib defines"), fixing the second umbrella-quantifier falsifier. Amendment 11 (test, this pass): added `test_base_ignore_filter_string_silences_viz_categories` (1 function, 3 parametrized viz cases) decorated `@pytest.mark.filterwarnings("ignore::hiveplotlib.exceptions.HiveplotlibWarning")`, exercising the user-facing filter-STRING resolution path no object-based test covered; the marker resolves the string through the same import + `getattr` export path as stdlib `-W` / ini filters, so a dropped or renamed export fails at filter application, and the deprecation category still escapes to pin the base string's exclusion. File now 13 functions / 33 parametrized cases; `warning_categories.py` coverage holds at 100% (5/5); 33 passed; `make format` clean, no CRLF flip (`git status`).
- 2026-07-06: CHANGELOG heading-level fix (docs-engineer, WS C follow-up). The v0.29.0 `Documentation` category heading was underlined `^^^^^^^^^^^^^` (a sub-topic nesting it under `Added`, per the v0.28.0 `NetworkX Compatibility` convention where `^` is a child of `Added`); swapped to `~~~~~~~~~~~~~` (13 tildes, matching the word length and the sibling `Added`/`Changed`/`Tooling Changes` category-underline style) so it now renders as its own category. Left in place between `Added` and `Changed` (no content reorder). `make docs` clean (no RST title/underline warning), `make format` clean, no CRLF flip (`git status`, `file`, `\r` probe all clean).
