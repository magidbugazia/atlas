# Command: Lint

This file is loaded on demand by `~/.claude/skills/atlas/SKILL.md` when the user invokes `/atlas lint`. It contains the complete operational logic for the lint command, including the three parallel agent prompts.

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

**Step B: Launch 3 agents in PARALLEL.**

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
- Pages flagged needs_update: list every page with `status: needs_update` and how long it has held that status. These need human attention
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

## Phase 2: Consolidate and Save

After all three agents return, merge their findings into a report. Write the report to `wiki/lint/lint-YYYY-MM-DD.md` (create `wiki/lint/` if it doesn't exist) AND display it in the terminal.

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

[If any hub concept has status: needs_update or has stale frontmatter, flag it inline next to the row.]

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
```

## Phase 3: Auto-Fix and Git Commit

Phase 3 runs in two parts. Part A always runs and records that lint was performed. Part B is optional and only runs if the user approves auto-fixes. Splitting them guarantees the lint report is committed even when the user declines fixes, and that auto-fixes land in a separate, revertable commit.

**Part A: Record the lint run (always runs).**
1. Update `KB.md`: set `last_linted` to today, reset `compiles_since_lint` to 0, set `last_lint_health` to the overall-health value from the report.
2. Git commit if in a repo: `git -C [KB root] add wiki/lint/ KB.md` then `git -C [KB root] commit -m "atlas: lint run - [HEALTH] - [YYYY-MM-DD]"`. This commit contains only the lint report and the KB.md metadata update.

**Part B: Apply auto-fixes (opt-in).**
After Part A, ask: "Should I auto-fix the simple structural issues? (broken links, backlink asymmetry, orphan index entries, registry drift)"

If the user says yes:
1. Fix broken links by removing or correcting them
2. Add missing backlinks
3. Add orphan pages to INDEX.md
4. Remove ghost entries from INDEX.md
5. Sync `.atlas/concepts.json` with actual concept files (add missing entries, remove entries with no file)
6. Git commit if in a repo: `git -C [KB root] add wiki/ INDEX.md` then `git -C [KB root] commit -m "atlas: lint fixes - [N] issues resolved"`. This is a second, independent commit so it can be reverted without losing the lint report.

If the user says no, Phase 3 ends after Part A — no second commit is created.
