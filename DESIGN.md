# Atlas Design Notes

Engineering notes that don't belong in user-facing README. Rationale for decisions, deferred work, and future considerations.

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
