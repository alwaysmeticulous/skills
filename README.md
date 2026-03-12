# Meticulous Agent Skills

A collection of agent skills for [Meticulous](https://meticulous.ai) — the automated visual regression testing platform for web frontends.

These skills are designed to be used with AI coding assistants (Cursor, Claude Code, etc.) to help investigate and debug issues in Meticulous test runs and replays.

## What Are Skills?

Skills are Markdown files that provide AI coding assistants with domain-specific knowledge and step-by-step workflows. When an agent reads a skill, it gains targeted expertise for a specific task — without needing that context baked into every conversation.

Skills follow the [`SKILL.md` format](https://docs.cursor.com/context/rules-for-ai) used by Cursor and Claude Code.

## Installation

Install skills into your project using [`npx skills`](https://github.com/vercel-labs/skills):

```bash
# Install all skills
npx skills add alwaysmeticulous/skills

# Preview available skills without installing
npx skills add alwaysmeticulous/skills --list

# Install a specific skill
npx skills add alwaysmeticulous/skills --skill debugging-flakes

# Install globally (available across all projects)
npx skills add alwaysmeticulous/skills -g
```

`npx skills` automatically detects your coding agent (Cursor, Claude Code, Codex, and [37+ more](https://github.com/vercel-labs/skills#available-agents)) and installs to the right location.

## Usage

### In a `met_debug` workspace

These skills are automatically placed in `.claude/skills/` inside every workspace created by `met_debug`. Invoke them by asking the AI assistant:

> "Use the debugging-flakes skill to investigate this replay."

### Manually

Copy the relevant `SKILL.md` into your project's `.claude/skills/<name>/SKILL.md` (or the equivalent path for your agent).

## Skill Categories

| Directory | Purpose |
|-----------|---------|
| `meticulous-cli/` | Reference documentation for each CLI command and its options |
| `using-meticulous/` | Higher-level workflow skills for common agent tasks |

## Contributing

To add or improve a skill:

1. Create a new folder under the appropriate category (e.g. `met-debug/<skill-name>/`)
2. Add a `SKILL.md` with a YAML front-matter block (`name`, `description`) and a Markdown body
3. Open a pull request

### Skill format

```markdown
---
name: skill-name
description: One-line description of when to use this skill.
---

# Skill Title

...content...
```

## License

[ISC](LICENSE) © Meticulous, Inc.
