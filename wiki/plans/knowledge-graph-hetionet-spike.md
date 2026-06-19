# Spike: Hive-plot views of Hetionet (knowledge-graph ergonomics)

<!--
LIGHTWEIGHT SPIKE CHARTER, not a normal implementation plan. The maintainer
chose "lightweight spike" over a full implementation plan. It is question-driven
exploration. No API ships from this spike. The template is implementation-shaped;
sections that don't apply to an exploration say so explicitly rather than being
dropped. Scratch, version-controlled, not curated wiki content (see
plans/README.md). Non-ADR plan (ships a notebook + findings, no structural API
decision) — archived post-task, not promoted to an ADR.
-->

## What this is

The empirical validation pass for the analysis at
`wiki/wiki/analyses/hive-plots-for-knowledge-graphs.md` (the charter). That
analysis is theory: a 7-pattern taxonomy of hive-plot views over a KG, a
"what hiveplotlib supports today" capability table, 6 ranked ergonomic gaps, a
Hetionet worked example, and 4 open questions. **Those are hypotheses.** This
spike prototypes hive-plot views of the real **Hetionet** biomedical knowledge
graph (Himmelstein et al.; ~47K nodes / 11 node types, 2.2M edges / 24
relationship types; CC0) on the **current** hiveplotlib API to confirm, refute,
or refine them against real data, before any new API is designed.

The actual spiking happens later, on a hiveplotlib git worktree, possibly weeks
out. This doc must read cold and self-contained, in the spirit of
`wiki/wiki/analyses/cora-prototype-plan.md`. **Do not create a worktree now;**
this is the charter, written in the wiki.

## Goal

Answer three questions empirically and write the answers into the
**Findings / gaps observed** section so they can feed a later implementation
plan:

1. Which of the analysis's 6 ranked gaps actually obstruct a real Hetionet
   workflow on today's API (vs. read worse on paper than they bite in practice)?
2. Which patterns the analysis marks "supported today" truly just work, with no
   friction worth fixing?
3. Where does the friction concentrate, so a later plan can spend its leverage
   there rather than spreading thin across all 6 gaps?

The deliverable is **two** scratch/exploratory notebooks (A: ingestion /
wrangling; B: views / conclusions) plus this plan's own
**Findings / gaps observed** writeup. No code ships to `src/`.

**Stop condition.** Cook while productive. The dispatcher halts and surfaces for
maintainer review when it's spinning wheels, hits a genuine fork, or turns up
anything worth the maintainer's attention, rather than improvising API or
workarounds past a real blocker. The maintainer reviews and commits at a roughly
daily cadence (agents do not commit).

### Non-goals (read these first)

1. **No API changes from this spike.** Any API this exploration suggests
   (`from_metapath`, RDF / property-graph converters, schema/aggregation
   helpers) is a **finding**, recorded in **Findings / gaps observed** and
   labeled *not-for-implementation*. Such proposals are deliberately kept OUT of
   "API usage examples" so api-critic is not triggered on a strawman that this
   spike has no mandate to ship.
2. **No whole-graph multi-axis layout.** The analysis settled that forcing all
   11 entity types onto axes recreates the hairball (Krzywinski; Nöllenburg &
   Wallinger: NP-complete axis assignment). The unit of work is a **scoped
   metapath/predicate slice**, never the whole graph. A probe that drifts toward
   "draw all of Hetionet" is out of scope.

## Alignment (grill)

**Maintainer shared-understanding pass (grill), Wave 1 (2026-06-19):** purpose,
ingestion testability, probe coverage, notebook durability and structure, stop
condition. Wave 1 converged; no open items.

Aligned positions:

- **Purpose.** The spike answers "how hard is it to interact with KG data," and
  both outcomes are valuable: (a) obvious easy additions that would make it
  simple, and (b) fundamental flaws that would require changing the package.
  Findings are bucketed into easy-adds vs fundamental-flaws.
- **Ingestion testability (gap 1).** Ingestion gets a real, dedicated probe
  rather than coming back "indirect." Realized via the two-notebook split below.
- **Notebook structure (maintainer call).** Two notebooks, not one: (A) an
  ingestion/wrangling notebook that wrestles with getting Hetionet data in and
  surfaces ingestion issues, and (B) a views/conclusions notebook for the
  hive-plot slices and the science implications. The spike cares about both the
  wrangling and the science.
- **Durability/placement (maintainer call, supersedes the /tmp default).** Work
  lives in the repo on the worktree branch, not /tmp, so the maintainer can
  review and commit at a roughly daily cadence; /tmp risked losing valuable work
  if weeks pass before review. Agents still do not commit; the maintainer reviews
  and commits everything.
- **Graduation target (maintainer).** If it pans out, the vendored Hetionet
  subset (from a `make_hetionet_*` runner mirroring `make_trade_network_dataset.py`)
  plus a reader mirroring `international_trade_data()` (robust to user-supplied
  data) ship with the package, and Notebook B becomes a CI-tested "Hive Plots for
  Knowledge Graphs" example running on the shipped subset. W1's runner and reader
  follow the trade-data convention from the start to keep this path open.
- **Probe coverage (delegated to dispatcher).** Promote the schema/TBox view (#4)
  from stretch to core, since a fundamental flaw would surface there. Aggregate
  (#5) and ego (#7) stay stretch. Slice choice, probe depth, and subset size are
  delegated to the dispatcher's research judgment.
- **Stop condition (maintainer).** Cook while productive; halt and surface for
  review when spinning wheels, on a genuine fork, or on anything worth the
  maintainer's attention, rather than improvising API or workarounds past a real
  blocker.

Resolved structural changes routed to orchestrator amend-plan: two-notebook split,
ingestion probe, schema view to core, findings bucketing, /tmp-to-repo placement
with graduation framing, stop condition. No open items.

Candidate alignment questions that seeded this grill pass (kept for the record;
all answered by the aligned positions above and folded into the charter via the
2026-06-19 amendment):

- **Probe coverage.** Is validating the 4 strongest "supported today" patterns
  (#1 metapath, #2 predicate facet, #3 role decomposition, #6 P2CP) the right
  cut, or should a gap-heavy pattern (#4 schema, #5 aggregation) be promoted from
  stretch to core so the spike stresses where the analysis predicts the most
  pain?
- **W1 vendoring scope.** Is one CbGaD metapath slice enough to exercise #1/#2,
  or does the spike need a second slice (a role-decomposition-friendly one) to
  answer the friction-concentration question without contorting one dataset?
- **Predicate-as-column commitment.** ADR 0001 forces predicate-as-edge-column
  (multigraph keys are dropped on conversion). Is the spike content to model
  every probe through a `predicate` column from the start, or does it want one
  probe that deliberately tries the multigraph path to document the collapse
  warning as a first-hand finding?
- **Notebook fate.** Should the scratch notebook live in `examples/` from day one
  (betting it graduates) or as a `/tmp` artifact until a finding justifies
  graduation? (See Notebook review.)
- **Stop condition.** What's "enough"? Is the spike done when all 6 gaps have a
  verdict, or when the friction-concentration question has a defensible answer
  even if some stretch gaps stay unprobed?

## Prior ADRs / design docs

- `wiki/wiki/adr/0001-networkx-integration.md` — **Accepted, v0.28, BINDING.**
  `graph=` is the NetworkX ingestion entry point on `HivePlot.__init__` /
  `HivePlotMatrix.from_partition` / `from_variable_sweep`. The `from_networkx`
  classmethod was built and torn out; **do not reintroduce a `from_*`
  classmethod.** `from_tags` is intentionally graph-less (tags are an
  `Edges`-level concept). **Predicates must ride as edge attributes/columns, not
  as NetworkX multigraph keys:** inbound conversion (`networkx_to_nodes_edges`
  via `nx.to_pandas_edgelist`) drops multigraph keys, so a predicate stored as a
  key is silently lost. Hetionet is a genuine multigraph, so a simple-graph build
  trips `warn_on_parallel_edge_collapse` unless edges keep multigraph form or
  model predicate-as-column. `node_graph_metrics` / `edge_graph_metrics` supply
  axis-sort metrics (degree, centrality) and directional-role inference
  (in/out-degree) by string name, computed before partitioning, replacing the old
  manual `pd.DataFrame(G.degree...) + merge` boilerplate.

## Structural precedent

- `wiki/wiki/analyses/cora-prototype-plan.md` — a self-contained, fresh-session
  real-data graph prototype, phased data setup → ingestion → structural-property
  extraction → view construction → iterate/scale. Pattern this spike's shape on
  it. **One correction:** it predates v0.28, so its manual `pd.DataFrame(G.degree)
  + merge` is the OLD pattern; this spike uses `graph=` + `node_graph_metrics`
  instead (per ADR 0001).

## Scale-plan consistency (not gating)

- `wiki/wiki/plans/scaling-large-networks.md`,
  `wiki/wiki/plans/performance-regression-harness.md`. Whole-graph Hetionet is
  their territory; scoped slices are 2-3 orders of magnitude smaller. The spike
  **records per-slice edge counts** in its findings. If a slice is large enough
  to want datashader, that is a data point for those plans, not a spike blocker.
  Stay consistent with their stance: very-large rendering is the datashader
  backend's job, not a reason to cap axes.

## Concept pages touched (link, one line each on why)

- `wiki/wiki/concepts/knowledge-graph.md` — the domain and its data models.
- `wiki/wiki/concepts/metapath.md` — the CbGaD slice IS pattern #1's theory.
- `wiki/wiki/concepts/edge-rendering.md` — edge **tags** = predicates (pattern #2).
- `wiki/wiki/concepts/hive-plot-matrix.md` — `from_tags()` faceting
  (matplotlib + datashader only).
- `wiki/wiki/concepts/node-assignment.md` — entity-type and directional-role
  partitions (patterns #1, #3).
- `wiki/wiki/concepts/graph-features.md` — the metric catalog feeding sorts/roles.
- `wiki/wiki/concepts/p2cp.md` — the attribute facet (pattern #6).
- `wiki/wiki/concepts/applications-bioinformatics.md` — why Hetionet. **Note: no
  Hetionet entry there yet — a post-task add, not part of this spike.**
- `wiki/wiki/concepts/differential-hive-plot.md` — **OUT of scope** (DHP
  unimplemented; compare slices with `from_tags()` panels). Listed only to
  forestall scope creep, not to be exercised.

## Patterns this replaces

None — exploration, no new surface.

## Default justifications

None — exploration, no new surface. (W1 ships a dataset loader whose `path=None`
default mirrors the existing house convention in
`src/hiveplotlib/datasets/international_trade.py`; that is convention-matching,
not a new default to justify.)

## Naming audit

None — exploration, no new surface. Any proposed name (`from_metapath`, RDF /
Neo4j converters) is a finding in **Findings / gaps observed**, not a committed
name, so the naming check is deferred to the later implementation plan that would
actually ship it.

## API usage examples

**No API surface change — exploration only.** The Proposed / API-Critic
sub-sections are intentionally omitted. Probes call the **current** documented
surface (`HivePlot(graph=...)`, `HivePlotMatrix.from_tags(...)`,
`node_graph_metrics=...`); they invent no new API. Any hypothetical API surfaced
by the exploration lives in **Findings / gaps observed**, labeled
not-for-implementation, precisely so api-critic is not invoked on a strawman.

## Notebook review

**Two notebooks, both genre = scratch / exploratory**, NOT gallery or tutorial:

- **Notebook A — ingestion / wrangling.** Wrestles Hetionet in from a native /
  native-ish form and surfaces ingestion friction first-hand (the dedicated probe
  for gap 1; see W2).
- **Notebook B — views / conclusions.** The hive-plot slices (metapath view,
  predicate facet, role decomposition, schema view, P2CP) and the structural /
  science implications. The graduation candidate (see W3–W8, Q4).

Flag both for editorial-critic so they are reviewed against the right bar (or
deferred until/unless a finding justifies graduating Notebook B to a polished
example). Neither is held to gallery dataset-coherence or tutorial-arc standards
while it is a scratch probe.

**Placement / durability (maintainer call, supersedes the prior /tmp default):**
both notebooks live in the **repo on the worktree branch**, not `/tmp`, so the
maintainer can review and commit at a roughly daily cadence; `/tmp` risked losing
valuable work if weeks pass before review. **Agents do not commit; the maintainer
reviews and commits everything** (standing rule unchanged). Per CLAUDE.md, only
`examples/` notebooks are editable (`docs/source/...` is auto-generated), so the
notebooks live under `examples/`.

**Graduation framing.** If the spike pans out: the vendored Hetionet subset (from
a `make_hetionet_*` runner mirroring `runners/make_trade_network_dataset.py`)
plus a reader mirroring `international_trade_data()` (robust to user-supplied data,
download-it-yourself) ship with the package, and **Notebook B** becomes a
CI-tested "Hive Plots for Knowledge Graphs" example running on the shipped subset.
W1 builds the runner and reader on the trade-data convention from the start to
keep this path open. Until/unless B graduates, both notebooks remain scratch
probes (genre note above).

## Workstreams

Lighter and question-driven. W1 is the must-settle-first feasibility item. W2 is
the ingestion probe (gap 1) and writes Notebook A; the rendering probes (W3–W8)
write Notebook B. Each probe has **Done when = rendered + friction recorded in
the Findings log.** A probe is a success whether the pattern works smoothly OR is
painful; either way it produces a verdict. Probes may run in any order after W1;
W1 gates all of them.

**Slice / depth / size are delegated to the dispatcher's research judgment.** The
maintainer doesn't know the dataset internals and explicitly delegated slice
choice, probe depth, and subset size. W1's CbGaD slice below is a **recommended
starting point**, not a fixed decision; the dispatcher may adjust it for what the
data supports (e.g. a directional-role-rich slice if role decomposition needs it).
Record the choice and rationale in the Findings / Implementation log; no
amend-plan round-trip is needed for a slice adjustment.

### Workstream W1 (feasibility, MUST settle first): Hetionet acquisition + subsetting

**Status:** not started
**Files:** `runners/make_hetionet_dataset.py` (new, generator), a vendored slice
under `src/hiveplotlib/datasets/hetionet/` (small, pre-subsetted), and a reader at
`src/hiveplotlib/datasets/hetionet.py` following the `international_trade_data()`
convention. Per the settled graduation framing (Notebook review), the runner and
reader follow the trade-data convention from the start, so the home is no longer
an open "scratch-only" fork.
**Done when:** at least one Hetionet slice is loadable in the spike environment as
`(data, metadata)` with the predicate carried as an **edge column** (per ADR
0001); the slice's node/edge counts are recorded in the Findings log; and the
acquisition path (where Hetionet is downloaded, what the runner subsets) is
documented in the runner docstring.

House dataset convention to mirror (from
`src/hiveplotlib/datasets/international_trade.py` +
`runners/make_trade_network_dataset.py`):

- **Ship only small pre-subsetted slices**, never the full 2.2M-edge graph.
  Hetionet is publicly downloadable (project repo / Neo4j dump / SemmedDB-derived
  TSVs); the **runner does the subsetting**, the package ships the result.
- **`(data, metadata)` return.** `metadata` carries provenance, citation
  (Himmelstein et al., *eLife* 2017, "Systematic integration of biomedical
  knowledge prioritizes drugs for repurposing"), license (**CC0**), the
  subsetting recipe, per-column descriptions, and counts.
- **Paired `runners/make_hetionet_*.py` generator**, runnable from repo root,
  matching the trade runner's structure (hardcoded params block → download/load →
  subset → write `(csv, json)` pair under `datasets/hetionet/`).
- **Helpful error listing available slices** when a requested slice is absent
  (mirror the trade reader's "only the following values are supported" `ValueError`).
- **`path=None` defaults to shipped data.**
- **Optional-backend gotcha (binding):** `datasets/__init__.py` star-imports
  every datasets module (`from hiveplotlib.datasets.hetionet import *`). If the
  Hetionet reader needs networkx (e.g. to hand back a graph), it must use a
  **function-level** networkx import with the helpful-missing-extra error, NOT a
  module-level import, or `import hiveplotlib.datasets` breaks for users without
  the `[networkx]` extra. The trade reader needs no optional backend and so is
  not a template for this part.

**Proposed W1 vendoring decision (orchestrator recommendation, maintainer
confirms):** vendor **one CbGaD-family metapath slice** as the primary fixture:
the `Compound–binds–Gene–associates–Disease` instance subgraph the analysis names
as the workhorse worked example. This single slice exercises probes #1 (metapath
instance view: three node types Compound/Gene/Disease on three axes) and #2
(predicate facet: the Gene–Disease relations `associates` / `upregulates` /
`downregulates` as tags) directly, and supports #3 (role decomposition by edge
direction) and #6 (P2CP on one entity type's attributes) without a second
dataset. Roughly target a few thousand edges so it is committable, renders without
datashader, and still shows real heterogeneity (exact size is a W1 finding to
record; subset by, e.g., top-degree compounds or a disease-area filter, documented
in the runner). This is a **recommended starting point**: per the delegation
above, if the dispatcher finds #3 needs a directional-role-rich slice that CbGaD
doesn't provide, it may add or swap a second slice on its own research judgment
(record the choice and rationale in the Findings / Implementation log; no
amend-plan round-trip).

### Workstream W2 (core probe): Native ingestion probe (gap 1) — Notebook A

**Status:** not started
**Files:** Notebook A (ingestion / wrangling), under `examples/`.
**Done when:** Hetionet loads from a **native / native-ish form** (e.g. the
hetio/hetionet hetnet JSON, or an `rdflib` / SPARQL pull) into hiveplotlib on a
small piece and renders something — AND the ingestion friction is recorded in the
Findings log (gap 1): the type-mapping work, the predicate-as-edge-column shaping
(per ADR 0001 multigraph-key loss), directedness handling, and any
multigraph-collapse warning (`warn_on_parallel_edge_collapse`) hit along the way.

This is the **dedicated** probe for gap 1, which the analysis flags as the biggest
adoption lever and which the rest of the spike (pre-subsetted CSV via W1) only
touches indirectly. It is separate from W1's full vendoring pass: W2 loads a small
piece from a native form to surface ingestion issues first-hand. **Both** the
W1 runner-authoring friction AND W2's first-hand friction feed gap 1's verdict.

### Workstream W3 (core probe): Metapath instance view (#1, the workhorse) — Notebook B

**Status:** not started
**Files:** Notebook B (views / conclusions), under `examples/`.
**Done when:** a `Compound–binds–Gene–associates–Disease` view renders — three
axes (Compound, Gene, Disease) from the entity-type column, edges filtered to the
metapath's relations, each axis sorted by a meaningful attribute (compound by
trial phase if present, gene by `node_graph_metrics="degree"`, disease by a
prevalence/degree proxy) — AND the friction is recorded in the Findings log:
specifically, how much manual assembly the "no `from_metapath`" gap (#2 in the
analysis's ranked gaps) actually costs, and whether per-axis sorting and edge
filtering are ergonomic or fiddly.

### Workstream W4 (core probe): Predicate facet via `from_tags()` (#2) — Notebook B

**Status:** not started
**Files:** Notebook B.
**Done when:** with the Gene and Disease axes fixed, a `HivePlotMatrix.from_tags()`
panel-per-predicate (`associates` / `upregulates` / `downregulates` as edge tags)
renders on the matplotlib backend, AND the Findings log records: how the predicate
column becomes tags (manual bucketing cost = analysis gap #4 "predicate-faceting
ergonomics"), whether a shared legend across panels is missing and how much it
hurts, and the per-panel edge counts.

### Workstream W5 (core probe): Role decomposition (#3) — Notebook B

**Status:** not started
**Files:** Notebook B.
**Done when:** a partition by directional role — pure subjects (sources), pure
objects (sinks), and both (intermediates, via repeat/cloned axes) — renders for
the slice, computed from edge direction (using in/out-degree from
`node_graph_metrics` per ADR 0001), AND the Findings log records how much the
"no role-partition convenience" gap (analysis gap #5) costs to hand-roll and
whether the repeat-axis handling of subject=object entities is smooth.

### Workstream W6 (core probe): P2CP attribute facet (#6) — Notebook B

**Status:** not started
**Files:** Notebook B.
**Done when:** a P2CP renders the attribute (literal) facet of one entity type
from the slice (e.g. Compound attributes as polar parallel coordinates), AND the
Findings log records whether this is, as the analysis predicts, friction-free
(gap verdict: confirmed-no-gap) or whether KG-attribute shaping surfaces an
unanticipated rough edge.

### Workstream W7 (core probe): Schema / TBox view (#4) — Notebook B

**Status:** not started
**Files:** Notebook B.
**Done when:** the schema view renders — the 11 entity types as nodes,
relationship types as edges weighted by instance count — AND the Findings log
records the cost of **hand-building the type-level graph** yourself (analysis gap
#3 "schema + aggregation helpers"). Promoted from stretch to core: the analysis
predicts a fundamental flaw would surface here, so the spike stresses it directly
rather than leaving it time-permitting.

### Workstream W8 (stretch probes): aggregate / ego scoping (#5, #7) — Notebook B

**Status:** not started (stretch)
**Files:** Notebook B.
**Done when (per probe attempted):** rendered + friction recorded. Lower-priority;
the analysis already marks them gap-heavy ("doable by hand if you first build the
type-level graph yourself"). Attempt as time allows:
- **#5 aggregated supergraph:** roll instance edges to type-to-type with
  weight = count; record what edge statistic reads best as bundle weight and
  whether the slice is large enough that datashader is implicated (feeds the
  scale plans + open question Q3).
- **#7 ego / query scoping:** scope to one entity's k-hop neighborhood, then
  re-run a core probe on it — record whether the "hand it an already-scoped
  subgraph" story holds (analysis gap #1 "ingestion converters").

## Findings / gaps observed (the spike's center of gravity)

This is what feeds the later implementation plan. Keyed to the analysis's **6
ranked gaps**; each gets a **confirmed / refuted / refined** verdict against real
Hetionet data once the probes run. Empty until execution; populate as probes
complete, citing the probe (W2–W8) and per-slice edge counts that ground each
verdict.

**Two buckets the maintainer cares about.** Beyond the per-gap verdicts, sort
every observation into: **(a) obvious easy additions** that would make KG
interaction simple (an ergonomic helper, a convenience constructor, a default),
and **(b) fundamental flaws** that would require changing the package itself (a
data-model assumption, a contract that doesn't fit KGs). The friction-concentration
synthesis names which bucket dominates.

```
Not yet populated — fill during execution. Each gap below gets a one-line verdict
(confirmed obstructs / refuted, didn't bite / refined to <narrower statement>)
plus the probe and edge-count evidence, and is tagged easy-add (a) or
fundamental-flaw (b). Record per-slice node/edge counts here for the scale plans.
Any API a finding suggests is labeled not-for-implementation.
```

- **Gap 1 — Ingestion converters (RDF / property graphs).** Verdict: _pending_
  (evidence from W2's dedicated native-form ingestion probe + W1 runner-authoring
  friction + W8 #7 scoping). The analysis calls this the biggest adoption lever;
  W2 probes it directly from a native form (no longer only indirect via
  pre-subsetted CSV).
- **Gap 2 — Metapath constructor (`from_metapath`).** Verdict: _pending_
  (evidence from W3). Record how much manual assembly the workhorse view costs.
- **Gap 3 — Schema + aggregation helpers.** Verdict: _pending_ (evidence from W7
  schema + W8 #5 aggregate).
- **Gap 4 — Predicate-faceting ergonomics (tag-building + shared legend).**
  Verdict: _pending_ (evidence from W4).
- **Gap 5 — Role-partition convenience.** Verdict: _pending_ (evidence from W5).
- **Gap 6 — Edge-filtering convenience.** Verdict: _pending_ (cross-cutting;
  evidence from how often W3–W8 hand-slice the edge DataFrame).

**Friction-concentration synthesis (the headline finding):** _pending_ — once
verdicts are in, one paragraph naming where a later plan should spend its leverage
(which 1-2 gaps actually bite hardest) **and which bucket dominates (easy-adds vs
fundamental-flaws)**, so the implementation plan is not a 6-gap shotgun.

## Open questions to seed (from the analysis, now testable)

The analysis's 4 open questions, restated as things this spike can produce
evidence on (it need not fully resolve them; it feeds the decision):

- **Q1 — Metapath constructor: feature or docs recipe?** W3's measured assembly
  cost informs whether `from_metapath` earns library surface or is better as a
  gallery recipe.
- **Q2 — RDF / property-graph ingestion: in-library or companion package?**
  Heavy optional deps (`rdflib`, Neo4j drivers) argue for a companion; W2's
  native-form ingestion experience (plus W1's acquisition) is a data point, not a
  resolution.
- **Q3 — Aggregated supergraph: best edge statistic for bundle weight, and does
  datashader carry it?** W8 #5 produces direct evidence; consistent with the
  scale plans' "datashader's job" stance.
- **Q4 — Does a worked Hetionet notebook make the case better than any API
  addition?** The whole spike is evidence here; the synthesis paragraph should
  take a position, since it bears on whether **Notebook B** graduates to a
  CI-tested example (placement itself is now settled — see Notebook review).

**Placement open question (surface, do NOT assert):** whether a polished Hetionet
figure later graduates to the `hiveplotlib-bioinformatics-examples` repo vs. stays
an in-hiveplotlib example is a downstream decision. The authoritative context
lives in the maintainer's memory (outside the wiki), so this is flagged for the
maintainer to confirm, not decided here.

## Plan amendments

Append-only. Charter-level adjustments after the original write.

### 2026-06-19 — Wave 1 grill outcomes folded into the charter

**Trigger:** maintainer grill-me alignment pass, Wave 1 (2026-06-19); source
capture in `## Alignment (grill)` above. Seven agreed changes, all in-scope to the
spike's "no API ships" framing (still a lightweight charter, not implementation-ified):

1. **Two notebooks, not one.** The single scratch notebook splits into **Notebook
   A** (ingestion / wrangling) and **Notebook B** (views / conclusions). Goal,
   Notebook review, and every probe's "Files" line updated accordingly. Notebook B
   is the graduation candidate.
2. **Ingestion workstream added (W2).** New core probe loading Hetionet from a
   native / native-ish form into hiveplotlib on a small piece, writing Notebook A.
   This is the dedicated probe for gap 1, which was previously only "indirect."
   Both W1 runner-authoring friction and W2's first-hand friction feed gap 1.
3. **Schema / TBox view (#4) promoted from stretch to core (now W7).** Pulled out
   of the old stretch bundle into its own core workstream (a fundamental flaw is
   predicted to surface there). Aggregate (#5) and ego (#7) stay stretch (now W8).
4. **Notebook placement: repo on the worktree branch, NOT /tmp** (supersedes the
   prior /tmp recommended default). Notebooks live under `examples/` so the
   maintainer can review and commit at a roughly daily cadence; agents still don't
   commit. Graduation framing recorded: if the spike pans out, a `make_hetionet_*`
   runner + a reader mirroring `international_trade_data()` ship the vendored
   subset, and Notebook B becomes a CI-tested example; W1 follows the trade-data
   convention from the start to keep this open. W1's "scratch-only" reader-home
   fork removed.
5. **Findings bucketed into easy-adds vs fundamental-flaws.** Findings now sort
   every observation into (a) obvious easy additions and (b) fundamental flaws, on
   top of the per-gap verdicts; the friction-concentration synthesis names the
   dominant bucket.
6. **Stop condition recorded** (in Goal). Cook while productive; halt and surface
   for maintainer review on wheel-spinning, a genuine fork, or anything worth the
   maintainer's attention, rather than improvising API or workarounds past a real
   blocker. Maintainer reviews / commits at a roughly daily cadence.
7. **Slice / depth / subset size delegated to the dispatcher's research judgment.**
   W1's "one CbGaD slice" softened from a fixed decision to a recommended starting
   point the dispatcher may adjust (e.g. for a directional-role-rich slice); record
   the choice and rationale in the Findings / Implementation log, no amend-plan
   round-trip for a slice adjustment.

Renumbering from the split + promotion: W1 (feasibility, unchanged) · W2 ingestion
probe / gap 1 [new] · W3 metapath #1 (was W2) · W4 predicate #2 (was W3) · W5 role
#3 (was W4) · W6 P2CP #6 (was W5) · W7 schema #4 [promoted to core] · W8 stretch
aggregate #5 + ego #7 (was the W6 stretch bundle, minus #4).

## Holdouts

None — exploration, no replace-and-sweep audit applies.

## Implementation log

Append-only. One line per workstream as it completes (record per-slice edge
counts here too, for the scale plans).
