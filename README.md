# Meticulous Agent Skills

A collection of agent skills for [Meticulous](https://meticulous.ai) — the automated visual regression testing platform for web frontends.

These skills enable AI coding assistants (Cursor, Claude Code, etc.) to use Meticulous: review test runs, investigate replays, debug diffs, etc.

## Setup

Full setup instructions live in the Meticulous docs: **[app.meticulous.ai/docs/agents/setup](https://app.meticulous.ai/docs/agents/setup)**. They cover installing the CLI, authenticating, and installing these skills.

The short version:

```bash
# 1. Install/update the Meticulous CLI
npm install --global @alwaysmeticulous/cli@latest

# 2. Authenticate (opens a browser to sign in and pick a default project)
meticulous auth login

# 3. Install these skills into your project
npx skills add alwaysmeticulous/skills --skill "*" --agent claude-code --agent codex --agent cursor -y
```

### Claude Code plugin

This repository is also packaged as a [Claude Code plugin](https://code.claude.com/docs/en/discover-plugins), which installs all the skills below (namespaced as `/meticulous:<skill-name>`) and automatically connects the hosted [Meticulous MCP server](https://app.meticulous.ai/api/mcp). To install, run in Claude Code:

```shell
/plugin marketplace add alwaysmeticulous/skills
/plugin install meticulous@meticulous
```

Then type `/mcp` and choose **Authenticate** for the Meticulous MCP server (login happens in your browser; not needed if you already authenticate via `meticulous auth login` for the CLI).

### Cursor Marketplace

This repository is also packaged as a [Cursor plugin](https://cursor.com/docs/plugins). Once listed in the Cursor Marketplace, Cursor users can install Meticulous skills from **Customize → Plugins** (or search the marketplace) without using `npx skills`. The `npx skills` flow above remains the recommended path when you need the same skills across multiple agents.

## Skills

Skills are Markdown files that provide AI coding assistants with domain-specific knowledge and step-by-step workflows. When an agent reads a skill, it gains targeted expertise for a specific task — without needing that context baked into every conversation.

Skills follow the [`SKILL.md` format](https://docs.cursor.com/context/rules-for-ai) used by Cursor and Claude Code.

| Skill                                                                          | Description                                                                                                                                                                                                                                                                                                                                   |
| ------------------------------------------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Review skills**                                                              |                                                                                                                                                                                                                                                                                                                                               |
| [`meticulous-review`](skills/meticulous-review/SKILL.md)                       | Analyze a completed Meticulous test run — compare the diffs against the PR description to see what's expected, then focus on finding and flagging potential regressions. Use when asked to review Meticulous test results, right after implementing a frontend change, or while babysitting a pull/merge request's Meticulous Tests CI check. |
| [`meticulous-fix`](skills/meticulous-fix/SKILL.md)                             | Fix the visual diffs that have been reviewed and rejected on a test run, following their review comments if given. Use when a user has reviewed the results of a test run and is handing off to an agent to implement the fixes.                                                                                                              |
| [`meticulous-test`](skills/meticulous-test/SKILL.md)                           | Run a Meticulous test run after implementing a frontend change, then hands off to the `meticulous-review` skill to classify each visual change as intended or unintended.                                                                                                                                                                     |
| [`meticulous-zero-diff-task`](skills/meticulous-zero-diff-task/SKILL.md)       | Implement a task for which no visual diffs are expected end to end, using Meticulous as the loop you iterate against until the diff is clean, then open a PR. Use when the task's whole premise is "the UI shouldn't change" — a dependency/version upgrade, a code refactor, a migration, or similar.                                        |
| **Experimental skills**                                                        |                                                                                                                                                                                                                                                                                                                                               |
| [`meticulous-iterative-dev`](skills/meticulous-iterative-dev/SKILL.md)         | Iterative frontend development loop using Meticulous for per-step visual validation. Use when implementing a multi-step frontend change and want to catch visual regressions and unintended side effects at each step, before the final cloud test run.                                                                                       |
| [`meticulous-simulate-and-diff`](skills/meticulous-simulate-and-diff/SKILL.md) | Run a Meticulous session simulation against a live URL and analyze the visual output — either by inspecting screenshots directly (quick-check mode) or by comparing pixel and HTML diffs against a base replay. Use when checking whether a code change has introduced visual regressions for a specific session.                             |
| [`meticulous-use-session-data`](skills/meticulous-use-session-data/SKILL.md)   | Download and use structured Meticulous session data (user flows + network mocks) for testing code changes locally. Use when you need to understand what user interactions and API calls a test covers, or when you want network mocks for writing tests.                                                                                      |
| **Supporting skills**                                                          |                                                                                                                                                                                                                                                                                                                                               |
| [`meticulous-cli`](skills/meticulous-cli/SKILL.md)                             | Overview of the Meticulous CLI tool and its global options. Use when asking about the meticulous CLI in general, available commands, or global flags that apply to all commands.                                                                                                                                                              |
| [`meticulous-cli-update`](skills/meticulous-cli-update/SKILL.md)               | Check whether the Meticulous CLI (@alwaysmeticulous/cli) and skills are installed and up to date, and install/update them if not. Invoked at the start of every other Meticulous skill, since the CLI and skills are under active development with frequent changes and improvements.                                                         |

## Contributing

All Markdown, JSON and YAML in this repository is formatted with
[oxfmt](https://www.npmjs.com/package/oxfmt), and CI enforces it:

```bash
pnpm install
pnpm format        # format in place
pnpm check-format  # what CI runs
```

This isn't only about house style. Skills from this repository get installed
into other repositories (via `npx skills add`), some of which run oxfmt over
their whole tree — including the installed skill files. If a skill isn't
already oxfmt-clean at the source, their formatter rewrites it on the way in,
the installed copy no longer matches the upstream hash recorded in
`skills-lock.json`, and every subsequent `npx skills update` there fights the
formatter. Keeping the source formatted the same way avoids that entirely, so
`.oxfmtrc.json` here mirrors the settings used in the repositories these skills
are consumed from (notably `printWidth: 80`).

## License

[ISC](LICENSE) © Meticulous, Inc.
