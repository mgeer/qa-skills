# QA Skills for Claude Code

A set of four QA skills for [Claude Code](https://claude.ai/code) that provide structured, assertion-driven quality assurance workflows.

## Skills

| Skill | Purpose |
|-------|---------|
| [qa-analyze](qa-analyze/skill.md) | Analyze code against requirements → assertion checklist with coverage gaps |
| [qa-cover](qa-cover/skill.md) | Generate missing test code to fill coverage gaps |
| [qa-review](qa-review/skill.md) | Run tests, map results to assertions, produce review report |
| [qa-mutate](qa-mutate/skill.md) | Mutation testing to verify test suite effectiveness |

## Pipeline

```
qa-analyze ──▶ qa-cover ──▶ qa-review ──▶ qa-mutate
  (what's      (fill        (run &       (test the
  missing?)    gaps)        verify)      tests)
```

Each skill supports **scoped runs** (e.g., `qa-analyze billing`) and produces a single markdown report in the `_qa/` directory.

## Installation

Copy the skill directories into your Claude Code skills directory:

```bash
cp -r qa-analyze qa-cover qa-review qa-mutate ~/.claude/skills/
```

## Usage

In Claude Code, invoke skills with:

```
/qa-analyze              # full analysis
/qa-analyze billing      # scoped to billing module
/qa-cover P0             # generate tests for critical gaps only
/qa-review unit          # run and review unit tests only
/qa-mutate account       # mutation test account module only
```

## Design Principles

- **Scope support** — every skill can target a specific module/priority/level
- **Single output** — one markdown report per skill, no redundant structured files by default
- **Tool call budgets** — guardrails to prevent runaway execution
- **Incremental mode** — qa-analyze supports incremental updates when re-run
- **Grep-first strategy** — batch search before reading files, minimizing I/O

## License

MIT
