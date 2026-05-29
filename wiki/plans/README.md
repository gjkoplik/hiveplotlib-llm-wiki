---
title: Plans
type: index
updated: 2026-05-17
---

# Plans

Working plans live here. They are scratch work in progress, not curated wiki content. Expect verbose, churn-heavy documents that change shape from day to day, with dead branches the author explored along the way.

## Plans vs. ADRs

- **`plans/`** — living working documents. Don't expect them to read cleanly end-to-end.
- **`adr/`** — durable, distilled records of structural decisions, written after a plan ships. Append-only, terse, every entry earned its place.

If you're browsing the wiki for the *current state* of a design call, read the ADR. The plan is the work-in-progress version.

## Active vs. archived

- **`plans/`** (this directory) — active or unstarted plans only. Browsing here shows what's still in flight.
- **`plans/archived/`** — shipped plans, kept for history. A plan moves there once its work has fully shipped, typically right after it promotes to an ADR, or after shipping for plans that won't become ADRs.

Archiving is a manual move on the human's confirmation. Research Liaison proposes it (at ADR promotion, or in its post-task pass for non-ADR plans) but never moves files itself. Hold a technically-done plan in `plans/` if it's still bundled with unshipped work; decline the archive suggestion until the bundle ships.

## Conventions

- Filenames are kebab-case, no numeric prefix: `networkx-metric-expansion-and-backend-refactor.md`, not `0003-…`.
- Plans are append-friendly during execution but not append-only after shipping; the ADR captures the durable summary.
- New plans always start in `plans/`, never directly in `plans/archived/`.

## See Also

- [[index|Wiki Index]]
