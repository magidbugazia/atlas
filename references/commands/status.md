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
Concepts: [N] | Summaries: [N] | Reports: [N] | Slides: [N]
Concept registry: [N] entries in .atlas/concepts.json

### Health
[If raw count > wiki source count: "Wiki is [N] sources behind. Run `/atlas compile --incremental` to update."]
[If untracked raw files found (in raw/ but not in hashes.json): "[N] raw files not yet hashed. These will be processed on next compile."]
[If last compile was more than 7 days ago: "Last compile was [N] days ago."]
[If counts match: "Wiki is up to date."]

### Top Concepts
[Read INDEX.md, list the first 10 concepts with their one-line summaries]
```
