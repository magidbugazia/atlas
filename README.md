# Atlas

A Claude Code skill that turns scattered research material into a structured, interconnected wiki maintained by an LLM. Based on [Andrej Karpathy's LLM Knowledge Bases workflow](https://x.com/karpathy/status/2039805659525644595) (April 2, 2026), which describes the ingest → compile → query → lint loop that Atlas implements as a skill.

You collect raw sources. Atlas organizes them into concept pages with summaries, cross-links, and a two-tier hierarchical index. You query it, and the LLM researches answers from the wiki and writes them as files that feed back into the knowledge base. The knowledge compounds over time.

## The Problem It Solves

You research a topic deeply. You read 50 articles, 20 papers, browse 10 repos. The knowledge ends up scattered across browser tabs, PDFs in Downloads, and your unreliable memory. Or you write everything into one massive file that becomes unnavigable past 2000 lines.

Atlas solves both. Raw material goes into one directory. The LLM compiles it into a structured, navigable, queryable wiki. The LLM maintains the wiki. You rarely edit it directly.

## Key Design Decisions

**One file per concept, not per source.** If three articles discuss "prompt engineering," they become ONE concept page that synthesizes all three. Knowledge converges into concepts, not sources.

**Two-tier hierarchical indexing.** INDEX.md lists categories (kept under 60 lines). Each category has its own sub-index in `wiki/indexes/` with detailed concept summaries. The LLM never reads more than ~30 summaries at once, even at 500+ concepts. At small scale (under 15 concepts), a single sub-index is used.

**Scale-adaptive query.** Small wikis: direct concept reads. Medium wikis: single research agent. Large wikis (150+ concepts): parallel agents per category, each scoped to its own sub-index. Prevents context rot at any scale.

**Dual indexing: navigation vs. lookup.** INDEX.md is the human-readable table of contents, organized by category for browsing in Obsidian. `.atlas/concepts.json` is the machine-parseable registry mapping every concept slug to its display name and aliases. They can't be collapsed into one: INDEX.md is organized by theme (useless for answering "does a page for RAG already exist?"), and concepts.json is a flat key-value map (useless for browsing). During compile, agents check concepts.json to avoid creating duplicate pages. During search and query, it expands aliases. INDEX.md handles everything else.

**Alias-aware search.** The concept registry (`.atlas/concepts.json`) maps every concept to its canonical name and known aliases. Searching for "RAG" finds "Retrieval-Augmented Generation." Used by query, search, and export.

**Source traceability.** Every claim in the wiki cites the raw source it came from. Lint includes a Source Verifier agent that spot-checks claims against cited sources to catch hallucinations introduced during compilation.

**Git-backed operations.** Every compile, query filing, and lint fix auto-commits if the knowledge base is in a git repo. Rollback, history, and diffable changes.

**Outputs as files, not chat.** Answers get written as markdown files in the wiki. You can file them back in to enrich future queries. Each question makes the wiki more complete.

## How Atlas and Mentor Work Together

Atlas organizes knowledge. It does not judge whether a source is worth including (that is `/mentor evaluate`), test your understanding (that is `/mentor interview`), or check industry alignment (that is `/mentor audit`).

Mentor decides WHAT belongs. Atlas does the WORK of organizing it. Atlas lint checks internal consistency and source accuracy. Mentor audit checks external alignment with the job market.

## What to Expect

**Image handling.** When you ingest a URL, Atlas downloads significant images from the article and saves them to `raw/images/`. Decorative images (logos, avatars, icons, tracking pixels, anything under 5KB) are skipped. Markdown image references in the saved article are rewritten to point to the local copies. During compile, image references carry through into concept pages automatically. If a download fails, the original remote URL is kept.

**Dataset ingestion.** When you ingest a `.csv`, `.parquet`, `.jsonl`, or `.xlsx` file, Atlas saves it to `raw/datasets/` and creates a companion `-schema.md` file with column names, row count, and the first 3 rows. Compile agents read the schema file, not the raw data (they cannot parse binary formats directly).

**Duplicate detection.** Before adding a new raw source, Atlas checks if the same URL or filename already exists in `raw/`. If it finds a match, it asks whether to update, duplicate, or skip.

**Manual drops.** You can place files directly into `raw/` (via Obsidian Web Clipper, file explorer, drag-and-drop). During compile, Atlas detects files without frontmatter, adds minimal metadata (title from filename, date from file timestamp, source marked as "manual drop"), and includes them in the compile.

**Query filing.** After answering a query, Atlas offers to extract new concepts from the answer back into the wiki. Accepting this creates or updates concept pages, enriching future queries.

**Lint health scores.** Lint produces a health rating: HEALTHY (0 critical issues, under 3 important, 0 unsupported claims), NEEDS ATTENTION (1-3 critical or 5+ important), or DEGRADED (4+ critical, contradictions, or 3+ unsupported claims from the hallucination audit).

**Web search quarantine.** If a query requires web search, Atlas always asks first. Web results are saved to `raw/` and compiled through the normal pipeline. They are never injected directly into wiki pages.

## Lint-Driven Improvement

Running `/atlas lint` produces a health report saved to `wiki/lint/lint-YYYY-MM-DD.md`. The report identifies issues, suggests improvements, and includes ready-to-run commands for follow-up. Nothing is lost between sessions.

The lint report has five sections, each with a different follow-up path:

| Lint Section | What it finds | How to act on it |
|-------------|--------------|-----------------|
| **Issues** | Broken links, orphan pages, contradictions, registry drift | Accept auto-fix when prompted, or fix manually |
| **Source Verification** | Claims that don't match the cited source (hallucination audit) | Edit the concept page to match what the source actually says |
| **Suggested Connections** | Concept pairs that should cross-link but don't | Add links manually, or wait for next compile |
| **New Article Candidates** | Topics referenced by many pages but with no page of their own | Run the `/mentor evaluate` command in the report (includes `--context` paths to relevant wiki pages) |
| **Research Questions** | Gaps the wiki almost answers but doesn't fully cover | Run the `/atlas query` command in the report |

The key design: atlas finds the gaps and prepares the commands. Mentor evaluates whether to fill them. Atlas organizes the result. The user decides what to pursue.

Atlas works without mentor. Lint still saves reports and auto-fixes structural issues. The `/mentor evaluate` commands in the report are suggestions, not dependencies. If mentor isn't installed, chase the leads manually.

## Limitations

- **No source evaluation.** Atlas ingests everything you give it. Use `/mentor evaluate` to decide what's worth adding.
- **Obsidian recommended.** Atlas generates standard markdown viewable anywhere. Obsidian adds backlinks, graph views, and Marp slide support on top.

## Pairing with a Mentor Skill

Atlas is domain-agnostic. The optional mentor skill adds opinionated curation: filtering what's worth ingesting, auditing against job market demands, and testing your understanding. Atlas works standalone. Mentor makes it smarter about what goes in. See the mentor skill's README for how to adapt it to any domain.

## Always-On Setup (Optional)

By default, atlas only acts when you explicitly invoke `/atlas <command>`. If you want atlas to surface itself automatically whenever you grep or glob inside a KB directory, install the `PreToolUse` hook. With the hook installed, every time Claude is about to call Glob or Grep in a directory containing a `KB.md` (in cwd or up to 3 parent directories), Claude sees an injected notification telling it the KB exists and to consider `/atlas search` or `/atlas query` first. The hook never blocks; it only adds context.

This turns atlas from "tool you have to remember to invoke" into an always-on context layer. Especially useful inside large KBs where grepping raw files would miss the synthesized concept pages atlas already built.

### Installation

The hook ships with atlas at `~/.claude/hooks/atlas-kb-context.py`. To enable it, add the following entry to the `PreToolUse` array in `~/.claude/settings.json`:

```json
{
  "matcher": "Glob|Grep",
  "hooks": [
    {
      "type": "command",
      "command": "python3 ~/.claude/hooks/atlas-kb-context.py"
    }
  ]
}
```

Place this entry alongside any other `PreToolUse` hooks you already have. Restart your Claude Code session for the new hook to take effect.

### How it works

The hook reads the session's working directory from stdin, walks up at most 3 parent directories looking for `KB.md`, and if found:

1. Extracts the KB's `subject` from the KB.md frontmatter (best effort)
2. Builds a context message naming the KB, pointing at `wiki/INDEX.md`, and suggesting `/atlas search` or `/atlas query` as alternatives to raw grep
3. Outputs the message via the `hookSpecificOutput.additionalContext` channel
4. Exits 0 (never blocks the underlying Glob or Grep)

If no KB.md is found in the walk, the hook exits silently and the Glob or Grep proceeds normally. The hook is fast: a single filesystem walk plus a small file read takes microseconds. It runs on every Glob and Grep, so cost adds up only if you grep thousands of times per session.

### Disabling

To disable, remove the `Glob|Grep` entry from the `PreToolUse` array in `settings.json` and restart Claude Code. The hook script can stay on disk; without the settings entry it never runs.

## Directory Structure

After `init`, `ingest`, and `compile`, your repo looks like this:

```
./
├── KB.md                          # KB metadata (subject, scope, compile stats)
├── .atlas/                        # Internal machinery (gitignored, regenerable via compile)
│   ├── hashes.json                # SHA-256 per raw file — incremental compile change detection
│   ├── concepts.json              # Slug/name/alias registry — dedup, alias search, compile coordination
│   └── splits/                    # Temp working copies when large files get split for compile
├── raw/                           # Append-only archive of source material
│   ├── articles/                  # .md files (ingested articles, guides)
│   ├── papers/                    # .pdf files
│   ├── repos/                     # Code files (.py, .js, .ts)
│   ├── datasets/                  # .csv, .parquet, .jsonl, .xlsx
│   ├── images/                    # .png, .jpg, .svg
│   └── notes/                     # Everything else
└── wiki/                          # Compiled output — the part you read
    ├── INDEX.md                   # Entry point. Lists categories, links to sub-indexes. Under 60 lines.
    ├── indexes/                   # Navigation — answers "what concepts exist in category Y?"
    ├── concepts/                  # Knowledge — answers "what is X?" One file per concept.
    ├── summaries/                 # Provenance — answers "what did source Z contribute?"
    ├── reports/                   # Query answers and exported reports.
    ├── lint/                      # Saved lint reports with actionable follow-up commands.
    ├── slides/                    # Marp slide decks from export.
    └── images/                    # Charts and downloaded images.
```

**Navigation flow:** `INDEX.md` → `indexes/[category].md` → `concepts/[topic].md` → cross-links to related concepts. `summaries/` is the reverse lookup: raw source → which concepts it contributed to.

**`.atlas/` vs `wiki/`:** `.atlas/` is machine state (hashes, registry, temp splits). `wiki/` is the human-readable output. The naming overlap between `concepts.json` (lookup table) and `concepts/` (wiki pages) is intentional — one is a flat registry for dedup and alias search, the other is the actual content.

## Commands

See SKILL.md for full operational details.

| Command | What it does | Reads | Creates / Updates |
|---------|-------------|-------|-------------------|
| `init <subject>` | Scaffold a new KB | nothing | `KB.md`, `.atlas/*`, `raw/` dirs, `wiki/INDEX.md` |
| `ingest <source>` | Add a file, URL, or directory to raw | source file/URL | `raw/[type]/`, `.atlas/hashes.json`, `KB.md` |
| `compile` | Full rebuild of wiki from all raw material | `raw/*`, `.atlas/*` | `wiki/concepts/`, `wiki/summaries/`, `wiki/indexes/`, `wiki/INDEX.md`, `.atlas/concepts.json`, `KB.md` |
| `compile --incremental` | Rebuild only new/changed sources | `raw/*`, `.atlas/hashes.json` | same as compile, only changed files |
| `query "question"` | Research answer from wiki | `wiki/INDEX.md`, `wiki/indexes/`, `wiki/concepts/` | `wiki/reports/` |
| `search "term"` | Full-text search with alias expansion | `.atlas/concepts.json`, `wiki/concepts/`, `wiki/reports/` | nothing (display only) |
| `lint` | Health check and hallucination audit | all `wiki/` files, `raw/`, `.atlas/concepts.json` | `wiki/lint/lint-YYYY-MM-DD.md`, fixes if approved, `KB.md` |
| `status` | Quick stats | `KB.md`, `.atlas/*`, `wiki/INDEX.md` | nothing (display only) |
| `export --slides "topic"` | Marp slide deck | `wiki/concepts/`, `.atlas/concepts.json` | `wiki/slides/` |
| `export --report "topic"` | Long-form report | `wiki/concepts/`, `.atlas/concepts.json` | `wiki/reports/` |
| `export --chart "desc"` | matplotlib visualization | `wiki/concepts/` | `wiki/images/` |

## Skill File Structure

The skill uses a sub-indexed pattern to keep `SKILL.md` small. SKILL.md contains the dispatcher, Phase 0 (auto-detect knowledge base), Mode Detection, and Rules for All Commands. Per-command operational logic lives in separate reference files that get loaded on demand:

```
atlas/
├── SKILL.md                          # Dispatcher (under 200 lines)
├── README.md                         # This file
├── atlas-guide.html                  # Standalone guide
└── references/
    └── commands/
        ├── init.md                   # Create KB structure
        ├── ingest.md                 # Add raw sources
        ├── compile.md                # Build wiki (3 sequential agents)
        ├── query.md                  # Hierarchical retrieval
        ├── lint.md                   # Health check (3 parallel agents)
        ├── status.md                 # Quick stats
        ├── export.md                 # Slides, reports, charts
        └── search.md                 # Alias-aware full-text search
```

When the user invokes a command, SKILL.md tells Claude to STOP and Read the corresponding file before proceeding. This keeps each invocation's context cost ~75% smaller than loading the full SKILL.md. A `/atlas status` call loads ~170 lines (SKILL.md) + ~45 lines (status.md) instead of the previous ~1160-line monolith.

## Deferred Work

Things that have been considered, decided to defer, and parked here so they're not lost.

### Tree-sitter AST extraction for `raw/repos/`

**Status:** deferred (2026-04-07).

**Idea:** Run tree-sitter (a fast, local, deterministic AST parser with bindings for Python, JavaScript, TypeScript, Go, Rust, etc.) on every code file in `raw/repos/` before the Concept Compiler agent runs. Extract classes, functions, imports, and call graphs into a structured artifact. Hand the structured data to the Concept Compiler instead of raw source text. The agent then writes concept pages from clean parsed facts rather than re-parsing source code itself.

**Why it would help:** Atlas currently treats `.py`, `.js`, `.ts` files as text and pays Claude tokens to re-parse the code semantically. For code-heavy KBs (a repo dump, a research codebase) this is expensive, slow, and error-prone. Tree-sitter does the parsing in milliseconds for free, locally.

**Why it's deferred:** Tree-sitter adds a new external dependency (Python or Node bindings, plus per-language grammar packages). Atlas is currently dependency-free aside from `git` and `shasum`. Adding tree-sitter is a real install step the user must perform, and it changes atlas from a pure markdown tool into something with a binary toolchain. Worth doing eventually if atlas usage shows code-heavy KBs are common, but not until then.

**What would need to change when revisited:**
- Add a tree-sitter dependency (probably `tree_sitter` Python package) and document the install
- Add a new compile pre-pass that runs tree-sitter on every file in `raw/repos/` and writes structured AST output to a temp location
- Update the Concept Compiler agent prompt to read the structured AST output for code sources instead of the raw file
- Add a fallback path: if tree-sitter is not installed, fall back to the current behavior (Claude parses code as text)

**Source of the idea:** graphify (`safishamsi/graphify`), which uses tree-sitter for the same purpose in its two-pass extraction pipeline. Atlas borrows the idea but does not adopt graphify's NetworkX graph data model.
