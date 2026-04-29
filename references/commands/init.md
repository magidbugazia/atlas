# Command: Init

This file is loaded on demand by `~/.claude/skills/atlas/SKILL.md` when the user invokes `/atlas init`. It contains the complete operational logic for the init command.

**Prerequisite:** Phase 0 (auto-detect knowledge base) is skipped for init since this command creates a new KB. SKILL.md routes `init` here without requiring an existing KB.

**Invocation:** `/atlas init "AI Engineering"` or `/atlas init "Fuel Operations"`

## Phase 1: Create Structure

1. Parse the subject name from arguments
2. Check if `KB.md` already exists in the current directory. If yes: warn the user and ask if they want to reinitialize (this would NOT delete existing files, just recreate missing directories and update KB.md)
3. Create the directory structure:

```
./
├── KB.md
├── .atlas/
│   ├── hashes.json
│   ├── concepts.json
│   ├── splits/
│   └── compile-runs/             # Per-compile manifests (created on first compile, used by lint Agent 6 for stale-by-content report detection)
├── raw/
│   ├── articles/
│   ├── papers/
│   ├── repos/
│   ├── datasets/
│   ├── images/
│   └── notes/
├── wiki/
│   ├── INDEX.md
│   ├── indexes/
│   ├── concepts/
│   ├── summaries/
│   ├── reports/
│   ├── lint/
│   ├── slides/
│   └── images/
```

4. Use AskUserQuestion: "Briefly describe the scope of this knowledge base. What topics should it cover? What's out of scope? (This guides how atlas organizes concepts during compilation.)"

5. Write `KB.md`:

```markdown
---
subject: [user-provided subject]
scope: [user's scope description from step 4]
created: [today's date]
last_compiled: never
last_linted: never
raw_count: 0
wiki_count: 0
compile_count: 0
compiles_since_lint: 0
---

# [Subject] Knowledge Base

Managed by Atlas. Raw sources in `raw/`, compiled wiki in `wiki/`.

## Scope
[user's scope description: what's in, what's out]
```

6. Write `.atlas/hashes.json`:

```json
{}
```

7. Write `wiki/INDEX.md`:

```markdown
# [Subject] Knowledge Base

Last compiled: never
Total concepts: 0
Total sources: 0
Total categories: 0

## Categories

[none yet, run `/atlas compile` after adding raw material]

Each category has its own sub-index in `wiki/indexes/[category].md` with detailed concept summaries.

## Recent Reports

[none yet]
```

8. Write `.atlas/concepts.json`:

```json
{}
```

The concept registry is a JSON object where each key is a concept slug and each value has `name` (display name) and `aliases` (array of alternate names). Example after compilation:

```json
{
  "retrieval-augmented-generation": {
    "name": "Retrieval-Augmented Generation",
    "aliases": ["RAG", "augmented retrieval", "retrieval augmented generation"]
  },
  "prompt-engineering": {
    "name": "Prompt Engineering",
    "aliases": ["prompt design", "prompt optimization"]
  }
}
```

## Phase 2: Confirm

Output:
```
## Knowledge Base Initialized: [Subject]

Directory: [path]
Structure: .atlas/ (hashes, concepts, splits, compile-runs), raw/ (6 subdirectories), wiki/ (8 subdirectories)

Next steps:
1. Add raw material to `raw/` (articles, papers, notes, images)
2. Run `/atlas compile` to build the wiki
```
