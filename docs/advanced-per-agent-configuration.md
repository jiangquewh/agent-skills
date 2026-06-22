# Advanced per-agent configuration

Skills in this repository are intentionally portable. Every `SKILL.md` keeps its frontmatter limited to the fields the Agent Skills specification defines, so the same file works across Claude Code, Cursor, Gemini CLI, Antigravity, and any other spec-conformant client.

Several agents also support extra runtime controls: model routing, tool restrictions, turn limits, execution isolation. These controls are valuable, but they are vendor-specific and do not belong in the shared, portable frontmatter. This document explains which controls each agent supports, where they belong, and how to apply them as an opt-in without breaking portability.

## Principles

1. **Keep `SKILL.md` portable.** The Agent Skills specification reserves top-level frontmatter for `name`, `description`, `license`, `compatibility`, `metadata`, and the experimental `allowed-tools`. Vendor or client-specific properties belong under `metadata`, not at the top level.
2. **Unknown top-level fields are not guaranteed to be ignored.** Many parsers only read `name` and `description` and silently drop the rest, but a strict spec-conformant validator or client may flag or reject unrecognized top-level keys. Putting vendor fields under `metadata` keeps the file conformant everywhere.
3. **Opt-in, not global.** Runtime controls should be something a user adds for their own agent and workflow, not a default baked into every skill in the repository.
4. **Prefer generation over hand-editing.** Where a configuration is repetitive and mechanical, generate it from a single source of truth instead of maintaining copies by hand. See [Automating with scripts](#automating-with-scripts).

## Where vendor fields belong

| Field type | Example | Correct home |
|---|---|---|
| Spec fields | `name`, `description`, `license`, `compatibility` | Top-level frontmatter in `SKILL.md` |
| Vendor metadata | model hints, routing tags | Under the `metadata` key in `SKILL.md` |
| Runtime orchestration | subagent definitions, turn limits, tool allowlists | A separate per-agent adapter file (see each agent below) |

## Claude Code

Claude Code reads `name` and `description` for discovery. Slash commands under `.claude/commands/*.md` support an extra `context` field.

- **`context: fork`** runs a command or skill in an isolated subagent context instead of the active conversation. This gives context isolation, repeatable execution from a clean state, execution tokens spent in the fork rather than the main session, and a cleaner handoff where only the result surfaces back.
- **`allowed-tools`** restricts the tool set available during a command, supporting least-privilege for passive stages such as review or audit.

These are Claude Code conventions. Apply them in your own `.claude/commands/` copies rather than in the shared `SKILL.md` files.

## Gemini CLI and Antigravity

Gemini CLI and Antigravity discover skills the same way: they match `name` and `description`, then load the full skill on demand. Runtime orchestration lives in a separate resource, not in the skill frontmatter.

- **Subagent definitions** live in `.gemini/agents/*.md`, which are distinct from skills. Their schema is `kind` (`local` or `remote`), `model`, `temperature`, `max_turns`, and `tools`. Note that this is a subagent schema, not a skill schema: the two should not be merged into one `SKILL.md`.
- **Model routing** can send lighter tasks (formatting, documentation, git operations) to a faster model and reserve a stronger model for cognitively demanding work (debugging, security auditing, interface design).
- **Tool restrictions** enforce least-privilege per subagent, which also trims unused tool schemas from the prompt.
- **Turn limits** (`max_turns`) cap execution to avoid runaway correction loops.

### Model guidance

Google recommends omitting `temperature` for Gemini 3.x models and using `thinking_level` instead. Do not hardcode `temperature` on skills or subagents routed to 3.x models; prefer `thinking_level` and revisit this guidance as the models evolve.

## Automating with scripts

Maintaining per-agent metadata by hand across every skill is repetitive, drifts out of sync, and is easy to get wrong. A better approach is to keep a single source of truth and generate the agent-specific output:

- A generator reads a per-skill configuration map (which model, tool allowlist, turn limit, and isolation each skill should use for a given agent).
- For metadata-style fields, it injects them under the `metadata` key of each `SKILL.md`, keeping the file spec-conformant.
- For orchestration-style fields, it emits the separate adapter files an agent expects, for example `.gemini/agents/*.md` subagent definitions, leaving `SKILL.md` untouched.

Generation also lets the tooling validate **semantics, not just shape**: confirm that a model name is a known model, that a tool name is a real tool, that `max_turns` is a positive integer, and that `temperature` (where still applicable) is a valid number in range. Shape-only checks let invalid values such as `parseInt("1.5")` or `parseFloat("0.2oops")` slip through.

## Status

This document is a draft consolidating the discussion from [#272](https://github.com/addyosmani/agent-skills/pull/272) and [#36](https://github.com/addyosmani/agent-skills/pull/36). The goal is a single reference for per-agent configuration that keeps the shared skills portable while documenting the advanced controls each agent offers as an opt-in. The script-based automation described above is a proposal pending agreement before any implementation.
