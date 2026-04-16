# Command: Export

This file is loaded on demand by `~/.claude/skills/atlas/SKILL.md` when the user invokes `/atlas export`. It contains the complete operational logic for the export command across all three sub-modes (slides, report, chart).

**Prerequisite:** Phase 0 (auto-detect knowledge base) must have already run from SKILL.md before this file is loaded. If it has not, STOP and return to SKILL.md.

**Invocation:** `/atlas export --slides "topic"` or `/atlas export --report "topic"` or `/atlas export --chart "description"`

## Phase 1: Research (same for all formats)

1. Read `wiki/INDEX.md`
2. Read `.atlas/concepts.json` for aliases
3. Identify concept pages relevant to the topic (using alias-aware matching)
4. Grep `wiki/concepts/` for the topic terms and their aliases
5. Read relevant articles using the same scale-adaptive pattern as `/atlas query`:
   - **Small wiki (fewer than 50 total concepts) or 5 or fewer relevant concepts:** read concept pages directly in main context.
   - **Medium wiki (50-150 concepts) with 6+ relevant concepts:** spawn a single research agent to read all relevant concepts and return synthesized notes for the export. Do NOT load 6+ files into main context.
   - **Large wiki (150+ concepts) with 3+ relevant categories:** spawn one research agent PER relevant category in PARALLEL. Each agent reads its category's sub-index, identifies relevant concepts within that category, reads them in full, and returns category-scoped notes (max 500 words per agent). The main context synthesizes the agent notes into the export.

## Phase 2: Branch on Sub-Mode

Based on the flag passed in the invocation:
- `--slides` → execute Phase 2a only, skip Phase 2b and Phase 2c
- `--report` → execute Phase 2b only, skip Phase 2a and Phase 2c
- `--chart` → execute Phase 2c only, skip Phase 2a and Phase 2b

If no sub-mode flag was passed, STOP and ask the user which format they want.

## Phase 2a: Slides (--slides)

1. Generate a slug: `[topic-slug]-slides`
2. Write a Marp-formatted markdown file at `wiki/slides/[slug].md`:

```markdown
---
marp: true
theme: default
paginate: true
---

# [Topic Title]

[Subject] Knowledge Base
[Today's date]

---

## [Concept 1 Title]

- [Key point 1]
- [Key point 2]
- [Key point 3]

---

## [Concept 2 Title]

- [Key point 1]
- [Key point 2]

---

[Continue for each relevant concept, one slide per concept]

---

## Summary

[3-5 bullet points synthesizing the key takeaways across all slides]

---

## Sources

[List concept pages consulted]
```

3. Rules for slides:
   - Maximum 5 bullet points per slide
   - One concept per slide
   - Use plain language, no jargon without inline definition
   - Total slides: 8-15 depending on topic breadth

4. Output: "Slides saved to `wiki/slides/[slug].md`. View in Obsidian with the Marp plugin, or export to HTML/PDF with: `npx @marp-team/marp-cli wiki/slides/[slug].md --html`"

## Phase 2b: Report (--report)

1. Generate a slug: `[topic-slug]-report`
2. Write a long-form report at `wiki/reports/[slug].md`:

```markdown
# [Topic Title]: A Comprehensive Report

Generated: [today's date]
Knowledge base: [Subject]
Concepts consulted: [count]

---

## Executive Summary

[3-5 sentences capturing the key findings]

## [Section 1 Heading]

[Detailed content with citations to concept pages]

## [Section 2 Heading]

[Continue...]

## [Additional sections as needed]

## Conclusion

[Key takeaways, open questions, suggested next steps]

---

## Sources

[Every concept page and raw source consulted, with links]
```

3. Target length: 1500-3000 words depending on topic breadth
4. Every major claim must cite the concept page it comes from
5. Offer to file new concepts back into wiki (same as Query Phase 3)

## Phase 2c: Chart (--chart)

1. Query the wiki for data relevant to the chart description
2. Generate a slug: `[topic-slug]-chart`
3. Write a Python script to `/tmp/claude_scratch_atlas_chart_[slug].py` that uses matplotlib to create the visualization
4. The script must:
   - Use data extracted from wiki concept pages (hardcoded into the script, not read at runtime)
   - Save output to `wiki/images/[slug].png`
   - Use a clean style (`plt.style.use('seaborn-v0_8-whitegrid')` or similar)
   - Include title, axis labels, and legend where appropriate
5. Run the script via Bash: `python3 /tmp/claude_scratch_atlas_chart_[slug].py`
6. Delete the temp script after execution: `rm /tmp/claude_scratch_atlas_chart_[slug].py`
7. If the chart is useful as a wiki asset, ask: "Chart saved to `wiki/images/[slug].png`. Should I create a concept page or report that references this chart?"
8. Output: "Chart saved to `wiki/images/[slug].png`. View in Obsidian or any image viewer."
