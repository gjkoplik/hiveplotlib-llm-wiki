# Research plan: Bioinformatics hive-plot exploration — grounding the counterfactuals

<!--
Research-plan shape (deliberately light per the plan-template). No workstream / done-when
ceremony; conventions (shallow-panel dispatch, standing lenses, bounds, grounding, durable
landing) live in the research-track skill. This plan carries only the per-plan fill-in.
Consumer: hiveplotlib. Evidence mode: literature.
-->

## Question

The `hiveplotlib-bioinformatics-examples` satellite repo built C. elegans connectome and E. coli
RegulonDB regulatory-network hive plots hunting for a "hot damn" figure that shows something only a
hive plot can. A research-liaison pass is separately recording **what we did** (the backbone at
`wiki/wiki/analyses/hiveplotlib-bioinformatics-examples.md`). This bounded run is the other half: go
find **what we don't already know**, so the durable record's counterfactuals are real prior art, not
the maintainer's recollection. Concretely, validate-or-refute the exploration's own five conclusions
(below) against the published literature on hive-plot / connectome / gene-regulatory-network
visualization, and surface the counterfactual bio framings and data sources we did not try, so the
backbone page's negative result ("real networks are not the hero; the hero is Datasaurus-for-networks")
is grounded rather than asserted.

The load-bearing outputs are the two standing lenses: is any conclusion below a **reinvention** of
located prior art (prior-art lens), and does a **celebrated counterexample** falsify the "single real
network is never a hero" thesis (counter-evidence lens). The orthogonality lenses feed the domain
detail those two need.

## Candidate stories / hypotheses

The five exploration conclusions, cast as things to validate or refute (each has a refuter the
counter-evidence lens hunts):

- **#1 Single-network-collapses-to-density.** "A single hive plot of a single real biological network
  collapses to a **density** story, not readable structure; the format's payoff is being a *fixed
  coordinate system* that only pays off in **comparison**" **vs.** "there exists a celebrated single
  real-network hive plot that *is* a hero figure reading as structure, not density" (the counter-evidence
  refuter).
- **#2 C. elegans dimorphism excluded by construction.** "The male-vs-hermaphrodite comparison reads
  as a **density** difference; the real dimorphism (male tail / mating circuitry) was *excluded* by the
  shared-neuron restriction, and showing it as structure would need anatomical-position metadata"
  **vs.** "an anatomical-position sort, or including the male-specific neurons, is a known better
  hive-plot (or hive-plot-adjacent) framing that *does* read as structure" / "the position metadata is
  cleanly sourceable and we wrongly concluded it was not."
- **#3 GRN sign reads only in the core; role partition is Krzywinski's.** "For E. coli the **sign** of
  regulation (activation vs repression) reads clearly only in the regulatory **core**, not the full
  hierarchy; the regulator / dual / target **role partition is Krzywinski's original construction**"
  **vs.** "someone has shown regulatory sign structure legibly at *full* scale" / "the role partition is
  not actually Krzywinski's / is not the canonical GRN hive-plot construction."
- **#4 The hero is engineered-control, not a real network.** "The real instant-'hot damn' comparison
  artifact is **engineered-control** (Datasaurus-for-networks: matched summary stats, identical hive
  positions, different *readable* structure), because a real network controls no confounds" **vs.**
  "matched-statistics / different-structure is a gimmick reviewers discount, not a compelling methods
  figure" / "a real differential connectome or GRN figure *does* deliver the gut-punch, so the hero
  need not be engineered."
- **#5 Biology-hands-you-the-axes.** "'Biology hands you the axes for free' (neuron class, TF-vs-target
  role are *given facts*, not design choices to defend) is the core adoption pitch for bio hive plots"
  **vs.** "the biology-given axis is contested / non-canonical / not actually the pitch practitioners
  respond to; axis choice is still a fight in practice."

## Failure-mode rubric

Populated in full by grill-me's failure-mode wave (research branch); seeded here, tuned to this run.
Consumed by the adversary at convergence. Structure-is-artifact first, then the standard modes.

- **Structure-is-artifact (the run's central risk, applied to the *counterfactuals*).** A "this would
  have read as structure" counterfactual (e.g. anatomical-position sort for #2, full-scale sign for #3)
  could be asserted from a paper's *claim* without evidence the resulting hive plot actually reads. The
  literature mode cannot render the plot (the data-validity mode is deferred), so any "would read as
  structure" verdict must be **named as a literature-grounded pointer, not a demonstrated result**, and
  the page must say which counterfactuals remain untested-by-rendering.
- **Already-known (reinvention).** Any of the five conclusions is a reinvention of located prior art:
  a published differential connectome hive plot (Krzywinski 2017 and after), a published signed-GRN hive
  plot at full scale, a published male-vs-hermaphrodite connectome viz that already does what we
  "discovered." The counter-evidence lens must surface the **specific refuter** (the paper, its data,
  what it revealed), not merely fail to find one, and the prior-art lens must settle whether our
  approaches are novel or reinvention.
- **Uncontrolled-comparison / novelty-overclaim.** Claiming "no one has shown X" from *absence* of
  located art. Hive-plot-adjacent bio viz is a thin, scattered literature (dormant tools: HiveR,
  hive_networkx), so an unindexed thesis / workshop / domain-journal figure could exist. Every
  novelty-or-first verdict carries this qualifier and the page states it.
- **Grounding-failure.** A claim (a prior-art verdict, "the role partition is Krzywinski's", a data-source
  license term, a "position metadata is/ isn't sourceable" call) asserted without a faithful supporting
  quote from the cited source. Claim-maker != voucher; default-refuted at verify.
- **Source-trustworthiness / provenance.** A faithfully-quoted claim from weak provenance (a blog, a
  vendor page, an unreviewed preprint, a third-party data mirror) passes verify untouched; record the
  trust judgment alongside each citation. Load-bearing here for the **data-source and licensing**
  findings (the RegulonDB-mirror fragility, the Abasy Atlas license terms, C. elegans position-metadata
  sourceability), where a wrong license read has real downstream cost.
- **Taste-not-fact (the hero-figure question).** Whether "matched-stats different-structure is
  compelling to a reviewer" (#4) is a *taste* judgment the literature can only inform, not settle. The
  run must frame the Anscombe / Datasaurus lineage evidence as **precedent that the move lands in
  practice**, not as proof this specific network application will; overclaiming reviewer reception is
  the failure.

## Lenses + bound

Six lenses, all disjoint, each blind to the others, hard **20-agent cap**. Two standing seats floored
first, then four orthogonality slices (the maximum the addable model allows). The "what to find" list
from the brief is folded into these six.

- **[STANDING] Prior-art / counterfactual lens.** Published hive-plot and hive-plot-adjacent
  visualizations of **connectomes**, of **male-vs-hermaphrodite / differential connectomes**, and of
  **gene regulatory networks**. Who, what data, what it revealed, whether our approaches are novel or
  reinvention. Anchors: Krzywinski's *original* RegulonDB / E. coli hive plot and its role partition
  (grounds #3, #5), Krzywinski's **differential hive plot** technique (2017; grounds #2, #4), the dormant
  HiveR / hive_networkx bio applications. Central question across #1-#5: "how do people normally show
  this, and have they already published what we built?"
- **[STANDING] Counter-evidence lens (blind to the confirming lenses).** Actively steelman the
  *opposite* of #1-#5 and hunt the refuters: a **celebrated single-network hive plot that IS a hero
  figure** (kills #1); a published **regulatory sign structure legible at full scale** (kills #3's
  core-only claim); a real **differential connectome / GRN figure that delivers the gut-punch** without
  engineering (kills #4); evidence that **matched-stats-different-structure reads as a gimmick** to
  reviewers rather than a compelling figure (also bears on #4); a contested / non-canonical
  biology-given axis (bears on #5). Carries not-fully-confirming hits to synthesis labelled as such;
  never drops them. This is the generation-time disconfirmation counterweight for the whole set.
- **Connectome-visualization state-of-the-art lens (orthogonality).** How neuroscientists **actually**
  visualize connectomes and sexual dimorphism (connectivity matrices, circular / chord `circos`-style
  layouts, anatomical / ball-and-stick renderings, force layouts), and specifically whether an
  **anatomical (anteroposterior / ganglion) position sort** or **including the male-specific neurons** is
  a known better framing for the dimorphism. Also: is C. elegans neuron **position / ganglion metadata**
  cleanly sourceable (OpenWorm, WormAtlas, Cook 2019 SI, Witvliet 2021)? Grounds #2. Distinct body and
  search from the hive-plot-specific prior-art lens.
- **Regulatory-network viz + data-source lens (orthogonality).** Canonical **signed-GRN**
  visualizations (RegulonDB / Ecocyc figures, `circos`, cytoscape conventions, hierarchical layouts) and
  whether the **regulator / dual / target role partition is standard** practice or Krzywinski-specific.
  Plus the **data-source / licensing** thread: a stable canonical RegulonDB source vs. the fragile
  student-repo mirror currently used, and a **permissively licensed signed-GRN alternative** (Abasy
  Atlas and its terms; others). Grounds #3, #5, and the licensing wrinkle the backbone flags as
  must-resolve-before-shipping. Distinct body and search from the connectome lens.
- **Hero-figure-lineage lens (orthogonality).** The **matched-statistics / different-structure** figure
  lineage: Anscombe's quartet, the Datasaurus dozen (Matejka & Fitzmaurice 2017), and any **network-viz
  analog** of matched-summary-stats-different-structure. What makes a *methods-paper* visualization
  land, and whether the "engineered control" move is an accepted, celebrated rhetorical device or a
  discounted gimmick. Grounds #4's "the hero is engineered-control" claim and its refuter. Distinct
  body (dataviz / statistics-education literature) from the three bio lenses.
- **Untried-bio-framings lens (orthogonality).** The counterfactual bio framings we did **not** try that
  might deliver the hero, and which have prior art vs. are open: brain connectome with **hemisphere
  axes** (the dataset that also motivates the scaling story), **PPI with cellular-compartment (GO) axes**,
  **differential hive plots as a native library feature**, metabolic / signaling pathway networks, and
  any other biology-hands-you-the-axes case with a natural partition. For each: has anyone published it
  as a hive plot, and does the literature suggest it would read as structure? Grounds the backbone's
  "open threads" and #4/#5. Distinct forward-looking search from the four backward-looking lenses.

**Bound math (addable model, research-track skill):** `1 scope-refine + 6 lenses + ~11 verify vouchers
+ 1 synth = 19 agents (cap 20)`. Six lenses is the maximum the model allows (2 standing + up to 4
orthogonality). The ~11 vouchers cover roughly **8 load-bearing claims** at one voucher each (the
five conclusions' prior-art verdicts, the "role partition is Krzywinski's" claim, the Abasy-Atlas
license terms, the C. elegans position-metadata sourceability call) plus **~3 generation-flagged
second vouchers** on single-quote load-bearing claims. The likeliest second-vote candidates: the
**no-celebrated-single-network-hero** absence claim (#1, an absence-of-art claim, the most fragile),
the **differential-hive-plot-is-Krzywinski's** attribution (#2/#3), and any **license-term** read
(where a single mis-quote has real downstream cost). This lands one under the cap with **zero
headroom to spare**, so the pre-flight estimate is where a wider decomposition would surface rather
than silently proceed.

## Validation criteria

A validated finding must be **grounded** (each claim carries a faithful supporting quote),
**independently verified** (claim-maker != voucher, default-refuted at verify, a second voucher on
generation-flagged single-quote load-bearing claims), and **adversary-convergence-cleared** against the
failure-mode rubric above. The counter-evidence lens's not-fully-confirming hits are carried to
synthesis, never dropped, and framed as such. Findings are framed as **correlational / exploratory
pointers, not causal claims**; a "would read as structure" counterfactual is a literature-grounded
pointer, never a demonstrated render (the data-validity mode that could render it is deferred).

The run classifies its terminal outcome honestly into one of three legitimate outcomes:

- **validated finding** — the expected outcome: the five conclusions land validated or amended against
  located prior art, with the counterfactuals (untried framings, data sources, licensing) surfaced. The
  page enriches the backbone with real prior-art grounding.
- **validated inconclusive** — e.g. if the "is matched-stats compelling to reviewers" question (#4) or a
  specific untried-framing's prior-art status cannot be settled either way, that finding lands as a
  pursued-to-a-negative "the literature will not settle this" reference in the page's
  `what-was-inconclusive-and-why` slot. A first-class outcome, not a failure. Plausible here given the
  thin, scattered hive-plot-bio literature.
- **nothing-cohered** — the degenerate case (unlikely, since the backbone already states five defensible
  conclusions this run re-grounds); would land only a minimal breadcrumb, never a thin finding-shaped
  page.

The expected shape is **mixed**: most conclusions validate or amend as findings, while the taste-heavy
#4 reviewer-reception sub-question and some untried-framing prior-art questions may land
validated-inconclusive. The page carries findings and inconclusives side by side; both are durable.

## Destination artifact

**Enrich** the existing backbone `wiki/wiki/analyses/hiveplotlib-bioinformatics-examples.md` via
research-liaison's **producer path** (not a new page, and not a rewrite of the liaison's "what we did"
backbone). This run supplies the grounded prior-art and counterfactual layer: the located prior art
behind each conclusion, the specific refuters found or not found, the data-source / licensing findings,
and the untried framings with their prior-art status, folded into the backbone's existing sections
(the per-dataset verdicts, "What was tried and dropped", "Status and open questions", "See Also"),
plus any new sources/entities the wiki gains. Yield determines shape: validated findings + any
validated-inconclusive enrich the page; a nothing-cohered run lands only a breadcrumb noting the
prior-art hunt found nothing to add. Landed under maintainer approval (not auto-committed).

## Minor pivots vs. amend-plan

Iteration lives inside the bounded run: refining a lens or dropping a dead angle does not force an
amend-plan. Only a fundamental change of the research question mid-run is an amend-plan.

---

## Prior ADRs / design docs

None govern this research question. The prior work to reference is the **backbone analysis being
enriched** (`wiki/wiki/analyses/hiveplotlib-bioinformatics-examples.md`), the satellite repo's own
`README.md` / `DATASETS.md` / `notes/` (canonical per-dataset provenance), and the linked wiki pages
the backbone already cites: `[[same-stats-different-graphs]]` (the Datasaurus hero this run's #4 tests),
`[[differential-hive-plot]]`, `[[martin-krzywinski]]`, `[[applications-bioinformatics]]`,
`[[nollenburg-2023-computing-hive-plots]]`. ADR 0001 (networkx-integration) is referenced only as the
API the figures used, not as a design constraint on this run.

## Failure modes

See `## Failure-mode rubric` above (the research-plan shape carries the rubric there). grill-me's
failure-mode wave (research branch) populates it in full before dispatch.

## Adversary review

### Adversary's challenge (planning mode)

```
Status: challenge (5 items)
Plan reviewed: wiki/wiki/plans/bioinformatics-hive-plot-exploration-research.md (cold)

Angles worked (all three mandated, per rule 18):
  - Premise (why now / real problem): item 1 bites hardest. The run's payoff is
    its counterfactuals, and two of the five conclusions (#1, #2) are stated as
    ABSENCE claims that a literature panel structurally cannot settle. The
    problem (grounding recollection) is real and live; the framing of what
    "grounded" means for an absence is where the premise leaks.
  - Approach (alternatives / smaller move): item 2 bites. The counter-evidence
    standing lens is the plan's only structural disconfirmation, and the way the
    five conclusions are cast (each as "X vs. a refuter the counter-evidence lens
    hunts") pre-loads the confirming side as the default. A smaller, harder move
    was not considered: run the disconfirmation lens FIRST / independently on the
    raw exploration, not as a refuter-hunt bolted onto a pre-written conclusion.
  - Size / could-this-not-exist: the run itself should exist (grounding the
    backbone's negative result is worth 19 agents). But items 3 and 4 subtract
    surface: the six lenses double-book sources across three backward-looking
    slices, and the voucher budget is quietly maxed to fit the cap with the
    absence claims (the fragile ones) getting the SAME single-vote floor as a
    verifiable presence claim.

Verification done before challenging (so items map to facts, not taste):
  - The addable-model bound math IS faithful to the shipped skill
    (research-track/SKILL.md:68-83): N lenses <= 6 (2 standing + up to 4
    orthogonality), verify vouchers <= 12, worst case 1+6+12+1=20. The plan's
    1+6+~11+1=19 lands one under with the stated zero headroom. "6 is the max the
    addable model allows" is correct, not invented. The bound is NOT the problem;
    what the 19 buys is (item 4).
  - The two standing seats the skill mandates (prior-art/counterfactual +
    counter-evidence, SKILL.md:31-38) are both floored here. The floor is
    honored. The counter-evidence lens's DESIGN (generation-time, blind to the
    confirming lenses, carries not-fully-confirming hits) is faithfully copied
    into the plan's [STANDING] Counter-evidence bullet. The convention is right;
    item 2 is about whether the CONCLUSIONS as framed let it bite.
  - The skill's counter-evidence lens is generation-time disconfirmation by
    design (SKILL.md:36), not a convergence duty — so the plan cannot fix item 2
    by leaning on the convergence adversary; that wall is the same one the
    parent plan's finding-2 hit. Any disconfirmation strengthening stays at
    generation.

Items:
  - [must-fix] Two of the five conclusions are ABSENCE-OF-EVIDENCE claims a
    literature panel will "confirm" by failing to find a refuter, and the plan
    does not name the asymmetry. #1 ("no celebrated single-network hive plot IS
    a hero") and, in its stated refuter, #2 ("no published male-vs-hermaphrodite
    differential connectome hive plot") are unfalsifiable from a bounded web
    search: not-finding-X is the DEFAULT outcome of a thin, scattered literature
    (the plan itself flags HiveR / hive_networkx as dormant, unindexed, and
    concedes "an unindexed thesis / workshop / domain-journal figure could
    exist"). A panel that searches and comes up empty will report "no refuter
    located," and the synth will read that as "conclusion validated" — which is
    exactly laundering absence into a cited-looking positive. The failure-mode
    rubric's "Uncontrolled-comparison / novelty-overclaim" entry names this, but
    naming it in the rubric does not stop the synth from doing it; nothing in the
    Validation criteria forces an absence claim to land DIFFERENTLY from a
    presence claim. — at ## Candidate stories (#1, #2) / ## Validation criteria
    Rubric: no rubric yet — pre-grill (maps to the seeded rubric's
      "Uncontrolled-comparison / novelty-overclaim" mode)
    Push: State, in Validation criteria, that an absence claim ("no one has
    published X") can NEVER reach `validated finding`; its ceiling is
    `validated inconclusive` framed as "searched N venues/tools, found none,
    absence is weak evidence." Require every absence-shaped conclusion to name
    the specific search surfaces it covered (which databases, which dormant
    tools' example galleries, which conference proceedings) so the reader can see
    the search was bounded and judge the absence, not trust it. A located POSITIVE
    refuter can validate/kill; a located ABSENCE only ever informs. Without this,
    the run's headline counterfactual ("the hero is not a real network") is
    grounded on the panel not looking hard enough.

  - [must-fix] The five conclusions are cast as "our claim vs. a refuter," which
    hands the counter-evidence lens a conclusion to defend rather than a story to
    break. Every one of #1-#5 is written confirming-side-first: the exploration's
    conclusion is the proposition, the refuter is the "vs." clause the
    counter-evidence lens "hunts." That is confirmation-shaped even with a
    disconfirmation lens floored, because the lens's job is framed as *find the
    thing that would kill OUR conclusion* — it inherits our conclusion as the
    null. The skill's counter-evidence lens is supposed to "actively search for
    evidence AGAINST the emerging conclusion" (SKILL.md:36) generation-time and
    blind; here "the emerging conclusion" is fixed to the five we already believe,
    so the lens steelmans against a moving target we chose. The tell: all five
    refuters are things we'd be SURPRISED to find (a celebrated single-network
    hero, full-scale legible sign) — the lens is set up to come back empty and
    rubber-stamp, not to surface the boring ways we're ordinarily wrong (we
    mis-attributed the role partition; the position metadata was sourceable all
    along and we didn't look). — at ## Candidate stories (framing) / ## Lenses
    ([STANDING] Counter-evidence lens)
    Rubric: no rubric yet — pre-grill (maps to seeded "Structure-is-artifact" and
      the parent skill's mode 5 "seeks only confirming evidence")
    Push: Re-charter the counter-evidence lens to search for the MUNDANE refuters,
    not just the celebrated ones, and write at least two conclusions
    disconfirmation-first. Concretely: for #2 and #3 the lens's primary job is
    "was the position metadata cleanly sourceable / is the role partition NOT
    Krzywinski's" (the ways we're quietly wrong), stated as the emerging
    conclusion to break, ahead of "is there a celebrated better framing." A lens
    told to hunt a hero it expects not to find is a lens told to confirm; a lens
    told "prove we mis-attributed / mis-sourced" is a lens that can bite.

  - [worth-discussing] The six lenses are asserted "all disjoint" but three
    backward-looking bio lenses will hit the same sources and can double-count a
    single figure as three independent findings. Prior-art (connectomes +
    differential + GRN), Connectome-viz-SOTA, and Regulatory-network-viz all
    search Krzywinski's original E. coli / RegulonDB hive plot and his 2017
    differential technique — the plan NAMES Krzywinski as an anchor for the
    prior-art lens (#3, #5, #2, #4) AND routes the role-partition question to the
    regulatory-viz lens AND routes the differential-connectome question to the
    connectome lens. The disjointness is by SUBJECT (connectome vs. GRN vs.
    hive-plot-specific), but the SOURCE SET overlaps at exactly the load-bearing
    papers. Two costs: (a) the same Krzywinski figure gets "found" by two lenses
    and counted as corroboration when it's one source seen twice (false
    independence — the verify step checks claim-maker != voucher, but not
    lens-A-source == lens-B-source); (b) the ~11 vouchers spread across findings
    that are secretly the same underlying fact, thinning real coverage. — at
    ## Lenses (prior-art vs. connectome-viz vs. regulatory-viz)
    Rubric: no rubric yet — pre-grill (maps to seeded "Already-known
      (reinvention)")
    Push: Either (a) partition by SOURCE not just subject — give the prior-art
    lens exclusive ownership of the Krzywinski/HiveR/hive-plot-specific corpus and
    forbid the two domain lenses from re-deriving hive-plot prior art (they cover
    NON-hive-plot viz: matrices, circos, chord, force layouts, and the
    data-source/licensing thread), so a Krzywinski figure is found once; or (b)
    add a synth-step dedup rule: when two lenses cite the same source for the same
    fact, it counts as one finding with one voucher, not two. Name which. As
    written "all disjoint" is asserted at the subject level and false at the
    source level for the three bio lenses.

  - [worth-discussing] The voucher budget is maxed to fit the cap, and the
    fragile absence claims get the same single-vote floor as verifiable presence
    claims — the allocation is backwards. The bound math spends ~11 vouchers on
    ~8 load-bearing claims + ~3 generation-flagged seconds, landing at 19 "with
    zero headroom to spare." The plan correctly names the two absence claims (#1,
    the #2/#3 attribution) as the likeliest second-vote candidates. But a second
    vote re-reads a cited SOURCE — and an absence claim has NO source to re-read
    (you can't voucher "I didn't find it"). So the ~3 flagged seconds are being
    aimed at claims where a second voucher does the least good (absence), while a
    verifiable presence claim (the Abasy license terms, the role-partition
    attribution to a REAL Krzywinski figure) is where a mis-quote actually bites
    and a second vote actually helps. The zero-headroom framing means there's no
    slack to fix this by adding votes; the fix is REALLOCATION. — at ## Lenses +
    bound (bound math) / ## Validation criteria
    Rubric: no rubric yet — pre-grill (maps to seeded "Grounding-failure" +
      "Source-trustworthiness")
    Push: Redirect the ~3 generation-flagged second vouchers away from the
    absence claims (where re-reading a non-existent source is theater) toward the
    verifiable single-quote presence claims where a mis-quote has downstream cost:
    the Abasy-Atlas license terms (a wrong license read ships a bad
    recommendation), the "role partition is Krzywinski's" attribution (a real
    figure to re-read), and any "position metadata IS sourceable" call (a real SI
    to re-read). Absence claims get the item-1 treatment (named search surface,
    inconclusive ceiling), not a second vote. State the reallocation in the bound
    math so the ~3 seconds land where a source exists to re-read.

  - [worth-discussing] "Enrich the backbone" is a laundering vector if a lens
    returns a source that SOUNDS adjacent but doesn't actually support the
    conclusion it's folded under. The producer path folds located prior art "into
    the backbone's existing sections" (per-dataset verdicts, "What was tried and
    dropped", etc.). The risk the destination creates: a lens searching "how do
    people visualize connectomes" returns a circos/chord connectome paper, and
    the synth folds it under #1 ("single real network collapses to density") as
    supporting prior art — when the paper is about a DIFFERENT layout making a
    DIFFERENT point, adjacent in topic but not evidence for our specific claim.
    Once it's a citation in the backbone with a wikilink, it reads as if the
    literature grounds our conclusion. The grounding contract checks the quote
    supports the claim, but "supports" is exactly the judgment that slips when a
    topically-adjacent source is folded into a pre-existing narrative slot. — at
    ## Destination artifact (producer-path fold) / ## Validation criteria
    Rubric: no rubric yet — pre-grill (maps to seeded "Grounding-failure" — the
      claim-maker != voucher / faithful-quote entry)
    Push: Require the grounding quote for each folded source to support the
    SPECIFIC conclusion it's filed under, not merely be topically adjacent — the
    verify voucher's default-refuted test must ask "does this source make OUR
    point, or a neighboring one," and a topically-adjacent-but-not-on-point source
    lands in "See Also" as related reading, NOT as prior-art grounding for a
    conclusion. State that the producer-path fold is a validated-finding
    operation: only convergence-cleared findings enrich a verdict section;
    un-cleared adjacent sources are related-reading links, kept visibly separate
    from the grounding.

  Note: this is NOT an existential-must-fix. The run should exist — grounding the
  backbone's negative result against real prior art is worth the 19 agents, the
  bound math is faithful, and both standing lenses are correctly floored. The
  five items subtract: two absence claims that can only ever inform, not validate
  (1); a confirmation-shaped conclusion framing the disconfirmation lens inherits
  as its null (2); a source-level double-count across three subject-disjoint bio
  lenses (3); a second-vote budget aimed where no source exists to re-read (4);
  and a fold-into-the-backbone step that can launder adjacency into grounding (5).
  The highest-leverage single risk — that this run produces a tidy lit-review
  validating what we already believe rather than surfacing what we got wrong —
  is items 1 and 2 together: absence read as confirmation, and a
  refuter-hunt framing that expects to come back empty. Fix those two and the
  run can actually bite; leave them and the "grounded, not asserted" claim in the
  Question is itself asserted, not grounded.
```

### Adversary post-impl (convergence gate)

```
Pending — the adversary validates each finding against the failure-mode rubric at
the convergence gate, reading distilled conclusions (not sources), and emits the
"Adversary verdicts" block in the run summary.
```

## Alignment (grill)

```
Not yet run — recommended before the run for this plan. The whole run's value is its
counterfactuals (standing lenses load-bearing) and the failure-mode wave sharpens the
structure-is-artifact and reinvention modes the counterfactuals live or die on. Run the
grill-me skill (research failure-mode branch) or knowingly skip; record each wave here.
Route any resulting change to amend-plan.
```

## Run outcome

Terminal outcome: **validated finding** (mixed shape, as expected). Six lenses + three verification
vouchers ran to completion; every folded claim is grounded against a primary source. The run
validated the headline (neither built example is the hero; the engineered
[[same-stats-different-graphs|Datasaurus-for-networks]] work is), and *corrected* the backbone on the
points the panel got wrong. Folded into
`wiki/wiki/analyses/hiveplotlib-bioinformatics-examples.md` (producer-path enrichment, under
maintainer approval, not committed).

**The adversary's two planning-mode `must-fix` items were both accepted, and here is how the run
honored them:**

- **Item 1 (absence claims can never reach `validated finding`; ceiling is `validated inconclusive`
  with a named search surface).** Honored. The two headline absence-shaped claims were *not* laundered
  into cited positives. The panel located **positive refuters** for both, which is the outcome item 1
  was protecting against a false version of: the "no one has published a C. elegans connectome hive
  plot" and "the hairball framing is ours" absences were *killed* by a located positive
  ([[tabacof-2013-connectome-hive-plots|Tabacof 2013]], with its own verbatim first-application
  claim), and the "no published male-vs-hermaphrodite comparison" absence was killed by
  [[cook-2019-both-sexes-connectome|Cook 2019]]. The one absence that survived (no confirmed published
  *differential-connectome hive plot*) is landed as **weak-evidence absence**, explicitly framed
  ("searched but unindexed literature could hide one"), never as a validated first. So no absence
  reached `validated finding`.
- **Item 2 (re-charter the disconfirmation lens to hunt the mundane refuters, not just the celebrated
  ones; write conclusions disconfirmation-first).** Honored, and it is what produced the run's most
  valuable output. The lens did not come back empty rubber-stamping our conclusions; it surfaced the
  **boring ways we were wrong**: the role-partition vocabulary is Yu & Gerstein 2006's, not
  Krzywinski's (mis-attribution); the position metadata *was* cleanly sourceable all along and we
  wrongly called it a dead end (mis-sourcing); the GRN license is a restrictive EULA, not CC-BY, and
  our student-mirror fetch is itself a violation (mis-read with downstream cost). These mundane
  refuters, exactly the ones item 2 demanded the lens be chartered to find, are the corrections now in
  the page.

**Fold discipline applied (adversary `worth-discussing` item 5).** The FOLD-GUARD held: a source
grounds a conclusion only where its quote makes *that* specific point. Topically-adjacent material was
kept out of the verdict sections and routed to related-reading framing instead: the field-wide
"comparing networks by density is hard" limitation (NodeTrix) is named as *related reading* under
"What was tried and dropped," not as primary grounding for our specific single-plot-collapses claim.
The refuted Abasy third-party-rights sub-claim was dropped entirely (no such text on the live site).

**Headline result.** Both built examples are **credible real-data re-tellings of already-published
figures** (Tabacof 2013 for the connectome hairball-vs-hiveplot; Cook 2019 for the sex dimorphism;
Krzywinski 2012 for the E. coli role-partition GRN), so the honest delta is **execution and medium**
(a deterministic, reproducible hiveplotlib re-encoding), not primacy. Two attributions corrected (GRN
role vocabulary = Yu & Gerstein 2006; GRN data = restrictive EULA, not CC-BY, with no clean permissive
swap). Two "dead ends" re-opened as buildable opportunities (C. elegans AP-position structural figure;
the male-tail-position gap the one residual blocker). The Datasaurus reframe is grounded, not asserted:
Kobourov 2018 did the matched-stats device on networks first with a spring reveal, so hiveplotlib's
narrow honest delta is the **deterministic** (data-is-a-function, not a seed) hive-plot medium.
