# Research plan: NN-training hive-plot novelty grounding (rebuild `nn-training-dynamics-p2cp-exploration`)

<!--
Research-plan shape (deliberately light per the plan-template). No workstream / done-when
ceremony; conventions (shallow-panel dispatch, standing lenses, bounds, grounding, durable
landing) live in the research-track skill. This plan carries only the per-plan fill-in.
Consumer: hiveplotlib. Evidence mode: literature.
-->

## Question

Under the bounded, grounded, adversary-validated research-track machinery, re-ground and re-validate the novelty / prior-art position of the four figures the throwaway `hiveplotlib-nn-viz` repo built (softmax-probability P2CP over training, cross-layer neuron **co-activation** in polar form, the output-vs-hidden-space data finding, and the newly-added **lock-in vs bounce** persistence figure), and land a rebuilt durable analysis page current with all four findings and both datasets (MNIST + Fashion-MNIST). The prior verdict came from the OLD ~97-agent deep-research workflow and was never put through the new bounded panel; finding #4 has had **zero** prior-art check.

The four findings are not equal weight: #2 (co-activation, polar) is the headline novelty claim; #4 (lock-in) is the un-scanned new question; #1 (softmax P2CP) is a positioning-against-Grand-Tour claim already partly scanned; #3 is a data finding the literature cannot settle but the page must frame.

**Honest marginal yield (state up front).** This is a *re-run* of a prior scan, and the panel's genuinely new work is narrow: #4's first-ever prior-art check, a fresh confirm that no #2 refuter has surfaced since the last scan, and a grounded rebuilt page. #1 and #3 are near-certain re-confirmations of the prior scan's verdict, not fresh adjudications. Two of the three staleness fixes (Fashion-MNIST coverage, the HivePlotMatrix entry-point correction) are **zero-agent page edits** the liaison makes at write-time, not panel work. The panel is worth its ~15 agents for #4 + the #2 refuter-recheck + grounding, not for re-deriving #1/#3.

## Candidate stories / hypotheses

Per finding, the competing explanations the run weighs:

- **#1 Softmax-probability P2CP positioning.** "Incremental-toward-novel: the Grand Tour owns the *use case* (animating MNIST softmax over training, per-class convergence to simplex corners); our differentiator is the *layout* (generalizable `p2cp_n_axes()` on the softmax, confusion as secondary lobes) vs. a bespoke linear-projection interactive" **vs.** "the layout difference is not enough to claim any novelty; frame purely as an application of the Grand Tour's use case."
- **#2 Co-activation, polar (HEADLINE).** "Defensibly novel: no located prior art draws neuron-to-neuron *observed co-activation* (not weights) in radial or parallel-coordinates form over training; the activation-viz canon (ActiVis, CNNVis, M-PHATE, Activation Atlas, Grand Tour) uses matrices, t-SNE/projections, or edge-bundled DAGs, never PC/polar" **vs.** "this is a reinvention of a known circuit / co-activation graph view (the refuter the counter-evidence lens hunts)."
- **#3 Output-vs-hidden-space (data finding).** "This MLP's hidden code is distributed (~20-38 of 64 hidden1 neurons active per image, roughly static over training), so the discriminative structure lives in output space and there is no clean per-digit hidden pathway; this is *why* Figure B keys off class-selectivity, not raw activation." Literature cannot validate the measurement; the lens role is to place the "distributed vs. sparse/modular hidden codes in small MLPs" claim against known results, and the page must frame it as a data observation, not a lit-validated finding.
- **#4 Lock-in vs bounce (NEW, un-scanned).** "The committed-vs-transient co-activation split (present in the selective set at every checkpoint of a trailing 6-window vs. transient, committed edges crystallizing out of a transient-gray fog, commitment tracking class difficulty) is a *novel visual* of a phenomenon" **vs.** "pathway lock-in / crystallization of connectivity over training is a known, named phenomenon" (lottery-ticket hypothesis, pruning-during-training, critical learning periods, emergent modularity, gradient fixation). Two separable sub-questions: is the *phenomenon* named in the literature, and is the *visual* of it novel? **#4 caps at validated-inconclusive (post-impl adversary must-fix, grill-confirmed).** #4's data-reality (knob-artifact) question, whether the committed/transient split is a real connectivity phenomenon or an artifact of the selectivity threshold / 6-checkpoint trailing window, lands as **validated-inconclusive**; the literature evidence mode cannot settle it and that check is explicitly deferred to a future **data-validity-mode** run (threshold/window sweep). #4 may **not** land as a plain "validated finding." The visual-novelty sub-claim may be stated **only** with the selectivity-threshold / 6-checkpoint-window caveat physically attached in the same sentence, never parked in a separate caveats slot the reader skims past. Naming the visual as novel while the phenomenon it depicts is untested would dress a knob-artifact as a finding.

## Failure-mode rubric

Populated in full by grill-me's failure-mode wave (research branch); seeded here, tuned to this run. Consumed by the adversary at convergence.

- **Already-known (reinvention).** The novelty claim, especially #2 and #4, is actually a reinvention of a located prior work. The counter-evidence lens must surface the specific refuter (a paper that drew neuron co-activation in PC/polar, or one that visualized pathway lock-in over training in a comparable form), not merely fail to find one.
- **Uncontrolled-comparison / novelty-overclaim.** Claiming "novel" from *absence* of located art. "None located after an adversarial search" is not "provably first"; polar-PC-for-NNs is recent (~2021+), so an unindexed thesis / workshop / blog could exist. Every novelty verdict must carry this qualifier and the page must state it. **Grey-lit reach clause (rigor fix):** a "none located" conclusion is admissible **only if** the named grey-literature venues (VISxAI, ML-viz / ICLR / NeurIPS workshop tracks, theses, distill-style blogs) were actually swept; a qualifier on the page does **not** substitute for having looked there. **Search-scope, not grade (rigor fix):** the convergence verdicts on #1 and #2 must be phrased as **search-scope statements** ("no refuter surfaced under a bounded search that swept [named venues]"), never as a first-ness grade. This is the single most likely overclaim site.
- **Inherited-vs-earned verdict laundering (rigor fix).** This is a re-run. The re-validation must state, per finding, which verdicts the panel **changed** vs. **inherited** from the prior scan; the rebuilt page must not launder an inherited verdict as freshly earned (the confident tone of a fresh page must not hide that the panel moved no conclusion on #1/#3).
- **Grounding-failure.** A claim (novelty verdict, canon characterization, phenomenon name) asserted without a faithful supporting quote from the cited source. Claim-maker != voucher; default-refuted at verify.
- **Data-side caveat (finding #4, must-name-not-settle).** The committed-vs-transient lock-in split could be an **artifact of the selectivity threshold or the trailing-window length** (6 checkpoints), not a real connectivity phenomenon. The literature evidence mode cannot settle this (it is a data-validity question, that mode is deferred); the run must **name it as an open caveat on the page**, not silently present the split as established. Same class of caveat applies to #3's "~20-38 active neurons" measurement.
- **Source-trustworthiness.** A faithfully-quoted claim from a weak-provenance source (a blog, an unreviewed preprint) passes verify untouched; record the trust judgment alongside each citation so the page does not lean a headline novelty verdict on thin provenance.

## Lenses + bound

Five lenses, all disjoint, each blind to the others, hard **20-agent cap**. Two standing seats floored first, then three orthogonality slices.

- **[STANDING] Prior-art / counterfactual lens.** How NN training is normally visualized: Grand Tour (distill.pub/2020/grand-tour), ConfusionFlow (arXiv 1910.00969), ActiVis (arXiv 1704.01942), CNNVis (arXiv 1604.07043), M-PHATE, Activation Atlas, TensorBoard, loss-landscape, CKA/SVCCA. Central question: where, if ever, do **radial or parallel-coordinates** layouts appear for NNs? Grounds #1's Grand-Tour positioning and the canon characterization behind #2.
- **[STANDING] Counter-evidence lens (blind to the confirming lenses).** Actively hunt the refuter: anyone who **has** drawn neuron-to-neuron co-activation in PC/polar form, or shown pathway lock-in over training in a comparable visual. Carries not-fully-confirming hits to synthesis labelled as such; never drops them. This is the generation-time disconfirmation counterweight for #2 and #4. **Grey-lit reach clause (rigor fix):** must actually sweep the named grey-lit venues (VISxAI, ML-viz / ICLR / NeurIPS workshop tracks, theses, distill-style blogs) before any "none located" conclusion may stand.
- **Co-activation / circuit-viz lens (orthogonality).** Neuron-to-neuron co-activation and **circuit / mechanistic-interpretability** graph views specifically (not general activation viz): the interpretability-circuits literature, feature-visualization circuit diagrams, attribution graphs. Distinct body of material and search strategy from the general-activation-viz canon the prior-art lens covers. Grounds #2's headline. **Grey-lit reach clause (rigor fix):** must sweep the named grey-lit venues before any "none located" conclusion may stand.
- **Training-dynamics-of-connectivity lens (orthogonality).** The phenomenon behind #4: lottery-ticket hypothesis, pruning-during-training, critical learning periods, emergent modularity, gradient fixation / weight freezing over training. Answers "is pathway lock-in a known, named phenomenon" independent of whether the *visual* is novel. Grounds #4 and touches #3 (distributed vs. modular hidden codes).
- **Visual-encoding / PC-radial-technique lens (orthogonality, NEW — closes the adversary's blind-spot item).** The one axis every other lens skips: **the encoding itself**, blind to the ML canon and to the other lenses. The parallel-coordinates lineage (Inselberg PC), radial-vs-cartesian perception studies, and parallel-coordinates / radial layouts for high-dimensional **classifier output** *outside* the NN-viz canon. Rationale: the whole #1/#2 novelty claim is about the *encoding*, yet every other lens is organized by ML topic, so the encoding axis (where a #1/#2 refuter is as likely to sit as in the ML-viz canon) was uncovered. **Grey-lit reach clause applies:** must sweep the named grey-lit venues before any "none located" here stands. Grounds the layout-differentiator half of #1 and #2.

**Bound math (addable model, research-track skill):** `1 scope-refine + 5 lenses + ~7-8 verify vouchers + 1 synth ≈ 15`, under the 20 cap. Standing-lens floor unchanged: 2 standing seats (prior-art/counterfactual, counter-evidence) + 3 orthogonality slices (circuit-viz, training-dynamics-of-connectivity, visual-encoding). The vouchers cover roughly 5-6 load-bearing claims at one voucher each (the canon characterization, the Grand-Tour use-case ownership, the co-activation-in-PC absence, the lock-in-phenomenon naming, the distributed-hidden-code result, the PC/radial-encoding prior-art absence) plus ~2 generation-flagged second vouchers on single-quote load-bearing claims (the #2 absence claim and the #4 phenomenon-naming claim are the likeliest to rest on a single source and are the most load-bearing, so they are the expected second-vote candidates).

## Validation criteria

A validated finding must be **grounded** (each claim carries a faithful supporting quote), **independently verified** (claim-maker != voucher, default-refuted at verify, second voucher on generation-flagged single-quote load-bearing claims), and **adversary-convergence-cleared** against the failure-mode rubric above. The counter-evidence lens's not-fully-confirming hits are carried to synthesis, never dropped, and framed as such.

Three additional criteria bind this re-run (grill + post-impl adversary must-fix, all confirmed):

- **#4 cannot land as a plain "validated finding."** Its best terminal outcome is **validated-inconclusive** on the data-reality (knob-artifact) question; the visual-novelty sub-claim ships only with the selectivity-threshold / 6-checkpoint-window caveat attached in the same sentence. The threshold/window sweep is deferred to a future data-validity-mode run.
- **Novelty verdicts phrased as search-scope, not grade.** The #1 and #2 convergence verdicts must read "no refuter surfaced under a bounded search that swept [named venues]," never a first-ness claim; and "none located" is admissible only if the grey-lit venues were actually swept.
- **Inherited-vs-earned stated per finding.** The run must record which verdicts the panel changed vs. inherited from the prior scan; the page must not launder an inherited #1/#3 re-confirmation as freshly earned.

The run classifies its terminal outcome honestly into one of three legitimate outcomes:

- **validated finding** — confident, grounded verdict on the four findings (the expected outcome given the prior scan already reached a defensible verdict on #1-#3); lands the full rebuilt page.
- **validated inconclusive** — e.g. if #4's phenomenon-naming question cannot be resolved either way, that finding lands as a pursued-to-a-negative "the literature will not settle whether lock-in is a named phenomenon" reference, and the page carries it in its `what-was-inconclusive-and-why` slot. A first-class outcome, not a failure.
- **nothing-cohered** — the degenerate case (unlikely here, since #1-#3 already have a prior verdict to re-validate); would land only a minimal breadcrumb, never a thin finding-shaped page.

Note the mixed shape this run expects: #1-#3 as validated findings, #4 potentially validated-inconclusive on the naming sub-question while its visual-novelty sub-question may validate. The page carries findings and inconclusives side by side; both are durable.

## Destination artifact

Rebuild `wiki/wiki/analyses/nn-training-dynamics-p2cp-exploration.md`. **Keep the slug / filename** (grill-confirmed): the 5 inbound wikilinks and the examples-catalog row keep resolving with no repoint. **Update the displayed H1 and frontmatter `title`** to the new headline (cross-layer co-activation + lock-in across MNIST and Fashion-MNIST). No superseding-pointer dance is needed since the slug is retained. The rebuild must fix the three staleness defects of the current page:

1. it predates finding #4 (lock-in) entirely,
2. it is **MNIST-only**; Fashion-MNIST has since been added and is the stronger headline dataset, so both datasets belong on the rebuilt page. The liaison sources Fashion-MNIST data facts (test accuracy, garment confusions, etc.) from the repo's `PLAN.md` / mlflow **at write-time**, not from this brief,
3. it lists `BaseHivePlot` as Figure B's entry point, but Figure B was refactored onto the generic `HivePlotMatrix` (with its built-in edge colorbar and cross-cell density unification); the usage notes must be corrected. (Fixes 2 and 3 are zero-agent page edits, not panel work.)

Landed via research-liaison's producer path under maintainer approval (not auto-committed). Yield determines shape: full page for the validated findings + any validated-inconclusive; the throwaway repo's `PLAN.md` remains the canonical prose summary the page points at.

## Minor pivots vs. amend-plan

Iteration lives inside the bounded run: refining a lens or dropping a dead angle does not force an amend-plan. Only a fundamental change of the research question mid-run is an amend-plan.

---

## Prior ADRs / design docs

None relevant (per the brief; do not hunt). The prior work to reference is the STALE analysis page being rebuilt (`wiki/wiki/analyses/nn-training-dynamics-p2cp-exploration.md`), the throwaway repo's `PLAN.md` (canonical prose), and the OLD-process scan verdict this run re-validates.

## Failure modes

See `## Failure-mode rubric` above (the research-plan shape carries the rubric there). grill-me's failure-mode wave (research branch) populates it in full before dispatch.

## Adversary review

### Adversary's challenge (planning mode)

```
Status: challenge (6 items)
Plan reviewed: wiki/wiki/plans/nn-hiveplot-research-run.md (cold, pre-grill)
Angles worked: premise (why re-run) / approach (lens decomposition, search reach) /
  size-and-maintenance (scope vs. the two staleness fixes that don't need a panel)

Items:
  - [worth-discussing] The panel's headline value is scoped to ONE finding (#4). #1-#3
    already have a defensible prior verdict this run only re-validates; the plan says so
    (Question para 2, Validation-criteria "expected outcome given the prior scan already
    reached a defensible verdict on #1-#3"). Two of the three staleness defects
    (Fashion-MNIST coverage, HivePlotMatrix entry-point fix) are page edits that need no
    literature panel at all. So the run's real new-yield surface is: #4's naming question
    + a fresh confirm that no #2 refuter surfaced since the last scan. That is a narrower
    justification than "re-ground and re-validate all four." — at Question / Validation
    criteria
    Rubric: no rubric yet — pre-grill
    Push: State the run's marginal yield explicitly up front: which findings does the
    panel actually move vs. which are page-mechanics the liaison could fix without agents.
    If #1/#3 re-validation is near-certain to re-confirm, say so and consider flooring
    fewer lenses on them (spend the cap on #2's refuter-hunt and #4). Answer for the grill:
    is the panel worth its ~19 agents, or is it #4 + two page edits wearing a panel's coat?

  - [worth-discussing] "No located prior art" (#2) is an absence claim, and the plan's
    own safeguards are the rubric qualifier + the counter-evidence lens — but nothing in
    the lens spec forces the search into the venues where a #2 refuter would actually live.
    The three content lenses name indexed/famous sources (Grand Tour, ActiVis, CNNVis,
    M-PHATE, lottery-ticket) and "the interpretability-circuits literature." A polar/PC
    neuron-co-activation drawing is exactly the kind of thing that shows up in a viz-
    conference SHORT paper, a workshop track (VISxAI, ICLR/NeurIPS workshops), a thesis
    chapter, or a distill-style blog — none of which the lens list points at by name.
    Absence found only in the canon is weaker than the headline wants. — at Lenses (circuit-
    viz + counter-evidence)
    Rubric: maps to "Uncontrolled-comparison / novelty-overclaim" (the ~2021+ unindexed
    thesis/workshop caveat) — the caveat is named but the SEARCH REACH that would honor it
    is not specified
    Push: Name the grey-literature venues the counter-evidence and circuit-viz lenses must
    sweep (VISxAI, ML-viz workshops, theses, distill/blog) before "none located" is allowed
    to stand. A qualifier on the page doesn't substitute for actually looking there.

  - [worth-discussing] "None located" → "defensibly novel" is a load-bearing inference the
    plan makes lightly. The candidate-story text for #2 already carries the word "Defensibly
    novel" as the pro-position; the risk is the run treats a null search result as evidence
    FOR novelty rather than as failure-to-refute. Absence of a refuter after a bounded
    20-agent panel is a much weaker warrant than the phrase "defensibly novel" implies. —
    at Candidate stories #2 / Failure-mode rubric
    Rubric: maps to "Already-known (reinvention)" and "Uncontrolled-comparison"
    Push: Require the convergence verdict on #2 to be phrased as "no refuter surfaced under
    a bounded search that swept [named venues]" and NOT as a novelty grade. The page's #2
    verdict must be a search-scope statement, not a first-ness claim. This is the single
    most likely place the run overclaims.

  - [must-fix] Finding #4's knob-artifact question is UNANSWERABLE in this run's evidence
    mode, and the plan half-admits it but still lets #4 carry a "novel visual" sub-verdict.
    The lock-in/bounce split depends on the prototype's own selectivity threshold and the
    6-checkpoint trailing window (rubric "Data-side caveat" says exactly this). Literature
    mode is deferred from settling it. So the run CAN name whether lock-in is a phenomenon,
    but it CANNOT know whether THIS visual shows a real phenomenon or a threshold artifact —
    yet the plan's #4 candidate story ("a novel visual OF a phenomenon") and the Validation-
    criteria "its visual-novelty sub-question may validate" both let the visual sub-verdict
    proceed as if the phenomenon-behind-it were established. A validated "the visual is
    novel" reads to a page reader as "the thing it shows is real." — at Candidate stories #4
    / Validation criteria / Failure-mode rubric
    Rubric: maps to "Data-side caveat (finding #4, must-name-not-settle)"
    Push: The visual-novelty sub-verdict for #4 must be gated so it cannot ship without the
    knob-artifact caveat physically attached in the same breath on the page — not in a
    separate caveats slot the reader skims past. Better: forbid #4 from landing as a
    "validated finding"; cap its best outcome at "validated inconclusive" until a data-
    validity run (the deferred mode) can test whether the split survives a threshold/window
    sweep. Naming the visual as novel while the phenomenon it depicts is untested is dressing
    a knob-artifact as a finding.

  - [worth-discussing] Lens-decomposition blind spot: no lens covers HUMAN-FACTORS / PC-
    perception prior art. The whole #1/#2 novelty argument leans on "polar/PC layout for NNs
    is the differentiator," but nobody searches the parallel-coordinates and radial-viz
    literature ON ITS OWN TERMS (Inselberg PC lineage, radial-vs-cartesian perception
    studies, PC-for-high-dim-classifier-output work outside the NN-viz canon). A refuter
    could live in the viz-technique literature that the NN-specific lenses (prior-art,
    circuit-viz, training-dynamics) all skip because they're organized by ML topic, not by
    visual encoding. The claim is about the ENCODING; no lens is organized by encoding. — at
    Lenses + bound
    Rubric: no entry — flagging anyway (the rubric is organized around the ML-side refuter,
    not the encoding-side one)
    Push: Either add a viz-encoding lens (PC/radial-for-classifiers, blind to the ML canon)
    or explicitly record that the encoding-side prior art is out of scope and why. As
    written the disjoint-lens claim ("all disjoint") is true but leaves a whole axis
    (encoding literature) uncovered, and that axis is where a #1/#2 refuter is as likely to
    sit as the ML-viz canon.

  - [low-confidence] Rubric completeness: the failure-mode rubric names reinvention,
    overclaim, grounding, data-caveat, and source-trust — a solid set for a literature run.
    The one mode I'd expect for THIS run that isn't crisply named: "re-run reconfirms the
    stale verdict and adds nothing, but the fresh page's confident tone hides that the new
    panel changed no conclusion." A no-change re-validation is a legitimate outcome, but the
    rebuilt page should not read as if the panel earned the verdict when it inherited it. —
    at Failure-mode rubric
    Rubric: no entry — flagging anyway
    Push: Consider a rubric line: "Re-validation must state which verdicts the panel changed
    vs. inherited from the prior scan; a rebuilt page must not launder an inherited verdict
    as freshly earned." Ties to item 1's premise concern.
```

Note: no `## Failure modes` rubric was present at this cold pre-grill read beyond
the seeded `## Failure-mode rubric` (grill's failure-mode wave populates it in full
before dispatch); the three mandated angles were worked without a completed rubric.
No item rose to `existential-must-fix` — the run has genuine marginal yield (#4 naming
+ two page fixes + a fresh refuter-check), so the premise is thin, not dead.

### Adversary post-grill rubric-check (conditional)

```
Knowingly SKIPPED. Every amendment above implements the adversary's planning-mode
challenge verbatim (the encoding-lens blind spot = item 5; the grey-lit reach = item 2;
novelty-as-search-scope = item 3; the #4 knob-artifact cap = the must-fix item 4;
inherited-vs-earned = item 6), and the grill's failure-mode wave named no new mode the
cold pre-grill pass had not already covered. Per the harness invocation trigger, a
delta-check against newly-named modes is skipped when the cold pass already covered them:
Status: clean — no new modes to check.
```

### Adversary post-impl (convergence gate)

```
Status: propose (2 must-fix, 3 worth-discussing) — panel yield UPHELD as mixed;
  no finding killed, but three phrasings must tighten before the page is written.
Artifact reviewed: 5 blind lenses + 8 verify vouchers (all VERIFIED w/ nuance),
  synthesized findings #1-#4 + cross-cutting method argument. Read as distilled
  conclusions, not sources.
Dispositions held: yes. Every planning-mode challenge item shipped into the plan's
  criteria (encoding lens added, grey-lit reach clause, novelty-as-search-scope,
  #4 capped at validated-inconclusive, inherited-vs-earned rubric line) and every
  one is honored in the proposed findings. No scope balloon: the panel fired the
  ~15 agents the bound predicted, did not silently promote #4, did not launder an
  inherited verdict as earned.

Adversary verdicts (per finding, against the failure-mode rubric):

  #1 Softmax-P2CP — VALIDATED as positioning, not novelty. [worth-discussing below]
     Grand-Tour use-case ownership: voucher VERIFIED. Radial/PC absent from canon:
     grounded by the prior-art lens. The proposed framing ("generalizable P2CP
     complement to the Grand Tour," not "first") is exactly the search-scope, not
     first-ness phrasing the rubric demands. Rubric "Uncontrolled-comparison":
     satisfied. One soft spot flagged below (RadViz-on-simplex under-searched).

  #2 Cross-layer co-activation, polar (HEADLINE) — VALIDATED as a bounded
     search-scope statement, NOT killed, NOT gerrymandered. This is the item to
     scrutinize hardest (question 1). Walking the three-near-miss union as a
     hostile reader: does Horta + NeuroBreak + TF-Playground TOGETHER preempt the
     intersection? No — but only because each fails a DIFFERENT axis, and I checked
     that the failing axes are real distinctions, not a novelty-preserving gerrymander:
       - Horta 2021 (co-activation graph, even Fashion-MNIST, voucher VERIFIED):
         has the SEMANTIC (observed co-activation, not attribution/weights) but
         force-directed Gephi, zero polar/PC. The rendering axis is a real
         distinction, not a cosmetic one — force-directed vs. polar is the whole
         hive-plot thesis.
       - NeuroBreak (radial neuron view, voucher VERIFIED): has the RENDERING
         (radial) but attribution edges + dashboard, not observed co-activation.
       - TF-Playground: has TRAINING-DYNAMICS + connectivity but Cartesian.
     The union covers {semantic}, {rendering}, {dynamics} pairwise but no single
     located work carries all three. That is a genuine three-property intersection,
     not a gerrymander — the tell of a gerrymander is a distinction with no
     load-bearing reason (e.g. "ours is on MNIST not CIFAR"); here each axis is a
     first-class design property. VERDICT STANDS, but with the must-fix phrasing
     gate below and one grounding downgrade.
     Rubric: "Already-known" (refuter demanded, none located across all three axes
     simultaneously) + "Uncontrolled-comparison" (search-scope phrasing honored).

  #3 Output-vs-hidden-space — VALIDATED as a data observation, correctly NOT
     lit-validated. The measurement (~20-38/64 active) is data-side; distributed
     dense codes are textbook; "distributed specialization" cautions against
     reading a distributed code as modular pathways. Framed as observation, not
     finding. Rubric "Data-side caveat": honored (same class as #4's measurement
     caveat). Clean.

  #4 Lock-in vs bounce — VALIDATED-INCONCLUSIVE, correctly held. This is the
     scrutiny question 2 (does residual "novel visual" phrasing smuggle in "the
     phenomenon is real"?). Checked: the phenomenon IS named at layer/weight level
     (critical periods, LTH early-phase, voucher VERIFIED) but ONLY as an analogy
     (voucher confirmed Achille never says "co-activation") and is ACTIVELY
     CONTESTED (dynamic sparse training / ITOP finds churn beneficial, voucher
     VERIFIED). The committed/transient-over-6-window construct is UNNAMED. The
     knob-artifact question is unsettleable in literature mode, deferred to
     data-validity. The proposal states visual-novelty may be stated ONLY with the
     knob-artifact + literature-is-split caveats "attached in the same breath."
     That is the correct gate. Rubric "Data-side caveat (#4 must-name-not-settle)":
     honored. BUT see must-fix #2 — the caveat-in-same-breath rule must survive the
     translation from this verdict to the PAGE prose, and the "literature is split"
     (ITOP contests it) half must ride alongside the knob-artifact half, not just
     the threshold/window caveat.

  Cross-cutting (polar-vs-straight-PC) — VALIDATED as scoped extrapolation. The
     674-participant PubMed study (voucher VERIFIED) finds radial underperforms
     Cartesian on time/position, but NOT tested on PC/P2CP (honest extrapolation
     flagged), and radial's single-dimension-focus advantage is the red-home-axis
     design's exploit. Honestly bounded. Clean as long as "extrapolation" and "not
     tested on PC specifically" ride with the claim on the page.

  Inherited-vs-earned (scrutiny question 3) — HONEST. The "panel changed the
     verdict" claim holds: the panel demoted encoding-novelty (was implicitly
     novel, now prior art per #2/encoding), sharpened #2 from a bare novelty claim
     to a bounded three-axis intersection, and ADDED #4's counter-current (ITOP
     contest) that the prior ~97-agent scan never had (#4 had zero prior check).
     That is genuine marginal yield beyond re-confirmation, not laundering. #1/#3
     are correctly labelled near-certain re-confirmations, not dressed as fresh
     adjudications. Rubric "Inherited-vs-earned laundering": satisfied.

Concerns:
  - [must-fix] #2's "VINCI-2010 is PC-of-an-NN" near-miss and the "Horta even on
    Fashion-MNIST" claim are load-bearing to the intersection argument and each
    appears to rest on a SINGLE voucher. The intersection's defensibility depends
    on these near-misses being characterized correctly (if VINCI actually rendered
    co-activation rather than training-samples, the intersection collapses). Before
    the page ships, confirm each of the four near-miss characterizations (Horta,
    NeuroBreak, TF-Playground, VINCI) carries a faithful supporting quote pinning
    the specific axis it FAILS on — the claim is not "these exist" but "each lacks
    property X," and the X-it-lacks is what must be quoted, not merely the paper's
    existence. — at synthesized #2 / near-miss table
    Rubric: "Grounding-failure" (claim-maker != voucher; the disqualifying-axis
    claim needs its own faithful quote, not just the paper's presence)

  - [must-fix] #4's page prose must carry BOTH caveats in the same breath, not one.
    The proposal names the knob-artifact caveat (selectivity-threshold / 6-window)
    but the literature-is-SPLIT half (ITOP/dynamic-sparse-training actively contests
    that early connectivity fixes) is an independent load-bearing caveat: a reader
    seeing "critical periods names this" without "but dynamic-sparse-training
    contests it" gets a one-sided phenomenon. The plan's rule says "visual-novelty
    only with the threshold/window caveat attached"; extend that on the page to
    "...and with the literature-is-contested note attached." Otherwise #4 smuggles
    in "the phenomenon is settled" via a one-sided literature summary even while the
    knob-artifact caveat is present. — at #4 / Validation criteria
    Rubric: "Data-side caveat (#4 must-name-not-settle)" + "Uncontrolled-comparison"

  - [worth-discussing] #1/#2's "RadViz-on-a-probability-simplex" was flagged
    UNDER-SEARCHED by the encoding lens itself, and Figure A is "conceptually
    adjacent" to it. RadViz IS a named radial technique for simplex/probability
    data; if a RadViz-of-softmax-over-training exists, it is a direct #1 refuter and
    a partial #2 one. This is an admitted hole in the search REACH, not a null
    result. The page's #1/#2 search-scope statement must name RadViz-on-simplex as
    an explicitly-unswept venue (like the theses/ACM-complete/VISxAI-per-submission
    gaps already named), so the "none located" is honest about what it did not
    reach. — at synthesized #1 / #2 search-scope statement
    Rubric: "Uncontrolled-comparison / novelty-overclaim" (grey-lit reach clause:
    a named-but-unswept venue must be disclosed, not silently omitted)

  - [worth-discussing] The #2 verdict phrasing "no refuter surfaced under a bounded
    search that swept arXiv/distill/transformer-circuits/VIS-index; ACM-complete,
    VISxAI-per-submission, theses NOT fully swept" is the honest form — but confirm
    the PAGE reproduces the NOT-swept list verbatim, not just the swept list. The
    rubric's search-scope-not-grade fix fails silently if the page states what was
    searched and drops what was not. The distinction between "swept" and "not fully
    swept" venues is the entire warrant separating a search-scope statement from a
    first-ness grade. — at Destination artifact / page prose
    Rubric: "Uncontrolled-comparison" (search-scope-not-grade)

  - [low-confidence] Source-trustworthiness is not recorded per-citation in the
    distilled findings I was handed. The rubric asks that a trust judgment ride
    alongside each citation so a headline (#2) does not lean on thin provenance.
    Horta 2021, NeuroBreak, NeuroBreak, VINCI-2010 provenance (peer-reviewed vs.
    preprint vs. workshop) is not surfaced here. Likely fine (these read as
    published venues), hence low-confidence, but the page should carry the trust
    tag the rubric asks for, especially on the four #2 near-misses the headline
    rests on. — at #2 near-miss citations
    Rubric: "Source-trustworthiness"

No finding killed. The mixed yield (validated #1/#2/#3 as scoped + validated-
inconclusive #4) is UPHELD as honest and un-laundered. The two must-fix items are
phrasing/grounding gates on the PAGE, not verdict reversals: #2's near-miss
disqualifying-axis quotes must be pinned, and #4 must carry the literature-split
caveat alongside the knob-artifact one. Both route to orchestrator amend-plan
(page-write gates), like the other post-impl critics.
```

## Alignment (grill)

```
Ran (dispatching session, inline with the maintainer), armed with the adversary's
cold pre-grill challenge. Waves covered:
  - premise / scope wave: confirmed the run has genuine marginal yield (#4's first
    prior-art check + a fresh #2-refuter recheck + a grounded page); #1/#3 are near-
    certain re-confirmations and two of the three staleness fixes are zero-agent page
    edits. Premise held (not existential); marginal yield now stated up front in the
    Question.
  - failure-mode wave (research branch): named the grey-lit reach, novelty-as-search-
    scope, and inherited-vs-earned modes into the rubric.

Maintainer dispositions (folded into this plan via amend-plan):
  - Encoding lens ADDED (fifth lens, visual-encoding / PC-radial-technique) — closes
    the adversary's blind-spot item.
  - #4 CAPPED at validated-inconclusive on the knob-artifact question; visual-novelty
    sub-claim only with the threshold/window caveat attached in the same sentence;
    threshold/window sweep deferred to a data-validity-mode run.
  - Destination: KEEP the slug, UPDATE the H1/title only.
  - Four rigor fixes ADOPTED: grey-lit reach clause, novelty-phrased-as-search-scope,
    inherited-vs-earned rubric line, marginal-yield-stated-up-front.

Pre-flight estimate: ~15 agents (cap 20) — auto-proceed.
```
