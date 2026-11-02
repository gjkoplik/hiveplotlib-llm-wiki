# Spec: the streaming threshold and the "going very big" notebook

Stub, 2026-11-02: not ready to sign. Every number below is contingent on [[datashader-route-at-scale]] shipping and being re-measured; "Open questions" is the list of what must resolve before this draft becomes a spec, and each item names what clears it. A reader who finds this page with the first spec closed and these items unresolved has found the next piece of work, not a finished intent.

## Outcome

Provisional, to be rewritten from the maintainer's words once the numbers exist.

The datashader route streams at the size where streaming measurably stops costing time, not at a million items chosen before the measurements existed, and the docs' numbers describe the routes the library takes. A user who reads the published scaling figures (the blog notebook, the decision table) sees figures measured on the shipped routes of the first spec, not on the flat combine and the per-partition merge they replaced. And a user who wants to know what happens past the measured sizes, at a billion edges or a billion nodes, reads one theory notebook that explains where the time goes and where the memory goes, with no code to run: the node table as the only term that grows with nodes, the raster canvas as the term that does not grow at all, and the sort-then-sweep pass as the out that works past memory. In the maintainer's words, a notebook about going very, very big, and it is not optional.

## Call shape

```python
# Unchanged for the user: the default constant moves, the call does not
import hiveplotlib as hpl
from hiveplotlib.viz.datashader import datashade_hive_plot_mpl

hp = hpl.HivePlot(...)
fig, ax = datashade_hive_plot_mpl(hp)
```

Path: `datashade_hive_plot_mpl` through `_rasterization_route` in `src/hiveplotlib/viz/datashader.py`, where `_DEFAULT_STREAM_CHUNK_THRESHOLD` (1,000,000 today) decides single-shot versus streamed; the constant changes, possibly into an edge constant and a node constant, and `stream_chunk_threshold=` keeps overriding it. The five prose sites that restate the literal "1,000,000" move with it.

Deliverables, at the altitude of the deliverable: the "going very big" theory notebook (home undecided, see below), the WS6 blog notebook and the WS7 decision table from the scaling plan, both re-measured on the shipped routes and published only after the constant lands.

## Out of scope

- Everything the first spec ships: the node-table modes, lazy nodes, the sorted lookup, nogil samplers, the tree combine, `persist_curves=False` streaming at every size. This spec moves a constant and publishes numbers.
- The default `num_steps`: the signed curve-resolution spec owns it, and this spec's expected-time figures are stated at whatever default that spec sets.
- New measurement past one machine (cluster, GPU, Parquet reads): the theory notebook says those are plausible and unmeasured, as the first spec's docs do.

## Alternatives considered

- None yet.

## Failure modes

- <failure mode>: <the wave has not run; it runs once the open questions below resolve and this draft is rewritten from the maintainer's words>

## Open questions

Resolve before signing. Each item names what clears it.

- **The constant's value.** The scaling plan's WS11 recommended 50,000 from one topology (nine edge chunks every time) measured on the old routes. Clears with the WS11 sweep re-run on the shipped routes of the first spec, plus the second topology WS11 asked for (fewer populated axis pairs, so fewer, larger chunks), which is the one caveat that can move the crossover.
- **Split or shared.** The constant gates node rasterization as well as edge rasterization, and WS11 found the two sides cost differently: forcing node streaming is nearly all of the small-plot time penalty and buys no memory. Either an edge constant and a node constant, or one lowered constant accepting that. A design call for the maintainer once the re-measured sweep is on the page; measurement alone cannot settle it.
- **Whether the lazy-node node raster needs its own streaming policy.** The first spec adds a node raster fed from a lazy table; the scaling plan's shared-threshold interaction (its section 1) assumed in-memory nodes. Clears when the first spec's implementation shows whether that raster takes the existing route or its own.
- **The re-measured WS10 figures.** WS10's Dask memory figures (the thread-cap pair at 4.0 to 4.5 GiB) describe the flat combine and per-partition merge the first spec replaces; the medium regression scenario (250,000 edges) also flips route at a 50,000 constant, so the medium-over-tiny peak-RSS gate margin has to be re-measured. Clears with one run of the regression workloads and the WS10 figures on the shipped routes; WS6 and WS7 stay blocked until then.
- **The theory notebook's home and scope.** A notebook with no code to run does not fit the examples gallery, which CI executes; candidates are a blog notebook under `docs/` (the one place CLAUDE.md allows a hand-written notebook outside `examples/`) or prose in the Dask notebook. Its source material is the scale-architecture research page in this wiki. Two watch items feed it and must be checked first: whether drawing each distinct curve once pays on a real degree-sorted example (the maintainer doubts it; check the OGB graph before closing branch 53), and whether log color scaling skews at a billion edges on real data.
- **Which plan carries it.** WS11 left the container open (the scaling plan, or a follow-up plan). The first spec's plan is the natural follow-on home now; confirm when that plan is written.

## Sign-off

Intent extraction: brief mode ran (2026-11-02, as part of the first spec's brief mode, where the maintainer chose two specs because this one's numbers depend on the first shipping, and named the theory notebook non-negotiable); failure-mode wave not yet run (runs once the open questions resolve).

No agent ever signs a spec change, autonomous runs included. An agent whose write can change what this spec promises marks it on an "Unapproved modifications:" line here; only the maintainer clears that marker, by re-signing. The full governance rules live in the harness CLAUDE.md under "Specs and plans".

The fence below is append-only: one line per event, in the form `Re-signed YYYY-MM-DD by <name(s)>: <one paragraph on what changed and why>.` or, once the outcome statement above is true, `Closed YYYY-MM-DD by <name(s)>.`

```
Stub, 2026-11-02; unsigned and not signable until the open questions resolve.
```

## Plans

A plan closing never closes this spec. The last plan listed here closing is the trigger to check the outcome statement above, which stays the arbiter; plan-end QA runs that check.

- None yet. The scaling plan's WS11 entry (`wiki/wiki/plans/scaling-large-networks.md`, Amendment of 2026-08-22) holds the measurements this stub was written from.
