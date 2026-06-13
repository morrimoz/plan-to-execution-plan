# Plan to Execution Plan

A [Cursor Agent Skill](https://cursor.com/docs/context/skills) that turns large markdown plans into lean, chunk-by-chunk execution plans an AI agent can follow without loading the full original into context.

Works with architecture specs, feature plans, refactors, migrations — anything long enough that an agent starts losing the thread.

## What it does

Given a detailed plan markdown file, the skill produces:

1. **`{ORIGINAL}_IMPLEMENTATION_PLAN.md`** — a navigation layer with phased chunks, file lists, steps, and verification blocks
2. **Quick Section Index** (plans ≥ 500 lines) — appended to the bottom of the original for line-range lookups

The execution plan **references** the original by path + line range. It does not duplicate schemas, flows, or acceptance criteria.

## Install (Cursor)

**Option A — personal skill (works in any project)**

```bash
mkdir -p ~/.cursor/skills/plan-to-execution-plan
curl -fsSL https://raw.githubusercontent.com/morrimoz/plan-to-execution-plan/main/SKILL.md \
  -o ~/.cursor/skills/plan-to-execution-plan/SKILL.md
```

**Option B — project skill (shared with your team via git)**

```bash
mkdir -p .cursor/skills/plan-to-execution-plan
curl -fsSL https://raw.githubusercontent.com/morrimoz/plan-to-execution-plan/main/SKILL.md \
  -o .cursor/skills/plan-to-execution-plan/SKILL.md
```

Restart Cursor or start a new agent session after installing.

## Use

In Cursor Agent chat:

```
/plan-to-execution-plan
```

Then point at your plan file, e.g.:

```
Turn docs/my_feature_plan.md into an execution plan.
```

Or attach the plan and ask for an implementation / execution / build breakdown.

## Claude Code

Copy `SKILL.md` into your project's `.claude/skills/plan-to-execution-plan/` directory (same content, same workflow).

## License

MIT — see [LICENSE](./LICENSE).
