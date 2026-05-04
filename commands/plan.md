---
description: Break work into small verifiable tasks with acceptance criteria and dependency ordering
x-provenance:
  origin: addyosmani/agent-skills
  upstream_path: .claude/commands/plan.md
  upstream_commit: 19e49a094d79540e635b107cb3490926ddeac7a3
  status: verbatim
  last_synced: 2026-05-01
---

Invoke the planning-and-task-breakdown skill.

Read the existing spec (SPEC.md or equivalent) and the relevant codebase sections. Then:

1. Enter plan mode — read only, no code changes
2. Identify the dependency graph between components
3. Slice work vertically (one complete path per task, not horizontal layers)
4. Write tasks with acceptance criteria and verification steps
5. Add checkpoints between phases
6. Present the plan for human review

Save the plan to tasks/plan.md and task list to tasks/todo.md.
