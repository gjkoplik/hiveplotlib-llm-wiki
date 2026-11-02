# Spec: the datashader route at any scale

Length: six call shapes, because the outcome has six user-visible halves and each one has a wrong implementation a `Path:` line catches; the content is the outcome, not a plan.

## Outcome

Branch 53 settles scaling once. A user who builds a `HivePlot` the usual way, with node and edge data that happen to be Dask frames, renders it through the datashader backend at any size the machine can hold on disk, with no new call to learn: the library picks how to hold the node table from the table's size and the memory available, and the user types nothing different for ten thousand nodes or a billion. When the table fits in memory the render is fast; when it does not, the render gets slower and keeps working. In the maintainer's words: a good way to handle things that is much more performant assuming sufficient memory, and a fallback case that might be much more sluggish but would never fail regardless of scale. Past-memory sizes are an out that works, not first-class support.

Nothing in-memory gets slower, for integer and string node ids alike. Today's in-memory users keep today's API, today's `node_placements` pandas table, and today's output, and the in-memory Dask-edge route renders faster than today at the same memory, because the per-partition node lookup stops rebuilding a pandas hash table of the whole axis for every partition.

The measured anchor, edge render only, edges and nodes generated in memory, reading from storage not included: 1B edges against 100M nodes on one 30 GiB workstation in about 3 minutes at `num_steps=16` and about 5.5 minutes at 50 with the node table in memory; at 1B nodes the table moves to disk, and a 100M-edge render then takes about 11 minutes of lookup and render after a table build of about 15 minutes. The shipped routes reproduce these on the same probes in `hiveplotlib-performance`, re-run by hand and recorded there, while the regression harness gates small workloads for regressions; the docs state the anchors as edge-render figures and say the storage read is unmeasured. One further row ships with the work: the sprint's 62M-edge OGB graph, a real graph with real ids, rendered through the new routes and recorded next to the anchors in `hiveplotlib-performance`, never in a notebook or a test that CI executes.

Two deliverables ride with the code: one page in the Dask notebook on lazy nodes and the node-table modes, on small synthetic data as the rest of that notebook is (a short table of what to expect at a few node and edge sizes, how to force a mode, and the integer-id recommendation with the one-liner for Dask, since users at this scale are not in pandas), and docstrings corrected where they are false today. The shipped documentation stops saying that node data must stay in memory and that the Dask route is exact for every reduction (`by` and `where` are not), and restates partition sizing as the measured tradeoff on the tree combine, more partitions for a lower peak at more time, instead of "smaller partitions only lower the ceiling" and "repartition smaller is the usual fix", which were true of the old flat combine's memory and not of its time.

`persist_curves=False` means what it says on the in-memory datashader edge route: it streams, at every size, instead of being inert below the streaming threshold.

## Call shape

```python
# Lazy nodes and lazy edges, built the usual way; the library picks the node-table mode
import dask.dataframe as dd
from hiveplotlib import Edges, HivePlot, NodeCollection
from hiveplotlib.viz.datashader import datashade_hive_plot_mpl

node_ddf = dd.read_parquet("nodes.parquet")  # columns: unique_id (integers), low
edge_ddf = dd.read_parquet("edges.parquet")  # columns: from, to

nodes = NodeCollection(data=node_ddf, unique_id_column="unique_id")
partition_variable = nodes.create_partition_variable(
    data_column="low", cutoffs=3, labels=["A", "B", "C"]
)
hp = HivePlot(
    nodes=nodes,
    edges=Edges(data=edge_ddf),
    partition_variable=partition_variable,
    sorting_variables="low",
    backend="datashader",
)
fig, ax, im_nodes, im_edges = datashade_hive_plot_mpl(hp)
```

Path: `NodeCollection(data=<Dask frame>)` is accepted today; `create_partition_variable` raises on lazy input today and must run as one lazy pass (explicit cutoffs exact, quantile cutoffs through Dask's approximate quantiles, documented as such); `HivePlot.__init__` places nodes today by collecting each axis into pandas, and must instead compute each axis's vmin and vmax in one lazy pass and build one sorted, id-range-bucketed node table, held for the object's lifetime and rebuilt only when placement changes (a re-sort, new vmin or vmax, a rotation or axis update); `datashade_hive_plot_mpl` renders edges through `_dask_native_edge_raster`, whose per-partition builder looks endpoints up in that table and whose `cvs.line` call takes the `["x"], ["y"]` form so datashader combines partition rasters as a tree, and renders nodes from the lazy placements through datashader's Dask points path, with the backend input check reading the lazy placements rather than `.shape` on a pandas table. The Parquet read is the one step here no probe measured. Lazy node ids may be any type; non-integer ids cost one extra lazy pass over the nodes and one over the edges to map them to integer codes, so the notebook page strongly recommends integer ids, dense ones best, and shows how to make them in Dask (`ddf["id"].astype("category").cat.as_known().cat.codes`) as well as in pandas (`pd.factorize`).

```python
# Force a node-table mode at construction, or change it afterwards
hp = HivePlot(
    nodes=nodes,
    edges=Edges(data=edge_ddf),
    partition_variable=partition_variable,
    sorting_variables="low",
    backend="datashader",
    node_table_mode="disk",
)
hp.set_node_table_mode("memory")
```

Path: `HivePlot.__init__(node_table_mode=...)` and `BaseHivePlot.set_node_table_mode(...)`, neither of which exists today, following the `persist_curves` pattern (constructor keyword, attribute, setter), and mirrored on the `HivePlotMatrix` constructors with a setter that forwards per cell, as `persist_curves` is. Values: `"memory"` (the table in RAM: fast), `"disk"` (the table on disk: works at any size the disk holds, slower once the table outgrows the page cache), `"disk_large"` (the table on disk and known to be far past memory: the slowest fixed cost, taken from the first partition without the trial `"disk"` runs; the name says the user's situation, not the mechanism), and the default `None`, which picks for the user: `"memory"` only when the table plus the render's in-flight curves fit the memory limit with headroom, `"disk"` otherwise. Inside `"disk"` the library has two ways to read the table, direct lookups (fast while the table mostly fits the page cache, hours once it does not) and the sort-then-sweep pass (a fixed cost that does not depend on the cache), and it does not guess between them: it starts with direct lookups, times the first few edge partitions, and switches the remaining partitions to the sweep when the measured rate says direct lookups would lose, so a wrong pick costs a few partitions, never hours. The switch is logged at `INFO` on the library's logger, as `graph_features` already logs backend dispatch, saying what was measured and what it switched to; it is not a warning, since nothing went wrong. `"disk_large"` skips the trial and the log line. The guarantee a user sees: the automatic choice never picks `"memory"` where that render would exceed the machine while `"disk"` would have finished, and a user who does not trust the estimate forces `"disk"` and cannot be crashed by it. The limit is the Dask worker's memory limit when the render runs under `distributed`, and the host's available memory otherwise. Changing the mode drops the node table and rebuilds it on the next render. On in-memory node input only `None` and `"memory"` are accepted; the two disk values raise, since an in-memory table never goes to disk. This is the one new parameter on the Dask path; the no-opt-in rule for Dask input otherwise stands, and the signal here is the node table's size, which one cheap pass knows, not partition sizes, which the earlier ruling said no pass could know. The on-disk table lives under Dask's `temporary_directory` setting (the same place the route's shuffles spill), costs about 66 bytes per node while being built and 33 after, and is removed when the object is, on failure included; the `"memory"` mode never touches disk.

```python
# Lazy node placements stay lazy; compute them if you want the table
placements = hp.axes["A"].node_placements
placements.compute()
```

Path: `Axis.node_placements` returns the pandas table for in-memory input, as today, and for lazy input the Dask frame of the placed columns and the node metadata columns, which `.compute()` materializes; the backend input check raises the same kind of legible `TypeError` the lazy edge subsets raise today for the four vector backends and `to_json`, with the remedy "render with `hiveplotlib.viz.datashader`, or compute the node data and rebuild". The placement invalidates on a re-sort as today.

```python
# In-memory nodes with Dask edges: this exact code renders faster than today at the same memory
import dask.dataframe as dd
from hiveplotlib import Edges, HivePlot
from hiveplotlib.datasets import example_edge_data, example_node_collection
from hiveplotlib.viz.datashader import datashade_edges_mpl

nodes = example_node_collection(num_nodes=20_000)
partition_variable = nodes.create_partition_variable(data_column="low", cutoffs=3, labels=["A", "B", "C"])
edge_ddf = dd.from_pandas(example_edge_data(nodes=nodes, num_edges=2_000_000), npartitions=20)
hp = HivePlot(nodes=nodes, edges=Edges(data=edge_ddf), partition_variable=partition_variable, sorting_variables="low", backend="datashader")
fig, ax, im = datashade_edges_mpl(hp)
```

Path: `_curves_from_id_pairs` today casts ids to object and merges against each axis's full `node_placements` per partition; it must look endpoints up in an id-sorted array per axis, built once at placement for whatever id type the user has (integers fast, strings sorted once per axis instead of hashed once per partition), with an explicit equality check so an id missing from the table raises an internal consistency error from `hiveplotlib.exceptions` naming the id and the axis, rather than silently matching a neighbour. The numba samplers it calls release the GIL (`nogil=True`). The combine takes the `["x"], ["y"]` form. Same raster, measured against today's within ADR 0002's bands (bit-exact for count, any, max and min; relative 1e-12 for sum and mean; raster-level for var and std), and the no-regression gate runs on a string-id workload as well as an integer one.

```python
# In-memory everything: unchanged, and no slower
from hiveplotlib.datasets import example_hive_plot
from hiveplotlib.viz.matplotlib import hive_plot_viz

hp = example_hive_plot(num_nodes=2_000, num_edges=5_000, seed=7)
fig, ax = hive_plot_viz(hp)
```

Path: `HivePlot.__init__` with pandas input, the same construction and the same persisted curves; the lookup change inside `_curves_from_id_pairs` reaches every backend, so the regression harness's no-regression gate on the default path is the test, and `node_placements` stays pandas.

```python
# persist_curves=False streams on the in-memory datashader edge route at every size
hp = example_hive_plot(num_nodes=2_000, num_edges=5_000, seed=7, persist_curves=False)
fig, ax, im = datashade_edges_mpl(hp)
```

Path: `_rasterization_route` in `viz/datashader.py` selects the streamed edge route when `persist_curves` is `False`, regardless of edge count and regardless of `stream_chunk_threshold`, which keeps its meaning for `persist_curves=True`; node rasterization keeps its count-based policy. Today the flag is inert below the threshold, so this call concatenates every curve anyway.

## Out of scope

- The streaming threshold's value and whether it splits into an edge and a node constant: a second spec, [[scale-thresholds-and-very-big-notebook]] (a stub until this one ships), set by re-measuring on the shipped routes, together with the "going very big" theory notebook and the re-measured WS10 figures.
- The default `num_steps`: the signed curve-resolution spec owns it.
- Any measurement on a real multi-machine cluster, a GPU, or edges read from Parquet; the docs call those plausible and unmeasured. On a cluster the on-disk modes need storage every worker can read (a network filesystem or a bucket), which is how Dask clusters normally run; for the rare cluster without it, the docs say how to do the join with Dask's own merge and shuffle, as a power-user recipe the library does not wrap.
- Keeping the node table across sessions. Within one object's lifetime it persists on disk; an explicit save and load is a later item if anyone needs it.
- Converting node ids to dense integers on the user's behalf: the notebook page recommends it, the library does not do it.
- `P2CP`, whose data is pandas by signature; lazy nodes do not reach it.
- DuckDB or any query engine, a compiled core, a language switch, drawing each distinct curve once, and antialiased lines.
- `subset` and `highlight` on lazy node data: they assume nodes fit in memory and stay that way until a follow-on.

## Alternatives considered

Wording a re-sign displaces from the outcome statement moves here rather than being deleted, on a dated "re-sign, narrowed from:" line. Those lines are append-only: add one as each re-sign happens, and never edit or drop an earlier one. Revising the rest of this list is normal.

- The on-disk chunk-mapped table as the primary route (the experiment session's position): rejected as a separate design; it is the middle mode of the one table, and the in-memory mode stays the fast one because it measured 10 to 30% faster when the table fits.
- Dask's own shuffle as the past-memory route: rejected on one machine (1.4 to 3x slower than the on-disk table's direct lookups and slower still than its sort-then-sweep pass in every measured setting); kept as the documented cluster fallback.
- A query engine (DuckDB) for the join: not needed at the target; at 1B nodes it would replace the sort-then-sweep pass with a new dependency, and nothing times its join past memory.
- A compiled core or a language switch: nothing measured is interpreter overhead; the one Python-level defect is the GIL-held sampler, fixed by a decorator argument. The term no route here addresses is datashader's own line aggregation, 64 to 72% of single-thread time at 1B edges; a faster rasterizer is upstream work and the one place "two minutes could have been two seconds" could still hide.
- Automatic selection by partition size: rejected before (the Path A ruling); the signal here is the node table's size, which is knowable.
- A hidden cross-session cache of the node table: rejected; not how the surrounding libraries behave.

## Failure modes

Maintainer-named (failure-mode wave, 2026-11-02).

- **Things slow down in some drastic way** for anyone who was fine before.
- **Automatic mode picks wrong**: someone who need not understand any of this notices it, or is affected by it. The bar is that a current user upgrades, starts using it, and either notices no change or thinks things are faster; effectively no change in user experience, only now they can do things at a higher scale.
- **It does not work for people out of the box to scale up**, or the documentation is wrong, or it is unbelievably complicated for even a power user to figure out: someone who understands everything they want to do with hive plots on big graphs is too confused about Dask shenanigans to get there. The maintainer is the test bed: reviewing all the code, he should not be in any way confused when he tries to use the tool at scale.
- **It does not work on a real graph**, or the defaults chosen are very non-performant on real graphs, as opposed to the uniform random ones the probes ran.
- **The anchor numbers do not generalize at all**: the choices were made on them, someone else uses it and finds it unusable.
- **Two minutes that could have been two seconds**: a year on, a different tool, a structural change or an architectural revision turns out to make this effectively real time instead of async, and the research path missed it.

Named and dismissed by the maintainer: misuse of the mode override (nothing wrong to do with it), and lazy nodes feeling second-class (lazy placements you cannot look at and approximate quantiles are the inherent compromises of working at scale, as with xarray on Dask; there should be something second-class about it).

## Open questions

- How big the buckets are in the sort-then-sweep pass (`"disk_large"`, and `"disk"` after its switch). That strategy sorts every edge's endpoints into id-range buckets matching the table's, then sweeps each bucket once; the sorting step costs one small file append per bucket per edge partition, so its time grows with buckets times partitions. The probe used one bucket per million nodes (1,000 at 1B nodes), which predicts about 98 minutes of sorting at 1B edges; buckets sized to a share of available memory (say 1 GB of table each, 33 at 1B nodes) would cut that about 30x. The rule that sets the size is unmeasured, and clears with one measured row at 1B nodes under a memory-sized bucket count.

## Sign-off

Intent extraction: brief mode ran (2026-11-02, three forks: WS11 fold-in, one spec or two, where the override lives); failure-mode wave ran (2026-11-02, eight tailored prompts, six modes named, two dismissed).

No agent ever signs a spec change, autonomous runs included. An agent whose write can change what this spec promises marks it on an "Unapproved modifications:" line here; only the maintainer clears that marker, by re-signing. Resolving an open question above is such a change. The full governance rules live in the harness CLAUDE.md under "Specs and plans".

The fence below is append-only: one line per event, in the form `Re-signed YYYY-MM-DD by <name(s)>: <one paragraph on what changed and why>.` or, once the outcome statement above is true, `Closed YYYY-MM-DD by <name(s)>.`

```
Unsigned draft, 2026-11-02; adversary spec-stage pass and api-critic planning pass folded in the same day.
```

## Plans

A plan closing never closes this spec. The last plan listed here closing is the trigger to check the outcome statement above, which stays the arbiter; plan-end QA runs that check.

- None yet.
