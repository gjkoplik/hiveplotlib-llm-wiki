---
title: Specs
type: index
updated: 2026-08-30
---

# Specs

A spec is the signed intent artifact for a piece of work: one page stating the outcome in the vocabulary a user of the library would use, and the literal code a user types to obtain it. It is signed before any plan is written, and re-signed, dated, whenever it changes. Agents draft and propose; the signature is always the maintainer's.

Specs are short and meant to be read straight through. If a page here runs long or reads like an execution document, it has drifted from what the directory is for.

## Specs vs. plans vs. ADRs

- **`specs/`** (this directory): what the work is for, and what a user gets. Signed.
- **`plans/`**: how it gets built. Living scratch work, verbose and churn-heavy; don't expect it to read cleanly end-to-end.
- **`adr/`**: durable, distilled records of structural decisions, written after a plan ships.

If you're browsing to find out what a piece of work was supposed to deliver, read the spec. For the shape of a design call already made and shipped, read the ADR.

## Lifecycle

A spec closes when its outcome statement is true, which may take several plans. Two archive triggers, kept apart: a spec is not archived when a plan serving it archives, and it is archived once its own outcome statement is true, moving to `specs/archived/`. The sign-off block on each page carries the dates, the closing date included.

Specs and plans are many-to-many: one spec may govern several plans, and one plan may serve two specs. The relationship lives in the links on each page, never in directory structure or filenames.

## Conventions

- Filenames are kebab-case, no numeric prefix.
- Sharing a slug with the plan serving the spec is the default for the common one-to-one case, not a rule.
- Where the work has a tracking issue, that issue's body mirrors the outcome, call shape, and failure modes and permalinks the page. The page is canonical.
- The sign-off block is append-only. Earlier entries are never edited; a change is a new dated re-sign line.

## See Also

- [[index|Wiki Index]]
