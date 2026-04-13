# Command: Search

This file is loaded on demand by `~/.claude/skills/atlas/SKILL.md` when the user invokes `/atlas search`. It contains the complete operational logic for the search command.

**Prerequisite:** Phase 0 (auto-detect knowledge base) must have already run from SKILL.md before this file is loaded. If it has not, STOP and return to SKILL.md.

**Invocation:** `/atlas search "retrieval augmented generation"` or `/atlas search "RAG"`

No agents needed. Quick full-text lookup with alias expansion.

## Phase 1: Expand Search Terms

1. Read `.atlas/concepts.json` for the concept registry
2. For each search term the user provided, check if it matches any alias in the registry. If yes, collect the canonical slug and all its aliases as additional search terms.
   Example: user searches "RAG". Registry has `"retrieval-augmented-generation": {"aliases": ["RAG", ...]}`. Search terms expand to: "RAG", "retrieval-augmented-generation", "Retrieval-Augmented Generation".

## Phase 2: Search

1. For each search term (original + expanded aliases), Grep `wiki/concepts/` and `wiki/reports/` with context (3 lines before and after each match).
2. Deduplicate results by file. Group matches by file path.
3. Cap at 20 matching files. If more, show the first 20 and tell the user how many were omitted.

## Phase 3: Output

Show matched lines verbatim from grep output -- do not summarize, paraphrase, or synthesize. The user reads the raw matches and decides what to dig into.

Use plain file paths (not markdown links) since the primary rendering context is a terminal. Always include the parent directory (`wiki/concepts/` or `wiki/reports/`) so the user can navigate directly.

```
## Search Results: "[original query]"

[If aliases were expanded: "Also searched for: [expanded terms]"]

### [N] matches in [M] files

**wiki/concepts/slug.md**
  [matched line verbatim, with 1 line of context on each side, as returned by grep]

**wiki/reports/slug.md**
  [matched line verbatim, with 1 line of context on each side]

[Continue for each matching file...]

[If no matches: "No results found. Try a broader term or check the concept registry with `/atlas status`."]
```
