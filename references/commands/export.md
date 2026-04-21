# Command: Export

This file is loaded on demand by `~/.claude/skills/atlas/SKILL.md` when the user invokes `/atlas export`. It renders an existing report from `wiki/reports/` as an interactive HTML guide by delegating to the code-fluent skill.

**Prerequisite:** Phase 0 (auto-detect knowledge base) must have already run from SKILL.md before this file is loaded. If it has not, STOP and return to SKILL.md.

**Invocation:** `/atlas export <report-slug>`

`<report-slug>` matches the filename of a report in `wiki/reports/`, with or without the `.md` extension. Export does not research a topic. Use `/atlas query "question"` to produce a report first; then `/atlas export <slug>` to render it.

## Phase 1: Locate the Report

1. Parse `<report-slug>` from the invocation. If it ends in `.md`, strip the extension.
2. Look for `wiki/reports/<report-slug>.md` (relative to the KB root detected in Phase 0).
3. If the file does not exist:
   - List the contents of `wiki/reports/` (sort by mtime, newest first, cap at 10 filenames).
   - Tell the user: "No report found at `wiki/reports/<slug>.md`. Available reports: ..."
   - STOP. Do not proceed.
4. Read the report file fully. Capture its H1 title (`# ...`) for the guide title. If no H1 exists, use the slug humanized (replace hyphens with spaces, title-case).

## Phase 2: Prepare the Rendering Directory

code-fluent operates on directories, not single files. Set up a minimal temp directory that gives code-fluent exactly the content it needs and nothing more.

1. Compute the temp directory path: `/tmp/claude_scratch_atlas_export_<slug>/`.
2. If the temp directory already exists, remove it first: `rm -rf /tmp/claude_scratch_atlas_export_<slug>/`.
3. Create the temp directory.
4. Copy `wiki/reports/<slug>.md` into the temp directory, preserving the filename.
5. Copy the KB's `KB.md` into the temp directory, but **rename the copy to `README.md`**. This gives code-fluent project-level context and triggers its "top-level README detection" in Phase 1.
6. Do NOT copy concept pages, other reports, INDEX.md, or `.atlas/` metadata. The guide renders this one report, not the whole wiki.

## Phase 3: Invoke code-fluent via Subagent

Spawn a general-purpose subagent. The subagent reads code-fluent's SKILL.md and executes it against the temp directory, then returns the path to the built guide.

Use this exact prompt (substitute `<TEMP_DIR>`, `<SLUG>`, and `<REPORT_TITLE>` with computed values):

```
You are rendering an atlas report as an interactive HTML guide by executing the code-fluent skill.

1. Read /Users/magidbugazia/.claude/skills/code-fluent/SKILL.md in full. Also read /Users/magidbugazia/.claude/skills/code-fluent/templates/guide.html for JSON component schemas.

2. Execute code-fluent's Phase 1 (Scan and Route) against the directory <TEMP_DIR>. The directory contains a single report markdown file and a README.md (the knowledge base's KB.md, renamed for context). Routing will resolve to doc-path — zero source-code files are present.

3. Skip code-fluent's AskUserQuestion audience step. Do not attempt to ask the user. Use this audience label: "Technical reader, no role-specific tailoring." Do not pitch content to engineers, data scientists, PMs, or business analysts. Use the report's own technical vocabulary. Explain domain-specific or project-specific terms inline via data-definition spans. Do not simplify for a lay reader.

4. Execute Phase 2 (Gather Context) using only the README.md in <TEMP_DIR>. Do not attempt to read conversation history, memory files, or git history — none are relevant to this render.

5. Execute Phase 3 (Build the Guide) using doc-path module types only (overview, reference, narrative, framework, assessment). 5-7 modules. The report title is "<REPORT_TITLE>"; use it as the --title argument and as the guide's H1. Use <SLUG> as the --nav argument truncated to the first 2-3 words.

6. Derive the output filename as <SLUG>-guide.html. Write modules.html in <TEMP_DIR>, run build.py to produce <TEMP_DIR>/<SLUG>-guide.html, then delete modules.html. Do NOT open the browser.

7. Return the absolute path to the generated guide.html. Also report any rule violations or places the code-fluent instructions were unclear when applied here.
```

The subagent returns the absolute path to the built guide. If the subagent reports that code-fluent's instructions were unclear or that rule violations occurred, surface those to the user verbatim at the end of export.

## Phase 4: Finalize Output

1. Move the built guide from the temp directory to the KB's `wiki/reports/` directory:
   `mv /tmp/claude_scratch_atlas_export_<slug>/<slug>-guide.html wiki/reports/<slug>-guide.html`
2. Remove the temp directory: `rm -rf /tmp/claude_scratch_atlas_export_<slug>/`
3. Per atlas Rule 15, if the KB root is a git repo, stage `wiki/reports/<slug>-guide.html` and commit with message: `Export guide: <slug>`. If not a git repo, skip silently.
4. Output to the user:

   ```
   Guide saved to wiki/reports/<slug>-guide.html
   Source report: wiki/reports/<slug>.md
   Open: open wiki/reports/<slug>-guide.html
   ```

   If the subagent surfaced any issues in Phase 3, append them below this output under a `## Notes from rendering` heading.

## Rules for Export

1. Export does not research. If the user passes a topic string instead of an existing report slug (e.g., `/atlas export "RAG tradeoffs"` with no matching file), treat the missing-file path in Phase 1 step 3 and suggest they run `/atlas query` first.
2. Do not modify the source report markdown. The guide is a rendering; the source stays canonical.
3. Do not copy wiki concept pages into the temp directory. The report already synthesized them. Including them risks the guide drifting into "whole KB" territory, which is a different product (reserved as Position 3 browse-mode view — not implemented).
4. If the subagent fails or times out, report the failure with the agent's error message. Do not silently fall back to a different rendering path.
