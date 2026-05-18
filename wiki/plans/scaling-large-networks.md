# Plan: scaling hiveplotlib to larger networks

## Goal

Open doors for hiveplotlib users to render dramatically larger graphs (10M+ edges, OGB-arxiv-scale) without hitting peak-memory cliffs, and let users bring their own dataframe library (polars, modin, Dask) without the library forcing pandas at the input boundary. The pitch: today the datashader path builds a single concatenated curves array before rasterizing, which is the dominant memory cost at scale; refactor it to rasterize per chunk and aggregate the rasters instead, and route input-boundary dataframe ops through narwhals so non-pandas frames don't get force-converted. Workstream 1 unlocks OGB-arxiv as a stress test; Workstream 2 makes the library polite to the broader dataframe ecosystem; Workstream 3 layers Dask on top once the first two are in place. No observed RAM pain today; this is forward-looking infrastructure to unblock use cases the library cannot serve.

## Prior ADRs / design docs

No prior ADRs exist on this topic. The wiki has no `wiki/wiki/adr/` directory yet; ADR promotion has never been exercised. This plan is a strong candidate for that first promotion on close. Relevant prior thinking lives in design docs and source pages rather than ADRs:

- `wiki/wiki/sources/hiveplotlib-python.md` lines 79-84 — documents the "minimal base deps: matplotlib, numpy, pandas only" stance. The Workstream 2 decision on whether narwhals joins core deps is a deliberate deviation from this and needs explicit justification in the eventual ADR.
- `wiki/wiki/analyses/gnn-heterogeneity-hive-plots.md` line 113 — names OGB-arxiv (169k nodes, 40 classes) as the "large-scale stress test (datashader backend)" target. Workstream 1 is the unblock.
- `wiki/wiki/analyses/cora-prototype-plan.md` lines 411-434 — "Phase 5: Iterate and Scale" depends on the datashader path holding up at OGB-arxiv scale. Direct dependency on Workstream 1.
- `wiki/wiki/entities/hiveplotlib.md` lines 53-56 — "Development Priorities" does not currently list scaling. Entity page updates in the post-task research-liaison pass.
- `wiki/wiki/concepts/edge-rendering.md` lines 39-46 — "Rendering Pipeline" documents the four-stage pipeline. Workstream 1 changes the shape of step 4 for the datashader backend; the concept page gains a note on chunked aggregation but does not change overall shape.
- `wiki/wiki/concepts/applications-cybersecurity.md` lines 19-23 — DDoS-classification ML-featurization use case (Rivas 2019, Guarino 2020, Bragg 2025) is where billion-edge plots could plausibly arise.
- `wiki/wiki/sources/perez-2021-hype.md` — HyPE at 1,880 OTUs / 13,605 edges as state-of-the-art comparator. Useful framing for "several orders of magnitude beyond prior tools" when writing the eventual ADR(s).
- CHANGELOG line 238-239 documents the already-shipped float32 work in `src/hiveplotlib/hiveplot.py:1018-1055`. This plan builds on that base.

ADR promotion strategy at task close: one umbrella ADR with three numbered decisions, since the workstreams form a single architectural story (input → curves → raster). Decided over three separate ADRs because the decisions are coupled (narwhals at the boundary is what makes Dask passthrough cheap; streaming curves is what makes Dask integration tractable).

## Patterns this replaces

The replace-and-sweep audit cuts across all three workstreams. Counts come from a `grep -E '\.loc\[|\.isin\(|\.iloc\[|to_numpy\(\)|to_dict\(|\.apply\(|\.groupby\(|\.merge\(|\.values\b|pd\.concat|pd\.DataFrame|pd\.cut|pd\.qcut|pd\.Series|astype\('` sweep of `src/hiveplotlib/`: 206 occurrences across 19 files (significantly larger than the 126/17 estimate in the original brief). Not all are in scope; the audit below names the in-scope ones by workstream.

### Workstream 1 — streaming curve aggregation (datashader backend)

- **List-of-arrays accumulation followed by `np.concatenate` + `pd.DataFrame` construction**, found at `src/hiveplotlib/viz/datashader.py:181-234`. Replace with per-`(g1, g2, tag)`-chunk `cvs.line()` calls and additive raster aggregation. Drop chunk curves array after each call.
- **`pd.concat([hive_plot.axes[axis_id].node_placements for axis_id in hive_plot.axes])`** at `src/hiveplotlib/viz/datashader.py:386-388`. Replace with per-axis `cvs.points()` aggregation; concatenate rasters rather than DataFrames.
- **Same-shape accumulation patterns in `datashade_hive_plot_mpl`** at `src/hiveplotlib/viz/datashader.py:445-631` inherit the streaming refactor through its two helpers (`datashade_edges_mpl`, `datashade_nodes_mpl`); the wrapper itself does not duplicate the accumulation logic.
- **HivePlotMatrix datashader path** at `src/hiveplotlib/viz/hiveplot_matrix.py:40-59, 196-218, 438-464`. The matrix calls `datashade_hive_plot_mpl` per cell. Each cell internally rasterizes its own chunks now; per-cell streaming benefits flow up automatically. No matrix-level loop change needed, but the matrix-level peak depends on cell peak, so a memory test at the matrix level is also wanted.

### Workstream 2 — narwhals at input boundary

In-scope pandas-specific operations (those touching user-provided dataframes or doing internal frame work that polars/modin/Dask could carry):

- **`NodeCollection.__init__`** at `src/hiveplotlib/node.py:132-161`. Includes `data.copy()`, `data.columns`, `self.data.index.to_numpy()`, `self.data[col].duplicated(...)`. Replace with narwhals-wrapped operations; `NodeCollection.data` property returns whatever the user passed (or pandas when constructed from numpy).
- **`NodeCollection.check_unique_ids`** at `src/hiveplotlib/node.py:163-173`. Currently `np.unique(self.data[col].values)`. Replace with narwhals `is_duplicated` / row count.
- **`NodeCollection.create_partition_variable`** at `src/hiveplotlib/node.py:188-277`. Heavy lift: `pd.cut` and `pd.qcut` semantics need to be preserved through narwhals. Narwhals coverage of `cut` / `qcut` is partial; this is the most uncertain piece of Workstream 2. The `(-inf, c0]` left-open semantics with `[-inf, *cutoffs, inf]` will likely need an adapter that falls back to pandas inside the narwhals wrapper if the underlying frame lacks `cut`. Document the fallback path in the docstring.
- **`Edges._validate_edge_data`** at `src/hiveplotlib/edges.py:126-187`. The validation reads `.columns`, normalizes np arrays via `pd.DataFrame`, and stores the result. Replace `.columns` access and column-existence checks with narwhals; the numpy-input path can stay pandas internally (we're creating a default frame to hold the array; no user dataframe involved).
- **`Edges.add_edges`** at `src/hiveplotlib/edges.py:211-238`. The `pd.concat` is the issue. Replace with narwhals `concat`.
- **`Edges.export_edge_array`** at `src/hiveplotlib/edges.py:253-281`. Uses `.loc[:, [...]].to_numpy()` and `np.vstack`. Stay pandas-internal where convenient; the output is numpy regardless. (Narwhals `select(...).to_numpy()` is the equivalent; treat as a refactor for consistency rather than a load-bearing change.)
- **`HivePlot.add_edge_ids`** at `src/hiveplotlib/hiveplot.py:707-749`. Uses `node_placements.loc[..., unique_id_column].to_numpy()` plus `np.isin`. Replace with narwhals `is_in` if the underlying frame supports it; fall back to `to_numpy() + np.isin` otherwise.
- **The datashader metadata-extraction path** at `src/hiveplotlib/viz/datashader.py:201-222`. The `hive_plot.edges._data[tag][mask].drop(columns=[...])` and column-presence check pass through narwhals.
- **The datashader node-metadata path** at `src/hiveplotlib/viz/datashader.py:393-407`. `node_placements.columns`, `node_placements.loc[:, columns]`, and the `astype(np.float32)` calls pass through narwhals.

Out-of-scope pandas-specific operations (kept as pandas for documented reasons, listed in Holdouts):

- **`nx.to_pandas_edgelist`** in `src/hiveplotlib/converters.py:41`. Networkx output is pandas; converting through narwhals adds no value because the next thing we do with it is hand it to `Edges`. Holdout.
- **Legacy `dataframe_to_node_list`** at `src/hiveplotlib/node.py:430-450`. Independently flagged for deprecation; do not touch in this plan. Holdout (separate workstream/plan).
- **`Node.add_data`** at `src/hiveplotlib/node.py:58-73`. Dict-based, no dataframe ops. Out of scope by definition.
- **All `viz/{bokeh,plotly,holoviews,matplotlib}.py` paths** that consume pandas frames mid-pipeline. Those backends are downstream of `add_edge_ids` and `construct_curves`; once those produce numpy curves arrays, the viz backends do not see user dataframes. Out of scope.
- **`hiveplot_matrix.py` partition / variable-sweep construction**. Construction-time dataframe ops that read `NodeCollection.data` columns; narwhals coverage on the read side will fall out for free once `NodeCollection` is narwhals-backed at the input boundary. List in audit but do not separately tweak.

### Workstream 3 — Dask passthrough

Mostly net new, with one structural site that is being replaced (`__store_edge_ids` membership storage). The point of Workstream 3 is "Dask DataFrames flow through Workstreams 1 and 2 with no path-specific code in user-facing methods." If Workstream 2 routes everything through narwhals correctly, and Workstream 1's streaming aggregation produces per-chunk curves, the Dask path is mostly free. Specific spots to verify and replace:

- **`HivePlot.add_edge_ids` `np.isin`** at `src/hiveplotlib/hiveplot.py:707-749`. Narwhals `is_in` should compile to a Dask-supported `isin` on a Dask backend. Verify rather than re-implement.
- **`construct_curves`** at `src/hiveplotlib/hiveplot.py:1068+`. Curves construction reads pandas-shaped data and produces a numpy curves array. With Workstream 1's streaming, the curves array stays per-chunk, so Dask interplay is at the iteration boundary rather than inside the kernel. Verify the iteration accepts Dask `.itertuples()` / `.compute()` semantics or document the materialization point.
- **`datashade_edges_mpl` per-chunk loop** (post-Workstream 1). `cvs.line()` accepts Dask DataFrames natively; verify the chunk DataFrame produced by streaming is wrappable in a small Dask DataFrame when the input was Dask, and falls back to pandas when the input was pandas.
- **`BaseHivePlot.__store_edge_ids` membership storage** at `src/hiveplotlib/hiveplot.py:580-647`. Currently writes `edges.relevant_edges[a1][a2][tag] = indices_to_store` as a numpy bool ndarray keyed by axis-pair-tag. For Dask input, materializing a per-edge bool array into RAM defeats the out-of-core story (a 1B-edge graph forces a 1B-bool array per `(axis_pair, tag)`). Replace with a backend-aware storage that keeps the membership as a Dask Series (or a per-tag boolean column on the edge frame, or equivalent) when input is Dask, while keeping the existing numpy ndarray storage for pandas/polars input. Structural choice is an implementation decision; the constraint is no per-edge-bool materialization for Dask.

## Default justifications

Workstream-by-workstream new defaults. Workstream 1 introduces no user-facing parameters (pure internal refactor). Workstream 2 introduces no new parameters at the public surface; it changes what `data` parameters accept (`pd.DataFrame` → any narwhals-supported frame), and the return semantics of `NodeCollection.data` / `Edges.data` properties are preserved (the user gets back what they passed in). Workstream 3 introduces one new parameter, `use_dask`:

- **`use_dask: Optional[bool] = None` (recommended, post planning-mode API critic).** Default `None` means "if a Dask frame is detected, raise `TypeError` instructing the user to pass `use_dask=True` explicitly." The user reaching for `NodeCollection(data=dask_df, ...)` or `Edges(data=dask_df, ...)` is usually doing so deliberately, but the failure modes are asymmetric: a user with a small Dask frame paying overhead is a fixable annoyance (`df.compute()`), but a user whose large Dask frame silently materializes partition-by-partition inside the rasterization loop can OOM or thrash with no warning. Loud failure surfaces that mode at construction time; auto-detection hides it. Opt-in is also asymmetrically cheaper to relax later (a future minor could loosen `None` to "infer") than to tighten (a silent inference users depend on would need a deprecation cycle). `use_dask=True` opts in; `use_dask=False` with a Dask frame raises; `use_dask=True` with a pandas frame raises. The decision lives in `_validate_edge_data` and the parallel point in `NodeCollection.__init__`.
- **Alternative considered, rejected: no new parameter, infer from frame type.** Would let `NodeCollection(data=dask_df, ...)` "just work" with no extra ceremony. Rejected because the silent path is the dangerous one (OOM on the rasterization loop, no signal at construction time); the rule "loud failure where the cost of silence is high" applies here. Kept as the documented alternative for transparency; not pursued.

Worth flagging: `bezier_xy_with_nans`'s `dtype=np.float64` default (`src/hiveplotlib/utils.py:256`). Library code already passes `dtype=np.float32` explicitly at every call site (`hiveplot.py:1024, 1037, 1046, 1055`). Flipping the kernel default is cosmetic and unrelated to scaling work; called out here per the brief but explicitly not in any workstream.

### Workstream B default: narwhals as a core dependency (not an extra)

Decision: narwhals is added as a **core dependency** in `pyproject.toml`, pinned `>=1.x,<2` (specific minor pin chosen at implementation time within that bound range).

Rationale:

- Narwhals is pure-Python with no compiled extensions, no platform-specific binaries, and no threading layer. The risk profile that justified keeping numba optional (compiled wheels, segfaults across platforms, parallelism complexity) does not apply.
- Optional narwhals would require conditional code paths that CI either tests twice (matrix doubling) or doesn't test (silent rot). Core narwhals is paradoxically the lower-CI-complexity choice.
- The wiki's documented "minimal base deps: matplotlib, numpy, pandas only" stance (`wiki/wiki/sources/hiveplotlib-python.md:79-84`) is a deliberate deviation. The post-task research-liaison pass should update that source page to acknowledge narwhals as a pure-Python dispatcher in the core deps.
- A brief design note in `CLAUDE.md` documents the abstraction layer so future maintainers don't relitigate the decision.

### Workstream B default: narwhals usage pattern is pass-through, not opinionated-internal

Decision: when a user passes a polars frame, internal ops dispatch to polars; when they pass pandas, ops dispatch to pandas; when they pass Dask, ops dispatch to Dask. The library does **not** convert to a canonical internal frame type (does not always-convert-to-polars-internally regardless of input).

Rationale:

- The library's source-dataframe operations (`np.isin` axis-pair classification, boolean mask + drop, `pd.cut` / `pd.qcut`, `pd.concat`, `np.unique`) sum to roughly 10-50 ms on a 1M-edge graph in pandas vs 2-10 ms in polars. Total HivePlot construction time is 1-3 seconds, dominated by Bezier curve construction. Polars-internal would save roughly 2% of total wall time; not enough to justify the architectural complexity.
- "Always-polars internally" breaks the Dask and cuDF/GPU stories: converting Dask to polars defeats the out-of-core purpose; converting cuDF to polars pulls data off the GPU. Either you accept that breakage (excludes Workstream C entirely) or you special-case those backends, which is the complexity cost without the bottleneck win.
- Pass-through respects the user's choice of where their data lives (RAM vs out-of-core vs GPU) when the library doesn't gain enough from overriding that choice to justify the move.

Confirmation against the API usage examples: Example 2 ("polars in, polars out") and Example 3 ("pandas in, pandas out") already reflect pass-through behavior accurately; no wording change needed.

## Naming audit

### Workstream 1

No new user-facing names. Pure internal refactor of `datashade_edges_mpl`, `datashade_nodes_mpl`, `datashade_hive_plot_mpl` bodies.

### Workstream 2

No new parameter names. The `data` parameter to `NodeCollection.__init__` and `Edges.__init__` keeps its name; what changes is the accepted type annotation (was `pd.DataFrame`, becomes `IntoDataFrame` from narwhals). The docstring grows a note: "Accepts any narwhals-supported frame (pandas, polars, modin, pyarrow, Dask). Returns whatever you passed in the same library (round-trip contract); the library does not coerce frame types behind your back."

`IntoDataFrame` is narwhals's own name for this type; using it preserves user vocabulary (anyone already using narwhals or polars recognizes it; anyone using pandas-only doesn't need to know it exists because the docstring tells them "pandas works"). No friction expected.

Round-trip contract docstring sketch on both `NodeCollection.data` and `Edges.data` properties (per planning-mode api-critic Q2, must-fix):

> Returns the frame in the same library you passed in. Constructing with `pd.DataFrame` returns `pd.DataFrame`; constructing with `pl.DataFrame` returns `pl.DataFrame`; constructing with a `np.ndarray` (Edges only) returns `pd.DataFrame` (the library's default for raw-array input, since no original frame existed). The library does not coerce frame types behind your back.

The ndarray-input-becomes-pandas asymmetry is the one named exception and is documented in the docstring (not a bug, a documented carve-out).

### Workstream 3

`use_dask` is the chosen parameter name (per planning-mode api-critic Q1, must-fix: opt-in lands by default, not auto-infer). `dask` is the actual library, not a hiveplotlib-specific term; the parameter name maps to user vocabulary cleanly. Alternative names considered and rejected:

- `backend="dask"`: invites a broader "backend" concept where there isn't one. Polars users would expect `backend="polars"`, but we are routing through narwhals, not switching backends.
- `lazy=True`: muddles Dask-the-distributed-frame with Polars-LazyFrame and narwhals' `LazyFrame` wrapper. Three different "lazy" concepts collide.

Type is `Optional[bool]` with default `None`. Semantics: `None` + Dask frame → `TypeError` with a message telling the user to pass `use_dask=True`; `True` + Dask frame → opt in; `True` + non-Dask frame → `TypeError` (mismatched declaration); `False` + Dask frame → `TypeError` (explicit refusal); `False` + non-Dask frame → no-op (pandas/polars/etc. flow normally).

### Internal-only

Internal narwhals-related helper names (`_to_narwhals(df)`, `_from_narwhals(nw_df)`) are internal module/package names and out of scope for this audit per template guidance.

## API usage examples

### Proposed (from planner / Orchestrator)

```python
# Example 1: Workstream 1. User has a 10M-edge graph, wants a datashaded plot.
# Before: peak memory was O(total_edges * num_steps); a 10M-edge plot at num_steps=100
# would allocate ~800 MB float32 in a single contiguous array. After: peak memory is
# O(largest_chunk * num_steps), where chunks are per-(axis_pair, tag).
# *No code change for the user.* This is the punchline.
import networkx as nx
from hiveplotlib import HivePlot, networkx_to_nodes_edges
from hiveplotlib.viz.datashader import datashade_hive_plot_mpl

g = nx.karate_club_graph()  # in practice: ogbn-arxiv at 169k nodes, 1.16M edges
nodes, edges = networkx_to_nodes_edges(g)
hp = HivePlot(nodes=nodes, edges=edges)
# ... partition + construct_curves ...
fig, ax, im_nodes, im_edges = datashade_hive_plot_mpl(hp)


# Example 2: Workstream 2. User has a polars DataFrame; library accepts it.
import polars as pl
from hiveplotlib import Edges, NodeCollection

node_df_pl = pl.read_parquet("nodes.parquet")
edge_df_pl = pl.read_parquet("edges.parquet")

nodes = NodeCollection(data=node_df_pl, unique_id_column="id")
edges = Edges(data=edge_df_pl, from_column_name="src", to_column_name="dst")

# nodes.data is polars (round-trip preserved).
type(nodes.data)  # polars.DataFrame
type(edges.data)  # polars.DataFrame


# Example 3: Workstream 2 (mixed pandas + polars). User wants pandas in, polars out
# is not a goal; what is a goal is "polars in, polars out, no surprise pandas
# materialization mid-pipeline."
import pandas as pd
import polars as pl
from hiveplotlib import Edges, NodeCollection

# pandas-typed user keeps pandas behavior, no migration needed.
nodes_pd = NodeCollection(data=pd.read_csv("nodes.csv"), unique_id_column="id")
type(nodes_pd.data)  # pandas.DataFrame


# Example 4: Workstream 3. User has a Dask DataFrame, opts in explicitly.
# `use_dask=True` is required; without it, the constructor raises TypeError
# (loud-failure default, since silent partition-by-partition materialization
# inside the rasterization loop can OOM on large Dask frames with no warning).
import dask.dataframe as dd
from hiveplotlib import Edges, NodeCollection

edge_ddf = dd.read_parquet("huge_edges/*.parquet")  # 100M edges
node_ddf = dd.read_parquet("huge_nodes/*.parquet")

nodes = NodeCollection(data=node_ddf, unique_id_column="id", use_dask=True)
edges = Edges(data=edge_ddf, from_column_name="src", to_column_name="dst",
              use_dask=True)

hp = HivePlot(nodes=nodes, edges=edges)
# ... rest of the pipeline runs Dask-aware ...


# Example 4b: What forgetting `use_dask=True` looks like.
# >>> NodeCollection(data=node_ddf, unique_id_column="id")
# TypeError: Detected a Dask DataFrame but `use_dask` is not set.
#     Pass `use_dask=True` to opt in to the Dask passthrough. Note that
#     hiveplotlib will materialize partitions one-by-one inside the
#     rasterization loop; ensure each partition fits in memory.
```

### API Critic's take (planning mode)

Reviewed Examples 1-4 as a user typing the class names and hitting tab against polars and Dask frames. Three orchestrator questions answered, then per-example friction, then the cross-cutting pattern.

**Q1 — "Infer from frame type" vs. opt-in for Dask (Workstream C):** [must-fix]

Recommend opt-in via `use_dask: Optional[bool] = None` with the default `None` meaning "raise if a Dask frame is passed without an explicit opt-in." Reasoning:

- The two failure modes are asymmetric. A user with a small Dask frame for dev silently paying Dask overhead is a wasted-cycles annoyance (fixable by `df.compute()`). A user with a large Dask frame who didn't realize it was Dask, hitting `cvs.line()` on partitions that materialize one-by-one into pandas inside narwhals... can OOM on a single partition or, worse, succeed in 100x the wallclock with chunked I/O thrash and no warning. The OOM/wallclock mode is the silent one; surfacing it explicitly is the safer default.
- The plan's own "Naming audit" already notes `use_dask=True` with a pandas frame should raise `TypeError`, which means the parameter is already in the design space. The choice is whether the *default* is "infer" or "explicit." Explicit is cheaper to walk back later (loosen `None` to mean "infer" in a future minor) than the reverse (silent inference users came to depend on, now you'd ship a deprecation).
- Workstream C's done-when says "no path-specific code in user-facing methods" — that goal is still met with an opt-in: the parameter lives at `NodeCollection.__init__` and `Edges.__init__`, the rest of the pipeline is unchanged. Opt-in is a one-line user cost (`use_dask=True`) for a much louder safety net.
- The plan's Naming audit rejects `backend=...` and `lazy=True` for good reasons; `use_dask` is the right name. Reject the "don't add a parameter" recommendation in favor of the fallback (auto-detect via `None`) the plan already names.

Concrete recommendation: `use_dask: Optional[bool] = None`, with `None` meaning "if a Dask frame is detected, raise `TypeError` instructing the user to pass `use_dask=True` explicitly." This is the loud-failure default; users who want auto-detect can pass `use_dask=True` once they've understood the cost.

**Q2 — `.data` round-trip guarantee, documented or not:** [must-fix, document it]

Document it. Once `NodeCollection(data=polars_df).data` returns polars (per the plan), users will write `nc.data.filter(pl.col("x") > 0)` and ship it. If a future refactor switches `.data` to "always pandas because internal storage simplified," every downstream polars user breaks silently. The right time to commit to a contract is when the surface is born; the right place is the docstring on `.data` and `.__init__`.

Suggested docstring sketch for both `NodeCollection.data` and `Edges.data`:

> Returns the frame in the same library you passed in. Constructing with `pd.DataFrame` returns `pd.DataFrame`; constructing with `pl.DataFrame` returns `pl.DataFrame`; constructing with a `np.ndarray` (Edges only) returns `pd.DataFrame` (the library's default for raw-array input). The library does not coerce frame types behind your back.

The np.ndarray-input-becomes-pandas asymmetry is real and worth naming in the docstring rather than hoping nobody notices. It is a documented exception, not a bug.

**Q3 — Cross-example pattern:** [worth-discussing]

Yes, two patterns:

- *Frame-library introspection.* Across Examples 2-4 the user immediately wants to confirm "what did the library actually store?" — Example 2 does `type(nodes.data)`, Example 3 does the same. A `NodeCollection.frame_library` property returning a string like `"pandas"` / `"polars"` / `"dask"` is more durable than `type(...)` and cheaper to document. Not blocking; a quality-of-life add that should be considered alongside Workstream B.
- *Convenience converter.* A `to_pandas()` method on `NodeCollection` and `Edges` would help users who built a `HivePlot` from polars but want to drop into pandas for a one-off `.describe()` or for an integration with a pandas-only downstream tool. Cheap to add (`nw.from_native(self.data).to_pandas()`), and signals "you can always escape to pandas if you need to" which is a comforting safety property for the narwhals adoption. Strictly optional; flagging here so the code-engineer can add it in passing rather than as a follow-up.

Recommend adding `frame_library` as a property in Workstream B; defer `to_pandas()` to a worth-discussing follow-up unless code-engineer finds it cheap.

**Per-example walkthrough:**

- *Example 1 (Workstream A):* Agreed. The "no code change for the user" punchline is the right framing; this example is purely a memory-characteristics note in the docstring. The note in `datashade_hive_plot_mpl`'s docstring should name the per-chunk shape concretely (`O(largest_chunk * num_steps)` is the right magnitude).

- *Example 2 (Workstream B, polars in/out):* Mostly agreed. One friction: `unique_id_column="id"` and `from_column_name="src"`/`to_column_name="dst"` are required kwargs the user is expected to know. That is unchanged from today's pandas surface and out of scope here, but the docstring should make clear that polars column-name semantics are identical (no `.` vs `_` translation, no case folding). Suggested addition to the planner's example: a single line `# polars column names round-trip; no transformation` before the constructors. [nit]

- *Example 3 (mixed pandas + polars):* Agreed. The example correctly shows pandas users see no change. The Q2 docstring fix above covers the "is it always pandas or sometimes polars" confusion this example might raise.

- *Example 4 (Workstream C, Dask):* Per Q1 above, this example must change. Suggested rewrite:

  ```python
  # Workstream C. User has a Dask DataFrame, opts in explicitly.
  import dask.dataframe as dd
  from hiveplotlib import Edges, NodeCollection

  edge_ddf = dd.read_parquet("huge_edges/*.parquet")  # 100M edges
  node_ddf = dd.read_parquet("huge_nodes/*.parquet")

  # Explicit opt-in. Without `use_dask=True`, the constructor raises
  # TypeError to surface "this will materialize partitions one-by-one;
  # are you sure?" Default is loud-failure rather than silent inference.
  nodes = NodeCollection(data=node_ddf, unique_id_column="id", use_dask=True)
  edges = Edges(data=edge_ddf, from_column_name="src", to_column_name="dst",
                use_dask=True)

  hp = HivePlot(nodes=nodes, edges=edges)
  # ... rest of the pipeline runs Dask-aware ...
  ```

  The example should also include a "what error do I see if I forget?" snippet:

  ```python
  # Forgetting `use_dask=True` raises early:
  # >>> NodeCollection(data=node_ddf, unique_id_column="id")
  # TypeError: Detected a Dask DataFrame but `use_dask` is not set.
  #     Pass `use_dask=True` to opt in to the Dask passthrough. Note that
  #     hiveplotlib will materialize partitions one-by-one inside the
  #     rasterization loop; ensure each partition fits in memory.
  ```

  This is the loud failure mode the orchestrator's Q1 was asking about. If `use_dask=True` is passed but Dask is not installed, the error should name the missing extra (`pip install hiveplotlib[dask]` or whatever the chosen install marker is).

**Other concerns:**

- *Partially set-up Dask (`dask.delayed` vs `dask.DataFrame`):* The plan does not name what happens if a user passes a `dask.delayed` or a Dask `Bag` or a `dask.array`. Recommend: narwhals's `from_native` will raise on non-DataFrame Dask objects; surface that error with a clarifying message ("hiveplotlib accepts `dask.dataframe.DataFrame`; got `dask.delayed.Delayed`. Materialize first with `.compute()` or wrap in `dd.from_delayed(...)`."). [should-fix in Workstream C]

- *`create_partition_variable` and `cut`/`qcut` fallback:* Plan correctly flags this as the most uncertain piece. From an API perspective: if the fallback materializes a polars frame through pandas internally, the *returned* partition column should still land in the user's original frame type. A polars user calling `create_partition_variable` expects the resulting `nc.data` to remain polars; if the internal fallback silently makes it pandas, that breaks the Q2 round-trip contract. Add to Workstream B done-when: "if `create_partition_variable` falls back to pandas for `cut`/`qcut`, the result is converted back to the user's original frame type before being assigned to `self.data`." [must-fix in Workstream B]

- *Variable-naming consistency at the boundary:* Today `NodeCollection(data=...)` and `Edges(data=...)` both use `data`. Good. The narwhals docstring note should use the same word ("frame") consistently across both; the plan says "any narwhals-supported frame" in one place and "Returns whatever you passed" in another — both fine, just make sure both classes' docstrings say it the same way. [nit]

- *The `.data` getter on `Edges` already does dict-vs-singleton dispatch.* Worth verifying the round-trip contract holds across the multi-tag dict case too: a user passing `data={"tag_a": polars_df_a, "tag_b": polars_df_b}` should get back a dict-of-polars, not a dict-of-pandas. Add a multi-tag polars test case to Workstream B's test matrix. [should-fix, test coverage]

**Recurring pattern summary:**

The strongest cross-cutting concern is documenting the round-trip contract (Q2). Without it, the API silently grows a "polars in, pandas out under some conditions" mode that users will discover only when they hit it. The `use_dask` opt-in (Q1) is the second-strongest; opt-in is asymmetrically cheaper to relax later than to tighten. The `frame_library` property (Q3) is a nice-to-have. The `create_partition_variable` round-trip preservation (under Other concerns) is must-fix and ties Q2 to a concrete done-when bullet.

### API Critic — post-implementation review

```
Pending — invoke api-critic in post-implementation mode after Workstream 2
ships (because Workstream 2 changes accepted types on `NodeCollection.__init__`
and `Edges.__init__`).

Pending — invoke api-critic in post-implementation mode after Workstream 3
ships (because Workstream 3 changes user-visible behavior for Dask input,
even if no new parameter exists).

No post-impl critic invocation needed for Workstream 1; pure internal refactor
with no public surface change.
```

## Workstreams

### Workstream A: Streaming curve construction + per-chunk rasterization (datashader backend)

**Status:** not started
**Files:**
- `src/hiveplotlib/viz/datashader.py` (primary; refactor `datashade_edges_mpl`, `datashade_nodes_mpl`, verify `datashade_hive_plot_mpl` benefits flow through)
- `tests/viz/datashader_test.py` (or equivalent path; add memory-bound tests, gate by available RAM)
- `runners/performance/profile_hiveplot_ds.py` (extend `--edges-list` to validate at 10M+; add peak-memory measurement)
- `CHANGELOG.rst` (entry for the refactor)
- `docs/source/...` autodoc for the three datashader functions (no signature change, but docstring grows a note on memory characteristics)
- `wiki/wiki/concepts/edge-rendering.md` lines 39-46 ("Rendering Pipeline" gains note on chunked aggregation; post-task research-liaison pass)

**Done when:**
- Each `(g1, g2, tag)` chunk in `datashade_edges_mpl` calls `cvs.line()` separately and the resulting raster is added into a running aggregate; the chunk's curves array is dropped before the next chunk.
- Same shape for `datashade_nodes_mpl` over axes.
- All existing datashader tests pass with no regressions (visual or numeric).
- A new memory-bound test (skipped by default if RAM unavailable) confirms peak RSS for a synthetic 10M-edge graph stays under O(largest_chunk × num_steps × 8 bytes) rather than O(total_edges × num_steps × 8 bytes).
- Performance runner sweep extends to 10M edges and the resulting CSV is checked in (or written to a known artifact path).
- CHANGELOG entry added.
- Docstring note explaining chunked aggregation added to the three datashader entry points.

### Workstream B: Narwhals at the input boundary

**Status:** not started
**Files:**
- `pyproject.toml` (add narwhals as core dependency, pinned `>=1.x,<2`)
- `CLAUDE.md` (brief design note on the narwhals abstraction layer)
- `src/hiveplotlib/node.py` (`NodeCollection.__init__`, `check_unique_ids`, `create_partition_variable`)
- `src/hiveplotlib/edges.py` (`__init__`, `_validate_edge_data`, `add_edges`, `export_edge_array`)
- `src/hiveplotlib/hiveplot.py:707-749` (`add_edge_ids` `np.isin`-style filter)
- `src/hiveplotlib/viz/datashader.py:201-222, 393-407` (metadata extraction and node-placements `pd.concat`; narwhals refactor parallel to Workstream A)
- `tests/node_test.py`, `tests/edges_test.py`, `tests/hiveplot_test.py` (extend with polars-backend cases for input-boundary methods)
- `CHANGELOG.rst` (entry naming the polars/narwhals support and the deps decision)
- `docs/source/...` autodoc updates for `NodeCollection`, `Edges` (note narwhals-supported frame acceptance, "returns whatever you passed")
- `wiki/wiki/entities/hiveplotlib.md` (post-task research-liaison pass; "Development Priorities" gains scaling)
- `wiki/wiki/sources/hiveplotlib-python.md` lines 79-84 (post-task; "minimal base deps" note expanded to acknowledge narwhals as a pure-Python dispatcher)

**Done when:**
- Narwhals is declared as a **core dependency** in `pyproject.toml`, pinned `>=1.x,<2` (specific minor within that range chosen at implementation time). A brief design note is added to `CLAUDE.md` explaining the abstraction layer (so future maintainers don't relitigate the core-dep decision).
- `NodeCollection(data=polars_df, ...)` and `Edges(data=polars_df, ...)` work end-to-end; `.data` round-trips to polars.
- `NodeCollection(data=pandas_df, ...)` and `Edges(data=pandas_df, ...)` continue to work identically to today; no regression.
- Docstrings on `NodeCollection.data` and `Edges.data` properties explicitly document the round-trip contract: the property returns the frame in the same library the user passed in, the one named exception is numpy-ndarray input to `Edges` (which becomes pandas because no original frame existed), and the library does not coerce frame types. Wording aligns across both classes.
- Round-trip contract is enforced by tests: for each accepted frame library, construct, then assert `type(nc.data)` / `type(edges.data)` matches the input library; assert the numpy-ndarray-to-pandas exception on `Edges`; assert the contract holds across `Edges`'s dict-vs-singleton dispatch (multi-tag dict-of-polars round-trips to dict-of-polars, not dict-of-pandas).
- `create_partition_variable` works with polars input; if `cut` / `qcut` semantics force a pandas fallback inside the narwhals wrapper, the resulting partition column is converted back to the user's original frame type before assignment to `self.data` (a polars `NodeCollection` stays polars after the call). A test verifies the post-call `nc.data` is the same frame type as the pre-call frame.
- `add_edge_ids` works with polars-backed `NodeCollection`.
- `datashade_edges_mpl` and `datashade_nodes_mpl` accept hive plots built from polars input (the metadata extraction path goes through narwhals).
- Test matrix gains polars cases for at least: `NodeCollection.__init__`, `check_unique_ids`, `create_partition_variable`, `Edges.__init__`, `Edges.add_edges`, `HivePlot.add_edge_ids`, `datashade_edges_mpl` metadata-column reductions. Coverage stays 100%.
- CI shape: single `pytest` invocation. A new `@pytest.mark.polars` marker is added following the existing optional-backend marker pattern (`bokeh`, `datashader`, `holoviews`, `plotly`, `networkx`). Polars-parametrized cases use per-case marks via `pytest.param("polars", marks=pytest.mark.polars)`. Marker isolation works correctly: a polars-marked case does not run under `pytest -m datashader`.
- All warnings-as-errors still pass (narwhals does not introduce a deprecation warning at the version pinned).
- CHANGELOG entry.
- Docstring updates on `NodeCollection.__init__`, `Edges.__init__`, the `data` properties.
- API critic post-impl review section filled in this plan.

### Workstream C: Optional Dask passthrough

**Status:** not started
**Files:**
- `src/hiveplotlib/hiveplot.py:707-749, 1068+` (verify `add_edge_ids` `is_in` and `construct_curves` chunk iteration work on Dask-backed input)
- `src/hiveplotlib/hiveplot.py:580-647` (`BaseHivePlot.__store_edge_ids`; writes `edges.relevant_edges[a1][a2][tag] = indices_to_store`. The storage structure must become backend-aware so Dask input does not materialize a per-edge bool ndarray into RAM.)
- `src/hiveplotlib/viz/datashader.py` (verify the per-chunk DataFrame fed to `cvs.line()` can be Dask-typed when the input was Dask)
- `tests/integration/dask_passthrough_test.py` (new; in-memory partitioned synthetic Dask DataFrame; small enough to live in CI; gated by the datashader extra plus Dask)
- `CHANGELOG.rst` (entry)
- `docs/source/...` (one new example or notebook section demonstrating Dask passthrough, including partition-size guidance for the per-partition Bezier memory ceiling and the sort-cost / shuffle disposition for `place_nodes_on_axis`; could also live in `examples/scaling_with_dask.ipynb` as a gallery-style notebook)

**Done when:**
- `NodeCollection.__init__` and `Edges.__init__` accept `use_dask: Optional[bool] = None`. The parameter is required for Dask input to flow through; the default `None` raises rather than auto-inferring.
- Passing a Dask DataFrame with `use_dask` left as `None` raises `TypeError` at construction time. The error message matches (modulo line wrapping): `"Detected a Dask DataFrame but `use_dask` is not set. Pass `use_dask=True` to opt in to the Dask passthrough. Note that hiveplotlib will materialize partitions one-by-one inside the rasterization loop; ensure each partition fits in memory."` A test pins the message.
- Passing `use_dask=True` with a non-Dask frame raises `TypeError` ("mismatched declaration" path); passing `use_dask=False` with a Dask frame raises `TypeError` (explicit refusal path); passing `use_dask=False` (or `None`) with a non-Dask frame is a no-op. Tests cover each branch.
- `use_dask=True` with Dask installed: the constructor succeeds, and a small Dask DataFrame end-to-end test (in-memory, ~2 partitions, synthetic 10k edges) produces an equivalent rasterization to the pandas path.
- `use_dask=True` with Dask not installed raises `ImportError` naming the missing extra (`pip install hiveplotlib[dask]` or whatever the install marker is named).
- CI shape: single `pytest` invocation. A new `@pytest.mark.dask` marker is added following the existing optional-backend marker pattern (`bokeh`, `datashader`, `holoviews`, `plotly`, `networkx`, `polars`). Dask-gated tests carry the marker so they only run under `pytest -m dask` (or the umbrella `make test`). Marker isolation works correctly: a Dask-marked case does not run under `pytest -m datashader`.
- Performance runner gains a small Dask-input scenario (or it is left as a worth-discussing follow-up by the api-critic).
- The `relevant_edges` storage at `BaseHivePlot.__store_edge_ids` (`src/hiveplotlib/hiveplot.py:580-647`) is backend-aware: when input is Dask, the per-`(axis_pair, tag)` membership is **not** materialized as a per-edge bool numpy ndarray into RAM (which would silently defeat the out-of-core story for a 1B-edge graph). Implementation chooses between (a) storing the membership as a Dask Series, (b) adding a per-tag boolean column to the edge frame (kept as Dask), or (c) an equivalent structure that avoids materializing a per-edge bool array into RAM. The pandas case keeps the existing numpy ndarray storage. The choice between the three structural options is an implementation decision; the requirement is the no-materialization guarantee for Dask input.
- The Dask sort cost is addressed in the Dask example notebook (or equivalent doc). `HivePlot.place_nodes_on_axis` sorts nodes by a variable, which in Dask requires a shuffle (the most expensive Dask operation). Implementer chooses between (a) accept the cost (the Dask sort happens, document expected wall-time impact) or (b) document a "sort upstream before passing to hiveplotlib" pattern in the notebook. Either disposition is acceptable; the requirement is that the question is answered visibly to users, not left as a silent footgun.
- The Dask example notebook (or the Dask passthrough docs) documents partition-size guidance. The Bezier kernel materializes one Dask partition's curves into numpy at a time; the per-partition memory ceiling is roughly `largest_partition_size * num_steps * 4 bytes` (float32). A user with few-but-huge partitions can still exceed RAM, defeating the streaming win. Documentation names a concrete repartition guideline (e.g., "for very large edge sets, repartition to ensure no partition exceeds ~10M edges before passing to hiveplotlib"; the exact recommended ceiling is implementer's call based on benchmarks).
- Documentation example or notebook demonstrating Dask usage with the explicit `use_dask=True` opt-in.
- CHANGELOG entry naming the new parameter and the loud-failure default.
- API critic post-impl review section filled.

**Dependency:** Workstream C requires both A and B to land first. A produces per-chunk curves arrays that the Dask path consumes; B routes input dataframes through narwhals so Dask flows through validation/filtering without bespoke code paths.

## Plan amendments

### In-scope tweak: Workstream C ships `use_dask: Optional[bool] = None` with default = raise-on-Dask

**Date:** 2026-05-17
**Trigger:** Planning-mode api-critic Q1 (must-fix). Quoted in the "API Critic's take (planning mode)" subsection: "Recommend opt-in via `use_dask: Optional[bool] = None` with the default `None` meaning 'raise if a Dask frame is passed without an explicit opt-in.'"
**Workstream affected:** Workstream C (Optional Dask passthrough), with cascading edits to Default justifications, Naming audit (Workstream 3 subsection), and API usage examples (Example 4).
**Change:**
- Default justifications: the recommended path flips from "no new parameter, infer from frame type" to "`use_dask: Optional[bool] = None` with `None` raising on Dask input." The "no new parameter" path is now the documented alternative considered. Rationale: failure modes are asymmetric (silent OOM/thrash on partition-by-partition materialization is the dangerous mode; opt-in surfaces it loudly at construction), and opt-in is asymmetrically cheaper to relax later than to tighten.
- Naming audit (Workstream 3): `use_dask` is now in scope, not a fallback. `Optional[bool]` with `None` default. Branch semantics for `None` / `True` / `False` × Dask / non-Dask are spelled out.
- Workstream C "Done when": new bullets requiring the parameter, the raise-on-default behavior with pinned error message text, all four `(use_dask × frame-type)` branches tested, the `ImportError` path when Dask is not installed, and the documentation example showing the explicit opt-in.
- API usage examples: Example 4 rewritten to show `use_dask=True` invocation; new Example 4b shows the `TypeError` users see when they forget the opt-in.

### In-scope tweak: `.data` round-trip is documented contract, enforced by tests

**Date:** 2026-05-17
**Trigger:** Planning-mode api-critic Q2 (must-fix). Quoted: "Once `NodeCollection(data=polars_df).data` returns polars (per the plan), users will write `nc.data.filter(pl.col('x') > 0)` and ship it. If a future refactor switches `.data` to 'always pandas because internal storage simplified,' every downstream polars user breaks silently. The right time to commit to a contract is when the surface is born; the right place is the docstring on `.data` and `.__init__`."
**Workstream affected:** Workstream B (Narwhals at the input boundary), with a cascading edit to the Naming audit (Workstream 2 subsection).
**Change:**
- Naming audit (Workstream 2): the round-trip contract docstring sketch is now part of the planned docstring text on `NodeCollection.data` and `Edges.data`, including the one named exception (numpy-ndarray input to `Edges` becomes pandas because no original frame existed). Wording aligns across both classes.
- Workstream B "Done when": new bullet requiring the docstring text on both properties to document the round-trip contract; new bullet requiring tests that enforce the contract for each accepted frame library (pandas in → pandas out, polars in → polars out), assert the documented numpy-to-pandas exception, and assert the contract holds across `Edges`'s dict-vs-singleton dispatch (multi-tag dict-of-polars round-trips to dict-of-polars).

### In-scope tweak: `create_partition_variable` pandas fallback restores frame type before assignment

**Date:** 2026-05-17
**Trigger:** Planning-mode api-critic "Other concerns" (must-fix). Quoted: "If `create_partition_variable` falls back to pandas for `cut`/`qcut`, the result is converted back to the user's original frame type before being assigned to `self.data`."
**Workstream affected:** Workstream B (Narwhals at the input boundary).
**Change:**
- Workstream B "Done when": new bullet requiring that when `create_partition_variable` falls back to pandas for `cut`/`qcut` semantics, the resulting partition column is converted back to the user's original frame type before being assigned to `self.data`. A test verifies that the post-call `nc.data` is the same frame type as the pre-call frame (polars in → polars out across a `create_partition_variable` call). Closes the silent-conversion hole that would otherwise undercut the round-trip contract above.

### Deferred follow-up: `frame_library` property on both `NodeCollection` and `Edges`

**Date:** 2026-05-17 (initial); updated 2026-05-17 (maintainer follow-up: symmetric mirroring to `Edges`)
**Trigger:** Planning-mode api-critic Q3 (worth-discussing). Quoted: "A `NodeCollection.frame_library` property returning a string like `'pandas'` / `'polars'` / `'dask'` is more durable than `type(...)` and cheaper to document. Not blocking; a quality-of-life add that should be considered alongside Workstream B." Subsequent maintainer note: the round-trip contract applies symmetrically to both `NodeCollection` and `Edges`, so the property should mirror to both classes; implementing only on `NodeCollection` would leave an asymmetric ergonomic gap.
**Target:** Maintainer call. If cheap to add during Workstream B execution, the code-engineer implements a `frame_library` property on **both** `NodeCollection` **and** `Edges`; otherwise defer to a follow-up plan. Not promoted to a done-when bullet because the round-trip contract above already gives users the durable behavior the property names; the property is documentation-as-API for ergonomic convenience rather than a load-bearing piece of the surface.
**Rationale:** Quality-of-life on top of the round-trip contract; not part of the load-bearing surface. Symmetric on both classes because the round-trip contract is symmetric. Surfaced here so the code-engineer sees it before starting Workstream B and can fold it in opportunistically across both classes in one pass (or leave it for a future plan with explicit done-when bullets).

### Deferred follow-up: `to_pandas()` convenience method on `NodeCollection` and `Edges`

**Date:** 2026-05-17
**Trigger:** Planning-mode api-critic Q3 (worth-discussing). Quoted: "A `to_pandas()` method on `NodeCollection` and `Edges` would help users who built a `HivePlot` from polars but want to drop into pandas for a one-off `.describe()` or for an integration with a pandas-only downstream tool. Cheap to add (`nw.from_native(self.data).to_pandas()`), and signals 'you can always escape to pandas if you need to' which is a comforting safety property for the narwhals adoption."
**Target:** Maintainer call. If cheap during Workstream B, code-engineer adds it; otherwise defer. Same disposition as `frame_library` above.
**Rationale:** Adds a one-line escape hatch, doesn't change any load-bearing surface, but does grow the public method count by one per class. Worth a maintainer eyeball before promotion to done-when, since taste on "library should not grow methods that wrap existing one-liners" varies.

### Deferred follow-up: multi-tag polars dict test coverage explicitly enumerated in Workstream B

**Date:** 2026-05-17
**Trigger:** Planning-mode api-critic "Other concerns" (should-fix, test coverage). Quoted: "The `.data` getter on `Edges` already does dict-vs-singleton dispatch. Worth verifying the round-trip contract holds across the multi-tag dict case too: a user passing `data={'tag_a': polars_df_a, 'tag_b': polars_df_b}` should get back a dict-of-polars, not a dict-of-pandas. Add a multi-tag polars test case to Workstream B's test matrix."
**Target:** Already partially covered by the round-trip contract enforcement bullet in the in-scope tweak above (which explicitly names the dict-vs-singleton dispatch case). Surfaced here as a follow-up flag so the test-engineer treats the multi-tag polars dict case as a named scenario, not a hidden assumption.
**Rationale:** The done-when bullet now names this case, so it is in-scope rather than deferred; this entry exists so the trigger and provenance are visible alongside the other amendments.

### Deferred follow-up: clearer error for partially set-up Dask objects (`dask.delayed`, `dask.array`)

**Date:** 2026-05-17 (initial); updated 2026-05-17 (maintainer disposition: parked indefinitely)
**Trigger:** Planning-mode api-critic "Other concerns" (should-fix). Quoted: "The plan does not name what happens if a user passes a `dask.delayed` or a Dask `Bag` or a `dask.array`. Recommend: narwhals's `from_native` will raise on non-DataFrame Dask objects; surface that error with a clarifying message ('hiveplotlib accepts `dask.dataframe.DataFrame`; got `dask.delayed.Delayed`. Materialize first with `.compute()` or wrap in `dd.from_delayed(...)`.')."
**Target:** **Parked indefinitely (post-Workstream C reconsideration; likely notebook documentation rather than code change).** Revisit after Workstream C ships. Maintainer disposition: the underlying narwhals raise already prevents silent misuse, and any user-facing improvement is likely better documented in the eventual Dask example notebook than added as a custom error path in code.
**Rationale:** The `TypeError` raise from narwhals's `from_native` already prevents silent misuse, so safety is in place. Adding a custom error-message wrapper inside code grows the surface for negligible ergonomic gain; the same guidance ("hiveplotlib accepts `dask.dataframe.DataFrame`; materialize first with `.compute()` or wrap with `dd.from_delayed(...)`") sits more naturally in the Dask example notebook's prose, where it can be expanded with worked examples. Revisit only if post-Workstream C usage shows users hitting this path frequently and the notebook is insufficient.

### Design clarification: `frame_library` property mirrors to both `NodeCollection` and `Edges`

**Date:** 2026-05-17
**Trigger:** Maintainer follow-up on the existing "Deferred follow-up: `frame_library` property" entry above. Paraphrased from the conversation: "The deferred entry currently reads as if `frame_library` applies only to `NodeCollection`; it should mirror to `Edges` as well, since the round-trip contract applies symmetrically to both classes."
**Workstream affected:** Workstream B (Narwhals at the input boundary). Edits the existing deferred follow-up entry above to make the symmetry explicit.
**Change:**
- The deferred follow-up entry's title and body now read "a `frame_library` property on both `NodeCollection` and `Edges`," with a note on the symmetric round-trip contract motivating the symmetric API.
- No done-when change; the entry remains a deferred follow-up rather than a load-bearing requirement. The clarification ensures that if the code-engineer picks it up opportunistically during Workstream B, they implement both at once rather than leaving an asymmetric ergonomic gap.

### Deferred follow-up update: `dask.delayed` / `dask.array` clearer-error item parked indefinitely

**Date:** 2026-05-17
**Trigger:** Maintainer disposition on the existing "Deferred follow-up: clearer error for partially set-up Dask objects" entry above. Paraphrased from the conversation: "Slide the dask.delayed / dask.array clearer-error item from 'maintainer call, cheap fold-in' to 'parked indefinitely; revisit after Workstream C ships.' Rationale: the underlying narwhals raise already prevents silent misuse, and any user-facing improvement is likely better documented in the eventual Dask example notebook than added as a custom error path in code."
**Workstream affected:** Workstream C (Optional Dask passthrough). Edits the existing deferred follow-up entry above to update its status.
**Change:**
- The deferred follow-up entry's target status now reads "parked indefinitely (post-Workstream C reconsideration; likely notebook documentation rather than code change)" rather than the prior "maintainer call, cheap fold-in" framing.
- Rationale spelled out in the entry: safety is already in place via the narwhals `from_native` raise; the value-add is message polish, which sits more naturally in the Dask example notebook's prose than in a code-level error wrapper. Revisit only if post-Workstream C usage shows users hitting this path frequently and the notebook is insufficient.

### In-scope tweak: narwhals is a core dependency, pinned `>=1.x,<2`

**Date:** 2026-05-17
**Trigger:** Maintainer follow-up resolution of the open question Workstream B left for the api-critic + maintainer review ("Narwhals as core dep vs. extra"). Maintainer ask, paraphrased from the follow-up conversation: "Narwhals as a core dependency, pinned `>=1.x,<2`; the optional-extra path is rejected."
**Workstream affected:** Workstream B (Narwhals at the input boundary), with cascading edits to Default justifications and Workstream B Files.
**Change:**
- Default justifications: new "narwhals as a core dependency (not an extra)" section captures the rationale (narwhals is pure-Python with no compiled extensions, no platform-specific binaries, no threading layer; the risk profile that justifies keeping numba optional does not apply; optional narwhals would force CI matrix doubling or silent rot; and the wiki's documented "minimal base deps" stance is a deliberate deviation that the post-task research-liaison pass should reflect into `wiki/wiki/sources/hiveplotlib-python.md:79-84`).
- Workstream B Files: `pyproject.toml` entry rewritten as "add narwhals as core dependency, pinned `>=1.x,<2`"; new `CLAUDE.md` entry for the brief design note on the abstraction layer (kept so future maintainers do not relitigate the decision).
- Workstream B "Done when": the bullet that previously deferred the decision now reads "Narwhals is declared as a **core dependency** in `pyproject.toml`, pinned `>=1.x,<2`. A brief design note is added to `CLAUDE.md` explaining the abstraction layer."

### Design clarification: narwhals usage pattern is pass-through, not opinionated-internal

**Date:** 2026-05-17
**Trigger:** Maintainer follow-up clarification. Paraphrased from the conversation: "Make the narwhals usage pattern explicit: pass-through, not opinionated-internal. When a user passes polars, dispatch to polars; when they pass pandas, dispatch to pandas; when they pass Dask, dispatch to Dask. We do not convert to a canonical internal frame type."
**Workstream affected:** Workstream B (Narwhals at the input boundary), with a cascading edit to Default justifications. No example change required.
**Change:**
- Default justifications: new "narwhals usage pattern is pass-through, not opinionated-internal" section captures the rationale. Bottleneck math: source-dataframe operations sum to ~10-50 ms on a 1M-edge graph in pandas vs ~2-10 ms in polars; total HivePlot construction is 1-3 seconds dominated by Bezier curves; polars-internal saves roughly 2% of total wall time, not enough to justify the architectural complexity. "Always-polars internally" also breaks the Dask and cuDF/GPU stories (converting Dask to polars defeats out-of-core; converting cuDF to polars pulls data off the GPU); special-casing those backends is the complexity cost without the bottleneck win. Pass-through respects where the user's data lives.
- API usage examples: Examples 2 ("polars in, polars out") and 3 ("pandas in, pandas out") already reflect pass-through behavior accurately; confirmation note added inside the new Default justifications subsection. No rewording needed.

### Design clarification (Workstream C): `relevant_edges` storage must be backend-aware

**Date:** 2026-05-17
**Trigger:** Maintainer follow-up design note. Paraphrased from the conversation: "The `relevant_edges` storage at `__store_edge_ids` (`dict[a1][a2][tag] = numpy bool ndarray`) needs to be backend-aware. For Dask, materializing a 1B-bool numpy array per (axis_pair, tag) silently defeats the out-of-core story; the Workstream C implementation must avoid that materialization."
**Workstream affected:** Workstream C (Optional Dask passthrough), with cascading edits to Patterns this replaces (Workstream 3 subsection) and Workstream C Files.
**Change:**
- Patterns this replaces (Workstream 3): `BaseHivePlot.__store_edge_ids` at `src/hiveplotlib/hiveplot.py:580-647` added to the replace-list. Today writes `edges.relevant_edges[a1][a2][tag] = indices_to_store` as a numpy bool ndarray; Workstream C replaces this with a backend-aware structure when input is Dask.
- Workstream C Files: `src/hiveplotlib/hiveplot.py:580-647` (`__store_edge_ids`) added.
- Workstream C "Done when": new bullet requiring the storage to not materialize a per-edge bool ndarray into RAM when input is Dask. Implementation chooses between (a) Dask Series storage, (b) a per-tag boolean column on the edge frame (kept as Dask), or (c) an equivalent structure that avoids the materialization. Pandas case keeps the existing numpy ndarray storage.
- The structural choice between the three options is a Workstream C implementation decision; the requirement is the no-materialization guarantee.

### Design clarification (Workstream C): Dask sort cost (`place_nodes_on_axis`) must be addressed in docs

**Date:** 2026-05-17
**Trigger:** Maintainer follow-up design note. Paraphrased from the conversation: "`HivePlot.place_nodes_on_axis` sorts nodes on an axis by a variable. In Dask, sort requires a shuffle, which is the most expensive Dask operation. Surface this as a Workstream C design note with two possible dispositions for the implementer to choose between."
**Workstream affected:** Workstream C (Optional Dask passthrough), with cascading edit to Workstream C Files (notebook scope).
**Change:**
- Workstream C "Done when": new bullet requiring the Dask example notebook (or equivalent doc) to address the sort-cost question. Implementer chooses between (a) accept the cost (the Dask sort happens, document expected wall-time impact) or (b) document a "sort upstream before passing to hiveplotlib" pattern.
- Either disposition is acceptable; the requirement is that the question is answered visibly to users, not left as a silent footgun.

### Design clarification (Workstream C / docs note): per-partition Bezier memory ceiling and partition sizing

**Date:** 2026-05-17
**Trigger:** Maintainer follow-up design note. Paraphrased from the conversation: "The Bezier kernel materializes one Dask partition's curves into numpy at a time. The memory ceiling for the curves array is `largest_partition_size * num_steps * 4 bytes` (float32). If a user has a Dask frame with few-but-huge partitions, the per-partition curves array can still exceed RAM, defeating the streaming win."
**Workstream affected:** Workstream C (Optional Dask passthrough), with cascading edit to Workstream C Files (notebook scope) and Workstream C "Done when".
**Change:**
- Workstream C "Done when": new bullet requiring the Dask example notebook (or the Dask passthrough docs) to document partition-size guidance. Names a concrete repartition guideline (e.g., "for very large edge sets, repartition to ensure no partition exceeds ~10M edges before passing to hiveplotlib"; the exact recommended ceiling is implementer's call based on benchmarks).
- Together with the sort-cost disposition above, the Dask example notebook now owns two concrete user-facing guidance pieces (sort and partition sizing) so the "Dask passthrough is opt-in and not silent footguns" story is visible to readers.

### In-scope tweak (bookkeeping): CI shape is single pytest invocation with new polars and dask markers

**Date:** 2026-05-17
**Trigger:** Maintainer follow-up bookkeeping. Paraphrased: "Confirm the CI shape decision is recorded somewhere in the plan: single `pytest` invocation, with new `@pytest.mark.polars` and `@pytest.mark.dask` markers following the existing optional-backend marker pattern. Polars-parametrized cases use per-case marks via `pytest.param('polars', marks=pytest.mark.polars)`. Marker isolation works correctly (a polars-marked case does not run under `pytest -m datashader`)."
**Workstream affected:** Workstream B (polars marker), Workstream C (dask marker).
**Change:**
- Workstream B "Done when": new bullet recording the CI shape decision (single `pytest` invocation, new `@pytest.mark.polars` marker following the existing optional-backend pattern: `bokeh`, `datashader`, `holoviews`, `plotly`, `networkx`). Polars-parametrized cases use `pytest.param("polars", marks=pytest.mark.polars)`, marker isolation verified.
- Workstream C "Done when": parallel bullet for `@pytest.mark.dask`.

## Alternatives considered

- **Collapsing `Edges._data` from dict-of-DataFrames to single-DataFrame-with-tag-column.** Walked back during the planning conversation. Workstream A iterates per-tag anyway; the collapse would be a breaking change to the `Edges._data` shape and the public `Edges.data` property without proportional benefit. Not pursued.
- **Polars-only adoption (drop pandas, switch internals to polars).** Polars's wins are on operations that are not bottlenecks in hiveplotlib (joins, groupbys at scale). The interop cost with matplotlib, networkx, bokeh, and datashader does not earn its keep when the dominant memory cost is the curves array, which is numpy regardless of frame backend. Narwhals at the input boundary captures the polars benefit without the cost. Not pursued.
- **Dask-only adoption (route everything through Dask).** Would slow the median pandas case (small graphs, single-machine). Optional Dask via Workstream C captures the benefit without the regression. Not pursued.
- **Flipping `bezier_xy_with_nans`'s default `dtype` from float64 to float32.** Cosmetic; every internal call site passes `dtype=np.float32` already. Public-API change without downstream beneficiary. Noted and left out of all workstreams.
- **Legacy `Node` / `dataframe_to_node_list` deprecation** at `src/hiveplotlib/node.py:430-450`. Worth doing, but it is scaling-unrelated tech debt and would muddy this plan. Belongs to a separate plan.

## Holdouts

Patterns deliberately left as pandas-specific (the qa-engineer should not flag these post-execution):

- `src/hiveplotlib/converters.py:41` — `nx.to_pandas_edgelist`. NetworkX output is pandas; converting through narwhals adds no value because the immediate next step hands the frame to `Edges`, which (after Workstream B) accepts pandas natively.
- `src/hiveplotlib/node.py:430-450` — `dataframe_to_node_list`. Independently flagged for deprecation; out of scope for this plan.
- `src/hiveplotlib/edges.py:151-153, 181-184` — internal `pd.DataFrame` construction from numpy arrays. The user did not bring a dataframe; we are creating a default to hold the array. No reason to route through narwhals.
- All `src/hiveplotlib/viz/{matplotlib,bokeh,plotly,holoviews}.py` pandas-specific operations. These backends do not see user dataframes; they consume numpy curves arrays produced upstream.
- All `src/hiveplotlib/datasets/` pandas operations. Internal dataset utilities; do not constitute user input boundary.
- `src/hiveplotlib/graph_features/__init__.py` pandas operations. Compute path that produces metrics; downstream of the input boundary.
- The `bezier_xy_with_nans` numpy float64 default at `src/hiveplotlib/utils.py:256`. Documented in Alternatives.

## Implementation log

- (empty; populated by the executing specialists in the same turn each workstream completes)
