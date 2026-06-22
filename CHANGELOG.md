# Changelog

Hiveplotlib wiki built up using the
[Karpathy LLM Wiki](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f) model.

This wiki is intended to serve as a knowledge base for Hiveplotlib behavior, architectural decisions,
research ideas, and use cases for hive plots. This knowledge base will be referenced and augmented by
the agent harness used to develop Hiveplotlib.

Dated versioning, most recent release first.

## 2026.06.22

Checkpointing after first ADR released having shipped Hiveplotlib v0.28.

## 2026.05.10

The first pass - an LLM-curated knowledge base over hand-curated raw sources, with `entities/`, `concepts/`, `analyses/`, and `sources/` categories. Schema documented in [`CLAUDE.md`](CLAUDE.md).

The primary focus up to this point of wiki curation was Hiveplotlib summary information, research ideas, and use cases.

This entry marks the wiki state immediately before integration with the hiveplotlib agent harness, which adds `wiki/adr/` as a new category for Architecture Decision Records produced by the agentic dev loop.
