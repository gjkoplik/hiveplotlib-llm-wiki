# Hive Plot Research Wiki — Schema

This is a personal research wiki about **hive plots** and **hiveplotlib**, maintained by an LLM agent following the LLM Wiki pattern. The human curates sources and directs research; the LLM writes, updates, and maintains all wiki content.

## Domain

Hive plots are a rational, quantitative network visualization method that maps nodes to radially distributed linear axes based on meaningful node properties, then draws edges as curves between them. They were introduced by Martin Krzywinski as an alternative to force-directed layouts. **hiveplotlib** is the Python library (by the human user of this wiki) for creating hive plots.

Topics of interest include: hive plot theory and design, network visualization methods, hiveplotlib API and internals, graph/network analysis, applications of hive plots in various domains (biology, social networks, software engineering, etc.), and comparison with other visualization approaches.

## External Repos

These local repos are primary references. The LLM reads from them but never modifies them.

- **hiveplotlib (Python)**: `/home/garyk/repos/hiveplotlib`
- **hiveplotlib-javascript**: `/home/garyk/repos/hiveplotlib-javascript`

## Directory Structure

```
.
├── CLAUDE.md          # This file — the schema (LLM + human co-evolve)
├── raw/               # Immutable source documents (human curates)
│   ├── assets/        # Downloaded images, figures, data files
│   └── *.md           # Clipped articles, papers, notes, transcripts
├── wiki/              # LLM-generated knowledge base (LLM owns entirely)
│   ├── index.md       # Content catalog — updated on every ingest
│   ├── log.md         # Chronological operation log — append-only
│   ├── overview.md    # High-level synthesis of the entire wiki
│   ├── sources/       # One summary page per ingested source
│   ├── entities/      # Pages for specific things (people, libraries, tools)
│   ├── concepts/      # Pages for ideas, methods, techniques
│   └── analyses/      # Filed query results, comparisons, deep dives
└── .obsidian/         # Obsidian vault config
```

## Page Format

Every wiki page uses this template:

```markdown
---
title: Page Title
type: source | entity | concept | analysis
created: YYYY-MM-DD
updated: YYYY-MM-DD
sources: [list of source filenames that inform this page]
tags: [relevant tags]
---

# Page Title

Content here. Use [[wikilinks]] for cross-references to other wiki pages.

## See Also

- [[Related Page 1]]
- [[Related Page 2]]
```

**Conventions:**
- Use `[[wikilinks]]` (Obsidian-style) for all cross-references between wiki pages.
- Tags use lowercase-kebab-case: `hive-plot`, `network-visualization`, `hiveplotlib`, etc.
- Source summary filenames mirror the raw source: if raw is `krzywinski-2012.md`, summary is `wiki/sources/krzywinski-2012.md`.
- Entity and concept page filenames are kebab-case: `martin-krzywinski.md`, `node-assignment.md`.

## Operations

### Ingest

Triggered when the human adds a new source to `raw/` and asks to process it.

1. **Read** the raw source completely.
2. **Discuss** key takeaways with the human — what's interesting, what to emphasize.
3. **Create** a source summary page in `wiki/sources/`.
4. **Create or update** entity pages in `wiki/entities/` for people, tools, libraries, datasets mentioned.
5. **Create or update** concept pages in `wiki/concepts/` for methods, techniques, ideas.
6. **Update** `wiki/index.md` with new/changed pages.
7. **Update** `wiki/overview.md` if the new source materially changes the big picture.
8. **Append** to `wiki/log.md`.

The human stays involved — read summaries aloud, check cross-references, ask for adjustments before finalizing. Prefer ingesting one source at a time.

### Query

The human asks a question against the wiki.

1. **Read** `wiki/index.md` to find relevant pages.
2. **Read** the relevant wiki pages (not raw sources, unless needed for detail).
3. **Synthesize** an answer with `[[wikilinks]]` citations to wiki pages.
4. If the answer is substantial and reusable, **offer to file it** as a new page in `wiki/analyses/`.
5. **Append** to `wiki/log.md`.

### Lint

Periodic health check, triggered by the human or proactively suggested.

- Contradictions between pages.
- Stale claims superseded by newer sources.
- Orphan pages (no inbound links).
- Important concepts mentioned but lacking their own page.
- Missing cross-references.
- Data gaps — suggest new sources to find.
- **Append** findings and actions to `wiki/log.md`.

## Index Format (`wiki/index.md`)

Organized by category. Each entry is one line:

```markdown
- [[Page Name]] — one-line summary (N sources)
```

## Log Format (`wiki/log.md`)

Append-only. Each entry:

```markdown
## [YYYY-MM-DD] operation | Subject
Brief description of what was done. Pages created/updated listed.
```

## Rules

1. **Never modify files in `raw/`.** Sources are immutable.
2. **Always update `index.md` and `log.md`** on every ingest and every filed query result.
3. **Use wikilinks everywhere.** The link graph is a first-class artifact.
4. **Cite sources.** Every claim in the wiki should trace back to a source via the `sources` frontmatter field and inline references.
5. **Flag contradictions explicitly.** When a new source contradicts existing wiki content, note it on both pages.
6. **Keep pages focused.** One entity/concept per page. Split if a page grows beyond ~500 lines.
7. **Maintain the overview.** `wiki/overview.md` should always reflect the current state of knowledge.
8. **Ask before filing.** When a query result is worth keeping, ask the human before creating an analysis page.
9. **Suggest next steps.** After every ingest, suggest what to explore next — related papers, open questions, concepts that need their own page.
10. **Every interaction follows this schema.** No freewheeling — the LLM operates as a disciplined wiki maintainer.
