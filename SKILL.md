---
name: plan-to-execution-plan
description: Convert any large or detailed plan markdown file (architecture spec, feature implementation, refactor plan, migration, codebase update — typically anything 200+ lines that an AI code editor like Claude Code, Codex, or Cursor wrote) into a lean, executable companion plan that an agent can follow chunk-by-chunk without loading the full original into context. Use when the user has a plan and asks to make an "implementation plan", "execution plan", "build plan", "task plan", "executable plan", or "breakdown" from it, or says their plan is "too long for the agent" or wants to "phase / chunk / break down" the work.
---

# Plan to Execution Plan

Take any detailed markdown plan (an AI-generated architecture spec, feature plan, refactor plan, migration plan, etc.) and produce a lean **execution plan** that an agent can follow chunk-by-chunk, loading only the specific line ranges it needs from the original. Preserves every detail without duplicating it.

This skill is **rigid by design.** Every execution plan it produces has the exact same shape regardless of input. The only thing that varies is which chunks exist and what they reference.

---

## Core principles (never violate)

1. **Reference, don't duplicate.** The execution plan points back to the original by relative path + line range. Schemas, flows, rules, acceptance criteria stay in the original.
2. **Append refinements to the BOTTOM of the original, never the top.** Inserting at the top shifts every subsequent line number and silently breaks every reference in the execution plan.
3. **Verify line numbers after editing the original.** Re-grep header line numbers; confirm they match every reference.
4. **Reuse chunk numbering if the original has one.** If the original has Chunk 7, your execution plan also has Chunk 7.
5. **Same chunk shape every time.** Seven mandatory fields, identical order, no exceptions.
6. **The execution plan is a navigation layer, not a second source of truth.**

---

## Canonical outputs (what this skill always produces)

| Output | Location | Always produced? |
| --- | --- | --- |
| Execution plan | `{ORIGINAL_BASENAME}_IMPLEMENTATION_PLAN.md` next to the original | Always |
| Quick Section Index appended to bottom of original | end of original file | Only if original ≥ 500 lines |

Naming is fixed: take the original's basename without extension, append `_IMPLEMENTATION_PLAN.md`. Example: `my_feature_plan.md` → `my_feature_plan_IMPLEMENTATION_PLAN.md`. If the user has already established a different convention in the same directory (e.g. uppercase `MY_FEATURE_IMPLEMENTATION_PLAN.md`), follow theirs.

---

## Workflow (run in this exact order)

### Step 1 — Probe the original

Run these greps. Do NOT read the file body.

```bash
wc -l path/to/plan.md
rg -n "^# "   path/to/plan.md   # H1 (usually the title)
rg -n "^## "  path/to/plan.md   # major sections
rg -n "^### " path/to/plan.md   # subsections
rg -n -i "^#+ .*(build sequence|implement|chunk|step|task|phase)" path/to/plan.md
rg -n -i "^#+ .*(verif|accept|test|criteria|done when)" path/to/plan.md
rg -n -i "^#+ .*(schema|model|state|api|endpoint)" path/to/plan.md
```

Record three things:

- Total line count.
- Whether a **work-breakdown section** exists (Build Sequence / Implementation Steps / Chunks / Tasks / Phases).
- Whether a **verification section** exists.

### Step 2 — Classify by size

| Class | Lines | Chunks | Length target for execution plan |
| --- | --- | --- | --- |
| **Small** | < 300 | 2–5 | ~1/2 of original |
| **Medium** | 300–1,000 | 5–10 | ~1/3 of original |
| **Large** | 1,000–3,000 | 10–16 | ~1/5 of original |
| **Very large** | > 3,000 | 12–20 | ~1/6 of original |

Quick Section Index is appended to the original **if and only if** the original is ≥ 500 lines. No other thresholds matter.

### Step 3 — Targeted reads (only these)

Use `Read(path, offset, limit)` on **exactly** these slices, in this order. Skip a slice if it doesn't exist.

1. First 100 lines of the original (context / intent).
2. The work-breakdown section (Build Sequence / Implementation Steps / Chunks / Tasks).
3. The verification / acceptance / tests section.
4. Any section the user explicitly named.

**Do NOT read the whole plan.** That defeats the skill.

### Step 4 — Define chunks (single deterministic rule)

Apply this rule in order; stop at the first match:

1. **If the original has a numbered work-breakdown** (Build Sequence / Chunks / Tasks / Steps), reuse those numbers and titles verbatim. Map each to its line range from the Step 1 grep.
2. **Else, decompose by layer + file-set**:
   - Group changes into ordered layers: `data/schema → backend logic → API/routes → frontend → tests/docs`.
   - Within a layer, split by file-set so each chunk touches 1–3 related files.
   - Number sequentially starting at 1.
   - Target chunk size: one PR-sized review (~1–3 files, ~1–3 hours of work).

For every chunk, record:

- Title (≤ 70 chars).
- Goal (one sentence).
- Dependencies (chunk numbers, or `None`).
- "Read first" line ranges in the original.
- Files touched (`new` or `modify`).

### Step 5 — Group chunks into phases (always)

Phases always exist; small plans just get one phase. This keeps the shape uniform.

| Chunk count | Phases |
| --- | --- |
| 2–5 | 1 phase named `Implementation` |
| 6–10 | 2–4 phases by theme |
| 11+ | 3–6 phases by theme |

Phase seams should match natural review boundaries (e.g. `Foundation → Core logic → Edges → Polish`).

### Step 6 — Write the verification block for each chunk (single canonical shape)

Every chunk's **Verify** block has the same four lines, in this order. Omit a line only when it has no content; never reorder.

```
**Verify:**
- Run: `[exact command]` — expect `[expected result]`.
- Manual: [behaviour to confirm by hand].
- Regression: `[command to run existing test suite]` still passes.
- Original spec: [section name](./PLAN.md#anchor) (Lxx-Lyy).   ← only if the original has a verification section
```

If the original has a verification section, the last line links to its specific subsection by line range. If not, the first three lines stand alone.

### Step 7 — Write the execution plan

Save as `{ORIGINAL_BASENAME}_IMPLEMENTATION_PLAN.md` next to the original. Use the [Canonical execution plan template](#canonical-execution-plan-template) below — every section is mandatory; fill placeholders with explicit `None` when empty.

### Step 8 — Append the Quick Section Index (if ≥ 500 lines)

Append the [Quick Section Index template](#quick-section-index-template) to the **bottom** of the original. Never the top.

### Step 9 — Verify line ranges

After every edit to the original, re-run:

```bash
rg -n "^## "  path/to/plan.md
rg -n "^### " path/to/plan.md
```

Confirm every header still sits at the line your execution plan references. If anything shifted, you violated principle #2 — undo and re-do.

### Step 10 — Lint check both files

Run your agent tool's markdown linter on the execution plan and the modified original (e.g. `ReadLints` in Cursor, `markdownlint` on the CLI, or whatever your environment provides). Fix anything that surfaced.

---

## Canonical execution plan template

Every execution plan has these sections in this exact order. Every chunk has these seven fields in this exact order. Fill empty slots with `None` rather than omitting them.

````markdown
---
name: "[Project / Feature Name] — Execution"
overview: "Lean, executable companion to [ORIGINAL_FILENAME]. [N] chunks across [M] phase(s); each chunk references precise line ranges in the original."
---

[The frontmatter above is optional. Include only the keys your agent tool reads; some tools (e.g. Cursor) accept extra keys like `todos: []` and `isProject: false`. Plain markdown without frontmatter works in any tool.]

# [Project / Feature Name] — Implementation Execution Plan

This is the **executable companion** to [ORIGINAL_FILENAME](./ORIGINAL_FILENAME) ([N] lines).

> Do **not** read the original plan end-to-end. Each chunk below tells you exactly which line range to load. Use `Read(path, offset, limit)` to load only the referenced section.

[Include the next line only if a Quick Section Index was appended:]
A Quick Section Index lives at the bottom of `ORIGINAL_FILENAME` for section-name → line-range lookups.

---

## How to use this plan

1. Read this file end-to-end **once**.
2. Read the [Cross-cutting rules](#cross-cutting-rules-read-once) below **once** before starting Chunk 1.
3. For each chunk, in [Build order](#build-order):
   - Load only the line ranges listed in **Read first** — do not load the full original.
   - Implement the **Files** in the order listed.
   - Stop at the **Done when** checkpoint and surface the diff for review.
   - Run the **Verify** block.
4. Treat the dependency graph in [Build order](#build-order) as binding.

All file paths in this plan are relative to the repo root.

---

## Cross-cutting rules (read once)

These apply to every chunk. Read them before Chunk 1, then re-read only on demand.

| Topic | Lines in `ORIGINAL_FILENAME` | Why it matters |
| --- | --- | --- |
| [Rule 1] | [Lxx-Lyy](./ORIGINAL_FILENAME) | [why] |
| [Rule 2] | [Lxx-Lyy](./ORIGINAL_FILENAME) | [why] |

**Repository conventions** (from the repo's `AGENTS.md` / `CLAUDE.md` / `CONTRIBUTING.md` if present):
- [Convention 1]
- [Convention 2]

[If the original has no cross-cutting rules, use a single row with `None` and `—` for lines. If the repo has no conventions file, omit the Repository conventions block.]

---

## Phase map

| Phase | Chunks | Theme |
| --- | --- | --- |
| **1. [Name]** | 1, 2, 3 | [theme] |
| **2. [Name]** | 4, 5 | [theme] |

[For small plans the table has exactly one row: `1. Implementation` covering all chunks.]

---

## Build order

```
Chunk 1: [Title]   (depends on: None — start here)
Chunk 2: [Title]   (depends on: 1)
Chunk 3: [Title]   (depends on: 1, 2)
...
```

[If the original has its own dependency graph, mirror it verbatim.]

---

# PHASE 1 — [Name]

## Chunk 1 — [Title]

**Goal:** [One sentence. What this chunk delivers.]

**Depends on:** [Chunk numbers, or `None — independent`.]

**Read first:**
- [Section name in original](./ORIGINAL_FILENAME#anchor) (Lxxxx-Lyyyy)
- [Schema / rule / context needed](./ORIGINAL_FILENAME) (Lxxxx-Lyyyy)
- [Existing reference file in repo, if any]

**Files:**
- `path/to/file.ext` (new) — [one-line purpose]
- `path/to/other.ext` (modify) — [one-line purpose]

**Steps:**
1. [Concrete step]
2. [Concrete step]
3. [Concrete step]

**Done when:** [One-line summary. If original has a matching "Done when" / "acceptance" subsection, link it: see (Lxxxx-Lyyyy).]

**Verify:**
- Run: `[exact command]` — expect `[expected result]`.
- Manual: [behaviour to confirm].
- Regression: `[command]` still passes.
- Original spec: [verification subsection](./ORIGINAL_FILENAME#verification) (Lxxxx-Lyyyy).

---

[Repeat for every chunk. Insert `# PHASE N — [Name]` headers between phase boundaries.]

---

## Final acceptance

The work is complete when:
- Every chunk's **Verify** block has passed.
- [If original has a verification section] Every subsection in [Verification](./ORIGINAL_FILENAME#verification) (Lxxxx-Lyyyy) is checked off.
- End-to-end smoke test: [one paragraph in plain English describing the happy path through the finished feature].
````

---

## Quick Section Index template

Append to the bottom of the original (after all existing content) **only when the original is ≥ 500 lines**.

```markdown

---

## Quick Section Index

For the lean, executable companion plan that consumes this spec section-by-section, see [ORIGINAL_BASENAME_IMPLEMENTATION_PLAN.md](./ORIGINAL_BASENAME_IMPLEMENTATION_PLAN.md).

Use the line ranges below with `Read(path, offset, limit)` to load only the section you need — do **not** load this [N]-line file in full.

| Section | Lines |
| --- | --- |
| [H2 section name] | [start]-[end] |
| &nbsp;&nbsp;[H3 subsection name] | [start]-[end] |
```

Indent H3 subsections under their parent H2 in the table using `&nbsp;&nbsp;`.

---

## Quality checks (run before declaring done)

- [ ] Execution plan filename is exactly `{ORIGINAL_BASENAME}_IMPLEMENTATION_PLAN.md` and sits in the same directory as the original.
- [ ] Execution plan length is within the target ratio for its size class (Step 2 table).
- [ ] Frontmatter (if your agent tool uses one) is present and well-formed.
- [ ] All sections present in order: title → How to use → Cross-cutting rules → Phase map → Build order → Phases/Chunks → Final acceptance.
- [ ] Every chunk has all seven fields in order: Goal / Depends on / Read first / Files / Steps / Done when / Verify.
- [ ] Every "Read first" link includes a line range (`Lxxxx-Lyyyy`).
- [ ] Every file in "Files" is annotated `(new)` or `(modify)`.
- [ ] Every "Verify" block uses the four-line canonical shape (Run / Manual / Regression / Original spec — omit only when empty).
- [ ] At least one phase exists (small plans get a single `Implementation` phase).
- [ ] Build order block lists every chunk with its dependencies.
- [ ] No content from the original is duplicated — only referenced by line range.
- [ ] If original ≥ 500 lines, Quick Section Index is at the **bottom** of the original.
- [ ] Header line numbers in the original (`rg -n "^## "`) match every reference.
- [ ] Markdown linter (e.g. `ReadLints` in Cursor, `markdownlint`, etc.) shows no errors on either file.

---

## Anti-patterns (do not do these)

- **Loading the full original.** Always use `Read(offset, limit)`.
- **Re-explaining what's already in the original.** Reference, never restate.
- **Inserting an index at the top of the original.** Shifts every line number. Always append.
- **Skipping verification of line numbers** after editing the original.
- **Renumbering existing chunks.** If the original has Chunk 7, the execution plan has Chunk 7.
- **Omitting a chunk field** because it feels empty. Write `None` instead.
- **Omitting the Phase map** on a small plan. Always include it — just with one row.
- **Vague "Read first" entries** like "see the data section". Always include a line range.
- **Long prose in the execution plan.** Each chunk should fit on one screen.
- **Adding a Quick Section Index** to a plan under 500 lines.
- **Inventing a verification step** when the original already has one. Link it.
- **Treating the execution plan as a second source of truth.** It is purely a navigation layer.

---

## Worked example A — large architecture spec (pre-chunked)

A ~3,500-line architecture spec that already has a `Build Sequence` section listing 16 chunks and a `Verification` section listing acceptance criteria, becomes a ~660-line execution plan with 16 chunks across 5 phases.

What the skill did:
- **Step 1 — Probe:** `wc -l` reported ~3,500 lines; greps found both a `Build Sequence` and a `Verification` section.
- **Step 2 — Classify:** Very large → target ~1/6 length, 12–20 chunks.
- **Step 3 — Targeted reads:** first 100 lines, Build Sequence, Verification.
- **Step 4 — Define chunks:** reused the existing numbered Build Sequence (Chunks 1–16) verbatim, mapped to their line ranges.
- **Step 5 — Phases:** grouped the 16 chunks into 5 themed phases (`Foundation → Core domain → Integrations → Learning loop → External connectivity`).
- **Step 6 — Verify blocks:** each one's `Original spec` line links to its specific subsection of the original Verification block.
- **Step 7 — Wrote** the execution plan using the canonical template.
- **Step 8 — Appended** a Quick Section Index to the bottom of the original (file is ≥ 500 lines).
- **Steps 9-10 — Re-grepped** line ranges and ran the markdown linter.

## Worked example B — small feature plan (freeform, no chunks, no verification section)

A ~250-line feature plan (e.g. a UI toggle, a config flag, a single new endpoint) with no pre-existing chunking and no verification section.

- Size class: **Small** → target ~125 lines, 2–5 chunks.
- No verification section in the original → every chunk's `Verify` block uses only the first three canonical lines (Run / Manual / Regression).
- Step 4 falls through to the layer-and-file-set rule → 4 chunks: `data/storage` → `core logic` → `UI surface` → `existing-callsite updates`.
- Step 5 → one phase named `Implementation` covering all 4 chunks.
- < 500 lines → **no** Quick Section Index appended.
- Result: an ~80-line execution plan with the exact same shape as the large example above.
