# Command: Query

This file is loaded on demand by `~/.claude/skills/atlas/SKILL.md` when the user invokes `/atlas query`. It contains the complete operational logic for the query command.

**Prerequisite:** Phase 0 (auto-detect knowledge base) must have already run from SKILL.md before this file is loaded. If it has not, STOP and return to SKILL.md.

**Invocation:** `/atlas query "What are the tradeoffs between RAG and fine-tuning?"`

## Phase 1: Research (Two-Hop Hierarchical Retrieval)

**Hop 1: Category selection**
1. Read `wiki/INDEX.md` in full (always small, under 60 lines).
2. Read `.atlas/concepts.json` for the concept registry including aliases.
3. From the category summaries in INDEX.md, identify which categories are relevant to the question. Select 1-4 categories.

**Hop 2: Concept selection**
4. Read the sub-indexes for each selected category (`wiki/indexes/[category].md`).
5. **Alias-aware search:** Extract key terms from the question. Check each term against the concept registry aliases. If a term matches an alias, include that concept in the candidate list even if the sub-index summary didn't match. Then use Grep to search `wiki/concepts/` for the key terms AND their known aliases. This catches concepts that neither the index nor the registry surfaced.
6. Combine results from sub-index browsing, registry lookup, and search. Deduplicate.

**Concept reading (scale-adaptive):**
7. **Small wiki (fewer than 50 total concepts) or 5 or fewer relevant concepts:** Read concept pages directly in main context.
8. **Medium wiki (50-150 concepts) with 6+ relevant concepts:** Spawn a single research agent to read all relevant concepts and return a synthesized answer. Do NOT load 6+ files into main context.
9. **Large wiki (150+ concepts) with 3+ relevant categories:** Spawn one research agent PER relevant category in PARALLEL. Each agent reads its category's sub-index, identifies relevant concepts within that category, reads those concept pages in full, and returns a category-scoped summary (max 500 words per agent). The main context synthesizes the agent summaries into a final answer. This prevents any single agent from reading too many concepts and keeps each agent's context focused.
10. If the question clearly requires information not present in any wiki article: tell the user "The wiki doesn't cover [specific gap]. Would you like me to search the web for this? (Web results will be saved to `raw/` for future compiles, not injected directly into the wiki.)"

## Phase 2: Synthesize

1. Generate a slug for the question: lowercase, hyphens, max 50 chars. Example: "rag-vs-finetuning-tradeoffs"
2. Write the answer as a markdown file at `wiki/reports/[slug].md`. Reports use YAML frontmatter as the canonical metadata block (matching concept and summary pages). Do NOT add a redundant header-style block (`Generated:` / `Concepts consulted:` lines) below the H1 — that is the legacy format. Do NOT add a trailing `## Sources Consulted` section — `concepts_consulted` in frontmatter is the canonical record, and report bodies already cite each concept inline as clickable links throughout the prose, so a footer list adds no navigation value and creates a drift-prone second source of truth.

```markdown
---
title: "[Question as title, quote it if it contains a colon, question mark, or apostrophe]"
slug: [slug]
generated: [today's date, YYYY-MM-DD]
last_updated: [today's date, YYYY-MM-DD]
status: reviewed
concepts_consulted:
  - "../concepts/[slug-1].md"
  - "../concepts/[slug-2].md"
  - "../concepts/[slug-3].md"
---

# [Question as title]

[Synthesized answer. Structure with headings if the answer is complex.
Every major claim should cite the concept page it comes from inline as a
clickable markdown link:
"Attention mechanisms allow... ([Attention Mechanisms](../concepts/attention-mechanisms.md))"

The set of links cited inline must match `concepts_consulted` in frontmatter.]
```

**YAML field semantics:**
- `title`: the H1 text. Quote with double quotes whenever the title contains `:`, `?`, `'`, `"`, `#`, or starts/ends with whitespace.
- `slug`: the filename without `.md`. Must match the file's actual name on disk.
- `generated`: the date the report was first written. Never changes after creation.
- `last_updated`: the date of the most recent edit. Equals `generated` on creation. Bump whenever the report body is materially revised.
- `status`: `reviewed` for fresh atlas-generated reports (you wrote them, you stand behind them). Use `draft` only when the report is incomplete and the user has been told. Use `review_pending` only when a later edit introduced material that warrants a re-read.
- `concepts_consulted`: machine-readable list of relative paths to every concept page actually read. This is the canonical record of what the report drew from. Lint Agent 4 (Report Overlap Detector) reads this directly for cluster Jaccard analysis. Every concept-page link cited inline in the body should appear here; every entry here should appear at least once as an inline citation in the body.
- Optional fields: `revision_note` (one-line summary of the most recent edit, when `last_updated > generated`), `parent_report` (filename of a parent report when this is a satellite/recall card).

**When updating an existing report** (e.g., after a `--rerun` or follow-up): update `last_updated`, optionally add a `revision_note`, leave `generated` alone. Update `concepts_consulted` if the revision pulled in new concept pages. Do not duplicate metadata in the body.

3. Display the report content to the user directly in the terminal output.

## Phase 3: Offer Filing

After displaying the report, ask:

"Report saved to `wiki/reports/[slug].md`. Should I extract new concepts or enrich existing concept pages from this report? (This feeds your exploration back into the knowledge base.)"

If the user says yes:
1. Read the report
2. Read `.atlas/concepts.json` for the current registry
3. Identify concepts mentioned in the report that either don't have wiki pages or could be enriched
4. Create/update concept pages accordingly
5. Update `.atlas/concepts.json` with any new concepts
6. Update INDEX.md
7. Git commit if in a repo (detect by Globbing for `.git/` at the KB root, not by running `git rev-parse`): `git -C [KB root] add wiki/ KB.md && git -C [KB root] commit -m "atlas: query filed - [slug]"`
8. Confirm what was updated
