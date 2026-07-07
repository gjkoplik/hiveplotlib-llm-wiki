# Plan: HivePlot.unify_axes — mirror the HivePlotMatrix affordance on single hive plots

<!--
Hiveplotlib and wiki-structure plans go to wiki/wiki/plans/<topic>.md (tracked
in the wiki submodule). Harness-self plans go to agent-harness/.claude/plans/
<topic>.md (gitignored). The plan is the canonical reference; the conversation
transcript is not.
-->

## Goal

Add a `unify_axes` affordance to `HivePlot` that mirrors the existing one on `HivePlotMatrix`. Today, a user who wants apples-to-apples axis ranges on a single hive plot has to compute global `min`/`max` over their sorting variable(s) by hand and thread those values through either the constructor's `axis_kwargs` or per-axis `update_axis(vmin=, vmax=)` calls. After this lands, `HivePlot(..., unify_axes=True)` and `hp.unify_axes()` do that work, with the same `True | dict` / no-arg / `vmin`-or-`vmax`-only call shapes the HPM API already establishes. Both calling conventions on HPM stay byte-for-byte unchanged; the underlying resolver becomes shared code.

Grill-me: knowingly skipped per maintainer directive (2026-07-06); autonomous end-to-end execution authorized the same day.

## Alignment (grill)

Status: knowingly skipped (2026-07-06, maintainer directive; autonomous end-to-end execution authorized). With no grill to fight it in, the adversary's planning challenge routed to orchestrator amend-plan for explicit disposition: all 8 items disposed in Amendment 3.

## Failure modes

None named: grill knowingly skipped (2026-07-06), so no maintainer-authored rubric exists. Post-impl adversary passes run without a grill rubric; the planning challenge and its Amendment 3 disposition serve as the reconcile baseline.

## Prior ADRs / design docs

Populated by research-liaison (pre-task pass, 2026-07-06). `wiki/wiki/adr/` holds two ADRs; neither is displaced by this plan.

ADRs:

- `wiki/wiki/adr/0001-networkx-integration.md` — binding symmetry precedent: `graph=` was chosen over classmethods partly to "keep the two classes symmetric" (HivePlot ↔ HivePlotMatrix). Mirroring `unify_axes` from HPM onto `HivePlot` extends the same principle. Also the durable record of construction-time graph-metric attachment, which Workstream A's resolver must run after.
- `wiki/wiki/adr/0002-performance-regression-harness.md` — not load-bearing (dev/CI perf-gating infrastructure; this plan makes no performance claims).

Concept / analysis pages:

- `wiki/wiki/concepts/fixed-layout-comparability.md` — the conceptual home for this affordance: its "pin the axis ranges" corollary (a nominally fixed frame lies if each panel infers its own value-range from its own data) is the use case `unify_axes` serves. Constraint it implies: auto-computed `unify_axes=True` unifies within one instance; comparability across *separate* `HivePlot` instances holding different data still needs the explicit `{"vmin": V, "vmax": W}` form (or pinned `axis_kwargs`). Workstream C/D prose must not imply per-plot `unify_axes=True` yields a shared frame across different datasets.
- `wiki/wiki/analyses/hive-plots-for-knowledge-graphs.md` — recorded demand for this plan, cited by name twice: ergonomic gap #7 pairs the `hiveplot-unify-axes` plan with fixed-layout small multiples, and the released-API portability lesson names landing this affordance as the fix. This work realizes the axis-range-unification slice of that gap.
- `wiki/wiki/analyses/soccer-passing-hive-plots.md` — real-usage lesson: with domain-meaningful fixed bounds, explicit pinning supersedes unification ("so `unify_axes` is dropped"). Corroborates Workstream D's "known reference scale: leave alone" class.
- `wiki/wiki/analyses/hiveplotlib-bioinformatics-examples.md` — the shared-metric + `unify_axes=True` identical-node-positions pattern (HPM two-panel comparison); the working demonstration of what the affordance buys.
- `wiki/wiki/concepts/hive-plot-matrix.md` — HPM background (v0.27 class; v0.28 `graph=` shipped "mirroring `HivePlot`"); the same mirroring direction this plan runs in reverse.

Named plan references (both resolved to `archived/`):

- `wiki/wiki/plans/archived/i-want-to-plan-optimized-hoare.md` — load-bearing on Workstream A's ordering: records why `_apply_graph_metrics` runs *before* `_resolve_unify_axes` ("so unify_axes can act on metric columns if requested as `sorting_variables`").
- `wiki/wiki/plans/archived/networkx-metric-expansion-and-backend-refactor.md` — its `unify_axes` mention is an incidental matrix-density observation in the implementation log; not load-bearing.

Cross-plan coordination (active plans touching this surface):

- `wiki/wiki/plans/edge-coverage-and-gotchas-docs.md` — cites this plan as the HPM-mirroring precedent ("same name, same shape"); it also carries the warning-conventions pointer (plain `UserWarning`, `stacklevel=3`, from the archived graph-metric-conflict-validation plan) that Workstream A's displacement warning already matches.
- `wiki/wiki/plans/graph-metrics-tutorial-series.md` — plans HPM `unify_axes=True` in a *new* notebook; no collision with Workstream D, which sweeps existing `examples/*.ipynb` and excludes HPM call sites.

## Patterns this replaces

- "If you were generating multiple hive plots to compare to each other, you might want to restrict the comparable axes to the same `[vmin, vmax]` range" prose in `examples/modifying_axes.ipynb` cells around lines 304-308 and 418-422. Once `unify_axes` exists on `HivePlot`, the docstring/notebook callout routes this use case by scope (Amendment 2): the quoted prose is cross-plot, so it points at passing the same full `unify_axes={"vmin": V, "vmax": W}` dict to each plot; per-plot `unify_axes=True` auto-computes from each instance's own data and only matches across plots holding the same node data and sorting variables. Raw `vmin`/`vmax` stays for the outlier-clipping / known-reference-range cases.
- Direct imports of `_resolve_unify_axes` from `hiveplotlib.hiveplot_matrix` (three call sites today, all in `src/hiveplotlib/hiveplot_matrix.py:1319`, `:1724`, `:2163`). Replace with the new shared location once the helper moves.
- `Dict[str, float]` annotation on `HivePlotMatrix.__init__`'s `unify_axes` parameter at `src/hiveplotlib/hiveplot_matrix.py:346` is looser than the `Dict[Literal["vmin", "vmax"], float]` used on the three `from_*` classmethods. Tighten to match while we're in the area; do not loosen the new `HivePlot.__init__` annotation to match the looser one.

## Default justifications

- `HivePlot.__init__(unify_axes=False)`: matches the HPM default. A single hive plot's axes are independent ranges by design; unifying changes that and should be opt-in. The user paying the conceptual cost of unification is the one who knows they want it.
- `HivePlot.unify_axes(vmin=None, vmax=None)`: mirrors `HivePlotMatrix.unify_axes` at `src/hiveplotlib/hiveplot_matrix.py:976`. `None` means auto-compute from the data the instance already holds. No new defaults beyond the HPM mirror.

## Naming audit

- New parameter: `unify_axes` on `HivePlot.__init__`. Vs. user vocab: ok. Same name as `HivePlotMatrix.__init__`'s parameter; users learning one transfer to the other without surprise. One deliberate behavior asymmetry rides on the shared name (Amendment 3, item 3): combining `unify_axes` with a per-axis `axis_kwargs` `vmin`/`vmax` warns on `HivePlot` and resolves silently on HPM, because only the `HivePlot` combo can break unification on a single axis.
- New method: `HivePlot.unify_axes()`. Vs. user vocab: ok. Same name and signature shape as `HivePlotMatrix.unify_axes()`.
- Internal helper relocation (e.g. `hiveplotlib._unify_axes._resolve_unify_axes` or a sibling under `hiveplotlib/`): out of scope per "Internal module/package names are out of scope" in the template. Code engineer picks a private location that breaks no circular imports.

## API usage examples

Required when this work adds or modifies user-facing API.

### Proposed (from planner / Orchestrator)

Rewritten by Amendment 4 (api-critic planning pass): data construction uses the `graph=` constructor path throughout (`node_graph_metrics` lives on `HivePlot.__init__`, `hiveplot.py:2658`; `networkx_to_nodes_edges` takes no metrics parameter, `converters.py:20-24`). Runnability witness for the construction: `tests/hiveplot_test.py:6026-6052` builds exactly this form and asserts axes `{"Mr. Hi", "Officer"}` plus a `degree` column. The `unify_axes` lines are the surface Workstream A ships, pinned by its done-whens; the `axis_kwargs` shape in Example 5 matches today's `Dict[Hashable, Dict]` (`hiveplot.py:2649`).

```python
# Example 1: opt into unified ranges at construction time, fully auto-computed
import networkx as nx

from hiveplotlib import HivePlot

hp = HivePlot(
    graph=nx.karate_club_graph(),
    partition_variable="club",
    sorting_variables="degree",
    node_graph_metrics="degree",
    unify_axes=True,
)
```

```python
# Example 2: pin one side of the unified range, let the other auto-compute
import networkx as nx

from hiveplotlib import HivePlot

hp = HivePlot(
    graph=nx.karate_club_graph(),
    partition_variable="club",
    sorting_variables="degree",
    node_graph_metrics="degree",
    unify_axes={"vmin": 0},
)
```

```python
# Example 3: build first, unify post-hoc with no arguments
import networkx as nx

from hiveplotlib import HivePlot

hp = HivePlot(
    graph=nx.karate_club_graph(),
    partition_variable="club",
    sorting_variables="degree",
    node_graph_metrics="degree",
)
hp.unify_axes()
```

```python
# Example 4: post-hoc unify with an explicit vmin floor
import networkx as nx

from hiveplotlib import HivePlot

hp = HivePlot(
    graph=nx.karate_club_graph(),
    partition_variable="club",
    sorting_variables="degree",
    node_graph_metrics="degree",
)
hp.unify_axes(vmin=0)
```

```python
# Example 5: unify globally, override one axis explicitly via axis_kwargs (precedence demo)
import networkx as nx

from hiveplotlib import HivePlot

# `axis_kwargs` wins for "Mr. Hi"; "Officer" follows the unified range.
# Emits a warning that the per-axis vmin displaces what unify_axes would have set.
hp = HivePlot(
    graph=nx.karate_club_graph(),
    partition_variable="club",
    sorting_variables="degree",
    node_graph_metrics="degree",
    unify_axes=True,
    axis_kwargs={"Mr. Hi": {"vmin": 0}},
)
```

```python
# Example 6: cross-plot comparability across instances holding different data.
# Compute the shared range once; pass the same full dict to each plot.
import networkx as nx

from hiveplotlib import HivePlot

g_full = nx.karate_club_graph()
g_thinned = g_full.copy()
g_thinned.remove_edges_from(list(g_full.edges)[::2])

# shared bounds across both graphs' degree values; per-plot
# `unify_axes=True` would auto-compute a different range per instance
degrees = [deg for g in (g_full, g_thinned) for _, deg in g.degree()]
shared_range = {"vmin": min(degrees), "vmax": max(degrees)}

hive_plots = [
    HivePlot(
        graph=g,
        partition_variable="club",
        sorting_variables="degree",
        node_graph_metrics="degree",
        unify_axes=shared_range,
    )
    for g in (g_full, g_thinned)
]
```

### API Critic's take (planning mode)

Status: propose (one must-fix on the example snippets; the API surface itself is agreed).

Surface verdict: agreed on the constructor parameter shape (`True | Dict[Literal["vmin", "vmax"], float]`, default `False`), the post-hoc `unify_axes(vmin=None, vmax=None)` mirror, the layer-2 precedence chain, the keep-and-document warning asymmetry (Amendment 3 item 3), and the post-hoc clobber semantics (Amendment 3 item 4). Independently re-verified this pass: the resolver called with `axis_kwargs=None` returns exactly the flat resolved bounds (`hiveplot_matrix.py:159-183`; merge line `{**unified, **explicit, **existing}` at `:182` is the documented precedence), the HPM post-hoc method consumes current axis ranges then clobbers all axes (`:1000-1020`), and `update_axis` carries the `build_hive_plot` kwarg the single-rebuild loop needs (`hiveplot.py:4169-4181`).

Findings:

- [must-fix] Every example's data construction fails as written: `networkx_to_nodes_edges` has no `node_graph_metrics` parameter (its full signature is `(graph, unique_id_name="unique_id", check_uniqueness=True)`, `src/hiveplotlib/converters.py:20-24`), so all five snippets raise `TypeError` at the second line. Graph metrics attach on `HivePlot.__init__` itself (`node_graph_metrics`, `hiveplot.py:2658`), and the `graph=` keyword path makes construction one line. Suggested change: rewrite the data construction in all five examples to the form below (verified runnable against `tests/hiveplot_test.py:6026-6052`, which builds exactly this and gets axes `{"Mr. Hi", "Officer"}` plus a `degree` column); each example's `unify_axes` / `axis_kwargs` / post-hoc lines are correct and stay as proposed. Side benefit: this construction demonstrates the ordering fact Workstream A leans on (metrics attach before the resolver reads `sorting_variables`).

  ```python
  # Example 1, corrected: opt into unified ranges at construction time, fully auto-computed
  import networkx as nx

  from hiveplotlib import HivePlot

  hp = HivePlot(
      graph=nx.karate_club_graph(),
      partition_variable="club",
      sorting_variables="degree",
      node_graph_metrics="degree",
      unify_axes=True,
  )
  ```

- Owed decision (Amendment 2, cross-plot sixth example): **add it.** Two reasons: (a) the full-dict form `{"vmin": V, "vmax": W}` is the exact target the plan routes cross-plot users to in three places (Patterns-this-replaces bullet 1, Workstream C's docstring and notebook touches, Workstream D's different-data migration class), yet no example shows it; Example 2 pins only the partial dict. (b) This section is the shape-pinning contract for downstream workstreams: without an agreed snippet, Workstream C's docstring version and Workstream D's notebook version of "compute the range once, pass the same dict to each plot" get invented independently. The teaching prose still lives in Workstream C; the example only pins the idiom. Proposed snippet (same node set, different edge sets: `club` partition stays valid on both while degrees genuinely diverge, so per-plot `True` would demonstrably not match):

  ```python
  # Example 6: cross-plot comparability across instances holding different data.
  # Compute the shared range once; pass the same full dict to each plot.
  import networkx as nx

  from hiveplotlib import HivePlot

  g_full = nx.karate_club_graph()
  g_thinned = g_full.copy()
  g_thinned.remove_edges_from(list(g_full.edges)[::2])

  # shared bounds across both graphs' degree values; per-plot
  # `unify_axes=True` would auto-compute a different range per instance
  degrees = [deg for g in (g_full, g_thinned) for _, deg in g.degree()]
  shared_range = {"vmin": min(degrees), "vmax": max(degrees)}

  hive_plots = [
      HivePlot(
          graph=g,
          partition_variable="club",
          sorting_variables="degree",
          node_graph_metrics="degree",
          unify_axes=shared_range,
      )
      for g in (g_full, g_thinned)
  ]
  ```

- [worth-discussing] Method-shadowing trap for Workstream A: the constructor parameter and the new instance method share the name `unify_axes`, so storing the parameter via HivePlot's routine attribute pattern (`self.unify_axes = unify_axes`, cf. `self.num_steps_per_edge` at `hiveplot.py:2706`) silently shadows the bound method and turns `hp.unify_axes()` into `TypeError: 'bool' object is not callable`. HPM never stores it (its `__init__` consumes the param by calling the method, `hiveplot_matrix.py:427-429`). Suggested change: one sentence in Workstream A's `hiveplot.py` files bullet: do not store the parameter as an instance attribute. Workstream B's method tests would catch the mistake, so this is a save-a-cycle note, not a ship risk.

- [low-confidence] Example 1's `node_graph_metrics=["degree", "betweenness_centrality"]` computes a metric nothing downstream uses (sorting is degree-only); in a headline example every value reads as load-bearing to a first-time user. Trim to `"degree"` (the corrected snippet above already does).

- [low-confidence] The displacement warning has no opt-out, while the same constructor's kwarg-conflict warnings do (`warn_on_overlapping_kwargs`, `hiveplot.py:2655`); a user running the deliberate combo (Example 5 is that recipe) gets an unavoidable warning, which becomes an error under a downstream consumer's `filterwarnings = error`. Not pushing for a new parameter (surface growth for a rare case); suggested change: Workstream C's warning sentence also names standard suppression (`warnings.catch_warnings` / `filterwarnings`) for the deliberate-combo user.

Recurring pattern: both snippet findings share a root, examples authored from memory of the API rather than against shipped signatures. The `graph=` constructor path (shipped v0.28) supersedes converter-based construction for networkx-sourced example data; future plans should default to it and cite a shipped test or notebook per snippet as the runnability witness.

### API Critic — post-implementation review

Status: propose (no must-fix)
API surface reviewed: `HivePlot.__init__(unify_axes=...)`, `HivePlot.unify_axes(vmin, vmax)`, `HivePlot._merge_unified_axis_bounds` (internal), `_unify_axes._resolve_unify_axes` (shared internal), `HivePlotMatrix.__init__` annotation tighten.

Surface verdict: the shipped shape matches the plan and my planning-mode take exactly. Signature placement is correct (`unify_axes` sits immediately before `axis_kwargs`, `hiveplot.py:2176-2177`). The `Union[bool, Dict[Literal["vmin", "vmax"], float]]` annotation is right and now consistent with the tightened HPM `__init__` (`hiveplot_matrix.py`). The layer-2 precedence in `_merge_unified_axis_bounds` implements the documented chain (per-axis `axis_kwargs` > explicit dict > auto-computed): explicit-vs-auto is settled inside `_resolve_unify_axes`, axis_kwargs-vs-unified is settled in the merge, and `unified_bounds` always carries both keys when the merge runs, so no `KeyError` path. The post-hoc `HivePlot.unify_axes` is a faithful mirror of `HivePlotMatrix.unify_axes` (`hiveplot_matrix.py:934-978`) with the single-plot adaptation (drops `iter_populated_cells`), same clobber semantics, same one-rebuild-on-last-axis loop, and a correctly-adapted `.. note::` (points at the constructor param rather than HPM's convenience methods). Displacement-warning text is actionable: it names the axis, the displacing value, the displaced unified value, and the consequence. Docstring facts are all correct against the implementation (the "serviceable-not-final" thinness Workstream C owns is not flagged). Nothing shipped contradicts the planning take; the planning must-fix was against the plan's example snippets and was disposed in Amendment 4, not in shipped code.

Concerns:

- [worth-discussing] A typoed dict key is silently swallowed — at `_unify_axes.py:34-54` (resolver) and `hiveplot.py:2546-2557` (merge). `unify_axes={"vmim": 0}` (typo for `vmin`): the resolver's `need_vmin`/`need_vmax` both stay True, so both sides auto-compute; the stray `"vmim"` rides into `unified_bounds` but `_merge_unified_axis_bounds` only ever reads the literal `"vmin"`/`"vmax"` keys, so the typo vanishes with no error and the user gets a fully auto-computed range while believing they pinned a floor. The `Literal["vmin", "vmax"]` annotation advertises a closed key set that nothing enforces at runtime; the failure is silent-wrong, not silent-noop. Same swallow applies to a non-dict non-bool value (`unify_axes=5` behaves as `True`; `unify_axes=0` too, since the guard is `is False`). Weighed against the plan: the resolver moved byte-for-byte from HPM and the plan's explicit thesis is mirror fidelity, so this ships identically on HPM today and is not a Workstream-A defect. Not must-fix for that reason. Suggested change: route to orchestrator `amend-plan` for a small validation add on the shared resolver (raise on non-`vmin`/`vmax` dict keys and on non-bool/non-dict values) so both classes gain the guard together; it is a deliberate divergence from "import source only," so it needs plan authority, not an inline fix.

- [low-confidence] Repeat-axis override breaks unification with no warning — at `hiveplot.py:2542-2544`. The merge loop skips `ax in repeat_axis_names`, so `unify_axes=True` combined with a user-set repeat-axis `vmin`/`vmax` via `axis_kwargs` (`{"Mr. Hi_repeat": {"vmin": 5}}`) leaves that repeat axis off the unified frame silently, while the identical override on a non-repeat axis warns. This is asymmetric with the non-repeat displacement warning. Shipped per the plan's explicit letter (no displacement warning on a user-set repeat-axis override), and repeat-axis vmin overrides are a rare deliberate action, so within-spec and low-impact. Suggested change: none for this workstream; if Workstream C's docstring treatment enumerates the warning triggers, note there that the warning is non-repeat-axis-only.

Test-method-coverage audit: n/a this pass — Workstream B (tests) has not run; nothing to sample. The plan's WS-B roster does name `test_<method>_*` tests that call the method in the body; verify at WS-B post-impl.

## Adversary review

Cold-context dissent against the plan and the artifact it ships. Both subsections are mandatory on every plan. See `agent-harness/.claude/agents/adversary.md`.

### Adversary's challenge (planning mode)

Status: challenge (8 items)
Plan reviewed: wiki/wiki/plans/hiveplot-unify-axes.md (cold)

Angles worked. Premise: holds, no existential objection. The within-plot problem is real today (each axis infers its range from its own nodes only, per the `update_axis` docstring and `modifying_axes.ipynb`, so radial positions are not comparable across axes without manual `axis_kwargs` threading), demand is recorded (KG analysis gap #7), and ADR 0001's symmetry precedent covers the mirror direction. Approach: the mirror is the right shape, but the plan's central reuse mechanism is misspecified against the verified code (item 1) and two behavioral asymmetries ride along unacknowledged (items 3, 4). Size-and-maintenance: surface is bounded and mirrors shipped HPM surface; the one shrink candidate is the new displacement warning (item 3); the Workstream D sweep needs a fidelity gate to stay a refactor rather than a silent figure change (item 5).

Items:

- [must-fix] Workstream A's mechanism cannot work as written: `_resolve_unify_axes` merges flat `{"vmin": ..., "vmax": ...}` dicts (HPM's uniform `axis_kwargs` shape), but `HivePlot.axis_kwargs` is nested per-axis (`Dict[Hashable, Dict]`, `hiveplot.py:2649`) and the consumption loop raises `InvalidAxisNameError` on non-axis keys (`hiveplot.py:2793-2797`), so "call the shared resolver ... the resolver feeds into `axis_kwargs`" injects `"vmin"`/`"vmax"` as axis names and every `unify_axes=True` construction fails — at Workstream A (files bullet, repeat-axis done-when, and the Holdouts phrase "resolver-populated axis_kwargs")
  Rubric: no rubric — grill knowingly skipped
  Push: amend Workstream A before dispatch to a two-layer design: reuse the resolver unchanged for range resolution only (calling it with `axis_kwargs=None` returns exactly the resolved `{"vmin": V, "vmax": W}` bounds, preserving "HPM changes only in import source"), then a HivePlot-side per-axis merge that writes those bounds into each non-repeat axis's kwargs unless that axis carries its own explicit `vmin`/`vmax`, owns the displacement warning, and leaves repeat-axis inheritance to the existing branch (~`hiveplot.py:2822-2836`). Otherwise the autonomous run halts mid-workstream under rule 9, or the code engineer invents precedence and warning semantics with no plan authority.
- [worth-discussing] Every pre-Amendment-2 line anchor is stale (anchored against an older tree): HPM `unify_axes` method is at `hiveplot_matrix.py:976` not `:832`; the three resolver call sites are `:1319`/`:1724`/`:2163` not `:1127`/`:1480`/`:1872`; the loose `Dict[str, float]` annotation is at `:346` not `:301`; the precedence comment at `:215` not `:212`; the `hiveplot.py` `axis_kwargs` consumption loop at `:2793` not `:2026`; repeat-axis vmin/vmax inheritance at ~`:2822-2836` not `:2050-2069`; the HPM unify test block starts at `tests/hiveplot_matrix_test.py:1563`. "Verify by reading" instructions aimed at wrong ranges verify the wrong code silently — at Patterns this replaces / Default justifications / Workstream A / Workstream B / Holdouts
  Rubric: no rubric — grill knowingly skipped
  Push: refresh all anchors in the same amendment as item 1 (Amendment 2's `:143-183` is current; the rest predate recent churn on these files).
- [worth-discussing] The displacement warning is a deliberate mirror break the plan never acknowledges: HPM resolves the identical combo silently (documented priority at `hiveplot_matrix.py:215`, no warning in the resolver), while the naming audit sells "users learning one transfer to the other without surprise"; same inputs would warn on `HivePlot` and stay silent on `HivePlotMatrix` — at Workstream A done-when (warning bullet) / Naming audit
  Rubric: no rubric — grill knowingly skipped
  Push: dispose explicitly: (a) drop the warning (exact mirror, less surface), (b) keep it and state the asymmetry in both classes' docstrings, or (c) propagate it to HPM (a released-behavior change: a new warning can break `filterwarnings=error` consumers and needs a CHANGELOG entry). If kept, note for Workstream B that every test combining `axis_kwargs` vmin/vmax with `unify_axes` must expect the warning under `filterwarnings = error` (`test_init_axis_kwargs_vmin_overrides_unify` and `test_init_unify_axes_dict_vs_axis_kwargs_precedence`, not just the dedicated displacement test).
- [worth-discussing] Post-hoc precedence reverses the constructor chain and the plan documents only the constructor half: `unify_axes()` (HPM mirror, `hiveplot_matrix.py:1000-1020`) consumes current per-axis vmin/vmax, user overrides included, as autocompute inputs and then clobbers every axis; the Workstream A parenthetical "since per-axis ranges may already encode user overrides" reads as if overrides are preserved, when they are consumed and then flattened — at Workstream A done-when (post-hoc bullets) / Workstream C precedence-chain bullet
  Rubric: no rubric — grill knowingly skipped
  Push: Workstream C's docstring requirement should state both directions (constructor: per-axis `axis_kwargs` wins over unify; post-hoc: `unify_axes()` overrides all current per-axis ranges and derives auto values from them); Workstream B should pin the clobber with a test (`update_axis(vmin=<explicit>)` then `unify_axes()`, explicit value gone).
- [worth-discussing] Workstream D has no output-preservation gate: migrating manual global min/max threading to `unify_axes=True` is a faithful refactor only when the notebook's manual computation equals whole-collection `nanmin`/`nanmax`; the done-whens check `make test-nb` and prose only, and viz-critic reviews quality, not sameness, so a silently shifted figure ships with every stated gate green — at Workstream D done-when
  Rubric: no rubric — grill knowingly skipped
  Push: add a done-when: each migrated within-instance cell yields the same `(vmin, vmax)` as the code it replaced (assert in-notebook or confirm from rendered ranges); where the manual computation differs from whole-collection semantics, classify the cell left-alone instead of migrating.
- [worth-discussing] The load-bearing api-critic planning pass is armed only by prose: Amendment 2 defers the cross-plot dict-form example decision to the "pending planning pass", and this run is authorized autonomous with the grill skipped, so nothing mechanical blocks Workstream A dispatch while that section is still pending — at API usage examples / Amendment 2
  Rubric: no rubric — grill knowingly skipped
  Push: add an explicit gate to Workstream A ("Depends on: API Critic's take (planning mode) filled") so the autonomous dispatcher cannot start code engineering past a pending section.
- [low-confidence] Mixed-scale unification is a knowingly inherited footgun: dict `sorting_variables` concatenates all referenced columns (`hiveplot_matrix.py:170-179`), so unifying an axis sorted by degree with one sorted by a [0, 1] metric crushes the latter to the bottom of its axis; consistent with HPM, but single-plot users will hit it more directly — at Workstream C
  Rubric: no rubric — grill knowingly skipped
  Push: one docstring sentence in Workstream C warning that unification across different-scale sorting variables rarely reads well.
- [low-confidence] `test_resolve_unify_axes_import_stability` as named would violate the test-name = test-body contract if its body only imports: the trip-wire requires the named entry point be called in the body — at Workstream B done-when
  Rubric: no rubric — grill knowingly skipped
  Push: if the re-export branch is taken, either call the resolver in that test's body or name the test for what it actually verifies (an import-location test not named after the method).

### Adversary post-impl

Status: propose
Artifact reviewed: Workstream A (commit d9fd522) — shared resolver move + `HivePlot` `unify_axes` constructor param + `HivePlot.unify_axes()` method
Dispositions held: yes. The planning must-fix (item 1: flat resolver dict vs nested per-axis `axis_kwargs`) is closed in the artifact, not just on paper. `_resolve_unify_axes` moved byte-identical to `_unify_axes.py`; the constructor calls it with `axis_kwargs=None` (`hiveplot.py:2320`) to get flat bounds, then `_merge_unified_axis_bounds` (`hiveplot.py:2522-2562`) injects them nested per non-repeat axis, and repeat axes inherit through the untouched branch (`2359-2376`) because `update_axis` sets `inferred_vmin/vmax=False` on the injected floats (`3838`, `3843`) — the exact two-layer shape Amendment 3 item 1 adopted. Warning is constructor-only and names axis / override / unified value (item 3, held). Post-hoc clobber consumes-then-flattens, mirroring HPM (item 4, held). HPM resolver body unchanged, all three call sites + `__init__` annotation retargeted with no stranded imports (verified). No scope balloon; no holdout touched.

Concerns:

- [worth-discussing] Constructor and post-hoc `unify_axes` compute the auto range from different sources: the constructor resolver reads raw `nodes.data[sorting_variables]` (`_unify_axes.py:57-66`), while `hp.unify_axes()` reads current `axis.vmin`/`axis.vmax` (`hiveplot.py:3899-3908`). Same-named entry points can yield two different "unified" ranges on one object once any per-axis bound deviates from the raw extent. This is faithful to Amendment 3 item 4 (post-hoc semantics = consume current per-axis ranges) and mirrors HPM exactly, so it is dispositioned, not new — flagging only that the two arithmetics under one name is a discoverability trap the WS-C docstring should make explicit. Bears on Workstream C (docs) only; no bearing on B or D — batch to WS-C, do not halt.
  Rubric: no entry (grill skipped)
- [low-confidence] `_merge_unified_axis_bounds` unconditionally indexes `unified_bounds["vmin"]` and `["vmax"]` (`hiveplot.py:2550, 2556`); this is safe only because layer 1 is always called with `axis_kwargs=None`, forcing the resolver to return both keys. Correct as shipped, brittle if a future caller ever routes a partial `unified_bounds` (would `KeyError`). Recorded only; no amendment.
  Rubric: no entry (grill skipped)
- [low-confidence] The resolver takes `np.nanmin`/`nanmax` on `nodes.data[sorting_variables].to_numpy()` with no guard for an all-NaN or non-numeric sorting column (`_unify_axes.py:63-66`); an all-NaN column emits an `All-NaN slice` RuntimeWarning that `filterwarnings=error` would turn into a failure. Inherited byte-identical from the pre-move HPM resolver, so not introduced here and adjacent to the item-7 mixed-scale footgun already dispositioned to a WS-C docstring sentence. Recorded only; no amendment.
  Rubric: no entry (grill skipped)

---

Status: propose
Artifact reviewed: Workstream B (uncommitted worktree) — `TestHivePlotUnifyAxes` in `tests/hiveplot_test.py` (the `unify_axes` constructor-param + instance-method suite)
Dispositions held: yes. My WS-A planning must-fix (two-layer resolver) is not a test-side concern; the WS-A post-impl dispositions stand. No scope balloon in the test file: the one un-rostered test is legitimate 100%-coverage work for a branch WS-A introduced, not smuggled scope (finding 2 below). No holdout touched.

Reconcile note (blind attack ran first, plan read second): the blind pass judged the diff against the WS-B roster only. Reading WS-A done-when `hiveplot.py:332` ("the post-hoc method stays silent, like HPM's") and Amendment 5 confirmed the two findings that survive; both regrade *down* against plan context, neither reflexively.

Concerns:

- [worth-discussing] The post-hoc-path silence contract (WS-A done-when, `hiveplot.py:332`: post-hoc `unify_axes()` stays silent, unlike the warning constructor path) has no *named* assertion. Regraded from a blind must-fix: the silence is in fact protected transitively. All five `test_unify_axes_method_*` tests call `hp.unify_axes()` bare under `filterwarnings = error`, so any `UserWarning` the post-hoc path emitted would already fail them, and `test_unify_axes_method_overrides_prior_update_axis` (diff:121-136) runs the *displacing* scenario post-hoc (an override below data min, the exact case that warns on the constructor path) yet asserts only values, not silence. So the highest-risk silence case is covered by side effect; what is missing is a legible regression guard (a future maintainer who adds a post-hoc warning gets a confusing value-test failure rather than a named "post-hoc must stay silent" one). Fix is a one-line `warnings.catch_warnings`/record-and-assert-empty in that existing test. **Bearing: reopens WS-B (edits WS-B's own just-shipped test file); no bearing on C or D.** at `tests/hiveplot_test.py:121-136`
  Rubric: no entry (grill skipped)
- [low-confidence] `test_init_unify_axes_repeat_axis_user_override_survives` (diff:240-255) is not on the WS-B roster (plan lines 346-360). Confirmed legitimate, not smuggled: it covers the `_merge_unified_axis_bounds` trailing branch (`hiveplot.py:2559-2561`) that keeps a user's repeat-axis entry, a branch reachable ONLY via a user-set repeat-axis key and therefore required for the 100% gate; and it pins the repeat-override-survives-silently behavior that both api-critic and I flagged at low-confidence in WS-A post-impl (Amendment 5 context). Resolved: legitimate coverage. One residual nit inherited from finding 1: line 248-249 asserts the surviving value but not the *silence* of that survival (the "no warning on a repeat-axis override" half of the contract); if the finding-1 guard is added, extend it here. No amendment on its own. **Bearing: reopens WS-B only if bundled with the finding-1 fix; no bearing on C or D.** at `tests/hiveplot_test.py:240-255`
  Rubric: no entry (grill skipped)
- [low-confidence] `test_resolve_unify_axes_importable_from_both_homes` (diff:267-275) satisfies the test-name=test-body contract (imports from both homes, asserts identity, calls the resolver) and its `{"vmin": 0.0, "vmax": 10.0}` assertion is falsifiable. Noting only that it exercises the resolver's `axis_kwargs=None` path exclusively (the sole path the constructor uses, `hiveplot.py:2320`); the resolver's `existing`-dict precedence branch (`_unify_axes.py:37-38, 53`) is dead on every in-tree caller. That dead-branch question is the WS-A code-side concern already deferred in Amendment 5 item 1 (mirror-faithful input tolerance), not a WS-B test gap. Recorded only; no amendment. **No bearing on B, C, or D.** at `tests/hiveplot_test.py:267-275`
  Rubric: no entry (grill skipped)

Clean checks (worked, no finding): no-op trap disarmed (`test_unify_axes_method_no_args_auto_computes` asserts the disjoint pre-state before flattening, diff:79-90); precedence tests assert BOTH the overridden axis keeping the user value AND siblings carrying the unified value (`test_init_axis_kwargs_vmin_overrides_unify` diff:182-198, `test_init_unify_axes_dict_vs_axis_kwargs_precedence` diff:207-221); dict-`sorting_variables` test proves concatenation not per-axis column (0.0 from `value`, 105.0 from `other`, diff:170-180); displacement-warning `match` pins axis name + unified value (diff:203); no optional-backend import in the class, so no missing `@pytest.mark.*`; no process references in any test name or docstring (rule 15 clean). Every rostered done-when has a correctly-named present test; the re-export test is present and the resolver is genuinely re-exported.

---

Status: propose
Artifact reviewed: Workstream C (uncommitted worktree) — `unify_axes` docstrings on the `HivePlot` constructor param + `HivePlot.unify_axes` method; `CHANGELOG.rst` entry; `modifying_axes.ipynb` two-sentence touch; `hive_plot_matrix.rst` `.. automethod::` add; `llms.txt` correction
Dispositions held: yes. Every WS-C docstring mandate from my planning challenge shipped, verified against the code the docstrings describe. Item 3 (displacement warning kept on `HivePlot`, HPM silent) → constructor docstring `hiveplot.py:2074-2077` states both halves ("emits a ``UserWarning``"; "``HivePlotMatrix`` resolves the same combination silently: its ``axis_kwargs`` applies one value across the whole frame"), plus the Amendment 4 item-4b suppression note (`catch_warnings`/`filterwarnings`) at `:2077-2080`; matches `_merge_unified_axis_bounds` (`hiveplot.py:2571-2578`, warns only on a per-axis-set bound). Item 4 (post-hoc consume-then-flatten) → method docstring `:3913-3917` ("consumed as an input to the auto-compute and then flattened, not preserved... reverse of the constructor precedence"); matches the method body reading `axis.vmin`/`axis.vmax` then unconditionally `update_axis`-ing every axis (`:3936-3950`). Item 7 (mixed-scale caveat) → constructor docstring `:2087-2088` ("Unifying across *different-scale* sorting variables rarely reads well"). No scope balloon; no holdout touched. No shipped prose (docstring, notebook sentence, or `llms.txt`) implies per-plot `unify_axes=True` yields a shared frame across separate different-data instances — the WS-C done-when the whole scope-correctness thread exists to hold is met.

Reconcile note (blind attack ran first, plan read second): the blind pass judged the diff against the WS-C done-whens only. Reading Amendment 3 item 3 (HPM warns nowhere by design; its flat `axis_kwargs` keeps the frame uniform) and my own WS-A post-impl entry (the resolver moved byte-identical to `_unify_axes.py`, carries no warning) resolves my blind low-confidence finding on the HPM-silent claim: it is accurate, no residual doubt, no extra `HivePlotMatrix` read needed. **No must-fix emerged after reconcile; the blind pass's no-must-fix verdict holds.** The docstrings are truthful against the shipped code.

Concerns (both regrade from blind, neither reflexively):

- [worth-discussing] The constructor `unify_axes` param docstring is long (~44 lines: lead + complements-paragraph + `**Priority**` block + `**Scope**` block + different-arithmetic note). The simple "make my axes share a range" answer IS reachable (the lead sentence and the `True` clause front it, so rule 8 is met at the top), but a reader scanning the signature help wades through a complements paragraph before the one-line use closes. All of it is accurate and mandated (Amendments 2, 3, 4 each added a required sentence here), so this is a self-polish taste call, not a defect, and I would not cut behavioral content to shorten it. **Bearing: WS-C self-polish only; no bearing on WS-D** (the notebook sweep migrates call sites and touches prose in `examples/*.ipynb`, not this docstring). Batch to plan-end. at `src/hiveplotlib/hiveplot.py:2050-2093`
  Rubric: no entry (grill skipped)
- [low-confidence] The corrected `llms.txt` line reads "it shares one range within an instance, and lines separate instances up only when they hold the same node data and sorting variables". "lines separate instances up" is an awkward word-order slip for "lines up separate instances"; the clause parses clumsily for a human or an LLM consuming the index. The *substance* is correct and does not overclaim (it drops the false "HivePlotMatrix-only feature", scopes `True` to same-data instances, and routes different-data users to the explicit dict), so this is a prose-quality nit, not an accuracy miss. **Bearing: WS-C self-polish only; no bearing on WS-D.** Batch to plan-end. at `docs/source/_llms/llms.txt:26`
  Rubric: no entry (grill skipped)

Clean checks (worked, no finding): constructor precedence claim (`hiveplot.py:2048-2053`) TRUE against the resolver's `{**unified, **explicit, **existing}` merge (`_unify_axes.py:53`) plus the layer-2 per-axis win (`hiveplot.py:2571`); scope nuance (`:2081-2088`) TRUE for all four branches (`True` per-instance auto-compute, same-data identical ranges, full-dict verbatim, partial-dict per-instance auto-compute of the missing side) against `_resolve_unify_axes`'s `need_vmin`/`need_vmax` logic; different-arithmetic trap (`:2090-2093` and method `:3919-3921`) TRUE (constructor reads `nodes.data`, post-hoc reads `axis.vmin`); notebook sentence (identical under `### Vmin`/`### Vmax`) accurate and correctly routes the cross-plot compare-different-data case to the full dict; CHANGELOG entry exactly 4 wrapped lines, behavior-level, under 120 chars, no sub-bullets (rule 13 clean); `.. automethod:: hiveplotlib.HivePlotMatrix.unify_axes` documents an existing method, no overreach; no em-dashes or AI filler in any shipped prose; no process references in docstrings/CHANGELOG/notebook/rst/llms.txt (rule 15 clean).

---

Status: clean
Artifact reviewed: Workstream D (uncommitted worktree) — notebook `unify_axes` sweep; NEGATIVE result (zero notebooks modified, zero cells migrated)
Dispositions held: yes. This is a survey workstream; a zero-migration outcome is the deliverable when no faithful `HivePlot`-axis-range unification target exists (Amendment 1 context). No scope balloon (no notebook touched), no holdout touched.

Reconcile note (blind independent re-survey ran first, plan + implementer classification read second): I re-surveyed the full `examples/*.ipynb` corpus cold before reading the WS-D log, anchoring on the three axis-range API idioms (`axis_kwargs={ax:{"vmin"/"vmax"}}`, `update_axis(vmin=,vmax=)`, `place_nodes_on_axis(vmin=,vmax=)`) and reading cell source (not output blobs) for every one of the 30 `vmin`/`vmax`-bearing notebooks. My independent verdict corroborated the negative result before I saw the implementer's, so the agreement below is not read-back.

Agreement with the implementer's classification: **full agreement, no divergence.** Every genuine `HivePlot`-axis-range site resolves the same both ways:
- Only `hive_plots_more_than_three_groups.ipynb` carries real `HivePlot`-axis-range `vmin`/`vmax` pins, and it is correctly **left alone: hard-excluded (cartopy/mpl-3.11 freeze)**. The implementer's cell-level detail (cells 20/27/30 `axis_kwargs` + two `place_nodes_on_axis` calls in cell 27, `[0, 3e8]` pins) is the deferred-candidate record; my blind pass reached the same leave-alone verdict from the rubric's hard exclusion without needing to open the frozen notebook.
- `adding_repeat_axes.ipynb` (`update_axis(start=, end=)`) and `customizing_edge_curves.ipynb` (`axis_kwargs` with `angle`/`start`/`end`) use the axis API but set no `vmin`/`vmax` — not sites. Agreed.
- Every other `vmin`/`vmax` is colormap/color-scale (node/edge kwargs, `Normalize`, `nx.draw`, datashader `vmax=500`/`clim`), P2CP plural `vmins`/`vmaxes` (`election_96`, `introduction_to_p2cps`, `datashading_p2cps`), HPM already on its own `unify_axes` (`bitcoin_user_ratings` + the four `hpm_*`), the excluded pedagogical `modifying_axes.ipynb`, or output-blob noise (`hover_info`, `changing_viz_backends` `.Segments` metadata). Agreed on all.

Done-whens vs the negative result — all met:
- "Every unification-intent site migrated OR explicitly classified left-alone with reason" (Amendment 2-tightened): satisfied, every genuine site is classified left-alone with a rubric-named reason logged per notebook.
- Migration fidelity (Amendment 3 item 5): **vacuously satisfied** — no cell migrated, so no per-cell `(vmin, vmax)` equality check is owed. The gate's own escape clause (a non-equal manual computation → classify left-alone) is moot here.
- `make test-nb`: correctly NOT run on the full suite — the only notebook with real axis-range sites is the excluded one that freezes on mpl 3.11 (standing gotcha; a green full run there would be misleading, not reassuring). No notebook was modified, so none needed re-execution. This is the honest outcome, not a skipped gate.
- No overlap/contradiction with WS-C's `modifying_axes.ipynb` touch: held — WS-D did not re-touch it.

Standing item (not a WS-D gap): the deferred candidate is `hive_plots_more_than_three_groups.ipynb`'s `[0, 3e8]` `axis_kwargs`/`place_nodes_on_axis` pins, to re-examine when the cartopy/mpl-3.11 freeze lifts. This is correctly OUT of this change. One precision for the record: `[0, 3e8]` is a domain-meaningful fixed range, so under the rubric it is a **known-reference-scale** pin (leave-alone even if uniform) at least as much as a unification target — the freeze-lift revisit should re-classify it against the rubric, not assume a migration is owed. Either way, no gap in WS-D.

Concerns: none. No must-fix or worth-discussing surfaced in the blind pass, and none emerged on reconcile. Workstream D closes clean.

## Workstreams

### Workstream A: shared resolver + `HivePlot` constructor + instance method

**Status:** not started
**Depends on:** `### API Critic's take (planning mode)` filled (Amendment 3, item 6; the dispatching session enforces the same sequencing).
**Files:**

- `src/hiveplotlib/hiveplot_matrix.py` (move `_resolve_unify_axes` out unchanged; update its three call sites to import from the new location; tighten the loose `Dict[str, float]` on `__init__` to `Dict[Literal["vmin", "vmax"], float]`)
- New private module (code engineer's call: e.g. `src/hiveplotlib/_unify_axes.py`, or co-locate next to `BaseHivePlot`). Must not create a circular import: the helper reads `NodeCollection` (already used at the matrix-module top level) and `numpy`; it does not need to import `HivePlot` or `HivePlotMatrix`.
- `src/hiveplotlib/hiveplot.py`: add `unify_axes` to `HivePlot.__init__` signature (place it adjacent to `axis_kwargs` for discoverability), implemented as a two-layer mechanism (Amendment 3, item 1). Consume the parameter; never store it as an instance attribute: `self.unify_axes = ...` (the routine pattern used for `self.num_steps_per_edge` at `hiveplot.py:2706`) would shadow the bound method and turn `hp.unify_axes()` into `TypeError: 'bool' object is not callable`; HPM likewise consumes without storing (`hiveplot_matrix.py:425-429`) (Amendment 4, item 3). Layer 1, range resolution: after `_apply_graph_metrics` (`hiveplot.py:2748`; metric columns must be available as sorting variables), call the shared resolver with `axis_kwargs=None` on the instance's node collection and the user's `sorting_variables`; it returns `None` when `unify_axes=False`, else the flat resolved bounds `{"vmin": V, "vmax": W}` (explicit dict values pass through, missing sides auto-computed). The flat dict must NOT be fed into `HivePlot`'s per-axis `axis_kwargs` (`Dict[Hashable, Dict]`, `hiveplot.py:2649`): that consumption loop raises `InvalidAxisNameError` on non-axis keys (`hiveplot.py:2793-2797`). Layer 2, per-axis merge: just before that consumption loop, build the effective `axis_kwargs` (never mutating the user's dict) by writing the resolved bounds into each non-repeat axis's entry for any `vmin`/`vmax` key the user did not set on that axis; a user-set per-axis key wins and triggers the displacement warning. Repeat axes get no direct injection; the existing inheritance branch (`hiveplot.py:2817-2836`) propagates the bounds from the non-repeat sibling, the same nested non-repeat-key shape HPM's `from_partition` diagonal already hands `HivePlot` (`hiveplot_matrix.py:1348-1349`). Also add a new `HivePlot.unify_axes(vmin=None, vmax=None)` instance method mirroring `HivePlotMatrix.unify_axes` at `hiveplot_matrix.py:976`. The post-hoc method calls `self.update_axis(axis_id=..., vmin=..., vmax=..., build_hive_plot=(i == last))` so only one rebuild fires.

**Done when:**

- `HivePlot(..., unify_axes=True)` produces a hive plot whose every axis carries the same `(vmin, vmax)` pair, computed as global min/max across whichever sorting variables the user passed (single hashable or dict of axis→variable). With `repeat_axes`, the inheritance bullet below governs the repeat siblings.
- `HivePlot(..., unify_axes={"vmin": 0})` honors the explicit vmin and auto-computes vmax.
- `hp.unify_axes()` after construction reads the axes' current `vmin`/`vmax` (user overrides included) as auto-compute inputs, then applies the single resolved range to every axis, overriding all per-axis values: HPM's post-hoc semantics at `hiveplot_matrix.py:1000-1020`. Overrides are consumed as inputs and then flattened, not preserved (Amendment 3, item 4).
- `hp.unify_axes(vmin=0)` honors explicit vmin, auto-computes vmax from current axis ranges.
- Constructor precedence lives in the layer-2 merge and holds: per-axis `axis_kwargs[ax]["vmin"|"vmax"]` > explicit `unify_axes` dict values > auto-computed unified range (HPM's documented priority at `hiveplot_matrix.py:215`, enforced per axis). Post-hoc, `update_axis(vmin=, vmax=)` overrides the unified value on that axis specifically and persists until a later `unify_axes()` call flattens it (bullet above).
- When a per-axis explicit `vmin` or `vmax` displaces what `unify_axes` would have set, a warning fires via `warnings.warn(..., stacklevel=3)` naming the displaced axis, the unified value, and the user-supplied override. Use `UserWarning` (no new exception class needed; `src/hiveplotlib/exceptions/` carries Error classes only, and the displacement is a non-fatal advisory). The layer-2 merge owns this warning, constructor path only (the post-hoc method stays silent, like HPM's). Deliberate asymmetry with HPM, which resolves the same combo silently: HPM's flat `axis_kwargs` keeps the frame uniform (only the value's source changes), while a per-axis override here breaks unification on that axis (Amendment 3, item 3).
- Repeat-axis vmin/vmax inheritance at `hiveplot.py:2817-2836` continues to behave: the layer-2 merge writes unified bounds to non-repeat axes only, and the existing branch propagates them to a repeat axis exactly as it does for user-supplied explicit bounds (injected values are non-inferred, so same-sorting-variable repeat axes inherit; a repeat axis given a different sorting variable via the dict form keeps today's inheritance-gated behavior, mirroring HPM's `from_partition` diagonal at `hiveplot_matrix.py:1348-1349`). Verify by reading the branch against the merge output.
- `HivePlotMatrix.__init__`, `HivePlotMatrix.unify_axes`, and the three `from_*` classmethods continue to behave identically; only the import source for `_resolve_unify_axes` changes.
- `ruff format` + `ruff check` + `make ty` clean.

### Workstream B: tests

**Status:** not started
**Files:**

- `tests/hiveplot_test.py` — add a `Test_HivePlotUnifyAxes` class (or equivalent) mirroring `tests/hiveplot_matrix_test.py:1563-1945`'s coverage. Each test name follows `test_<method>_<scenario>` and calls the named method in its body.

**Done when:** coverage matches the HPM suite, adjusted for `HivePlot`'s single-plot shape (no `iter_populated_cells`):

- `test_unify_axes_method_no_args_auto_computes` — `hp.unify_axes()` auto-computes from current axis ranges and applies uniformly.
- `test_unify_axes_method_explicit_vmin_vmax` — both args honored, no autocompute.
- `test_unify_axes_method_explicit_vmin_only` — explicit vmin, auto vmax.
- `test_unify_axes_method_explicit_vmax_only` — explicit vmax, auto vmin.
- `test_unify_axes_method_overrides_prior_update_axis`: `update_axis(vmin=<value below the data min>)` on one axis, then `unify_axes()`; every axis (not just the touched one) now carries that vmin. Pins the post-hoc clobber: the override is consumed as an auto-compute input and flattened across axes rather than preserved per-axis (Amendment 3, item 4).
- `test_init_unify_axes_true` — constructor `unify_axes=True` produces uniform `(vmin, vmax)` across all axes; vmin equals global min of sorting variable, vmax equals global max.
- `test_init_unify_axes_dict_full` — `unify_axes={"vmin": V, "vmax": W}` uses the dict values uniformly.
- `test_init_unify_axes_dict_partial_vmin` / `_vmax` — partial dict uses provided value, auto-computes the other.
- `test_init_unify_axes_with_dict_sorting_variables` — `sorting_variables={ax: var, ...}` with `unify_axes=True` computes range across all referenced variables (concatenation; matches `_resolve_unify_axes` semantics today).
- `test_init_axis_kwargs_vmin_overrides_unify` — per-axis `axis_kwargs[ax]["vmin"]` wins over `unify_axes=True`; the other axes still carry the unified value. Wrap in `pytest.warns(UserWarning)`: the displacement warning fires, and `filterwarnings = error` turns an uncaught one into a failure (Amendment 3, item 3).
- `test_init_unify_axes_displacement_warns` — the precedence override emits a `UserWarning` whose message names the displaced axis and the unified value (caught with `pytest.warns(UserWarning, match=...)`).
- `test_init_unify_axes_dict_vs_axis_kwargs_precedence` — `axis_kwargs` wins over the explicit `unify_axes` dict (matches HPM's documented priority at `hiveplot_matrix.py:215`). Wrap in `pytest.warns(UserWarning)` likewise (Amendment 3, item 3).
- `test_init_unify_axes_with_repeat_axes` — `unify_axes=True` plays nicely with `repeat_axes=True`; repeat axes inherit the unified range from their non-repeat siblings.
- `test_init_unify_axes_false_is_default` — opt-out behavior unchanged from today's default.
- Re-export test, only if the code engineer re-exports `_resolve_unify_axes` from `hiveplot_matrix` for downstream import stability: a test whose name contains `resolve_unify_axes` must also call the resolver in its body (test-name = test-body contract), e.g. import from both homes, assert they are the same object, and call it once on a small `NodeCollection`. If not re-exported, no test required; the helper is private and its coverage arrives via the constructor tests (Amendment 3, item 8).
- Suite passes under `pytest -n 7`, 100% coverage maintained, no new warning escapes (`filterwarnings = error`).

### Workstream C: docs + docstrings + CHANGELOG

**Status:** not started
**Files:**

- `src/hiveplotlib/hiveplot.py` — docstring on the new `unify_axes` constructor parameter and on the new `HivePlot.unify_axes` method. Frame `unify_axes` (cross-axis convenience) and per-axis `vmin`/`vmax` (outlier clipping, known reference scales) as complementary, not alternatives. State the precedence chain explicitly, both directions (Amendment 3, item 4): constructor, per-axis `axis_kwargs` > explicit `unify_axes` dict values > auto-computed unified range; post-hoc, `unify_axes()` derives auto values from the axes' current ranges (user overrides included) and then overrides every axis. Make the shared-name / different-arithmetic trap explicit (Amendment 5, item 2): because the constructor auto-computes the range from this instance's node data while post-hoc `unify_axes()` auto-computes from the axes' current ranges, the two paths can yield different unified ranges on the same object once any per-axis bound has deviated from the raw data extent. Note the displacement warning on the constructor combo, and that `HivePlotMatrix` resolves the same combo silently because its flat `axis_kwargs` keeps the frame uniform (Amendment 3, item 3). The warning carries no opt-out parameter by design; the same docstring note names standard suppression (`warnings.catch_warnings` / `filterwarnings`) for the user running the deliberate combo, since under a downstream `filterwarnings = error` config the unavoidable warning becomes an error (Amendment 4, item 4b). One sentence that unifying across different-scale sorting variables rarely reads well, e.g. a `[0, 1]` metric crushed against a degree-scale range (Amendment 3, item 7). State the auto-compute scope (Amendment 2): `True` unifies within this instance, from this instance's node data; for shared ranges across separate `HivePlot` instances holding different data, pass the same full `{"vmin": V, "vmax": W}` dict to each (a partial dict auto-computes the missing side per instance and inherits `True`'s caveat); the idiom is pinned as Example 6 in API usage examples (Amendment 4, item 2).
- `src/hiveplotlib/hiveplot_matrix.py` — tighten or refresh the `unify_axes` docstring on `__init__` only if the import move or type-annotation tightening makes the current prose drift. No prose changes required if the move is purely mechanical.
- `examples/modifying_axes.ipynb` — the two "Why might you care" case-2 bullets under `### Vmin` and `### Vmax` ("generating multiple hive plots to compare to each other", cells around lines 304-308 and 418-422): that case is cross-plot, so route it to passing the same full `unify_axes={"vmin": V, "vmax": W}` dict to each plot, not to bare `unify_axes=True` (per-plot `True` auto-computes from each instance's own data; the docstring carries the full scope story, Amendment 2). Case 1 (outlier clipping) stays with raw per-axis `vmin`/`vmax`. Single-sentence touch each, not a rewrite.
- `CHANGELOG.rst` — entry under the unreleased section: "Add ``unify_axes`` constructor parameter and instance method to ``HivePlot``, mirroring ``HivePlotMatrix``."
- Gallery / tutorial notebook touch: optional, docs engineer's judgment. The existing notebook landscape covers HPM unify well; a single hive plot demo could land in `modifying_axes.ipynb` rather than a new file. Defer if it inflates scope.

**Done when:**

- `make docs` renders without new warnings.
- Updated docstrings render the precedence chain (constructor and post-hoc directions) cleanly in autodoc.
- Docstring and notebook prose state the within-instance scope of auto-computed unification; no shipped prose implies per-plot `unify_axes=True` yields a shared frame across separate instances holding different data (cross-plot guidance points at the shared full-dict form).
- `examples/modifying_axes.ipynb` reads naturally after the touch (no em-dashes, no AI filler).
- `make test-nb` passes.

## Plan amendments

### Amendment 1 (2026-05-27): notebook sweep for legacy `vmin`/`vmax` unification patterns

**Trigger:** user ask. Adds a workstream (rule 14 "adds a workstream").

**Tag:** Added workstream.

**Context:** today the only way to unify axes on a `HivePlot` is to manually thread `vmin` / `vmax` through `axis_kwargs` at construction or through per-axis `update_axis(vmin=, vmax=)` post-hoc. Once Workstream A lands, some existing `examples/*.ipynb` cells that do this the verbose way for apples-to-apples comparison reasons become candidates for simplification. This is distinct from cells using `vmin`/`vmax` for legitimate non-unification reasons (clipping outliers, pinning a known reference scale), which should be left alone.

### Workstream D: notebook sweep for legacy unify-axes patterns

**Status:** not started
**Depends on:** Workstream A (the new affordance must exist). Can run concurrent with or after Workstreams B and C.
**Files:**

- `examples/*.ipynb` — survey and selective migration. Only `examples/` is in scope per the project trip-wires ("Only edit notebooks in `examples/`"; `docs/source/notebooks/` and `docs/source/gallery_examples/` are auto-generated).

**Method:**

1. Grep `examples/*.ipynb` for explicit `vmin=` / `vmax=` usage in `HivePlot` call sites (constructor `axis_kwargs`, post-hoc `update_axis`, or anywhere else they thread through `HivePlot`). Exclude `HivePlotMatrix` hits; HPM already has `unify_axes` and the relevant notebooks already use it.
2. For each hit, classify the intent from surrounding prose and code:
   - **Within-instance unification:** the cell computes global `min`/`max` across one instance's sorting variable(s) and pins that instance's axes to the shared range. Migrate to `unify_axes=True` (or `unify_axes={"vmin": V}` / `{"vmax": W}` when one side is pinned to a meaningful literal).
   - **Cross-plot unification, same data:** multiple `HivePlot` instances built from the same node data and sorting variables (e.g. comparing edge subsets over one node set). Auto-compute reads the whole `NodeCollection`, so per-plot `unify_axes=True` yields identical ranges; migrate to it (Amendment 2).
   - **Cross-plot unification, different data:** the compared instances hold different node data, so per-plot `unify_axes=True` does not produce a shared frame (Amendment 2). Migrate to computing shared `V`/`W` across the compared data once and passing the same full `unify_axes={"vmin": V, "vmax": W}` dict to each instance; this replaces the per-axis `axis_kwargs` threading while the manual cross-data `min`/`max` stays. Partial dicts are not a valid target here: the auto-computed side would diverge per instance. Example 6 in API usage examples pins the target idiom (Amendment 4, item 2).
   - **Outlier clipping:** the cell pins one or both bounds to drop tail values that compress the rest of the axis. Leave alone.
   - **Known reference scale:** the cell pins to a domain-meaningful range (`[0, 1]` for a normalized metric, etc.) that the data may not span. Leave alone, even if it happens to be uniform across axes.
   - **Pedagogical demo of raw `vmin`/`vmax`:** the cell exists to teach the per-axis primitive. Leave alone; if anything, add a one-sentence cross-reference to `unify_axes` for the unification use case (this overlaps with Workstream C's `modifying_axes.ipynb` touch and should not be duplicated).
3. For each migrated cell, update surrounding markdown prose to match the simplified call site. No em-dashes, no AI filler, length discipline. Removed prose that explained the manual global-min/max computation goes away with the code; do not replace it with a longer explanation of `unify_axes` (the affordance is self-evident).
4. Re-run the affected notebooks end-to-end via `make test-nb` to confirm outputs still render.

**Done when:**

- Every unification-intent `vmin`/`vmax` site in `examples/*.ipynb` that targets a `HivePlot` is migrated to the scope-correct `unify_axes` form per the classification above (within-instance and same-data cross-plot to `unify_axes=True`; different-data cross-plot to the shared full-dict form), OR explicitly classified in the implementation log as "left alone: <reason>" with the reason matching one of the legitimate-use categories above.
- Migration fidelity (Amendment 3, item 5): each migrated cell resolves the same per-axis `(vmin, vmax)` as the code it replaced, verified at migration time by comparing `hp.axes[...].vmin` / `.vmax` (or node placements) before and after the edit (scratch check outside the tree per rule 16; no comparison asserts shipped in the notebooks), with the per-cell result recorded in the implementation log. A manual computation that does not equal whole-collection `nanmin`/`nanmax` over the referenced sorting variable(s) means the cell is classified left-alone, not migrated.
- Surrounding prose reads naturally and does not contradict the new code.
- `make test-nb` passes.
- No overlap or contradiction with Workstream C's `modifying_axes.ipynb` touch; if both workstreams want to edit the same cell, Workstream C's framing-level change lands first and Workstream D defers to it.

**Critic pass:** invoke viz-critic on the changed notebook cells (rendered output) after the sweep lands, since the migration touches user-facing example notebooks where the visual output is the contract.

### Amendment 2 (2026-07-06): scope-correct cross-plot `unify_axes` guidance

**Trigger:** research-liaison pre-task premise flag, disposition deferred to the orchestrator. Modifies done-whens of not-yet-run Workstreams C and D (rule 14 "modifies a done-when").

**Tag:** In-scope tweak (Workstreams C and D; workstream set unchanged).

**Premise (verified against `_resolve_unify_axes` at `src/hiveplotlib/hiveplot_matrix.py:143-183` and the notebook cells):** auto-computed `unify_axes=True` reads only the instance's own `NodeCollection` (whole-collection `nanmin`/`nanmax` over the referenced sorting-variable columns). The `modifying_axes.ipynb` prose this plan targets ("generating multiple hive plots to compare to each other") is the cross-plot use case: separate instances holding *different* data get no shared frame from per-plot `True`; passing the same full `{"vmin": V, "vmax": W}` dict to each instance does, and still beats threading per-axis `axis_kwargs`. Separate instances holding the *same* node data and sorting variables do get identical auto-computed ranges, since the auto-compute inputs are identical (the bioinformatics-analysis HPM pattern), so `True` remains a valid migration target there. A partial dict auto-computes the missing side per instance and inherits `True`'s caveat. This operationalizes the constraint research-liaison already recorded under `fixed-layout-comparability.md` in "Prior ADRs / design docs"; the plan's guidance sections had not been cascaded to match.

**Delta (cascaded in place; no workstream has run and no concurrent editor holds the file):**

- "Patterns this replaces" bullet 1: "point at `unify_axes` first" superseded by scope-routed guidance (cross-plot points at the shared full-dict form; `True` only matches across plots on same node data and sorting variables).
- Workstream C, `hiveplot.py` files bullet: docstring must state the auto-compute scope (within-instance), route different-data cross-plot comparison to the shared full-dict form, and carry the partial-dict caveat.
- Workstream C, `modifying_axes.ipynb` files bullet: the case-2 "multiple hive plots" bullets route to the shared full-dict form, not bare `unify_axes=True`; still a single-sentence touch each.
- Workstream C, done-when: one added (no shipped prose implies per-plot `True` yields a shared frame across different-data instances).
- Workstream D, method step 2: the "apples-to-apples unification" migration class split into three: within-instance (to `True`), same-data cross-plot (to `True`, provably identical ranges), different-data cross-plot (to the shared full dict; partial dicts invalid there).
- Workstream D, first done-when: migration target must be the scope-correct form per class.

**Done-whens touched:** Workstream C (one added), Workstream D (first bullet tightened). Workstreams A and B untouched: the resolver's semantics do not change, only the guidance and migration targets built on them. API usage examples untouched: all five are within-instance call shapes, which stay correct; whether the set wants a cross-plot dict-form example is left to api-critic's pending planning pass.

### Amendment 3 (2026-07-06): adversary planning-challenge disposition (grill skipped)

**Trigger:** adversary planning-mode challenge (8 items). Grill knowingly skipped, so the disposition that would happen in the grill lands here, explicitly, per item. Modifies done-whens of not-yet-run Workstreams A, B, C, D (rule 14 "modifies a done-when").

**Tag:** In-scope tweaks (all four workstreams touched in place; workstream set unchanged).

**Verification:** every anchor and behavior claim relied on below was independently re-verified against the working tree this pass, not copied from the challenge: resolver `hiveplot_matrix.py:143-183`; nested `axis_kwargs` `hiveplot.py:2649`; `InvalidAxisNameError` loop `hiveplot.py:2793-2797`; repeat inheritance `hiveplot.py:2817-2836`; HPM post-hoc method `hiveplot_matrix.py:976-1020`; resolver call sites `:1319`/`:1724`/`:2163`; loose annotation `:346`; priority docstring `:215`; `_apply_graph_metrics` in `__init__` at `hiveplot.py:2748`; HPM unify test block `tests/hiveplot_matrix_test.py:1563-1945`; `from_partition` diagonal nested-kwargs shape `hiveplot_matrix.py:1348-1349`.

**Dispositions (challenge items in order):**

1. Must-fix, Workstream A mechanism: **accepted; the adversary's two-layer design is adopted and specced in place.** Verified workable: the resolver called with `axis_kwargs=None` returns exactly the flat resolved bounds (or passes `None` through when `unify_axes=False`), and the layer-2 non-repeat-key injection is the same nested shape HPM's `from_partition` diagonal already hands `HivePlot` (`hiveplot_matrix.py:1348-1349`), so the design is mirror-faithful and keeps "HPM byte-for-byte unchanged, import source only" true. Precedence and the displacement warning live in the layer-2 merge; repeat axes inherit through the existing branch. Cascaded: WS-A files bullet, WS-A done-when bullets (unified-range, post-hoc, precedence, warning, repeat-axis), Holdouts bullet 1.
2. Stale line anchors: **accepted; refreshed in place** across Default justifications, Patterns this replaces, WS-A, WS-B, and Holdouts.
3. Displacement warning vs HPM silence: **accepted as keep-and-document (option b).** The asymmetry tracks a real semantic difference, not an oversight: HPM's flat `axis_kwargs` combined with `unify_axes` still yields one uniform frame (only the value's source changes), while a `HivePlot` per-axis override breaks the unification the user just asked for on that axis. Option (a) drop: rejected, loses real protection against a silently non-unified frame. Option (c) propagate to HPM: rejected, changes released behavior (a new warning breaks `filterwarnings=error` consumers and needs a CHANGELOG entry) to warn where the combo cannot produce divergence. Constructor path only; the post-hoc method stays silent like HPM's. Cascaded: WS-A warning bullet, Naming audit, WS-C docstring sentence, `pytest.warns` on WS-B's two precedence tests.
4. Post-hoc precedence reversal: **accepted.** Verified at `hiveplot_matrix.py:1000-1020`: current per-axis ranges, user overrides included, feed the auto-compute, then every axis is overridden. Cascaded: WS-A post-hoc done-when reworded (consumed then flattened, not preserved), WS-C docstring states both directions, WS-B gains `test_unify_axes_method_overrides_prior_update_axis`.
5. Workstream D output-preservation gate: **accepted.** Cascaded: new WS-D done-when, per-cell migration-time `(vmin, vmax)` equality (scratch verification, logged per cell), with non-equal cells classified left-alone instead of migrated.
6. api-critic planning gate: **accepted as a recorded gate.** Sequencing is enforced by the dispatching session (api-critic planning mode runs immediately after this amendment, before any code dispatch); a Depends-on line lands on WS-A so the gate is also mechanical in the plan.
7. Mixed-scale unification footgun: **accepted at the proposed size.** Cascaded: one docstring sentence in WS-C.
8. `test_resolve_unify_axes_import_stability` name-vs-body: **accepted.** Cascaded: WS-B bullet reworded; if the re-export branch is taken, the test body must call the resolver (identity assert plus one real call); if not re-exported, no test, as already stated.

Nothing deferred; no item rejected (two sub-options inside item 3 rejected with reasons above). API usage examples untouched: Example 5's warning comment already matches the item-3 keep decision. Structural addition in the same pass: `## Alignment (grill)` and `## Failure modes` sections added recording the skip, so the gate reads as a decision rather than an omission and post-impl adversary extracts have their expected slots.

**Done-whens touched:** WS-A (mechanism respecced, Depends-on added), WS-B (one test added, two tests gain `pytest.warns`, re-export bullet reworded, anchors refreshed), WS-C (docstring scope grows by three sentences: post-hoc direction, warning asymmetry, mixed-scale), WS-D (migration-fidelity gate added).

### Amendment 4 (2026-07-06): api-critic planning-pass disposition

**Trigger:** api-critic planning pass (one must-fix on the example snippets plus supporting findings; the API surface itself agreed). Routed per the repo trip-wire: broken data construction in "API usage examples" goes to orchestrator `amend-plan` for a feasibility check before code-engineer dispatch.

**Tag:** In-scope tweaks (API usage examples rewritten in place; one sentence each cascaded to WS-A, WS-C, WS-D; workstream set unchanged, A/B/C/D).

**Verification (independently re-checked against the working tree this pass, not copied from the critic's take):** `networkx_to_nodes_edges` signature is `(graph, unique_id_name="unique_id", check_uniqueness=True)` (`converters.py:20-24`; no metrics parameter, so all five prior snippets raised `TypeError` at their second line); `node_graph_metrics` lives on `HivePlot.__init__` (`hiveplot.py:2658`) with `graph=` keyword-only (`:2641`) and metrics attached before partitioning (`:2748`); the corrected construction is built verbatim by `tests/hiveplot_test.py:6026-6052`, which asserts axes `{"Mr. Hi", "Officer"}` and a `degree` column (the runnability witness); HPM consumes `unify_axes` without storing it (`hiveplot_matrix.py:425-429`); `axis_kwargs: Optional[Dict[Hashable, Dict]]` (`hiveplot.py:2649`) matches Example 5's shape; `warn_on_overlapping_kwargs` (`hiveplot.py:2655`) is the opt-out precedent weighed in item 4b. Example 6 feasibility: `g_full.copy()` preserves the `club` node attribute and `remove_edges_from` drops no nodes, so both instances partition on `club` while degrees genuinely diverge; no new entry point, no undocumented convention.

**Dispositions:**

1. Must-fix, broken example construction: **accepted; all five snippets rewritten in place** to the `graph=` construction path (one-line data construction; the two-step converter idiom predates the v0.28 `graph=` keyword). Each example's `unify_axes` / `axis_kwargs` / post-hoc lines kept as proposed (independently confirmed correct against the WS-A spec and today's `axis_kwargs` shape). A runnability-witness note now heads the Proposed section citing `tests/hiveplot_test.py:6026-6052`. Side benefit retained from the critic's take: the construction itself demonstrates the ordering WS-A leans on (metrics attach before the resolver reads `sorting_variables`).
2. Cross-plot sixth example (Amendment 2's deferred decision, resolved by the critic as add): **accepted; promoted into Proposed as Example 6** essentially as proposed (karate club full vs edge-thinned copy, shared bounds computed once, same full dict to both constructors), construction adjusted to match the rewritten Examples 1-5. It pins the idiom WS-C's docstring/notebook touches and WS-D's different-data migration class route to; pointers added at both sites so neither invents the shape independently. Teaching prose stays with WS-C.
3. Method-shadowing trap (worth-discussing): **accepted; one sentence added to WS-A's `hiveplot.py` files bullet**: the constructor parameter is consumed, never stored as an instance attribute (storing would shadow the bound `unify_axes` method; HPM likewise consumes without storing). Save-a-cycle note, as the critic sized it; WS-B's method tests would catch the mistake anyway.
4. Low-confidence items, each disposed explicitly:
   - (a) Example 1's unused `betweenness_centrality`: **accepted**; trimmed to `"degree"` as part of the item-1 rewrite.
   - (b) Displacement warning opt-out: **accepted as no-new-parameter plus documented standard suppression** (the critic's suggested shape). A dedicated opt-out parameter is surface growth for a rare, deliberate combo; `warn_on_overlapping_kwargs` is precedent but guards a far more common collision. Cascaded: WS-C's warning documentation names `warnings.catch_warnings` / `filterwarnings` for the deliberate-combo user (who otherwise hits an unavoidable error under a downstream `filterwarnings = error` config). WS-A/WS-B untouched: the warning mechanism and its tests already match the keep decision (Amendment 3, item 3).

Nothing deferred; no item rejected. The critic's recurring-pattern note (author examples against shipped signatures, cite a runnability witness per snippet) is process guidance for future plans, recorded in the orchestrator's expertise file rather than this plan. WS-A's Depends-on gate (`### API Critic's take (planning mode)` filled) is now discharged.

**Done-whens touched:** none (all four workstreams' done-when lists unchanged). Files bullets touched: WS-A (consume-don't-store sentence), WS-C (suppression sentence, Example 6 pointer), WS-D method step 2 (Example 6 pointer). API usage examples section rewritten (six examples, runnability-witness header).

### Amendment 5 (2026-07-06): Workstream A post-impl critic disposition

**Trigger:** Workstream A post-impl critics (api-critic + adversary), one worth-discussing each, no must-fix. Modifies a done-when of not-yet-run Workstream C (rule 14 "modifies a done-when"); the other finding is deferred with a revival trigger. Workstream set unchanged.

**Tag:** In-scope tweak (Workstream C) + Deferred follow-up (known limitation, separate decision).

**Verification (re-checked against the worktree `hiveplotlib-51-unify-axes` this pass, not copied from the critics):** the resolver guard is `if unify_axes is False: return axis_kwargs` and `explicit = unify_axes if isinstance(unify_axes, dict) else {}` with `need_vmin = "vmin" not in explicit and "vmin" not in existing` (`_unify_axes.py`, function body confirmed): a non-dict non-`False` value auto-computes as `True`; a typoed dict key (`"vmim"`) leaves both `need_*` True, both sides auto-compute, and the stray key rides into `{**unified, **explicit, **existing}` but is never read by the merge. The merge reads only literal `"vmin"`/`"vmax"` (`_merge_unified_axis_bounds`, `hiveplot.py`). The two arithmetics are confirmed distinct: the constructor resolves from raw `nodes.data[sorting_variables]` (`_unify_axes.py`), the post-hoc method from `[axis.vmin for axis in self.axes.values()]` / `axis.vmax` (`hiveplot.py` `unify_axes` body). Both are mirror-faithful to HPM (the resolver moved byte-for-byte; the post-hoc source-of-truth is HPM's own).

**Dispositions:**

1. **api-critic worth-discussing — silent swallow of invalid `unify_axes` input: deferred as a known limitation with a revival trigger; NOT accepted into this plan.** Confirmed silent-wrong, not silent-noop: `unify_axes={"vmim": 0}` returns a fully auto-computed range while the user believes they pinned a floor, and `unify_axes=5`/`=0` behaves as `True`. But this is inherited mirror-faithful behavior: the resolver moved byte-for-byte from HPM, ships identically on `HivePlotMatrix` today, and the plan's explicit thesis is mirror fidelity ("HPM byte-for-byte unchanged, import source only"). Adding runtime validation on the shared resolver is a deliberate divergence from that thesis and a cross-cutting decision touching both classes' released behavior (a new raised exception can break callers relying on the current tolerance, and needs its own naming / default-justification / docstring / CHANGELOG treatment and its own tests on both classes). That is a separate plan or follow-up, not an in-scope tweak here, and the api-critic itself framed the fix as needing plan authority rather than an inline change. Rejected as option (c) docs-only-note-in-WS-C too: a docstring line saying "invalid keys are silently ignored" documents a footgun without fixing it and adds a caveat the mirror source (HPM) does not carry, drifting the two classes' prose; the honest home for this is a validation decision that covers both classes at once, not a one-class docstring asterisk. **Revival trigger:** when a user (or a downstream consumer's `filterwarnings`/type-checker) is bitten by a swallowed invalid `unify_axes` value on either `HivePlot` or `HivePlotMatrix`, open a shared-resolver input-validation plan covering both classes together (raise on non-`vmin`/`vmax` dict keys and on non-bool/non-dict values, with the exception-class + CHANGELOG + both-class test work that a released-behavior change requires). Recorded as a known limitation, not silently dropped. No workstream, no done-when, no code touched this pass.

2. **adversary worth-discussing — constructor vs post-hoc auto-range data-source asymmetry: accepted as an in-scope sharpening of WS-C's existing docstring spec; no new done-when.** Already largely covered: WS-C's `hiveplot.py` files bullet (Amendment 3 item 4 + Amendment 2) already requires the docstring to state both arithmetics, that the constructor computes the auto range from this instance's node data and that post-hoc `unify_axes()` derives auto values from the axes' current ranges (user overrides included) and then overrides every axis. What the existing spec does not yet make explicit is the trap the adversary names: the same method name over the two data sources can yield two *different* unified ranges on one object once any per-axis bound has deviated from the raw data extent. Cascaded as a one-clause sharpening to WS-C's docstring spec (below); the WS-C dispatch brief should emphasize the shared-name / different-arithmetic trap when writing that docstring. The adversary self-recommended batching to WS-C and not halting; this honors that. No done-when added (the existing "render the precedence chain, both directions, cleanly in autodoc" done-when already gates the docstring; the sharpening rides inside it).

**Delta (cascaded in place; WS-C has not run and no concurrent editor holds the file):**

- Workstream C, `hiveplot.py` files bullet: the both-directions precedence sentence gains a trailing clause making the different-arithmetic trap explicit, that because the constructor auto-computes from instance node data and post-hoc `unify_axes()` auto-computes from current axis ranges, the two paths can produce different unified ranges on the same object once any per-axis bound deviates from the raw data extent.

**Done-whens touched:** none. Finding 2 folds into WS-C's existing docstring done-when; finding 1 is deferred with a revival trigger and touches no workstream.

**Workstream set:** unchanged (A/B/C/D). Neither finding adds, removes, or changes the identity of a workstream. Finding 1's accept-as-validation option (which WOULD have changed the set) is deliberately not taken; it is deferred to a separate plan.

## Holdouts

- `src/hiveplotlib/hiveplot.py:2817-2836` (repeat-axis vmin/vmax inheritance): intentionally untouched. It runs inside the `axis_kwargs` consumption loop on the entries the layer-2 merge builds, so repeat axes inherit the unified values through the existing mechanism.
- `BaseHivePlot.place_nodes_on_axis` `vmin`/`vmax`: untouched. Base-class plumbing; `unify_axes` lives at the user-facing layer.
- `HivePlot.update_axis(vmin=, vmax=)`: untouched. Per-axis post-hoc setter remains the right tool for one-axis overrides.

## Implementation log

Append-only. After each workstream completes, one line in the same turn:

- 2026-07-06: Workstream A complete. Moved `_resolve_unify_axes` unchanged to new `src/hiveplotlib/_unify_axes.py` (HPM now imports it, behavior identical; the name stays importable from `hiveplot_matrix` as a side effect of that import), tightened HPM `__init__`'s `unify_axes` annotation to `Dict[Literal["vmin", "vmax"], float]`, added `HivePlot.__init__` `unify_axes` (consumed, never stored; layer 1 resolves flat bounds via the shared resolver with `axis_kwargs=None` after `_apply_graph_metrics`, layer 2 merges them into every non-repeat axis's kwargs via new `_merge_unified_axis_bounds` without mutating the user dict, user-set per-axis vmin/vmax winning with a `UserWarning` at `stacklevel=3`; repeat axes inherit through the existing branch) and `HivePlot.unify_axes(vmin=None, vmax=None)` mirroring HPM's post-hoc clobber with one rebuild (`build_hive_plot` on last axis only). `make format` / `make ty` / full `make test` green (1359 passed); all six API usage examples plus repeat-axis, dict-sorting-variables, and post-hoc-clobber spot checks sanity-verified in a /tmp script.
- 2026-07-06: Workstream B complete. Added `TestHivePlotUnifyAxes` to `tests/hiveplot_test.py` (17 tests, no networkx marker: a `NodeCollection` built directly with disjoint per-group `value` slices makes every vmin/vmax an exact literal). Covers the post-hoc method (no-args, both-explicit, vmin-only, vmax-only, prior-`update_axis` clobber), the constructor (`True`, full/partial-vmin/partial-vmax dict, dict `sorting_variables` concatenation range, repeat-axis inheritance, false-is-default), precedence + displacement warning (per-axis `axis_kwargs` wins with the other axes still unified, `axis_kwargs` vs explicit dict, dedicated `pytest.warns(match=)` on the message naming the axis + unified value; the two precedence tests wrapped in `pytest.warns` for `filterwarnings=error`), a repeat-axis user-override-survives case (exercises the merge's surviving-user-entry branch), and the `resolve_unify_axes` re-export (same object from both homes, called once on a small `NodeCollection`). Drove the new `hiveplot.py` surfaces (merge call, `_merge_unified_axis_bounds` body, post-hoc `unify_axes()` body) to 0 uncovered; full `make test` green (1376 passed, `src/hiveplotlib` 100%), `make format` / `make ty` clean.
- 2026-07-06: Workstream C docstrings + CHANGELOG complete (notebook prose touch handled separately by notebook-author). Wrote the full `HivePlot.__init__` `unify_axes` param docstring and the post-hoc `HivePlot.unify_axes` method docstring in `src/hiveplotlib/hiveplot.py`: clear one-line lede up top, then complementary framing against per-axis `axis_kwargs`, both-directions precedence (constructor `axis_kwargs` > explicit dict > auto; post-hoc consume-then-clobber), the constructor-only displacement warning + HPM-silent asymmetry + standard-suppression note (no opt-out param), within-instance auto-compute scope with the cross-plot full-dict idiom, the mixed-scale caveat, and the shared-name/different-arithmetic trap (constructor reads raw node data, post-hoc reads current axis ranges). Cross-referenced `HivePlotMatrix.unify_axes`, which surfaced a pre-existing autodoc gap (the method was public but absent from `docs/source/autodoc/hive_plots/hive_plot_matrix.rst`); added the missing `.. automethod::` under Bulk Modification, which resolves the cross-ref. Added the `Added` CHANGELOG entry (4 lines, at the released-entry altitude). Fixed a now-stale `llms.txt` line that called `unify_axes` a `HivePlotMatrix`-only feature. HPM `__init__` docstring left untouched (annotation tighten was purely mechanical, no prose drift). `make docs` builds with zero warnings; `make format` / ruff clean; all touched files LF (git `--eol` verified).
- 2026-07-06: Workstream C notebook-prose touch complete (distinct from the docstring/CHANGELOG line above). Added one sentence to each of the two identical case-2 "generating multiple hive plots to compare to each other" bullets in `examples/modifying_axes.ipynb` (under `### Vmin` and `### Vmax`), routing the cross-plot comparison case to the shared full-dict `unify_axes={"vmin": V, "vmax": W}` form passed to each `HivePlot`, and explicitly warning off bare `unify_axes=True` (which auto-computes each plot's range from its own data). Framing matched to the shipped `HivePlot.__init__` `unify_axes` docstring's Scope paragraph. Markdown-only: no code cell touched (verified against HEAD, cell count unchanged, diff = 2 lines), existing raw-`vmin`/`vmax` teaching and the outlier-clipping case-1 bullets left intact. Notebook re-executed end-to-end via nbconvert on the worktree `python3` kernel with zero error outputs; committed notebook left with its existing outputs (markdown-only change needs no re-save). No `llms.txt` entry (routine cross-reference, not a consequential change). `git status` shows no CRLF/whitespace warning.
- 2026-07-06: Workstream D complete. Survey outcome: **migrated nothing** (no faithful `HivePlot`-axis-range unification target exists outside the hard-excluded cartopy notebook). Surveyed all 30 `examples/*.ipynb` carrying a raw `vmin`/`vmax` grep hit (261 raw occurrences); classified each real cell by scanning cell source (not outputs, which is where the plotly/datashader-JSON `vmin`/`vmax` count noise lives, e.g. plotly.ipynb 37 grep hits resolve to 1 code cell). Every `HivePlot`-axis-range `vmin`/`vmax` site (`axis_kwargs={ax: {"vmin","vmax"}}` and `place_nodes_on_axis(vmin=, vmax=)`) lives in `examples/hive_plots_more_than_three_groups.ipynb` (cells 20, 27, 30, and the two `place_nodes_on_axis` calls in 27) and is **left alone: excluded, cartopy/mpl-3.11 freeze, notebook held out of this change** (standing project gotcha); those are within-instance/reference-scale pins to `[0, 3e8]` that would otherwise be `unify_axes` candidates, but the notebook is not edited or re-executed. All other hits are legitimate non-targets: colormap / color-scale `vmin`/`vmax` on node/edge kwargs, `Normalize` colorbars, `nx.draw`, or datashader density (`datashader`, `datashading_statistical_summaries_of_metadata`, `matplotlib`, `plotly`, `setting_partition_variable`, `setting_sorting_variables`, `visualizing_node_metadata`, `exporting_hive_plots_to_networkx`, `hive_plots_for_large_networks`'s `imshow` `vmax=500`); P2CP axis-range `vmins`/`vmaxes` on `p2cp_n_axes` (`datashading_p2cps`, `election_96`, `introduction_to_p2cps`); `HivePlotMatrix` `unify_axes` + datashader density (`bitcoin_user_ratings`, `hpm_from_partition`, `hpm_from_tags`, `hpm_from_variable_sweep`, `hpm_generic`); and the pedagogical raw-primitive notebook `modifying_axes.ipynb` (teaches raw `vmin`/`vmax`; Workstream C already added the `unify_axes` cross-reference, not re-touched). `adding_repeat_axes.ipynb` (`update_axis` start/end only) and `customizing_edge_curves.ipynb` (`axis_kwargs` angle/start/end only) use the axis API but set no `vmin`/`vmax`, so they are not sites. No migration-fidelity check was needed (no cell migrated). Zero notebooks modified, so per the brief no re-execution and no `llms.txt`/CHANGELOG entry. `git status` clean (no working-tree change from this workstream).
