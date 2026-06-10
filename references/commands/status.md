# Command: Status

This file is loaded on demand by `~/.claude/skills/atlas/SKILL.md` when the user invokes `/atlas status`. It contains the complete operational logic for the status command.

**Prerequisite:** Phase 0 (auto-detect knowledge base) must have already run from SKILL.md before this file is loaded. If it has not, STOP and return to SKILL.md.

**Invocation:** `/atlas status`

No agents needed. Quick stats from reading existing files:

1. Read `KB.md` for metadata
2. Count files in each `raw/` subdirectory (use Glob)
3. Count files in `wiki/concepts/`, `wiki/summaries/`, `wiki/reports/`, `wiki/slides/`
4. Read `.atlas/hashes.json` and count entries vs. current raw file count to detect untracked files
5. Read `wiki/INDEX.md` first 5 lines for the last compiled date and totals
6. Read `.atlas/concepts.json` and count entries (number of top-level keys)
7. Count reports flagged for review: glob `wiki/reports/*.md`, parse YAML frontmatter on each, count those whose `status: review_pending`. This surfaces lint Agent 6's stale-by-content output without exposing manifest internals.
8. Count compile manifests since last lint: glob `.atlas/compile-runs/*.json`, count those whose `compile_completed` is after `KB.md:last_linted`. Treat `last_linted: never` (init's seed value) or a missing field as "no lint yet" and count ALL manifests (same three-state rule lint Agent 6 uses). Skip only if the manifests directory is missing.

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
Concepts: [N] | Summaries: [N] | Reports: [N] (flagged for review: [N]) | Slides: [N]
Concept registry: [N] entries in .atlas/concepts.json

### Health
[Compute the compilable-source count first: raw files minus images, PDF binaries, and dataset binaries — their companion `-extracted.md` / `-schema.md` files are the compilable units, and images never get summaries. If compilable count > wiki summaries count: "Wiki is [N] sources behind. Run `/atlas compile --incremental` to update." Comparing raw counts directly false-positives forever on any KB containing an image or PDF.]
[If untracked raw files found (in raw/ but not in hashes.json): "[N] raw files not yet hashed. These will be processed on next compile."]
[If last compile was more than 7 days ago: "Last compile was [N] days ago."]
[If review_pending report count > 0: "[N] reports flagged review_pending — run `/atlas lint` for context, or re-run `/atlas query \"<H1>\"` on each to refresh."]
[If compile manifests since last lint > 3: "[N] compiles since last lint — run `/atlas lint` to audit for stale-by-content reports."]
[If counts match and nothing else flagged: "Wiki is up to date."]

### Categories
[The index is two-tier: wiki/INDEX.md lists categories only; per-concept one-liners live in wiki/indexes/[category].md. List each category from INDEX.md with its concept count and one-line description, and point the user at the sub-indexes for concept listings.]
```
