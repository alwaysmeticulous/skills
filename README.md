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

### 2. Set your API token

The CLI requires a Meticulous API token. See [the docs](https://app.meticulous.ai/docs/github-actions-v2) for how to create one, then add it to your environment:

```bash
export METICULOUS_API_TOKEN=xyz...
```

Alternatively, store the token in `~/.meticulous/config.json`:

```json
{
  "apiToken": "xyz..."
}
```

### 3. Install skills

Install skills into your project using [`npx skills`](https://github.com/vercel-labs/skills):

```bash
# Claude Code only: create .claude/ first if it doesn't already exist
mkdir .claude

# Interactive: pick which skills to install and which agents to install them for
npx skills add alwaysmeticulous/skills

# Non-interactive: install all skills for the specified agents
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
      "Read(/HOME/DIR/.meticulous/agent-images/**)",
      "Bash(meticulous agent *)",
      "Bash(meticulous ci upload-assets --appDirectory * --repoDirectory * --waitForTestRunToComplete)",
      "Bash(meticulous ci upload-container --localImageTag * --repoDirectory . --waitForTestRunToComplete)",
      "Bash(meticulous debug *)",
      "Bash(meticulous download *)",
      "Bash(meticulous local *)",
      "Bash(meticulous schema *)",
      "Bash(meticulous simulate *)"
    ]
  }
}
```

#### 4.2 Codex

Add the following to `~/.codex/rules/default.rules`:

```starlark
prefix_rule(pattern=["meticulous", "agent"], decision="allow")
prefix_rule(pattern=["meticulous", "ci", "upload-assets"], decision="allow")
prefix_rule(pattern=["meticulous", "ci", "upload-container"], decision="allow")
prefix_rule(pattern=["meticulous", "debug"], decision="allow")
prefix_rule(pattern=["meticulous", "download"], decision="allow")
prefix_rule(pattern=["meticulous", "local"], decision="allow")
prefix_rule(pattern=["meticulous", "schema"], decision="allow")
prefix_rule(pattern=["meticulous", "simulate"], decision="allow")
```

#### 4.3 Cursor

Add the following entries to Cursor's Command Allowlist:

- `meticulous agent`
- `meticulous ci upload-assets`
- `meticulous ci upload-container`
- `meticulous debug`
- `meticulous download`
- `meticulous local`
- `meticulous schema`
- `meticulous simulate`

## Skills

Skills are Markdown files that provide AI coding assistants with domain-specific knowledge and step-by-step workflows. When an agent reads a skill, it gains targeted expertise for a specific task — without needing that context baked into every conversation.

Skills follow the [`SKILL.md` format](https://docs.cursor.com/context/rules-for-ai) used by Cursor and Claude Code.

| Skill | Description |
|-------|-------------|
| [`meticulous-iterative-dev`](meticulous-iterative-dev/SKILL.md) | Iterative frontend development loop using Meticulous for per-step visual validation. Use when implementing a multi-step frontend change and want to catch visual regressions at each step. |
| [`meticulous-review`](meticulous-review/SKILL.md) | Review a completed Meticulous test run by inspecting screenshot diffs, DOM diffs, and timelines, and classifying each visual change as intended or unintended. |
| [`meticulous-simulate-and-diff`](meticulous-simulate-and-diff/SKILL.md) | Run a Meticulous session simulation against a live URL and analyze the visual output — either by inspecting screenshots directly or by comparing pixel and HTML diffs against a base replay. |
| [`meticulous-test`](meticulous-test/SKILL.md) | Run after implementing any frontend change to verify its visual impact. Triggers a Meticulous test run, then hands off to `meticulous-review` to classify each visual change. |
| [`meticulous-use-session-data`](meticulous-use-session-data/SKILL.md) | Download and use structured Meticulous session data (user flows + network mocks) for testing code changes locally. Use when you need to understand what user interactions and API calls a test covers, or when you want network mocks for writing tests. |
| | |
| [`meticulous-cli`](meticulous-cli/SKILL.md) | Reference documentation for the Meticulous CLI tool, its commands, and global options. |
| [`meticulous-cli-update`](meticulous-cli-update/SKILL.md) | Check whether the Meticulous CLI is up to date and update it if not. Invoked at the start of every other Meticulous skill. |

## License

[ISC](LICENSE) © Meticulous, Inc.
