# Atlas

A Claude Code skill that turns scattered research material into a structured, interconnected wiki maintained by an LLM. Based on a workflow described by Andrej Karpathy for using LLMs to build personal knowledge bases.

You collect raw sources. Atlas organizes them into concept pages with summaries, cross-links, and a two-tier hierarchical index. You query it, and the LLM researches answers from the wiki and writes them as files that feed back into the knowledge base. The knowledge compounds over time.

## The Problem It Solves

You research a topic deeply. You read 50 articles, 20 papers, browse 10 repos. The knowledge ends up scattered across browser tabs, PDFs in Downloads, and your unreliable memory. Or you write everything into one massive file that becomes unnavigable past 2000 lines.

Atlas solves both. Raw material goes into one directory. The LLM compiles it into a structured, navigable, queryable wiki. The LLM maintains the wiki. You rarely edit it directly.

## Key Design Decisions

**One file per concept, not per source.** If three articles discuss "prompt engineering," they become ONE concept page that synthesizes all three. Knowledge converges into concepts, not sources.

**Two-tier hierarchical indexing.** INDEX.md lists categories (kept under 60 lines). Each category has its own sub-index in `wiki/indexes/` with detailed concept summaries. The LLM never reads more than ~30 summaries at once, even at 500+ concepts. At small scale (under 15 concepts), a single sub-index is used.

**Scale-adaptive query.** Small wikis: direct concept reads. Medium wikis: single research agent. Large wikis (150+ concepts): parallel agents per category, each scoped to its own sub-index. Prevents context rot at any scale.

**Alias-aware search.** A JSON concept registry (`.atlas/concepts.json`) maps every concept to its canonical name and known aliases. Searching for "RAG" finds "Retrieval-Augmented Generation." Used by query, search, and export.

**Source traceability.** Every claim in the wiki cites the raw source it came from. Lint includes a Source Verifier agent that spot-checks claims against cited sources to catch hallucinations introduced during compilation.

**Git-backed operations.** Every compile, query filing, and lint fix auto-commits if the knowledge base is in a git repo. Rollback, history, and diffable changes.

**Outputs as files, not chat.** Answers get written as markdown files in the wiki. You can file them back in to enrich future queries. Each question makes the wiki more complete.

## How Atlas and Mentor Work Together

Atlas organizes knowledge. It does not judge whether a source is worth including (that is `/mentor evaluate`), test your understanding (that is `/mentor interview`), or check industry alignment (that is `/mentor audit`).

Mentor decides WHAT belongs. Atlas does the WORK of organizing it. Atlas lint checks internal consistency and source accuracy. Mentor audit checks external alignment with the job market.

## Limitations

- **No source evaluation.** Atlas ingests everything you give it. Use `/mentor evaluate` to decide what's worth adding.
- **Obsidian recommended.** Atlas generates standard markdown viewable anywhere. Obsidian adds backlinks, graph views, and Marp slide support on top.

## Pairing with a Mentor Skill

Atlas is domain-agnostic. The optional mentor skill adds opinionated curation: filtering what's worth ingesting, auditing against job market demands, and testing your understanding. Atlas works standalone. Mentor makes it smarter about what goes in. See the mentor skill's README for how to adapt it to any domain.

## Commands

See SKILL.md for full operational details. Quick reference:

`/atlas init`, `/atlas ingest`, `/atlas compile`, `/atlas query`, `/atlas search`, `/atlas lint`, `/atlas status`, `/atlas export`
