# Agent Frontmatter Conventions

All 7 phase agents under `agents/*.md` (at the repo root) share the same
YAML frontmatter shape. Not all fields are equal — some are enforced by Claude
Code / the Agent SDK today, others are ai-tools conventions that are
**advisory for now** and may be wired up later.

This file is the single source of truth so we don't have to paste annotations
into every agent.

## Fields

| Field             | Enforced by            | Purpose                                                 | Notes |
|-------------------|------------------------|---------------------------------------------------------|-------|
| `name`            | Claude Code            | Unique agent id; used with the Task tool                | Must match the filename (without `.md`). |
| `description`     | Claude Code            | Routing/selection hint for the router + proactive use   | Keep under ~400 characters. |
| `model`           | Claude Code / SDK      | Which Claude model to run the agent on                  | Values: `opus`, `sonnet`, `haiku`. |
| `tools`           | Claude Code / SDK      | Allowlist of tools the agent may call                   | Everything else is denied. |
| `disallowedTools` | Claude Code / SDK      | Explicit denylist (applied on top of `tools`)           | Used to scope out `Edit` for read-only agents. |
| `color`           | Claude Code UI         | Badge color shown in the UI                             | Purely cosmetic. |
| `effort`          | ai-tools (advisory)    | Convention — hints at "try hard"                        | Not read by the SDK today. Kept for future tuning and documentation. |
| `maxTurns`        | Agent SDK (where supported) | Hard cap on back-and-forth turns before the agent yields | Honored by the SDK programmatic runner; ignored by older CLI invocations. |
| `memory`          | ai-tools (advisory)    | Convention — scope hint ("project", "session", "none")  | Not read by the SDK today. Used by docs + future memory layer. |

## Rule of thumb

- If you edit an **authoritative** field (`name`, `description`, `model`,
  `tools`, `disallowedTools`), re-test the agent — Claude Code may route
  differently.
- If you edit an **advisory** field, update this table if the semantics change
  but expect no runtime behavior change today.

## Why not drop the advisory fields?

Two reasons:

1. They make the frontmatter self-documenting — a reader can see the intended
   "personality" of each agent without reading the whole SKILL.md.
2. They are forward-compatible with planned features (effort-aware routing,
   per-agent memory scoping) so we don't have to re-introduce them later.

If you are writing a brand-new agent and do not want the advisory fields, just
omit them — nothing will break.
