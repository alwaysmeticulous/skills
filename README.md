# Meticulous Agent Skills

A collection of agent skills for [Meticulous](https://meticulous.ai) — the automated visual regression testing platform for web frontends.

These skills enable AI coding assistants (Cursor, Claude Code, etc.) to use Meticulous: review test runs, investigate replays, debug diffs, etc.

## Setup

Full setup instructions live in the Meticulous docs: **[app.meticulous.ai/docs/agents/setup](https://app.meticulous.ai/docs/agents/setup)**. They cover installing the CLI, authenticating, installing these skills, and the agent permission allowlists for Claude Code, Codex, and Cursor.

The short version:

```bash
# 1. Install/update the Meticulous CLI
npm install --global @alwaysmeticulous/cli@latest

# 2. Authenticate (opens a browser to sign in and pick a default project)
meticulous auth login

# 3. Install these skills into your project
npx skills add alwaysmeticulous/skills --all --agents claude-code,codex,cursor
```

### Cursor Marketplace

This repository is also packaged as a [Cursor plugin](https://cursor.com/docs/plugins). Once listed in the Cursor Marketplace, Cursor users can install Meticulous skills from **Customize → Plugins** (or search the marketplace) without using `npx skills`. The `npx skills` flow above remains the recommended path when you need the same skills across multiple agents.

## Skills

Skills are Markdown files that provide AI coding assistants with domain-specific knowledge and step-by-step workflows. When an agent reads a skill, it gains targeted expertise for a specific task — without needing that context baked into every conversation.

Skills follow the [`SKILL.md` format](https://docs.cursor.com/context/rules-for-ai) used by Cursor and Claude Code.

| Skill | Description |
|-------|-------------|
| **Review skills** | |
| [`meticulous-review`](skills/meticulous-review/SKILL.md) | Analyze a completed Meticulous test run — fetch the diff summary, inspect representative screenshots, DOM diffs, and timelines. Resolves the test run from the current commit. Use when asked to review Meticulous test results, or while reviewing or babysitting a pull/merge request to assess and fix a failing Meticulous Tests CI check. |
| [`meticulous-test`](skills/meticulous-test/SKILL.md) | Run a Meticulous test run after implementing a frontend change, then hands off to the `meticulous-review` skill to classify each visual change as intended or unintended. Use when implementing a feature autonomously end-to-end before creating a PR. |
| **Experimental skills** | |
| [`meticulous-iterative-dev`](skills/meticulous-iterative-dev/SKILL.md) | Iterative frontend development loop using Meticulous for per-step visual validation. Use when implementing a multi-step frontend change and want to catch visual regressions and unintended side effects at each step, before the final cloud test run. |
| [`meticulous-simulate-and-diff`](skills/meticulous-simulate-and-diff/SKILL.md) | Run a Meticulous session simulation against a live URL and analyze the visual output — either by inspecting screenshots directly (quick-check mode) or by comparing pixel and HTML diffs against a base replay. Use when checking whether a code change has introduced visual regressions for a specific session. |
| [`meticulous-use-session-data`](skills/meticulous-use-session-data/SKILL.md) | Download and use structured Meticulous session data (user flows + network mocks) for testing code changes locally. Use when you need to understand what user interactions and API calls a test covers, or when you want network mocks for writing tests. |
| **Supporting skills** | |
| [`meticulous-cli`](skills/meticulous-cli/SKILL.md) | Overview of the Meticulous CLI tool and its global options. Use when asking about the meticulous CLI in general, available commands, or global flags that apply to all commands. |
| [`meticulous-cli-update`](skills/meticulous-cli-update/SKILL.md) | Check whether the Meticulous CLI (@alwaysmeticulous/cli) is installed and up to date, and install/update it if not. Invoked at the start of every other Meticulous skill, since the CLI is under active development with frequent changes and improvements. |

## License

[ISC](LICENSE) © Meticulous, Inc.
