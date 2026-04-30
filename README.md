# skills

A personal collection of engineering skills for AI coding agents, focused on
durable workflows over stack-specific facts. Skills cover practices like
spec-driven development, code review, and debugging; project-specific commands
and conventions live in each project's `CLAUDE.md`.

## Attribution & License

This repository contains a mix of skills adapted from [addyosmani/agent-skills]
(used under the [MIT License][agent-skills license]) and skills written from scratch. Each skill's
frontmatter records its origin and whether it has been modified:

```yaml
---
name: spec-driven-development
description: ...
x-provenance:
  origin: addyosmani/agent-skills
  upstream_path: skills/spec-driven-development/SKILL.md
  upstream_commit: abc123
  status: modified
  last_synced: 2026-04-30
---
```

`origin` always points to the repo where the content first appeared. `status`
is one of `verbatim`, `modified`, or `authored`. For authored skills, only
`origin` and `status` are needed; the upstream-related fields don't apply.

This repository is licensed under the MIT License. See [LICENSE](./LICENSE) for
the full text.

[addyosmani/agent-skills]: https://github.com/addyosmani/agent-skills
[agent-skills license]: https://github.com/addyosmani/agent-skills/blob/main/LICENSE
