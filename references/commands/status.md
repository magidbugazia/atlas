# Command: Status

This file is loaded on demand by `~/.claude/skills/atlas/SKILL.md` when the user invokes `/atlas status`. It contains the complete operational logic for the status command.

**Prerequisite:** Phase 0 (auto-detect knowledge base) must have already run from SKILL.md before this file is loaded. If it has not, STOP and return to SKILL.md.

**Invocation:** `/atlas status`

No agents needed. Quick stats from reading existing files:

1. Read `KB.md` for metadata
2. Count files in each `raw/` subdirectory (use Glob)
3. Count files in `wiki/concepts/`, `wiki/summaries/`, `wiki/reports/`, `wiki/slides/`
4. Read `.atlas/hashes.json` and count entries vs. current raw file count to detect untracked files
5. Read `wiki/INDEX.md` in full (it is capped at 60 lines by design) for the last compiled date, totals, and the Categories list
6. Read `.atlas/concepts.json` and count entries (number of top-level keys)
7. Count concept pages awaiting review: glob `wiki/concepts/*.md`, parse YAML frontmatter on each, count those whose `status: review_pending`. (Reports carry no review flag; their currency is informational, surfaced by lint.)
8. Read `compiles_since_lint` from `KB.md` frontmatter (compile increments it; lint Part A resets it to 0). Treat a missing field as 0.

Output:

```
## Atlas Status: [Subject]

Created: [date]
Last compiled: [date]
Total compiles: [N]

### Raw Material
Articles: [N] | Papers: [N] | Repos: [N] | Datasets: [N] | Images: [N] | Notes: [N]
Total raw sources: [N]

### Wiki
Concepts: [N] (review_pending: [N]) | Summaries: [N] | Reports: [N] | Slides: [N]
Concept registry: [N] entries in .atlas/concepts.json

### Health
[Compute the compilable-source count first: raw files minus images, PDF binaries, and dataset binaries — their companion `-extracted.md` / `-schema.md` files are the compilable units, and images never get summaries. If compilable count > wiki summaries count: "Wiki is [N] sources behind. Run `/atlas compile --incremental` to update." Comparing raw counts directly false-positives forever on any KB containing an image or PDF.]
[If untracked raw files found (in raw/ but not in hashes.json): "[N] raw files not yet hashed. These will be processed on next compile."]
[If last compile was more than 7 days ago: "Last compile was [N] days ago."]
[If review_pending concept count > 0: "[N] concept pages review_pending — run: `/atlas verify pending`"]
[If compiles_since_lint > 3: "[N] compiles since last lint — run `/atlas lint` to audit for stale-by-content reports."]
[If counts match and nothing else flagged: "Wiki is up to date."]

### Categories
[The index is two-tier: wiki/INDEX.md lists categories only; per-concept one-liners live in wiki/indexes/[category].md. List each category from INDEX.md with its concept count and one-line description, and point the user at the sub-indexes for concept listings.]
```
