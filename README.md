# Plan to execution plan

**Category: AI development workflows | Format: Agent skill**

Turn a detailed architecture, feature or migration plan into a smaller execution companion. Each chunk identifies the files to work on, the relevant source sections and the checks needed before continuing.

The original plan remains the specification. The companion references it rather than duplicating schemas and acceptance criteria. This repository contains the agent workflow, not a standalone planning application.

## What it produces

| Input | Output | Purpose |
| --- | --- | --- |
| A detailed Markdown plan | A companion implementation plan | Break work into ordered, verifiable chunks |
| An original plan of at least 500 lines | A section index appended to the original | Find source sections without loading the whole document |

The workflow also handles plans that already contain useful chunking. See [SKILL.md](SKILL.md) for the decision rules and templates.

## Install

For Cursor:

```bash
mkdir -p ~/.cursor/skills
git clone https://github.com/morrimoz/plan-to-execution-plan.git ~/.cursor/skills/plan-to-execution-plan
```

For Claude Code, use `~/.claude/skills/plan-to-execution-plan`. For a shared project, use the corresponding project-local skills directory. Start a new agent session after installation.

## Example request

```text
Use plan-to-execution-plan on docs/integration-plan.md.
Keep the source plan authoritative and give each chunk a verification step.
```

The resulting companion gives the agent a bounded next task while keeping detailed requirements available by reference.

## Tradeoffs

Line references must be refreshed when the original plan changes. A clearer execution plan cannot repair missing requirements or prove that the proposed architecture is correct. Review the source plan first, then verify implementation against its acceptance criteria.

This repository includes workflow instructions and examples. It does not include an automated benchmark of planning quality.

## Related tools

- [Webflow to clean code](https://github.com/morrimoz/webflow-to-clean-code) - A migration workflow that can be split into execution chunks.
- [Directing motion films](https://github.com/morrimoz/directing-motion-films-with-claude-design) - A creative workflow with explicit stages and review gates.

[MIT license](LICENSE)
