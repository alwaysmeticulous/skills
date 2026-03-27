# Meticulous Agent Skills

A collection of agent skills for [Meticulous](https://meticulous.ai) — the automated visual regression testing platform for web frontends.

These skills are designed to be used with AI coding assistants (Cursor, Claude Code, etc.) to help investigate and debug issues in Meticulous test runs and replays.

> **Beta:** The agentic skills for Meticulous are in early beta. Expect some instability and frequent breaking changes. We recommend updating at least once a week (see [Updating](#updating) below).

## Setup

### 1. Install the Meticulous CLI

The [Meticulous CLI](https://github.com/alwaysmeticulous/meticulous-sdk) is required to run simulations and test runs from the command line:

```bash
npm install --global @alwaysmeticulous/cli
```

You can also install it locally per-project instead of globally.

### 2. Set your API token

The CLI requires a Meticulous API token. See [the docs](https://app.meticulous.ai/docs/github-actions-v2) for how to create one, then add it to your environment:

```bash
export METICULOUS_API_TOKEN=xyz...
```

### 3. Install skills

Install skills into your project using [`npx skills`](https://github.com/vercel-labs/skills):

```bash
# Install all skills
npx skills add alwaysmeticulous/skills --all --full-depth

# Preview available skills without installing
npx skills add alwaysmeticulous/skills --list

# Install a specific skill
npx skills add alwaysmeticulous/skills --skill test-with-meticulous

# Install globally (available across all projects)
npx skills add alwaysmeticulous/skills -g
```

`npx skills` automatically detects your coding agent (Cursor, Claude Code, Codex, and [37+ more](https://github.com/vercel-labs/skills#available-agents)) and installs to the right location.

### Updating

Both the CLI and the skills are under active development:

```bash
npm update --global @alwaysmeticulous/cli && npx skills update
```

## Skills

Skills are Markdown files that provide AI coding assistants with domain-specific knowledge and step-by-step workflows. When an agent reads a skill, it gains targeted expertise for a specific task — without needing that context baked into every conversation.

Skills follow the [`SKILL.md` format](https://docs.cursor.com/context/rules-for-ai) used by Cursor and Claude Code.

| Skill | Description |
|-------|-------------|
| [`iterative-dev`](iterative-dev/SKILL.md) | Iterative frontend development loop using Meticulous for per-step visual validation. Use when implementing a multi-step frontend change and want to catch visual regressions at each step. |
| [`simulate-and-diff`](simulate-and-diff/SKILL.md) | Run a Meticulous session simulation against a live URL and analyze the visual output — either by inspecting screenshots directly or by comparing pixel and HTML diffs against a base replay. |
| [`test-with-meticulous`](test-with-meticulous/SKILL.md) | Run after implementing any frontend change to verify its visual impact. Triggers a Meticulous test run, then inspects screenshot diffs to classify each visual change as intended or unintended. |
| | |
| [`meticulous-cli`](meticulous-cli/SKILL.md) | Reference documentation for the Meticulous CLI tool, its commands, and global options. |

## License

[ISC](LICENSE) © Meticulous, Inc.
