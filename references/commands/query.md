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

**Refreshing an existing report.** A user who wants to refresh a saved report runs `/atlas query` with a question whose text matches the existing report's `title` field. The title lives in the report's YAML frontmatter (`title: "..."`) and is also the H1 line at the top of the report body — both are kept in sync by this command. Step 1 below performs a normalized title lookup so that any reasonable case/whitespace variation routes to the existing slug, even if the LLM would have generated a different slug on a fresh run. When confirming a saved or refreshed report at the end of Phase 3, surface the exact title string so the user knows what to type to refresh it later.

1. **Reverse-lookup by title first.** Before generating a slug, check whether the input question already has a saved report under a known slug. Glob `wiki/reports/*.md`. For each existing report, read its `title` frontmatter field (preferred); if absent on a legacy report, fall back to the H1 (first `# ` line) of the body. Normalize both the input question and each candidate title for comparison: trim leading/trailing whitespace, strip surrounding double quotes, lowercase. If any existing report's normalized title matches the normalized input question exactly, reuse that report's existing slug from its filename and skip step 2 entirely. Proceed directly to step 3 with the matched slug. If multiple reports tie (rare; indicates a duplicate-title bug in the wiki), prefer the report with the most recent `last_updated`. This step is the primary re-run detection mechanism — it makes refresh independent of LLM-side slug-generation drift.

2. Generate a slug for the question: lowercase, hyphens, max 50 chars. Example: "rag-vs-finetuning-tradeoffs". Skip this step if step 1 already matched an existing report — reuse that report's slug instead.

3. **Detect re-run by slug (backstop).** Check whether `wiki/reports/[slug].md` already exists. When step 1 matched, the file is guaranteed to exist and this step always routes to the re-run path. When step 1 did not match but step 2's slug happens to collide with an existing file (e.g., a paraphrased question that slugifies identically), this step still catches the collision and routes to the re-run path. The slug-level check is the backstop for the title lookup.

   If the file exists:
   1. Read its YAML frontmatter.
   2. Preserve `generated` from the existing frontmatter — this date is immutable per `query.md` field semantics.
   3. Set `last_updated` to today's date.
   4. If the previous `status` was `review_pending`, this re-run resolves it: set `status: reviewed` and add a `revision_note` line summarizing the trigger (e.g., `revision_note: "re-synthesized 2026-04-29 against current KB state — resolves auto-flag from lint 2026-04-28"`). If the previous `status` was already `reviewed`, leave it as `reviewed` and add a `revision_note` summarizing what prompted the re-run if known (otherwise omit).
   5. Recompute `concepts_consulted` from the new synthesis pass. The list may differ from the prior run's list — that's expected when the wiki has grown.
   6. Overwrite the file. The body is fully replaced with the new synthesis; do not attempt to merge with the prior body.

   If the file does NOT exist: proceed with fresh-report defaults (created=today, last_updated=today, status=reviewed, no revision_note).

4. Write the answer as a markdown file at `wiki/reports/[slug].md`. Reports use YAML frontmatter as the canonical metadata block (matching concept and summary pages). Do NOT add a redundant header-style block (`Generated:` / `Concepts consulted:` lines) below the H1 — that is the legacy format. Do NOT add a trailing `## Sources Consulted` section — `concepts_consulted` in frontmatter is the canonical record, and report bodies already cite each concept inline as clickable links throughout the prose, so a footer list adds no navigation value and creates a drift-prone second source of truth.

The frontmatter template below shows the **fresh-report shape** (when step 3's existing-file check returned no match). On a re-run (step 1 matched a title, or step 3 found an existing file), use the override values that step 3 established: preserve `generated` from the prior frontmatter, set `last_updated` to today, set `status` per step 3.4 (resolve `review_pending → reviewed` if applicable), and add a `revision_note`. The slug, concepts_consulted, title, and body are recomputed in either case.

```markdown
---
title: "[Question as title, quote it if it contains a colon, question mark, or apostrophe]"
slug: [slug]
generated: [today's date for fresh reports; preserved prior date on re-run, see step 3]
last_updated: [today's date]
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

**When updating an existing report** (re-run via re-invoking `/atlas query` with the same question, or via a future `--rerun` flag alias): the operational logic for preserving `generated`, bumping `last_updated`, resolving `review_pending` → `reviewed`, and adding a `revision_note` is in step 3 above (with the title-lookup re-run path entered through step 1). Do not duplicate metadata in the body.

5. Display the report content to the user directly in the terminal output.

## Phase 3: Offer Filing

After displaying the report, confirm the save and surface the title so the user knows how to refresh it later. Use the template below — substitute `saved` for fresh reports, `refreshed` for re-runs that took the step 3 existing-file path.

```
Report [saved|refreshed] at `wiki/reports/[slug].md`.
Title: "[exact title string from frontmatter]"

To refresh this report later, run `/atlas query "[exact title string]"` — atlas matches on title and overwrites the existing file in place (preserves `generated`, bumps `last_updated`, resolves `review_pending` if set).

Should I extract new concepts or enrich existing concept pages from this report? (This feeds your exploration back into the knowledge base.)
```

If the user says yes:
1. Read the report
2. Read `.atlas/concepts.json` for the current registry
3. Identify concepts mentioned in the report that either don't have wiki pages or could be enriched
4. Create/update concept pages accordingly
5. Update `.atlas/concepts.json` with any new concepts
6. Update INDEX.md
7. Git commit if in a repo (detect by Globbing for `.git/` at the KB root, not by running `git rev-parse`): `git -C [KB root] add wiki/ KB.md && git -C [KB root] commit -m "atlas: query filed - [slug]"`
8. Confirm what was updated
