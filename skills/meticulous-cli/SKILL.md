---
name: meticulous-cli
description: Overview of the Meticulous CLI tool and its global options. Use when asking about the meticulous CLI in general, available commands, or global flags that apply to all commands.
user-invocable: true
---

# Meticulous CLI

The `meticulous` CLI records user sessions and replays them to catch visual regressions. It is installed as part of `@alwaysmeticulous/cli`.

> Before starting, run the `meticulous-cli-update` skill to ensure the Meticulous CLI is up to date — unless it has already run earlier in this conversation, in which case skip it.

## Invocation

```bash
meticulous <command> [options]
```

The skills assume `meticulous` is on `PATH`. The `meticulous-cli-update` skill installs it globally via `npm install --global @alwaysmeticulous/cli@latest` if missing. It can also be invoked as `npx @alwaysmeticulous/cli` when installed locally per-project.

## Command Groups

| Command | Purpose |
|---------|---------|
| `auth` | Authenticate with Meticulous (whoami, logout) |
| `debug` | Set up AI-ready debug workspaces for investigating replay diffs and replays |
| `download` | Download sessions, replays, and test runs locally |
| `local` | Find sessions relevant to the current branch's code changes |
| `project` | Inspect the project you're authenticated against |
| `simulate` / `replay` | Replay a recorded session against a URL |
| `schema` | Output the CLI command schema as JSON (for agent/programmatic use) |

See the reference for each group for full option details:
- [references/auth.md](references/auth.md)
- [references/debug.md](references/debug.md)
- [references/download.md](references/download.md)
- [references/local.md](references/local.md)
- [references/project.md](references/project.md)
- [references/simulate.md](references/simulate.md)
- [references/schema.md](references/schema.md)

## Global Options

These options are accepted by every command:

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `--logLevel` | string | `info` | Log verbosity: `trace`, `debug`, `info`, `warn`, `error`, `silent` |
| `--dataDir` | string | `~/.meticulous` | Directory where sessions, replays, and other data are stored |
| `--rawJson` | string | — | Pass all options as a JSON string (useful for programmatic/agent invocation) |
| `--dryRun` | boolean | `false` | Print what the command would do without making any changes |

## Authentication

Commands authenticate via OAuth. The login token is stored on disk and reused across sessions — if you're not logged in, an interactive command opens a browser sign-in automatically (or run `meticulous auth whoami` to trigger it). See [references/auth.md](references/auth.md).

An API token (via `--apiToken` or the `METICULOUS_API_TOKEN` environment variable, scoped to a specific organization/project) is also supported, and is the way to authenticate in non-interactive contexts such as CI.

## Example

```bash
# Print schema for all commands (agent use)
meticulous schema

# Run a single replay locally
meticulous simulate --sessionId=<id> --appUrl=http://localhost:3000
```
