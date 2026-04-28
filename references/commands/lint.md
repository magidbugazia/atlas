# Command: Lint

This file is loaded on demand by `~/.claude/skills/atlas/SKILL.md` when the user invokes `/atlas lint`. It contains the complete operational logic for the lint command, including the five parallel agent prompts.

**Prerequisite:** Phase 0 (auto-detect knowledge base) must have already run from SKILL.md before this file is loaded. If it has not, STOP and return to SKILL.md.

**Invocation:** `/atlas lint`

## Phase 1: Compute Hub Concepts and Spawn Agents

**Step A: Compute hub concepts (deterministic, runs in main context).**

This step identifies which concept pages are "load-bearing" by counting how many other concept pages link to them. Hub concepts deserve extra maintenance attention because errors on hub pages ripple farther than errors on leaf pages.

1. Glob `wiki/concepts/` for all `.md` files. Build the list of concept slugs (filename without `.md`).
2. For each concept slug, count its inbound link count: how many OTHER concept pages link TO it. Use Grep across `wiki/concepts/` for the pattern `[slug].md` (matching links like `[Name](../concepts/[slug].md)` and `[Name]([slug].md)`). Exclude self-references (the page linking to itself in its own Sources or footer).
3. Sort concepts by inbound link count, descending. Ties broken by alphabetical slug order.
4. Take the top 5. If the wiki has fewer than 5 concepts, take all of them. Store the list (concept name, slug, inbound link count) for Phase 2 to render in the report.

This step is fast: Glob + Grep across a few hundred small files takes well under a second. It does not need an agent.

**Step B: Launch 5 agents in PARALLEL.**

**Agent 1: Consistency Checker**
```
You are auditing a knowledge base wiki for internal consistency and structural integrity.

KB root: [path]
Subject: [from KB.md]

TASK:
1. Read wiki/INDEX.md to get the full article list
2. Read .atlas/concepts.json for the concept registry
3. Read ALL files in wiki/concepts/ and wiki/summaries/
4. Check for the following issues:

STRUCTURAL INTEGRITY:
- Broken links: for every markdown link in every wiki file, verify the target file exists (use Glob). Report each broken link with source file, line, and target path.
- Orphan pages: files in wiki/concepts/ or wiki/summaries/ that are NOT listed in INDEX.md
- Ghost entries: INDEX.md entries that link to files that don't exist
- Backlink symmetry: if page A links to page B, check that page B links back to page A. Report asymmetric links.
- Registry drift: concept pages in wiki/concepts/ that are not in concepts.json, or concepts.json entries with no corresponding file

CONTENT CONSISTENCY:
- Contradictions: two concept pages making conflicting claims about the same topic. Quote both claims.
- Terminology drift: the same concept referred to by different names across pages. Example: "context engineering" in one page, "prompt management" in another, if they mean the same thing. Check aliases in concepts.json to see if this is already tracked.
- Stale index summaries: INDEX.md one-line summaries that don't accurately reflect the current content of the concept page

COMPLETENESS:
- Referenced but missing: concept pages that mention another concept by name but no page exists for it
- Thin pages: concept pages with fewer than 100 words (likely stubs that need expansion)
- Unsummarized sources: raw files in raw/ with no corresponding summary in wiki/summaries/
- Source attribution gaps: concept pages with claims that don't cite any raw source

FRONTMATTER HEALTH:
- Missing frontmatter: any concept or summary page without a YAML frontmatter block at the top. Should auto-fix on next compile via Phase 1.5 backfill, but flag here so the user knows
- Stale frontmatter: pages where `last_updated` is more than 90 days old. Report the page name, last_updated date, and current age in days
- Pages flagged review_pending: list every page with `status: review_pending` and how long it has held that status. These need human attention. (Legacy `status: needs_update` should also be flagged here — it is the pre-rename name and any page still carrying it predates the rename.)
- Source-count drift: pages where the `source_count` value in frontmatter does not match the actual number of entries in the Sources section

For each issue report:
- Severity: CRITICAL (broken navigation, contradictions) / IMPORTANT (orphans, terminology drift) / MINOR (backlink gaps, stale summaries)
- File path and line number
- Description of the issue
- Suggested fix
```

**Agent 2: Connection Discoverer**
```
You are looking for hidden connections and knowledge gaps in a wiki.

KB root: [path]

TASK:
1. Read ALL concept pages in wiki/concepts/
2. Read .atlas/concepts.json for alias information
3. Analyze the full set of concepts for:

MISSING CONNECTIONS:
- Pairs of concepts that discuss related topics but don't link to each other
- Concepts that share key terms or themes but exist in isolation
- For each suggested connection: explain WHY these concepts should be linked

NEW ARTICLE CANDIDATES:
- Topics that multiple concept pages reference or assume knowledge of, but that don't have their own page
- Bridging concepts that would connect currently isolated clusters of the wiki
- For each candidate: which existing pages would link to it, and what would it cover

QUESTIONS WORTH EXPLORING:
- Based on the wiki's coverage patterns, suggest 5 research questions the user might want to investigate
- These should be questions the wiki ALMOST answers but has identifiable gaps on
- For each question: which concept pages are relevant and what's missing

Report everything as structured lists with clear reasoning.
```

**Agent 3: Source Verifier (Hallucination Audit)**
```
You are verifying that a knowledge base wiki's claims are actually supported by its cited sources. This catches LLM hallucinations introduced during compilation.

KB root: [path]

TASK:
1. Glob wiki/concepts/ for all .md files.
2. For each concept page, identify all source citations. These appear as "(Source: [filename])" in the text or in the "Sources" section at the bottom.
3. BEFORE verifying, detect exempt sources: for each cited source filename, check whether a summary page exists at wiki/summaries/[source-slug].md whose YAML frontmatter contains `exempt_from_raw_hash: true`. These summaries describe self-authored living documents (user-written references, not ingested external material). When a concept page cites such a source, do NOT look for it in raw/. Instead, read the matching summary page for verification, or mark the claim EXEMPT if word-level verification against the summary would be redundant. Never flag exempt sources as SOURCE MISSING.
4. SPOT-CHECK: For each concept page, select 2-3 specific factual claims that cite a non-exempt source. Read the cited raw source file in raw/. Verify the source actually says what the concept page claims it says.
5. Report findings.

For each claim checked, report:
- Concept page and the specific claim (quote it)
- Cited source file
- Verdict: SUPPORTED (source clearly says this), PARTIALLY SUPPORTED (source says something similar but the concept page overstates or simplifies), UNSUPPORTED (source does not say this, likely hallucinated), SOURCE MISSING (cited file doesn't exist AND is not marked exempt), EXEMPT (cited source matches a summary page with `exempt_from_raw_hash: true`; self-authored, verification optional)
- If PARTIALLY SUPPORTED or UNSUPPORTED: quote what the source actually says

SCOPE:
- Check a maximum of 20 concept pages per run (prioritize larger pages with more claims).
- Check 2-3 claims per page. Prioritize specific factual claims (numbers, dates, technical mechanisms) over general statements, since factual claims are easier to verify and more damaging when wrong.
- If a concept page has no source citations at all, flag it as "UNCITED PAGE" (this is a structural issue, not a hallucination, but worth reporting).

Do NOT verify opinions, synthesis, or the concept page's summary paragraph (those are the LLM's legitimate synthesis). Only verify claims attributed to a specific source.
```

**Agent 4: Report Overlap Detector**
```
You are detecting overlap between saved query reports in a knowledge base so the user can decide whether to consolidate or archive redundant ones. You DO NOT merge, delete, or edit anything. You only surface clusters of similar reports.

KB root: [path]

TASK:
1. Glob wiki/reports/ for all .md files. If the directory does not exist or has fewer than 3 reports, report "Not enough reports to analyze (need at least 3)" and stop.
2. For each report, read only the top ~40 lines. Extract:
   - The H1 title (the question it answers)
   - The `Generated:` date
   - The `Concepts consulted:` list (if present)
   - The filename slug
3. Build overlap signals WITHOUT reading full bodies yet:
   - Slug similarity: token-level Jaccard on slug words (e.g., `rag-vs-finetuning-tradeoffs` and `tradeoffs-rag-finetuning` share {rag, finetuning, tradeoffs})
   - Concept overlap: Jaccard on the concepts-consulted lists
   - Title similarity: shared key nouns/verbs in the H1 titles
4. Group reports into candidate clusters when any pairwise signal exceeds a reasonable threshold (e.g., 50% Jaccard on slugs OR 60% concept overlap OR clearly paraphrased titles).
5. For each candidate cluster with 2+ reports, deep-read the full bodies to confirm topical overlap. Discard clusters that looked similar in metadata but actually answer meaningfully different questions (note these briefly as "false positives, kept separate").

SCOPE:
- Maximum 50 reports analyzed per run. If there are more, prioritize the most recent 50.
- Maximum 10 clusters reported. If more than 10 pass the confirmation step, report the largest 10 by member count.

For each confirmed cluster, report:
- Cluster topic (one-line summary of what the reports collectively cover)
- Member reports: list each with filename, H1 title, generated date
- Recommended newest-wins candidate (the most recent report, assumed to reflect the most current KB state)
- Suggested older reports to archive or supersede (everything except the newest)
- Brief note on what differs between the reports (e.g., "April version cites 2 concepts the January version didn't — newer KB had those concepts compiled")

Do NOT recommend any destructive action. Your output is advisory only. The user will decide what to do with each cluster.

If no overlapping clusters are found after confirmation, report "No report overlap detected — all saved reports cover distinct topics."
```

**Agent 5: Alias Coverage Checker**
```
You are auditing alias coverage in a knowledge base's concept registry. Aliases are the KB's retrieval vocabulary: when a query uses a paraphrase, acronym, or variant form, the registry expands it to the canonical concept slug. Thin alias coverage silently degrades retrieval — queries miss the right page and fall back to full-text grep. This audit finds concepts whose alias lists don't cover how the concept is actually referred to in the wiki.

KB root: [path]

TASK:
1. Read .atlas/concepts.json for the full concept registry. Each entry has a slug, display_name, and aliases list.
2. Glob wiki/concepts/ for all .md files. Build a map from slug to concept page path.
3. For each concept, evaluate alias coverage along three dimensions:

STRUCTURAL VARIANTS (high confidence, mechanically derivable):
- ACRONYM: if the display_name is multi-word and the capital letters form a pronounceable or common acronym (e.g., "Retrieval-Augmented Generation" → "RAG", "Large Language Model" → "LLM"), the acronym should be in aliases. Flag if missing.
- PLURAL: if the display_name is a singular noun and the plural is a natural way to refer to the concept, flag missing plural (e.g., "Embedding" should have "Embeddings"; "Vector Database" should have "Vector Databases").
- HYPHEN/SPACE SWAP: if the display_name contains a hyphen, the space-separated form should be an alias, and vice versa (e.g., "fine-tuning" ↔ "fine tuning"; "retrieval-augmented generation" ↔ "retrieval augmented generation").
- CASE VARIANTS: aliases are expected to be case-insensitive at retrieval time, so do NOT flag case differences as gaps.

PARAPHRASE COVERAGE (medium confidence, requires judgment):
- Read the concept's page body (wiki/concepts/[slug].md). Look for alternative phrasings the page itself uses to refer to the concept — synonyms, older names, field-specific terms — that are NOT in the aliases list.
- Example: if a page titled "Retrieval-Augmented Generation" repeatedly uses "RAG pipeline" or "grounded generation" in its body, both are candidate aliases.
- Only suggest paraphrases that clearly refer to THE SAME concept, not related concepts. When uncertain, skip.

THIN REGISTRY (structural health):
- Flag any concept whose aliases list has 0 or 1 entries (beyond the canonical display_name itself) as THIN. Thin entries are the most at-risk for retrieval misses.

SCOPE:
- Audit every concept in the registry (this is cheap — registry is a single JSON file, pages are a Grep away).
- Do NOT invent aliases. Every suggested addition must come from either a mechanical rule (structural variant) or a literal phrase in the concept's own page body (paraphrase).
- COLLISION CHECK: before proposing any alias, verify it does not already match another concept's display_name or existing alias in the registry (case-insensitive comparison). If it would collide, skip the suggestion and note it inline under the concept as "skipped: collides with [other-concept-slug]". A shared alias would make retrieval ambiguous.

For each concept with findings, report:
- Concept: [display_name] (slug: [slug])
- Current aliases: [list or "none"]
- Proposed additions:
  - STRUCTURAL: [variant] — [rule: acronym / plural / hyphen-swap]
  - PARAPHRASE: "[phrase]" — found in page body at line [N], quoted context: "[surrounding sentence]"
- Severity: THIN (0-1 aliases and no additions proposed means the concept may be genuinely atomic, flag as INFO only), GAPS (structural variants missing — high-confidence auto-fixable), PARAPHRASE_REVIEW (body uses phrases not in aliases — requires human approval)

If every concept has adequate coverage, report "Alias coverage healthy — no gaps detected."
```

## Phase 2: Consolidate and Save

After all five agents return, merge their findings into a report. Write the report to `wiki/lint/lint-YYYY-MM-DD.md` (create `wiki/lint/` if it doesn't exist) AND display it in the terminal.

If a lint report with the same date already exists, overwrite it (re-running lint on the same day replaces the previous run).

```markdown
# Lint Report - [YYYY-MM-DD]

## Knowledge Base: [Subject]
Stats: [N] concepts | [N] sources | [N] reports | Last compiled: [date]

## Overall Health: [HEALTHY | NEEDS ATTENTION | DEGRADED]

HEALTHY: 0 critical issues, fewer than 3 important issues, 0 unsupported claims
NEEDS ATTENTION: 1-3 critical issues, or more than 5 important issues, or 1-2 unsupported claims
DEGRADED: 4+ critical issues, or contradictions found, or 3+ unsupported claims, or more than 20% of pages are orphaned/thin

---

## Hub Concepts

These pages have the most inbound links from other concept pages. Errors on hub concepts ripple farther than errors on leaf concepts. Prioritize maintenance work on these pages.

| Rank | Concept | Inbound Links |
|------|---------|---------------|
| 1 | [Concept Name](../concepts/slug.md) | N |
| 2 | [Concept Name](../concepts/slug.md) | N |
| 3 | [Concept Name](../concepts/slug.md) | N |
| 4 | [Concept Name](../concepts/slug.md) | N |
| 5 | [Concept Name](../concepts/slug.md) | N |

[If any hub concept has status: review_pending or has stale frontmatter, flag it inline next to the row.]

---

## Issues

### Critical
[Broken links, contradictions, ghost entries. Things that break navigation or mislead]

### Important
[Orphan pages, terminology drift, thin pages, source attribution gaps, registry drift]

### Minor
[Backlink asymmetry, stale summaries, formatting inconsistencies]

---

## Source Verification

Claims checked: [N]. Supported: [N]. Partially supported: [N]. Unsupported: [N]. Missing sources: [N]. Uncited pages: [N].

[For each UNSUPPORTED or PARTIALLY SUPPORTED claim:]
- **[concept-page.md]**: "[the claim]" cites [source-file.md]
  Verdict: [UNSUPPORTED/PARTIALLY SUPPORTED]
  Source actually says: "[what the source says]"

---

## Alias Coverage

Aliases are the retrieval vocabulary. Thin coverage causes silent query misses.

Concepts audited: [N]. Healthy: [N]. Structural gaps: [N]. Paraphrase-review: [N]. Thin (info only): [N].

[If no gaps:]
Alias coverage healthy — no gaps detected.

[Otherwise, for each concept with findings:]

### [Concept Name] (`[slug]`)
Current aliases: `[list or "none"]`

Proposed additions (auto-fixable, structural):
- [ ] `[variant]` — [acronym / plural / hyphen-swap]

Proposed additions (requires review, paraphrase):
- [ ] `[phrase]` — found at line [N]: "[quoted context]"

---

## Suggested Connections

[From Agent 2: concept pairs that should link to each other, with reasoning. Checkbox format:]
- [ ] **[Concept A] <-> [Concept B]** -- [why these should link]
- [ ] ...

---

## New Article Candidates

[From Agent 2. Each candidate includes a ready-to-run mentor command with --context paths pointing to the wiki pages that reference the topic:]

### [Candidate Topic]
Referenced by [N] pages: [list page names]

To evaluate whether this deserves a standalone page:
`/mentor evaluate "[topic]" --context [wiki/concepts/page1.md] [wiki/concepts/page2.md] ...`

### [Next Candidate]
...

---

## Research Questions

[From Agent 2. Each question includes a ready-to-run atlas query command:]

### [Question text]
Relevant pages: [list with paths]
Gap: [what's missing]

To explore: `/atlas query "[question text]"`

### [Next Question]
...

---

## Report Overlap

[From Agent 4. Clusters of saved reports that appear to cover the same topic. Advisory only; no action taken.]

[If no overlap detected:]
No report overlap detected — all saved reports cover distinct topics.

[Otherwise, for each cluster:]

### Cluster [N]: [one-line cluster topic]
Members:
- `wiki/reports/[slug-a].md` — "[H1 title]" (generated YYYY-MM-DD)
- `wiki/reports/[slug-b].md` — "[H1 title]" (generated YYYY-MM-DD)
- `wiki/reports/[slug-c].md` — "[H1 title]" (generated YYYY-MM-DD)

Recommended newest-wins: `wiki/reports/[slug-c].md`
Candidates to archive/supersede: `[slug-a].md`, `[slug-b].md`
Notes: [what differs between them, if anything meaningful]
```

## Phase 3: Auto-Fix and Git Commit

Phase 3 runs in two parts. Part A always runs and records that lint was performed. Part B is optional and only runs if the user approves auto-fixes. Splitting them guarantees the lint report is committed even when the user declines fixes, and that auto-fixes land in a separate, revertable commit.

**Part A: Record the lint run (always runs).**
1. Update `KB.md`: set `last_linted` to today, reset `compiles_since_lint` to 0, set `last_lint_health` to the overall-health value from the report.
2. Git commit if in a repo (detect by Globbing for `.git/` at the KB root, not by running `git rev-parse`): `git -C [KB root] add wiki/lint/ KB.md` then `git -C [KB root] commit -m "atlas: lint run - [HEALTH] - [YYYY-MM-DD]"`. This commit contains only the lint report and the KB.md metadata update.

**Part B: Apply auto-fixes (opt-in).**
After Part A, ask: "Should I auto-fix the simple structural issues? (broken links, backlink asymmetry, orphan index entries, registry drift, structural alias additions)"

If the user says yes:
1. Fix broken links by removing or correcting them
2. Add missing backlinks
3. Add orphan pages to INDEX.md
4. Remove ghost entries from INDEX.md
5. Sync `.atlas/concepts.json` with actual concept files (add missing entries, remove entries with no file)
6. Apply structural alias additions from Agent 5 (acronyms, plurals, hyphen/space swaps only — the items in the "Proposed additions (auto-fixable, structural)" lists). Merge into each concept's `aliases` array in `.atlas/concepts.json`, preserving existing aliases. Do NOT apply paraphrase suggestions — those require explicit user approval because a phrase in the page body can refer to the concept itself, a related concept, or a metaphor, and the agent cannot always tell which.
7. Git commit if in a repo (detect by Globbing for `.git/` at the KB root, not by running `git rev-parse`): `git -C [KB root] add wiki/ INDEX.md .atlas/concepts.json` then `git -C [KB root] commit -m "atlas: lint fixes - [N] issues resolved"`. This is a second, independent commit so it can be reverted without losing the lint report.

If the user says no, Phase 3 ends after Part A — no second commit is created.
