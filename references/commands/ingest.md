# Command: Ingest

This file is loaded on demand by `~/.claude/skills/atlas/SKILL.md` when the user invokes `/atlas ingest`. It contains the complete operational logic for the ingest command.

**Prerequisite:** Phase 0 (auto-detect knowledge base) must have already run from SKILL.md before this file is loaded. If it has not, STOP and return to SKILL.md.

**Invocation:** `/atlas ingest <URL>` or `/atlas ingest <file path>` or `/atlas ingest <directory path>`

## Phase 1: Load Source

Parse what follows `ingest`:

**If URL** (starts with `http`):

1. **Dedup check:** Grep `raw/` recursively for the URL string in file frontmatter. If found, tell the user: "This URL was already ingested as `[existing file path]` on [date]. Update the existing copy or add a duplicate?" If they say update, overwrite the existing file. If they say skip, stop. Only proceed with a new file if they explicitly want a duplicate.
2. WebFetch the URL with prompt: "Extract ALL content from this page. Include: title, author, date published, all text content, descriptions of important images/diagrams, code examples. Preserve the document structure with markdown headings."
3. If fetch fails (403, 404, timeout): STOP. Tell user the URL is not accessible.
4. Derive a filename from the title: lowercase, hyphens for spaces, strip special characters, max 60 characters. Example: "Understanding Transformer Architecture" becomes `understanding-transformer-architecture.md`
5. Write the extracted content to `raw/articles/[filename].md` with frontmatter:

```markdown
---
title: [extracted title]
source: [URL]
author: [if found]
date_published: [if found]
date_ingested: [today]
---

[extracted content]
```

6. If the extracted markdown contains image references (`![alt](url)`), attempt to download each image:
   - Skip decorative images, avatars, ads, logos, and tracking pixels (images under 5KB or with names like `logo`, `avatar`, `icon`, `pixel`, `tracking`).
   - For each significant image: use WebFetch to download it. Save to `raw/images/[article-slug]-[N].ext` where N is a counter and ext matches the original format.
   - Rewrite the markdown image references in the saved article to point to local paths: `![alt](../images/[article-slug]-[N].ext)`
   - If any image download fails (403, timeout, etc.), keep the original remote URL and log which images couldn't be fetched.
   - Tell the user: "Downloaded [N] of [M] images to `raw/images/`. [If any failed: '[K] images couldn't be fetched and still reference remote URLs.']"

**If file path** (contains `/` or `.`):

1. Verify the file exists using Read
2. **Dedup check:** If the file is already inside `raw/`, note it and stop. If not, derive the target filename and check if a file with that name already exists in the target `raw/` subdirectory. If it does, warn the user and ask how to proceed.
3. If the file is elsewhere: read it and write a copy to the appropriate `raw/` subdirectory:
   - `.md` files go to `raw/articles/`
   - `.pdf` files go to `raw/papers/`. After copying, extract text content using the Read tool (which supports PDFs with the `pages` parameter — read in batches of 20 pages). Write extracted text to a companion file at `raw/papers/[name]-extracted.md` with frontmatter noting the original PDF path and page count. During compile, agents read the extracted `.md` file, not the PDF binary.
   - `.py`, `.js`, `.ts` and other code files go to `raw/repos/`
   - `.csv`, `.parquet`, `.jsonl`, `.xlsx` go to `raw/datasets/`. After copying, extract a brief schema summary (column names, row count, first 3 rows for CSV/JSONL) and prepend it as a markdown comment at the top of a companion `.md` file: `raw/datasets/[name]-schema.md`. This schema file is what agents read during compile (they can't parse raw data formats directly).
   - `.png`, `.jpg`, `.svg` and other image files go to `raw/images/`
   - Everything else goes to `raw/notes/`

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
2. Compute content hash of the new file via Bash: `shasum -a 256 [filepath]`
3. Read `.atlas/hashes.json`, add the new entry (`"relative/path": "hash"`), write it back.

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
