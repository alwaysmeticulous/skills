---
name: meticulous-cli
description: Overview of the Meticulous CLI tool and its global options. Use when asking about the meticulous CLI in general, available commands, or global flags that apply to all commands.
---

# Meticulous CLI

The `meticulous` CLI records user sessions and replays them to catch visual regressions. It is installed as part of `@alwaysmeticulous/cli`.

## Invocation

```bash
npx @alwaysmeticulous/cli <command> [options]
```

The CLI can also be invoked as `meticulous` if installed globally or available on `PATH`, but `npx @alwaysmeticulous/cli` is the typical usage.

## Command Groups

| Command | Purpose |
|---------|---------|
| `auth` | Authenticate with Meticulous (whoami, logout) |
| `ci` | CI/CD workflows — run tests locally or in cloud, upload assets/containers, tunnel |
| `download` | Download sessions, replays, and test runs locally |
| `project` | Inspect the project linked to an API token |
| `record` | Open a browser and record a session |
| `simulate` / `replay` | Replay a recorded session against a URL |
| `schema` | Output the CLI command schema as JSON (for agent/programmatic use) |

See the skill for each group for full option details:
- [auth/SKILL.md](auth/SKILL.md)
- [ci/SKILL.md](ci/SKILL.md)
- [download/SKILL.md](download/SKILL.md)
- [project/SKILL.md](project/SKILL.md)
- [record/SKILL.md](record/SKILL.md)
- [simulate/SKILL.md](simulate/SKILL.md)
- [schema/SKILL.md](schema/SKILL.md)

## Global Options

These options are accepted by every command:

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `--logLevel` | string | `info` | Log verbosity: `trace`, `debug`, `info`, `warn`, `error`, `silent` |
| `--dataDir` | string | `~/.meticulous` | Directory where sessions, replays, and other data are stored |
| `--rawJson` | string | — | Pass all options as a JSON string (useful for programmatic/agent invocation) |
| `--dryRun` | boolean | `false` | Print what the command would do without making any changes |

## API Token

Most commands require an API token, passed via `--apiToken` or the `METICULOUS_API_TOKEN` environment variable. The token is scoped to a specific organization/project.

## Example

```bash
# Print schema for all commands (agent use)
meticulous schema

# Run a single replay locally
meticulous simulate --sessionId=<id> --appUrl=http://localhost:3000

# Run all tests in the cloud
meticulous ci run-with-tunnel --apiToken=$TOKEN --appUrl=http://localhost:3000
```
