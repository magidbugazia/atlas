# Command: Compile

This file is loaded on demand by `~/.claude/skills/atlas/SKILL.md` when the user invokes `/atlas compile`. It contains the complete operational logic for the compile command, including all three sequential agent prompts (Concept Compiler, Summary Writer, Index Builder) and all three compile paths (Small, Standard, Large).

**Prerequisite:** Phase 0 (auto-detect knowledge base) must have already run from SKILL.md before this file is loaded. If it has not, STOP and return to SKILL.md.

**Invocation:** `/atlas compile` (full rebuild) or `/atlas compile --incremental` (new sources only)

## Phase 1: Inventory

1. Glob `raw/` recursively for all files. Build a complete file list with paths.
2. If the file list is empty: tell the user "No raw material found in `raw/`. Add sources first with `/atlas ingest` or by placing files directly in the `raw/` subdirectories." and STOP.
3. Read `wiki/INDEX.md` to know what the wiki currently contains (if it exists).
4. Read `.atlas/concepts.json` for the concept registry (if it exists).
5. Read `KB.md` for `last_compiled` date.

**For `--incremental` (hash-based change detection):**
- Read `.atlas/hashes.json` for stored hashes
- Compute current hashes for ALL raw files: use Glob to get all file paths in `raw/` recursively, then pass the file list to a single Bash `shasum -a 256 [file1] [file2] ...` command. Parse the output into a hash map.
- Compare against stored hashes. Include a file in the compile list ONLY if: (a) it has no stored hash (new file), or (b) its current hash differs from the stored hash (modified file).
- If no files qualify: tell user "Wiki is up to date. No new or changed raw material since last compile on [date]." and STOP.
- Proceed with only the changed file list.

**Detect manual drops (both modes):**
After building the file list, check each raw file for frontmatter (Grep for `^---` in the first 5 lines). Files without frontmatter were likely dropped manually (via Obsidian Web Clipper, file explorer, etc.). For each:
- Create minimal frontmatter from what's available: `title` from filename (dehyphenate, title-case), `date_ingested` from file modification date (via Bash `date -r [file] +%Y-%m-%d`), `source: unknown (manual drop)`.
- Prepend the frontmatter to the file using Edit.
- Add the file to `.atlas/hashes.json` with its current hash.
- Log to output: "Detected [N] manually added files without metadata. Added minimal frontmatter."

**For full compile:**
- Process ALL raw sources. Warn the user if this will process more than 20 files: "Full compile will process [N] raw sources. This may take a few minutes. Proceed?"

## Phase 1.5: Frontmatter Backfill

Before launching agents, ensure every existing wiki page has YAML frontmatter. This step is a one-time backfill that auto-completes itself the first time atlas compiles a KB after frontmatter became required. After the first run, this step is a fast no-op because every page already has frontmatter.

1. Glob `wiki/concepts/` and `wiki/summaries/` for all `.md` files. If neither directory exists yet (brand-new KB), skip this phase entirely.
2. For each file, check whether the first non-empty line is `---` (the YAML frontmatter fence). If yes, skip the file (already has frontmatter).
3. For each file WITHOUT frontmatter:
   - Read the file
   - Extract the title from the first H1 heading. If no H1 exists, derive it from the filename slug (dehyphenate, title-case)
   - Get the file's modification date via Bash: `date -r [file] +%Y-%m-%d`. Use this for both `created` and `last_updated`
   - For concept pages (in `wiki/concepts/`): count entries in the `## Sources` section to derive `source_count`. If no Sources section exists, set `source_count: 0`. Set `status: review_pending` (the page predates frontmatter and is queued for review by either the next compile pass or by lint)
   - For summary pages (in `wiki/summaries/`): count entries in the `## Concepts Contributed To` section to derive `concept_count`. Set `status: review_pending`
   - Build the YAML frontmatter block and prepend it to the file using Edit. Concept page schema: `title`, `created`, `last_updated`, `source_count`, `status`. Summary page schema: `title`, `created`, `last_updated`, `concept_count`, `status`
4. Log to output: "Backfilled frontmatter on [N] concept pages and [M] summary pages." If both counts are 0, skip the log line.

This backfill is idempotent. Pages that already have frontmatter are skipped. Pages added by older compiles get their frontmatter on the first run after this step exists; subsequent runs are no-ops. Backfilled pages are tagged `status: review_pending` so lint can surface them for human review.

## Phase 2: Large Source Detection

Filter the file list to text-based formats only (`.md`, `.txt`, `.py`, `.js`, `.ts`, `.csv`, `.jsonl`) before checking line counts. Skip binary files (PDFs, images, parquet) from the splitting logic — their companion `.md` files will be in the list if they were ingested correctly.

For each text-based raw source in the file list, check line count via Bash (`wc -l`).

If any single source exceeds 1000 lines:
1. **Cleanup first.** Delete any existing files from previous compiles for this source via Bash: `rm -f .atlas/splits/[source-slug]-section-*.md`. This prevents stale section files from accumulating and must happen BEFORE step 3 writes the new split files (otherwise the cleanup would erase the writes).
2. Read ONLY the file's structure in main context: first 50 lines (for title and table of contents) and Grep for heading patterns (`^## `, `^### `, `^# `) to identify section boundaries.
3. Split the file into logical sections. Write each section to `.atlas/splits/[source-slug]-section-NN.md` with a header: `<!-- Split from [original filename], section: [heading] -->`. Create `.atlas/splits/` if it doesn't exist.
4. Replace the original monolith in the file list with the section files for compilation. The original file stays in `raw/` untouched. The section files are working copies.
5. Tell the user: "Split [filename] ([N] lines) into [M] sections for compilation. Original preserved in `raw/`."

## Phase 3: Batch Planning

Count total lines across ALL files in the compile list via Bash (`wc -l` on each file, sum the results).

**Why three paths exist.** The path choice exists to manage main-context window pressure. Reading a raw source in main context costs tokens proportional to source size; spawning subagents to read instead protects main context but costs orchestration overhead and requires the source to be re-read. The right path depends on how much of the new content can be absorbed in main context without crowding out the conversation, and whether the source is already in main context from a chained `/atlas ingest` in the same session (a common pattern — ingest reads the source for Phase 2 quick analysis, so it is already loaded by the time compile runs).

**Small compile (ALL conditions must be true):**
(a) Incremental compile, not full rebuild
(b) 1-3 new sources
(c) Total lines across new sources is under 2000

If ALL conditions are true, handle directly without agents (see "Small Compile Path" below). The 2000-line ceiling is sized for the realistic ingest+compile chain: a single ingested source up to ~1500 lines, or two-to-three smaller sources, can be processed in main context without spawning subagents. The previous 200-lines-per-source / 400-total cap was too tight for the chained-ingest case — it forced subagent spawning on already-loaded sources, which is wasted orchestration.

**Standard compile (total lines across all sources <= 8000):**
Process all sources in a single agent round (see "Standard Compile Path" below). Use this path for incremental compiles that exceed Small's 2000-line ceiling, OR for any full compile (which by definition processes all sources). The Standard Path's "Context Window Protection" rule (subagents read raw sources, main context only orchestrates) applies cleanly here because the sources are NOT already loaded in main context — full compile starts from a clean slate, and oversized incremental compiles are pulling in sources beyond what main context should hold.

**Large compile (total lines across all sources > 8000):**
Split the file list into batches of ~10 sources each (keeping related files together when possible by grouping files from the same `raw/` subdirectory). Process each batch sequentially. Each batch after the first reads existing concept pages from prior batches before writing, so knowledge accumulates correctly. See "Large Compile Path" below.

## Small Compile Path

Handle directly without agents. The context cost is minimal:
1. Read `.atlas/concepts.json` for the concept registry.
2. For each concept in the new sources:
   - Check the concept registry for existing slugs or aliases matching this concept
   - Grep `wiki/concepts/` for existing pages about this concept
   - If exists: Read the existing page. Edit it to incorporate new information from the new source. Add the new source to the "Sources" section at the bottom. Update cross-links. Update the YAML frontmatter: set `last_updated` to today, recompute `source_count` (count entries in the Sources section), leave `created` and `status` alone unless the new content contradicts existing claims (in which case bump status to review_pending). Pure enrichment (new source agrees with or extends existing synthesis) MUST NOT change status — review_pending is reserved for genuine contradictions or material additions that warrant human review.
   - If not exists: Create a new concept page at `wiki/concepts/[concept-slug].md` starting with YAML frontmatter (title, created=today, last_updated=today, source_count, status=draft) followed by the standard concept page structure.
   - Update `.atlas/concepts.json` with any new concepts (slug, display name, 2-3 aliases)
3. Create summary pages for new raw sources at `wiki/summaries/[source-slug].md` starting with YAML frontmatter (title, created=today, last_updated=today, concept_count, status=reviewed) followed by the standard summary page structure.
4. Update the relevant sub-index in `wiki/indexes/` with new concept entries. If the new concepts don't fit an existing category, create a new sub-index.
5. Update `wiki/INDEX.md` category summaries (update concept counts, add new categories if created).
6. Proceed to Phase 4.

## Standard Compile Path

**Full compile cleanup:** If this is a full compile (not incremental), delete all files in `wiki/concepts/`, `wiki/summaries/`, and `wiki/indexes/` before launching agents. Reset `.atlas/concepts.json` to `{}`. Preserve `wiki/reports/`, `wiki/lint/`, `wiki/slides/`, `wiki/images/`, and `wiki/INDEX.md`. This prevents stale pages from sources that were removed since the last compile.

**IMPORTANT: Context Window Protection.** Never read raw source files in the main conversation context. ALL reading and processing of raw sources happens inside subagents. The main context only manages agent orchestration and receives structured summaries in return.

Agents run SEQUENTIALLY. Agent 1 must finish before Agent 2 starts (so Agent 2 can reference actual concept filenames). Agent 3 runs after Agent 2.

**Step A:** Spawn Agent 1 via Agent tool (`subagent_type: general-purpose`). Wait for completion.
**Step B:** Spawn Agent 2 via Agent tool (`subagent_type: general-purpose`). Wait for completion.
**Step C:** Spawn Agent 3 via Agent tool (`subagent_type: general-purpose`).

**Agent 1: Concept Compiler**
```
You are building concept pages for a knowledge base wiki.

Subject: [from KB.md]
Scope: [from KB.md, what topics are in/out of scope]
Compile mode: [full | incremental]
KB root: [path]

CONCEPT REGISTRY (read first):
Read .atlas/concepts.json. This is a JSON object mapping slugs to {name, aliases}. Example:
{"prompt-engineering": {"name": "Prompt Engineering", "aliases": ["prompt design", "prompt optimization"]}}

When creating or updating concepts:
- Check the registry BEFORE creating a new page. If an existing slug or alias matches
  the concept you want to create, update the existing page instead.
- After creating any new concept page, add it to .atlas/concepts.json with its slug,
  display name, and 2-3 common aliases or alternate phrasings.
- If you rename or merge concepts, update the registry to reflect the change.

If compile mode is INCREMENTAL: existing concept pages are in wiki/concepts/. Read them
before writing to avoid duplicates. UPDATE existing pages rather than creating new ones
when the concept already has a page.
If compile mode is FULL: create all pages fresh. Rebuild concepts.json from scratch.

Raw sources to process:
[list every file path]

EXTRACTION PROFILES BY SOURCE TYPE:
The depth and shape of extraction depends on the type of the source. Infer type from the `raw/` subdirectory the source lives in, then apply the matching profile when reading that file:

- `raw/papers/` (whitepapers, academic papers, long-form reports, PDF-extracted files): Extract SECTION BY SECTION. Preserve the structure of the argument. For each section, capture the main claim, the evidence presented, and any cited authorities. Long sources deserve resolution; do not collapse a 50-page paper into one paragraph.

- `raw/articles/` (blog posts, web articles, opinion pieces): Extract KEY CLAIMS plus the evidence cited for each claim. Preserve the author's argument structure. Distinguish between claims that are well-supported in the article and claims asserted without evidence.

- `raw/notes/` when the file looks like a TRANSCRIPT (interview, meeting, podcast, conversation; speaker labels or timestamps present): Extract DECISIONS made, ACTION ITEMS mentioned, and 2-4 notable verbatim QUOTES. Decisions and action items are higher value than the surrounding chatter.

- `raw/notes/` when the file looks like a THREAD or TWEET (short, under ~500 words, no code, no speaker labels): Extract the CORE INSIGHT, the CONTEXT that makes it legible, and any DIRECTIONAL CLAIM (where the field is heading). Short sources deserve compact treatment; do not over-extract.

- `raw/notes/` when the file is a FREE-FORM NOTE (anything else): Treat as prose. Extract main ideas and any explicit decisions or open questions the author noted.

- `raw/repos/` (code files: `.py`, `.js`, `.ts`, `.go`, `.rs`, etc.): Extract ARCHITECTURE (key classes and functions and how they fit together), PATTERNS used, and DEPENDENCIES (external libraries imported). Capture rationale comments verbatim per the rationale extraction rule. Do not try to summarize what every line does; the architecture is the value.

- `raw/datasets/` (companion `-schema.md` files for tabular data): Extract the SCHEMA (columns, types) and any patterns visible in the first 3 rows. The dataset itself is the source of truth; the concept page just describes what kind of data lives there.

- `raw/images/` (image files with no companion text): SKIP. Atlas's compile agents do not have vision tools; image-only sources contribute nothing to concept pages until a future version adds vision support.

If the source type is ambiguous (e.g., a file in `raw/notes/` that could be either a transcript or a thread), use length and structure as tiebreakers: short with no speaker labels equals thread, long with speaker labels or timestamps equals transcript.

TASK:
1. Read .atlas/concepts.json for the current registry
2. Read all listed raw source files
3. Identify every distinct concept across all sources
4. For each concept, create or update ONE wiki page at wiki/concepts/[concept-slug].md
5. Update .atlas/concepts.json with all concepts (new and existing)

Each concept page must contain (in this order):
- YAML frontmatter block at the very top, fenced by `---` lines. Required fields: `title` (display name), `created` (YYYY-MM-DD), `last_updated` (YYYY-MM-DD), `source_count` (integer count of distinct raw sources cited on this page), `status` (one of: draft, reviewed, review_pending). Status semantics: `draft` = newly created and not yet reviewed; `reviewed` = human has signed off on the synthesis; `review_pending` = a recent compile changed something the human should re-check (genuine contradiction with existing claims, or material addition that meaningfully shifts the page's framing). For new pages: created=today, last_updated=today, status=draft. For updates to existing pages: preserve created, set last_updated=today, recompute source_count. Bump status to `review_pending` ONLY if the new source contradicts existing claims OR materially shifts the page's framing. Pure enrichment (new source agrees with or extends existing synthesis without challenging it) MUST leave status alone — over-flagging defeats the queue.
- Title as H1
- One-paragraph summary (what this concept is and why it matters)
- Detailed explanation synthesized from ALL sources that discuss this concept
- Key points as a bulleted list, with attribution: "(Source: [filename])" after each point
- "Counter-Arguments and Data Gaps" section: a `## Counter-Arguments and Data Gaps` heading followed by the strongest available critique of the page's synthesis. Regenerate this section ONLY when (a) the page is new, or (b) the new source contradicts or materially weakens an existing claim on the page. For pure extensions — when the new source agrees with or adds detail to the existing synthesis without challenging it — leave the existing Counter-Arguments section untouched. When regenerating, surface the most credible counter-position available: a known critique from the field, an unaddressed methodological limitation, an open empirical question, a missing data category, or the strongest opposing school of thought. Do NOT leave this section blank or write "no counter-arguments found" on new pages. If you cannot find a critique, dig harder. For new pages, this section is MANDATORY.
- "Related Concepts" section with markdown links to other concept pages: [Concept Name](related-concept.md)
- "Sources" section at the bottom listing which raw files contributed to this page
- "Rationale Notes" section (OPTIONAL, only when raw sources contain rationale comments): a `## Rationale Notes` heading followed by verbatim quotes of `WHY:` / `HACK:` / `TODO:` / `FIXME:` / `NOTE:` comments extracted from the raw sources, each with attribution: `> [verbatim comment text]` followed by `(Source: [filename]:[line])`. Only include this section if at least one rationale comment was found across the cited sources. Do NOT paraphrase or summarize the comments; quote them exactly as written.

RULES:
- ONE file per concept, NOT per source. Three articles about "prompt engineering" become ONE prompt-engineering.md.
- Respect the SCOPE. If the scope says "AI engineering, not data science," don't create concept pages for topics that fall outside scope. If a raw source mentions an out-of-scope topic in passing, note it briefly within a relevant in-scope concept page but don't give it its own page.
- Before creating a new concept page, check the concept registry AND existing files for the same idea under a different name. When in doubt, merge into one page rather than creating two.
- Concept slug: lowercase, hyphens, no spaces. Example: prompt-engineering.md. If a generated slug matches an existing file, read that file first. If same concept, merge. If different concept, append a disambiguator.
- Every claim must trace to a raw source.
- Create backlinks: if concept A references concept B, BOTH files must link to each other.
- If a raw source contains image references (`![alt](path)`), preserve them in the concept page. Rewrite paths to point to `../images/[filename]`.
- Use clear, direct language. Define jargon on first use.
- Do not invent information. Only include what the raw sources actually say.
- If a raw source is too garbled to extract meaningful concepts (bad OCR, corrupted text), skip it and include it in a "Skipped Sources" list at the end of your output with the reason.
- For incremental compiles: read existing concept pages first. Merge new information into existing pages. Do not overwrite. Append and synthesize.
- The Counter-Arguments and Data Gaps section is an epistemic integrity check distinct from source attribution. On NEW pages, always generate the strongest available critique even when every source praises the same approach — one-sided pages compound bias as the wiki grows, and if you ingest 5 sources from the same camp the page should still surface what the opposing camp would say. On EXISTING pages during incremental compile, only regenerate this section when the new source actually contradicts or weakens an existing claim; skip regeneration for pure extensions. This protects the wiki from becoming an echo chamber without spending reasoning cycles re-deriving the same critique when nothing epistemically changed.
- Rationale extraction: while reading each raw source file, scan for lines containing rationale tags. Tags to capture: `WHY:`, `HACK:`, `TODO:`, `FIXME:`, `NOTE:`. Tags may be preceded by comment markers (`#`, `//`, `/*`, `>`, `--`) and any amount of whitespace. Capture each match VERBATIM (do not paraphrase) along with the source filename and line number. Group captured rationales by the concept page they belong to (a rationale belongs to a concept page if the source file contributed to that concept). Render them in the optional `## Rationale Notes` section described above. Skip the section entirely if no rationale was found. Rationale is the first thing lost in summarization, so preserving the original wording is the point: do not interpret, do not collapse multiple rationales, just quote.
```

**Agent 2: Summary Writer**
```
You are creating source summaries for a knowledge base wiki.

Subject: [from KB.md]
Compile mode: [full | incremental]
KB root: [path]

Raw sources to process:
[list every file path]

IMPORTANT: Agent 1 has already created concept pages in wiki/concepts/. Read the
filenames in wiki/concepts/ (use Glob) so you can link to them accurately.
Also read .atlas/concepts.json for the canonical concept names and slugs.

TASK:
For each raw source, create a summary page at wiki/summaries/[source-slug].md

Each summary page must contain (in this order):
- YAML frontmatter block at the very top, fenced by `---` lines. Required fields: `title` (source title), `created` (YYYY-MM-DD), `last_updated` (YYYY-MM-DD), `concept_count` (integer count of concept pages this source contributed to), `status` (one of: draft, reviewed, review_pending). Optional fields if known from the raw source's own frontmatter: `author`, `date_published`, `source_url`, `date_ingested`. For new pages: created=today, last_updated=today, status=reviewed. For updates: preserve created, set last_updated=today, recompute concept_count.
- Title as H1
- 3-5 sentence summary of what this source covers and its main argument
- "Concepts Contributed To" section: bulleted list linking to actual concept pages in wiki/concepts/ using relative paths. Match concept names against .atlas/concepts.json aliases if the exact name doesn't match a filename.
- "Key Quotes" section: 2-3 notable quotes worth preserving verbatim (with context for each)

RULES:
- One summary per raw source, no exceptions
- Summary slug derived from source title or filename
- Summaries should be concise. Concept pages have the detailed synthesis
- Use relative paths for all links: `../concepts/[slug].md`
```

**Agent 3: Index Builder (runs AFTER Agents 1 and 2)**
```
You are maintaining a two-tier hierarchical index for a knowledge base wiki.

Subject: [from KB.md]
KB root: [path]

TASK:
1. Glob wiki/concepts/ for all .md files. Read each one (at minimum the first 10 lines for the title and summary paragraph).
2. Glob wiki/summaries/ for all .md files. Read each one (first 5 lines for title and metadata).
3. Glob wiki/reports/ for any existing report files.
4. Group all concepts into thematic categories (e.g., "Foundations", "Security", "Agents", "Evaluation"). Each category should contain 5-25 concepts. If a category would have fewer than 5, merge it into the most related category. If more than 25, split it.
5. For EACH category, write a sub-index at wiki/indexes/[category-slug].md
6. Write the top-level wiki/INDEX.md linking to all sub-indexes.

## Structure of each sub-index (wiki/indexes/[category-slug].md):

```
# [Category Name]

Part of: [Subject] Knowledge Base
Concepts in this category: [N]

## Concepts

[Alphabetical list. Each entry on one line:]
- [Concept Name](../concepts/concept-slug.md): [one-line summary, max 120 characters, genuinely informative]

## Sources Contributing to This Category

[List raw sources whose concepts fall in this category:]
- [Source Title](../summaries/source-slug.md) | [type] | [date if known]
```

## Structure of INDEX.md (top-level, kept small):

```
# [Subject] Knowledge Base

Last compiled: [today's date]
Total concepts: [count of files in wiki/concepts/]
Total sources: [count of files in wiki/summaries/]
Total categories: [N]

## Categories

[One entry per category. Keep this section under 40 lines total:]
- [Category Name](indexes/category-slug.md): [2-sentence summary of what this category covers and how many concepts it contains]

## Recent Reports

[List any files in wiki/reports/, most recent first:]
- [Report Title](reports/report-slug.md) | [date]
```

The top-level INDEX.md must stay small (under 60 lines total). All detailed concept
listings live in sub-indexes. This two-tier structure ensures the LLM can always read
INDEX.md in full without context pressure, then drill into relevant categories.

PREREQUISITE CHECK:
Before starting, verify wiki/concepts/ and wiki/summaries/ contain files. If either is empty, STOP and report: "No concept pages or summaries found. Earlier agents may have failed."

RULES:
- Every concept page MUST appear in exactly one sub-index. No orphans, no duplicates across categories.
- Every sub-index MUST be listed in INDEX.md. No orphan sub-indexes.
- One-line summaries must be specific. "Core mechanism enabling transformers to weigh token relevance" is good. "An important concept" is useless.
- Alphabetical within each sub-index.
- Category themes should emerge from the actual concepts, not be imposed. If all concepts are about one topic, one category with one sub-index is fine.
- Use relative links throughout (sub-indexes use `../concepts/` paths).
- If the wiki has fewer than 15 total concepts, use a SINGLE sub-index (no need for multiple categories at small scale). INDEX.md still links to it.
```

## Large Compile Path

When total source lines exceed 8000:

**Pre-batch cleanup (full compile only):** Before launching any batch, delete all files in `wiki/concepts/`, `wiki/summaries/`, and `wiki/indexes/`. Reset `.atlas/concepts.json` to `{}`. Preserve `wiki/reports/`, `wiki/lint/`, `wiki/slides/`, `wiki/images/`, and `wiki/INDEX.md`. This gives batch 1 a clean slate.

1. Split the file list into batches of ~10 sources. Group files from the same `raw/` subdirectory together when possible.
2. For **each batch**: run Agent 1 with compile mode **INCREMENTAL** (even during a full compile — the pre-batch cleanup already wiped old pages, so incremental mode correctly merges into the growing wiki). Then run Agent 2 (reads existing summaries and concepts.json for linking).
3. **Skip Agent 3 until all batches are done.** Running the index builder per-batch is wasted work since the next batch changes the wiki again.
4. After ALL batches complete, run Agent 3 ONCE to build the final index.

Tell the user before starting: "Large compile: [N] sources split into [M] batches of ~10. Processing sequentially to stay within context limits. This will take several minutes."

## Phase 4: Verify Completeness

After all agents finish:
1. Count files in `wiki/concepts/` and `wiki/summaries/`
2. Verify `.atlas/concepts.json` has entries (not an empty `{}`)
3. If any agent reported errors, produced no output, or created zero files: warn the user: "Compile partially failed. [N] concept pages and [M] summaries were created, but agents reported errors. Run `/atlas compile` again for a full rebuild." Do NOT update `last_compiled` in KB.md. This ensures incremental compile will retry.
4. If all agents succeeded, proceed to Phase 5.

## Phase 5: Update Metadata

1. Update `KB.md`:
   - Set `last_compiled` to today's date
   - Update `wiki_count` with count of files in `wiki/concepts/`
   - Update `raw_count` with count of files in `raw/`
   - Increment `compile_count`
   - Increment `compiles_since_lint`

2. Update `.atlas/hashes.json`: for incremental compiles, merge the current hash map from Phase 1 (which already has all hashes) into the stored file. For full compiles, replace the entire file with the current hash map. This avoids recomputing hashes that were already computed in Phase 1. After merge or replace, remove any entries whose files no longer exist in `raw/` (pruning deleted sources).

3. **Write a per-run compile manifest** to `.atlas/compile-runs/[YYYY-MM-DDTHHMM].json` (create `.atlas/compile-runs/` if it doesn't exist). The manifest records what changed in this run so downstream tools — specifically lint's Report Health Auditor — can determine the change set without re-deriving it.

```json
{
  "compile_started": "[ISO 8601 UTC timestamp from Phase 1]",
  "compile_completed": "[ISO 8601 UTC timestamp at this step]",
  "incremental": true,
  "created": ["[concept-slug-1]", "[concept-slug-2]"],
  "updated": ["[concept-slug-3]"],
  "deleted": ["[concept-slug-4]"]
}
```

Field semantics:
- `compile_started` / `compile_completed`: ISO 8601 UTC timestamps. Lint's Agent 6 normalizes any date-only or epoch values to ISO 8601 UTC at read time, but write them as ISO 8601 UTC for forward compatibility.
- `incremental`: boolean — true for `compile --incremental`, false for full rebuild.
- `created`: concept slugs (filename without `.md`) for pages NEW in this compile (the agents wrote a `wiki/concepts/[slug].md` that didn't exist when this compile started). Derived from comparing the post-Phase-3 contents of `wiki/concepts/` against the pre-compile slug list.
- `updated`: concept slugs whose pages existed before this compile and were edited by the agents. Derived from the same diff.
- `deleted`: concept slugs whose pages existed before this compile but no longer exist after (rare; happens only on full compiles where a source was removed and produced no output).

The manifest is advisory state. Compile does not read its own past manifests; only lint consumes them. If writing the manifest fails (disk full, permission issue), warn the user but do not treat it as a compile failure — the rest of Phase 5 has already succeeded.

**Why the manifest does NOT track `review_pending_now`.** Earlier drafts of this schema included a `review_pending_now` field listing concepts that compile flagged for review during this run. That field is omitted because it captures the wrong signal: lint's cascade arm needs to know which concepts are CURRENTLY in `review_pending` (a state on each concept page's frontmatter), not which ones happened to enter that state during this compile. A concept flagged `review_pending` two compiles ago and not yet resolved must still trigger the cascade arm, but would not appear in the most recent manifest's per-event field. Lint Agent 6 reads concept frontmatter directly to get the authoritative current state.

## Phase 6: Git Commit

Check if the KB root is inside a git repository by using Glob for `.git/` at the KB root (e.g., Glob pattern `.git` with `path` set to the KB root, or Glob `[KB root]/.git/HEAD`). Using Glob avoids running `git rev-parse`, which is not in the default allow-list and would prompt the user on every compile.

If yes:
1. Stage wiki changes: `git -C [KB root] add wiki/ KB.md .atlas/compile-runs/`
2. Commit: `git -C [KB root] commit -m "atlas: [full|incremental] compile - [N] concepts, [M] summaries"`
3. If the commit fails (nothing to commit, hook failure, etc.), warn the user but do not treat it as a compile failure.

If no (not a git repo): skip silently.

## Phase 7: Output

```
## Compile Complete

[Full/Incremental] compile of [Subject] knowledge base.

New concept pages created: [N]
Existing concept pages updated: [N]
New source summaries created: [N]
Total concepts: [N]
Total sources: [N]
Concept registry: [N] entries in .atlas/concepts.json

Index updated: wiki/INDEX.md
[If git commit succeeded: "Changes committed: [commit hash]"]

[If compiles_since_lint > 3: "Consider running `/atlas lint`. It's been [N] compiles since last health check."]
```
