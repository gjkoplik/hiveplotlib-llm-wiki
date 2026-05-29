# Plan: HivePlot.unify_axes — mirror the HivePlotMatrix affordance on single hive plots

<!--
Hiveplotlib and wiki-structure plans go to wiki/wiki/plans/<topic>.md (tracked
in the wiki submodule). Harness-self plans go to agent-harness/.claude/plans/
<topic>.md (gitignored). The plan is the canonical reference; the conversation
transcript is not.
-->

## Goal

Add a `unify_axes` affordance to `HivePlot` that mirrors the existing one on `HivePlotMatrix`. Today, a user who wants apples-to-apples axis ranges on a single hive plot has to compute global `min`/`max` over their sorting variable(s) by hand and thread those values through either the constructor's `axis_kwargs` or per-axis `update_axis(vmin=, vmax=)` calls. After this lands, `HivePlot(..., unify_axes=True)` and `hp.unify_axes()` do that work, with the same `True | dict` / no-arg / `vmin`-or-`vmax`-only call shapes the HPM API already establishes. Both calling conventions on HPM stay byte-for-byte unchanged; the underlying resolver becomes shared code.

## Prior ADRs / design docs

Populated by research-liaison at planning start. List relevant entries here once liaison has run.

Known wiki references to dig into during the liaison pass:

- `wiki/wiki/plans/networkx-metric-expansion-and-backend-refactor.md` — mentions `unify_axes` in the HPM construction sequence; useful background but not load-bearing.
- `wiki/wiki/plans/i-want-to-plan-optimized-hoare.md` — references the `_resolve_unify_axes` call order around graph-metric attachment.

No ADR directory exists yet (`wiki/wiki/adr/` empty), so this plan does not displace an accepted ADR.

## Patterns this replaces

- "If you were generating multiple hive plots to compare to each other, you might want to restrict the comparable axes to the same `[vmin, vmax]` range" prose in `examples/modifying_axes.ipynb` cells around lines 304-308 and 418-422. Once `unify_axes` exists on `HivePlot`, the docstring/notebook callout for this use case should point at `unify_axes` first and use raw `vmin`/`vmax` only for the outlier-clipping / known-reference-range cases.
- Direct imports of `_resolve_unify_axes` from `hiveplotlib.hiveplot_matrix` (three call sites today, all in `src/hiveplotlib/hiveplot_matrix.py:1127`, `:1480`, `:1872`). Replace with the new shared location once the helper moves.
- `Dict[str, float]` annotation on `HivePlotMatrix.__init__`'s `unify_axes` parameter at `src/hiveplotlib/hiveplot_matrix.py:301` is looser than the `Dict[Literal["vmin", "vmax"], float]` used on the three `from_*` classmethods. Tighten to match while we're in the area; do not loosen the new `HivePlot.__init__` annotation to match the looser one.

## Default justifications

- `HivePlot.__init__(unify_axes=False)`: matches the HPM default. A single hive plot's axes are independent ranges by design; unifying changes that and should be opt-in. The user paying the conceptual cost of unification is the one who knows they want it.
- `HivePlot.unify_axes(vmin=None, vmax=None)`: mirrors `HivePlotMatrix.unify_axes` at `src/hiveplotlib/hiveplot_matrix.py:832`. `None` means auto-compute from the data the instance already holds. No new defaults beyond the HPM mirror.

## Naming audit

- New parameter: `unify_axes` on `HivePlot.__init__`. Vs. user vocab: ok. Same name as `HivePlotMatrix.__init__`'s parameter; users learning one transfer to the other without surprise.
- New method: `HivePlot.unify_axes()`. Vs. user vocab: ok. Same name and signature shape as `HivePlotMatrix.unify_axes()`.
- Internal helper relocation (e.g. `hiveplotlib._unify_axes._resolve_unify_axes` or a sibling under `hiveplotlib/`): out of scope per "Internal module/package names are out of scope" in the template. Code engineer picks a private location that breaks no circular imports.

## API usage examples

Required when this work adds or modifies user-facing API.

### Proposed (from planner / Orchestrator)

```python
# Example 1: opt into unified ranges at construction time, fully auto-computed
# Example data:
import networkx as nx
from hiveplotlib.converters import networkx_to_nodes_edges

g = nx.karate_club_graph()
nodes, edges = networkx_to_nodes_edges(g, node_graph_metrics=["degree", "betweenness_centrality"])

# Call site:
from hiveplotlib import HivePlot

hp = HivePlot(
    nodes=nodes,
    edges=edges,
    partition_variable="club",
    sorting_variables="degree",
    unify_axes=True,
)
```

```python
# Example 2: pin one side of the unified range, let the other auto-compute
# Example data:
import networkx as nx
from hiveplotlib.converters import networkx_to_nodes_edges

g = nx.karate_club_graph()
nodes, edges = networkx_to_nodes_edges(g, node_graph_metrics=["degree"])

# Call site:
from hiveplotlib import HivePlot

hp = HivePlot(
    nodes=nodes,
    edges=edges,
    partition_variable="club",
    sorting_variables="degree",
    unify_axes={"vmin": 0},
)
```

```python
# Example 3: build first, unify post-hoc with no arguments
# Example data:
import networkx as nx
from hiveplotlib.converters import networkx_to_nodes_edges

g = nx.karate_club_graph()
nodes, edges = networkx_to_nodes_edges(g, node_graph_metrics=["degree"])

# Call site:
from hiveplotlib import HivePlot

hp = HivePlot(
    nodes=nodes,
    edges=edges,
    partition_variable="club",
    sorting_variables="degree",
)
hp.unify_axes()
```

```python
# Example 4: post-hoc unify with an explicit vmin floor
# Example data:
import networkx as nx
from hiveplotlib.converters import networkx_to_nodes_edges

g = nx.karate_club_graph()
nodes, edges = networkx_to_nodes_edges(g, node_graph_metrics=["degree"])

# Call site:
from hiveplotlib import HivePlot

hp = HivePlot(
    nodes=nodes,
    edges=edges,
    partition_variable="club",
    sorting_variables="degree",
)
hp.unify_axes(vmin=0)
```

```python
# Example 5: unify globally, override one axis explicitly via axis_kwargs (precedence demo)
# Example data:
import networkx as nx
from hiveplotlib.converters import networkx_to_nodes_edges

g = nx.karate_club_graph()
nodes, edges = networkx_to_nodes_edges(g, node_graph_metrics=["degree"])

# Call site:
from hiveplotlib import HivePlot

# `axis_kwargs` wins for "Mr. Hi"; "Officer" follows the unified range.
# Emits a warning that the per-axis vmin displaces what unify_axes would have set.
hp = HivePlot(
    nodes=nodes,
    edges=edges,
    partition_variable="club",
    sorting_variables="degree",
    unify_axes=True,
    axis_kwargs={"Mr. Hi": {"vmin": 0}},
)
```

### API Critic's take (planning mode)

Pending — invoke api-critic in planning mode before code engineering begins.

### API Critic — post-implementation review

Pending — invoke api-critic in post-implementation mode after Workstream A ships (constructor + method on `HivePlot`).

## Workstreams

### Workstream A: shared resolver + `HivePlot` constructor + instance method

**Status:** not started
**Files:**

- `src/hiveplotlib/hiveplot_matrix.py` (move `_resolve_unify_axes` out; update its three call sites to import from the new location; tighten the loose `Dict[str, float]` on `__init__` to `Dict[Literal["vmin", "vmax"], float]`)
- New private module (code engineer's call: e.g. `src/hiveplotlib/_unify_axes.py`, or co-locate next to `BaseHivePlot`). Must not create a circular import: the helper reads `NodeCollection` (already used at the matrix-module top level) and `numpy`; it does not need to import `HivePlot` or `HivePlotMatrix`.
- `src/hiveplotlib/hiveplot.py` — add `unify_axes` to `HivePlot.__init__` signature (place it adjacent to `axis_kwargs` for discoverability), call the shared resolver after `_apply_graph_metrics` and before the `axis_kwargs` consumption loop at `hiveplot.py:2026`. Add a new `HivePlot.unify_axes(vmin=None, vmax=None)` instance method mirroring `HivePlotMatrix.unify_axes` at `hiveplot_matrix.py:832`. The post-hoc method calls `self.update_axis(axis_id=..., vmin=..., vmax=..., build_hive_plot=(i == last))` so only one rebuild fires.

**Done when:**

- `HivePlot(..., unify_axes=True)` produces a hive plot whose every axis carries the same `(vmin, vmax)` pair, computed as global min/max across whichever sorting variables the user passed (single hashable or dict of axis→variable).
- `HivePlot(..., unify_axes={"vmin": 0})` honors the explicit vmin and auto-computes vmax.
- `hp.unify_axes()` after construction unifies based on the axes' current vmin/vmax (matching HPM's post-hoc semantics — read from `axis.vmin` / `axis.vmax`, since per-axis ranges may already encode user overrides).
- `hp.unify_axes(vmin=0)` honors explicit vmin, auto-computes vmax from current axis ranges.
- Precedence holds: per-axis `axis_kwargs[ax]["vmin"|"vmax"]` (constructor) and `update_axis(vmin=, vmax=)` (post-hoc) override the unified value on that axis specifically.
- When a per-axis explicit `vmin` or `vmax` displaces what `unify_axes` would have set, a warning fires via `warnings.warn(..., stacklevel=3)` naming the displaced axis, the unified value, and the user-supplied override. Use `UserWarning` (no new exception class needed; `src/hiveplotlib/exceptions/` carries Error classes only, and the displacement is a non-fatal advisory).
- Repeat-axis vmin/vmax inheritance at `hiveplot.py:2050-2069` continues to behave: the unified range applies to non-repeat axes through `axis_kwargs`, and the existing repeat-axis logic inherits from those non-repeat axes as it does today. Verify by reading; the resolver feeds into `axis_kwargs` before the repeat-axis branch runs, so the inheritance is already in the right place.
- `HivePlotMatrix.__init__`, `HivePlotMatrix.unify_axes`, and the three `from_*` classmethods continue to behave identically; only the import source for `_resolve_unify_axes` changes.
- `ruff format` + `ruff check` + `make ty` clean.

### Workstream B: tests

**Status:** not started
**Files:**

- `tests/hiveplot_test.py` — add a `Test_HivePlotUnifyAxes` class (or equivalent) mirroring `tests/hiveplot_matrix_test.py:1557-1930`'s coverage. Each test name follows `test_<method>_<scenario>` and calls the named method in its body.

**Done when:** coverage matches the HPM suite, adjusted for `HivePlot`'s single-plot shape (no `iter_populated_cells`):

- `test_unify_axes_method_no_args_auto_computes` — `hp.unify_axes()` auto-computes from current axis ranges and applies uniformly.
- `test_unify_axes_method_explicit_vmin_vmax` — both args honored, no autocompute.
- `test_unify_axes_method_explicit_vmin_only` — explicit vmin, auto vmax.
- `test_unify_axes_method_explicit_vmax_only` — explicit vmax, auto vmin.
- `test_init_unify_axes_true` — constructor `unify_axes=True` produces uniform `(vmin, vmax)` across all axes; vmin equals global min of sorting variable, vmax equals global max.
- `test_init_unify_axes_dict_full` — `unify_axes={"vmin": V, "vmax": W}` uses the dict values uniformly.
- `test_init_unify_axes_dict_partial_vmin` / `_vmax` — partial dict uses provided value, auto-computes the other.
- `test_init_unify_axes_with_dict_sorting_variables` — `sorting_variables={ax: var, ...}` with `unify_axes=True` computes range across all referenced variables (concatenation; matches `_resolve_unify_axes` semantics today).
- `test_init_axis_kwargs_vmin_overrides_unify` — per-axis `axis_kwargs[ax]["vmin"]` wins over `unify_axes=True`; the other axes still carry the unified value.
- `test_init_unify_axes_displacement_warns` — the precedence override emits a `UserWarning` whose message names the displaced axis and the unified value (caught with `pytest.warns(UserWarning, match=...)`).
- `test_init_unify_axes_dict_vs_axis_kwargs_precedence` — `axis_kwargs` wins over the explicit `unify_axes` dict (matches HPM precedence comment at `hiveplot_matrix.py:212`).
- `test_init_unify_axes_with_repeat_axes` — `unify_axes=True` plays nicely with `repeat_axes=True`; repeat axes inherit the unified range from their non-repeat siblings.
- `test_init_unify_axes_false_is_default` — opt-out behavior unchanged from today's default.
- `test_resolve_unify_axes_import_stability` — quick smoke import from both `hiveplotlib.hiveplot_matrix` (re-export, if added for back-compat) and the new home, OR a single import test from the new home. Code engineer's call whether to re-export from `hiveplot_matrix` for downstream import stability; if not re-exported, no test required (the helper is private).
- Suite passes under `pytest -n 7`, 100% coverage maintained, no new warning escapes (`filterwarnings = error`).

### Workstream C: docs + docstrings + CHANGELOG

**Status:** not started
**Files:**

- `src/hiveplotlib/hiveplot.py` — docstring on the new `unify_axes` constructor parameter and on the new `HivePlot.unify_axes` method. Frame `unify_axes` (cross-axis convenience) and per-axis `vmin`/`vmax` (outlier clipping, known reference scales) as complementary, not alternatives. State the precedence chain explicitly: per-axis `axis_kwargs` > explicit `unify_axes` dict values > auto-computed unified range.
- `src/hiveplotlib/hiveplot_matrix.py` — tighten or refresh the `unify_axes` docstring on `__init__` only if the import move or type-annotation tightening makes the current prose drift. No prose changes required if the move is purely mechanical.
- `examples/modifying_axes.ipynb` — cells around lines 304-308 and 418-422: the "you might want to restrict the comparable axes to the same `[vmin, vmax]` range" sentence should now route the reader to `unify_axes` first, then mention raw `vmin`/`vmax` for the outlier and known-reference cases. Single-sentence touch each, not a rewrite.
- `CHANGELOG.rst` — entry under the unreleased section: "Add ``unify_axes`` constructor parameter and instance method to ``HivePlot``, mirroring ``HivePlotMatrix``."
- Gallery / tutorial notebook touch: optional, docs engineer's judgment. The existing notebook landscape covers HPM unify well; a single hive plot demo could land in `modifying_axes.ipynb` rather than a new file. Defer if it inflates scope.

**Done when:**

- `make docs` renders without new warnings.
- Updated docstrings render the precedence chain cleanly in autodoc.
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
   - **Apples-to-apples unification:** the cell computes global `min`/`max` across the sorting variable(s) and pins every axis to that range so multiple hive plots compare cleanly. Migrate to `unify_axes=True` (or `unify_axes={"vmin": V}` / `{"vmax": W}` when one side is pinned to a meaningful literal).
   - **Outlier clipping:** the cell pins one or both bounds to drop tail values that compress the rest of the axis. Leave alone.
   - **Known reference scale:** the cell pins to a domain-meaningful range (`[0, 1]` for a normalized metric, etc.) that the data may not span. Leave alone, even if it happens to be uniform across axes.
   - **Pedagogical demo of raw `vmin`/`vmax`:** the cell exists to teach the per-axis primitive. Leave alone; if anything, add a one-sentence cross-reference to `unify_axes` for the unification use case (this overlaps with Workstream C's `modifying_axes.ipynb` touch and should not be duplicated).
3. For each migrated cell, update surrounding markdown prose to match the simplified call site. No em-dashes, no AI filler, length discipline. Removed prose that explained the manual global-min/max computation goes away with the code; do not replace it with a longer explanation of `unify_axes` (the affordance is self-evident).
4. Re-run the affected notebooks end-to-end via `make test-nb` to confirm outputs still render.

**Done when:**

- Every apples-to-apples `vmin`/`vmax` site in `examples/*.ipynb` that targets a `HivePlot` is migrated to `unify_axes`, OR explicitly classified in the implementation log as "left alone: <reason>" with the reason matching one of the legitimate-use categories above.
- Surrounding prose reads naturally and does not contradict the new code.
- `make test-nb` passes.
- No overlap or contradiction with Workstream C's `modifying_axes.ipynb` touch; if both workstreams want to edit the same cell, Workstream C's framing-level change lands first and Workstream D defers to it.

**Critic pass:** invoke viz-critic on the changed notebook cells (rendered output) after the sweep lands, since the migration touches user-facing example notebooks where the visual output is the contract.

## Holdouts

- `src/hiveplotlib/hiveplot.py:2050-2069` (repeat-axis vmin/vmax inheritance): intentionally untouched. It runs after the resolver-populated `axis_kwargs` are consumed, so repeat axes inherit the unified values through the existing mechanism.
- `BaseHivePlot.place_nodes_on_axis` `vmin`/`vmax`: untouched. Base-class plumbing; `unify_axes` lives at the user-facing layer.
- `HivePlot.update_axis(vmin=, vmax=)`: untouched. Per-axis post-hoc setter remains the right tool for one-axis overrides.

## Implementation log

Append-only. After each workstream completes, one line in the same turn:

- (empty)
