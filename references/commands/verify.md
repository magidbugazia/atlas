# Command: Verify

This file is loaded on demand by `~/.claude/skills/atlas/SKILL.md` when the user invokes `/atlas verify`. It contains the complete operational logic for the verify command.

**Prerequisite:** Phase 0 (auto-detect knowledge base) must have already run from SKILL.md before this file is loaded. If it has not, STOP and return to SKILL.md.

**Invocation:** `/atlas verify` (default scope: all unverified — `review_pending` and `draft`), `/atlas verify pending`, `/atlas verify drafts`, or `/atlas verify <concept-slug>`

## Why this command exists

Compile flags concept pages `status: review_pending` on genuine contradictions, material framing shifts, and the one-time frontmatter backfill. Compile ALSO creates every brand-new concept page as `status: draft` — machine-written, never vetted. Both `review_pending` and `draft` are unverified states, and verify's spot-check resolves them identically, so verify's default scope sweeps both: compiling new sources and then running `/atlas verify` is the normal path to move fresh pages to `reviewed` without naming each slug. (Before this scope was widened, `draft` pages fell outside the default sweep and could only be cleared per-slug, so a compile-heavy KB accumulated unverified drafts with no bulk clearing path.) Concept pages are the only thing in atlas that carries these flags (reports are dated snapshots with no review lifecycle; their currency is informational, surfaced by lint). Lint surfaces unverified concepts but never clears them, and compile never flips a concept back. Empirically the flags are high-recall, low-precision (two manual triage passes across ~140 spot-checked claims found zero drift). Verify is the spec-defined clearing path: spot-check each unverified page's claims against its raw sources, auto-clear the clean ones, and leave only genuine problems flagged.

Verify resolves; lint surfaces. Verify does not duplicate lint Agent 3 (the per-source Source Verifier sampling the whole wiki) — it reuses Agent 3's claim taxonomy and exemption rule, scoped to the pending queue, and unlike lint it writes the resolution.

## Phase 1: Determine Scope

Parse what follows `verify`:

- **No argument (default, scope `unverified`):** Glob `wiki/concepts/*.md`, parse each page's YAML frontmatter, collect every page whose `status` is `review_pending` OR `draft`. Both are unverified states that verify resolves identically: `review_pending` is a previously-reviewed page a later compile disturbed; `draft` is a newly compiled page never vetted. Sweeping both by default means a fresh compile's new pages clear on the next `/atlas verify` without the user naming each slug.
- **`pending`:** only pages with `status: review_pending` (the narrower "flagged for re-check" queue).
- **`drafts`:** only pages with `status: draft` (newly compiled, never vetted).
- **`<concept-slug>`:** the single page `wiki/concepts/<slug>.md`. If it does not exist, check `.atlas/concepts.json` aliases for a match; if still nothing, tell the user and STOP. A page in any status may be verified explicitly (re-verifying a `reviewed` page is allowed; it refreshes the `revision_note`). (`pending` and `drafts` are reserved scope keywords, not slugs; a concept page must not use either as its slug.)

If the scope is empty: tell the user "Nothing to verify. No concept pages are review_pending or draft." and STOP.

Tell the user the scope before spawning: "[N] pages to verify: [slugs]. Spawning [M] verifier agents."

## Phase 2: Spawn Verifier Agents

Batch the scoped pages 3-4 per agent. Spawn all agents IN PARALLEL via Task tool (`subagent_type: general-purpose`). Context protection: agents read pages and raw sources; main context only orchestrates.

**Verifier agent prompt:**

```
You are verifying concept pages in a knowledge base wiki against their raw sources.

Subject: [from KB.md]
KB root: [path]
(PATH RESOLUTION: Every relative KB path below — `wiki/...`, `raw/...`, `.atlas/...` — is anchored to KB root above. For every Read/Glob/Bash tool call, prefix the path with `[KB root]/`. Do not use bare relative paths against your CWD; your CWD may not equal KB root.)

Pages to verify:
[list of wiki/concepts/<slug>.md paths]

For EACH page:
1. Read the page in full, including frontmatter and the ## Sources section.
2. Select 4-6 specific factual claims that cite a source. Prefer checkable
   specifics: numbers, named tools, quoted positions, dated events. Skip
   claims too vague to falsify.
3. For each claim, identify the cited raw source and check exemption FIRST:
   if a summary page exists at wiki/summaries/[source-slug].md whose
   frontmatter contains `exempt_from_raw_hash: true`, mark the claim EXEMPT
   and do not check it (the source is expected to drift).
4. Otherwise Read the cited raw file and locate the supporting passage.
5. Verdict per claim (same taxonomy as lint's Source Verifier — do not
   invent a new one):
   - SUPPORTED: the source says what the page claims (quote the passage)
   - PARTIALLY SUPPORTED: directionally right, detail drifted (quote both)
   - UNSUPPORTED: the source does not say this (quote what it says instead)
   - SOURCE MISSING: the cited raw file does not exist
   - EXEMPT: per step 3
6. Page verdict:
   - REVIEWED-OK: every checked claim SUPPORTED or EXEMPT
   - MINOR-FIX: at least one PARTIALLY SUPPORTED, none UNSUPPORTED or
     SOURCE MISSING. Include a proposed one-line correction per drifted
     claim, with the exact current page text and exact replacement text.
   - NEEDS-REWRITE: any UNSUPPORTED or SOURCE MISSING claim

RULES:
- READ-ONLY. Never edit any file. The orchestrator applies all changes.
- Every verdict needs a verbatim quote. No quote, no verdict — mark the
  claim "could not check" with the reason, and substitute another checkable
  claim from the page so the checked count stays at 4-6. If the page has
  fewer than 3 checkable claims in total, its verdict cannot be REVIEWED-OK:
  leave it review_pending and report why.
- No fabrication. If a raw file is unreadable, that is SOURCE MISSING with
  the read error noted, not a guess.
- Report format per page: slug, claims checked (claim text + verdict +
  quote + source path), page verdict, proposed corrections (MINOR-FIX only).
```

If the Grep or Glob tools are unavailable, agents fall back per SKILL.md Rule 17.

## Phase 3: Apply Results (orchestrator, main context)

Process each page by verdict:

**REVIEWED-OK:** flip frontmatter to `status: reviewed` and set `revision_note: "verified YYYY-MM-DD — [N] claims spot-checked against raw sources, no drift"`. `revision_note` is hereby a defined optional concept-page frontmatter field (this spec is its producer; today the field is informational for humans and for verify's own re-runs — lint reads `revision_note` on reports only). Frontmatter edits must use line-list manipulation (parse the frontmatter block into lines, replace the status line, insert or replace the revision_note line), NOT regex replacement at line boundaries — a status line at the end of the frontmatter block has no trailing newline in a regex capture and the insert silently no-ops (this exact bug occurred 2026-05-01).

**MINOR-FIX:** present every proposed correction to the user in one batch (current text vs replacement, with the source quote). Apply only what the user approves; approved pages get the body edit, `last_updated` set to today (body changes always bump it, per compile's concept schema), and the REVIEWED-OK flip with `revision_note: "verified YYYY-MM-DD — [N] claims checked, [M] minor corrections applied"`. Declined pages stay `review_pending` untouched. (The REVIEWED-OK flip alone does NOT bump `last_updated`: frontmatter-only status changes follow lint Part B's precedent.)

**NEEDS-REWRITE:** never auto-fix. Leave `review_pending`. Report the failing claims with quotes and suggest the remedy per case: re-ingest the source if it changed upstream, or `/atlas compile --incremental` after fixing the raw file, or manual review.

## Phase 4: Update Metadata and Git Commit

1. No KB.md counters change (verify creates no pages). Do not touch `last_compiled` or `compiles_since_lint`.
2. Per SKILL.md Rule 15: if any page changed, stage and commit: `git -C [KB root] add wiki/concepts/` then `git -C [KB root] commit -m "atlas: verify - [N] pages reviewed, [M] minor fixes applied, [K] left flagged"`. Commit failure is a warning, not a verify failure.

## Phase 5: Output

```
## Verify Complete

Scope: [pending | slug] — [N] pages checked, [C] claims spot-checked

Cleared to reviewed: [N] pages
Minor fixes applied: [M] (user-approved)
Left flagged (needs rewrite): [K]
  - [slug]: [one-line reason with the failing claim]

[If K > 0: "Flagged pages keep status: review_pending. See the failing claims above."]
[If git commit succeeded: "Changes committed: [hash]"]
```
