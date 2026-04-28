# Command: Ingest

This file is loaded on demand by `~/.claude/skills/atlas/SKILL.md` when the user invokes `/atlas ingest`. It contains the complete operational logic for the ingest command.

**Prerequisite:** Phase 0 (auto-detect knowledge base) must have already run from SKILL.md before this file is loaded. If it has not, STOP and return to SKILL.md.

**Invocation:** `/atlas ingest <URL>` or `/atlas ingest <file path>` or `/atlas ingest <directory path>`

## Phase 1: Load Source

Parse what follows `ingest`:

**If URL** (starts with `http`):

1. **Fidelity warning first.** Before fetching, tell the user: "Direct URL ingest runs the content through an LLM (WebFetch), which can paraphrase or compress. For fidelity-critical sources (material you'll quote directly, cite publicly, or rely on for exact terminology), save the page locally first using a browser clipper (Obsidian Web Clipper, SingleFile, MarkDownload, or browser 'Save Page As...') and then run `/atlas ingest <saved path>` for a verbatim copy. Proceed with URL ingest anyway? (Best for routine topical coverage.)" If they decline, stop. If they proceed, continue.
2. **Dedup check:** Grep `raw/` recursively for the URL string in file frontmatter. If found, tell the user: "This URL was already ingested as `[existing file path]` on [date]. Update the existing copy or add a duplicate?" If they say update, overwrite the existing file. If they say skip, stop. Only proceed with a new file if they explicitly want a duplicate.
3. WebFetch the URL with a **verbatim-preservation prompt**: "Return the full article body as markdown. Preserve every paragraph, sentence, quote, heading, and code block exactly as written — do not summarize, paraphrase, compress, restructure, or editorialize. Preserve the author's exact wording. Strip only: ads, navigation menus, footers, sidebar widgets, cookie banners, social share buttons, author bios unrelated to the article, and 'related posts' sections. Keep: title, author, publish date, full body text, image alt-text for content-bearing images, code blocks, tables, and blockquotes. If uncertain whether to keep something, keep it."
4. If fetch fails (403, 404, timeout): STOP. Tell user the URL is not accessible.
5. Derive a filename from the title: lowercase, hyphens for spaces, strip special characters, max 60 characters. Example: "Understanding Transformer Architecture" becomes `understanding-transformer-architecture.md`
6. Write the extracted content to `raw/articles/[filename].md` with frontmatter:

```markdown
---
title: [extracted title]
source: [URL]
author: [if found]
date_published: [if found]
date_ingested: [today]
extraction_method: webfetch-verbatim-prompt
---

[extracted content]
```

The `extraction_method` field tells downstream steps (compile, lint Source Verifier, future you) the fidelity level of this source. Valid values: `webfetch-verbatim-prompt` (URL via WebFetch, best-effort verbatim but still LLM-mediated), `file-copy` (verbatim file copy, highest fidelity), `web-clipper` (file ingested from a browser clipper like Obsidian Web Clipper or SingleFile — verbatim), `pdf-extract` (text extracted from PDF via Read), `dataset-schema` (schema summary only, not the full data), `manual-drop` (file placed directly in `raw/` without frontmatter, backfilled during compile).

7. If the extracted markdown contains image references (`![alt](url)`), attempt to download each image:
   - Skip decorative images, avatars, ads, logos, and tracking pixels (images under 5KB or with names like `logo`, `avatar`, `icon`, `pixel`, `tracking`).
   - For each significant image: use WebFetch to download it. Save to `raw/images/[article-slug]-[N].ext` where N is a counter and ext matches the original format.
   - Rewrite the markdown image references in the saved article to point to local paths: `![alt](../images/[article-slug]-[N].ext)`
   - If any image download fails (403, timeout, etc.), keep the original remote URL and log which images couldn't be fetched.
   - Tell the user: "Downloaded [N] of [M] images to `raw/images/`. [If any failed: '[K] images couldn't be fetched and still reference remote URLs.']"

**If file path** (contains `/` or `.`):

1. Verify the file exists using Read
2. **Dedup check:** If the file is already inside `raw/`, note it and stop. If not, derive the target filename and check if a file with that name already exists in the target `raw/` subdirectory. If it does, warn the user and ask how to proceed.
3. **Detect clipper origin.** If the source file is `.md`, scan its content and existing frontmatter for browser-clipper signatures: Obsidian Web Clipper typically adds `source:` or `url:` frontmatter keys plus a "Clipped from" footer; SingleFile adds HTML comments like `<!-- Page saved with SingleFile`; MarkDownload adds a specific frontmatter shape. If any signature is present, set `extraction_method: web-clipper` in the copied file's frontmatter. If no signature is present but the file is a plain `.md`, use `extraction_method: file-copy`.
4. If the file is elsewhere: read it and write a copy to the appropriate `raw/` subdirectory with `extraction_method` set per step 3 (for `.md`) or per the mapping below:
   - `.md` files go to `raw/articles/` with `extraction_method: web-clipper` or `file-copy` (from step 3)
   - `.pdf` files go to `raw/papers/`. After copying, extract text content using the Read tool (which supports PDFs with the `pages` parameter — read in batches of 20 pages). Write extracted text to a companion file at `raw/papers/[name]-extracted.md` with frontmatter noting the original PDF path, page count, and `extraction_method: pdf-extract`. During compile, agents read the extracted `.md` file, not the PDF binary.
   - `.py`, `.js`, `.ts` and other code files go to `raw/repos/` with `extraction_method: file-copy`
   - `.csv`, `.parquet`, `.jsonl`, `.xlsx` go to `raw/datasets/`. After copying, extract a brief schema summary (column names, row count, first 3 rows for CSV/JSONL) and prepend it as a markdown comment at the top of a companion `.md` file: `raw/datasets/[name]-schema.md` with `extraction_method: dataset-schema`. This schema file is what agents read during compile (they can't parse raw data formats directly).
   - `.png`, `.jpg`, `.svg` and other image files go to `raw/images/` (no companion markdown, no extraction_method)
   - Everything else goes to `raw/notes/` with `extraction_method: file-copy`

**If directory path:**
1. Glob for all files in the directory (non-recursive unless user specifies)
2. List what was found and ask user to confirm before copying
3. Process each file as above

## Phase 2: Quick Analysis

After saving the raw source:
1. Read the ingested content
2. Extract 3-5 key topics/concepts mentioned
3. Check if any of these concepts already exist as wiki pages (Glob `wiki/concepts/` for matching filenames, and check `.atlas/concepts.json` for alias matches)

## Phase 3: Update Metadata

1. Update `KB.md`: increment `raw_count`.

**Do NOT touch `.atlas/hashes.json` here.** Hashes belong to compile, not ingest. The `hashes.json` file tracks which raw sources have already been processed into the wiki — it is the input to compile's incremental change-detection logic. If ingest writes a hash, the very next `/atlas compile --incremental` will see a stored hash that exactly matches the current hash and conclude "already processed, skip" — even though zero wiki pages exist for the source. Hash writes happen exclusively in compile's Phase 5, after the source has actually been turned into concept pages and summaries. The next compile's Phase 1 will compute the current hash from scratch, see no stored hash, and correctly include the file in its compile list.

## Phase 4: Output

```
## Ingested: [title or filename]

Saved to: [path within raw/]
Key topics detected: [list 3-5 concepts]
[If matching wiki pages exist: "Overlaps with existing concepts: [list with links]"]
[If no matches: "New topics, not yet in wiki"]

Raw sources total: [N]
Wiki last compiled: [date from KB.md]
[If wiki is behind: "Wiki is [N] sources behind. Run `/atlas compile --incremental` to update."]
```
