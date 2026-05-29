# Meticulous Agent Skills

A collection of agent skills for [Meticulous](https://meticulous.ai) — the automated visual regression testing platform for web frontends.

These skills are designed to be used with AI coding assistants (Cursor, Claude Code, etc.) to help investigate and debug issues in Meticulous test runs and replays.

> **Beta:** The agentic skills for Meticulous are in early beta. Expect some instability and frequent breaking changes. We recommend updating at least once a week (see [Install/update](#1-installupdate-the-meticulous-cli) below).

## Setup

### 1. Install/update the Meticulous CLI

The [Meticulous CLI](https://github.com/alwaysmeticulous/meticulous-sdk) is required to run simulations and test runs from the command line:

```bash
npm install --global @alwaysmeticulous/cli@latest
```

You can also install it locally per-project instead of globally.

The CLI is under active development with frequent breaking changes — re-run the same command to update to the latest version. The skills in this repo also invoke the [`meticulous-cli-update`](meticulous-cli-update/SKILL.md) skill at the start of every workflow to keep this in sync automatically.

### 2. Authentication

Authenticate the CLI with your Meticulous account:

```bash
meticulous auth whoami
```

This will open a browser to sign in if you're not already authenticated. If you're a member of multiple projects, also run `meticulous auth set-project` to pick which one to use (it's an interactive prompt).

Alternatively, use an API token (see [the docs](https://app.meticulous.ai/docs/github-actions-v2) for how to create one) — either via the `METICULOUS_API_TOKEN` environment variable, or by storing `{"apiToken": "xyz..."}` in `~/.meticulous/config.json`.

### 3. Install skills

Install skills into your project using [`npx skills`](https://github.com/vercel-labs/skills):

```bash
# Claude Code only: create .claude/ first if it doesn't already exist
mkdir -p .claude

# Install all skills for the specified agents
npx skills add alwaysmeticulous/skills --all --agents claude-code,codex,cursor

# Update already-installed skills to the latest version
npx skills update --project
```

### 4. Agent permissions

The Meticulous skills run a small set of CLI commands and read downloaded screenshots from `~/.meticulous/agent-images/`. Pre-allowing these in your agent configuration avoids repeated permission prompts during a session.

#### 4.1 Claude Code

Add the following to `.claude/settings.json` (project-level) or `~/.claude/settings.json` (user-level). Replace `/HOME/DIR` with your home directory path (e.g. `/Users/alice`).

```json
{
  "permissions": {
    "allow": [
      "Read(~/.meticulous/agent-images/**)",
      "Bash(meticulous *)",
      "Skill(meticulous-cli-update)",
      "Bash(npm view @alwaysmeticulous/cli version)",
      "Bash(npm install --global @alwaysmeticulous/cli@latest)",
      "Bash(npx skills update --project)"
    ]
  }
}
```

#### 4.2 Codex

Add the following to `~/.codex/rules/default.rules`:

```starlark
prefix_rule(pattern=["meticulous"], decision="allow")
prefix_rule(pattern=["npm", "view", "@alwaysmeticulous/cli", "version"], decision="allow")
prefix_rule(pattern=["npm", "install", "--global", "@alwaysmeticulous/cli@latest"], decision="allow")
prefix_rule(pattern=["npx", "skills", "update", "--project"], decision="allow")
```

#### 4.3 Cursor

Add the following entries to Cursor's Command Allowlist:

```bash
meticulous
npm view @alwaysmeticulous/cli version
npm install --global @alwaysmeticulous/cli@latest
npx skills update --project
```

## Skills

Skills are Markdown files that provide AI coding assistants with domain-specific knowledge and step-by-step workflows. When an agent reads a skill, it gains targeted expertise for a specific task — without needing that context baked into every conversation.

Skills follow the [`SKILL.md` format](https://docs.cursor.com/context/rules-for-ai) used by Cursor and Claude Code.

| Skill | Description |
|-------|-------------|
| [`meticulous-iterative-dev`](meticulous-iterative-dev/SKILL.md) | Iterative frontend development loop using Meticulous for per-step visual validation. Use when implementing a multi-step frontend change and want to catch visual regressions and unintended side effects at each step, before the final cloud test run. |
| [`meticulous-review`](meticulous-review/SKILL.md) | Review a completed Meticulous test run — fetch the diff summary, inspect representative screenshots, DOM diffs, and timelines, then classify each visual change as intended or unintended. Use when you have a `testRunId`, a PR URL, or a PR number and need to assess what visually changed. |
| [`meticulous-simulate-and-diff`](meticulous-simulate-and-diff/SKILL.md) | Run a Meticulous session simulation against a live URL and analyze the visual output — either by inspecting screenshots directly (quick-check mode) or by comparing pixel and HTML diffs against a base replay. Use when checking whether a code change has introduced visual regressions for a specific session. |
| [`meticulous-test`](meticulous-test/SKILL.md) | Run after implementing any frontend change to verify its visual impact. Triggers a Meticulous test run, then hands off to the `meticulous-review` skill to classify each visual change as intended or unintended. Use this before marking a frontend task as complete. |
| [`meticulous-use-session-data`](meticulous-use-session-data/SKILL.md) | Download and use structured Meticulous session data (user flows + network mocks) for testing code changes locally. Use when you need to understand what user interactions and API calls a test covers, or when you want network mocks for writing tests. |
| | |
| [`meticulous-cli`](meticulous-cli/SKILL.md) | Overview of the Meticulous CLI tool and its global options. Use when asking about the meticulous CLI in general, available commands, or global flags that apply to all commands. |
| [`meticulous-cli-update`](meticulous-cli-update/SKILL.md) | Check whether the Meticulous CLI (@alwaysmeticulous/cli) is installed and up to date, and install/update it if not. Invoked at the start of every other Meticulous skill, since the CLI is in active beta with frequent breaking changes. |

## License

[ISC](LICENSE) © Meticulous, Inc.
