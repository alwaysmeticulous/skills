---
name: meticulous-cli-debug
description: Meticulous CLI debug commands for creating AI-ready workspaces to investigate visual diffs, replay issues, and test run failures. Covers `meticulous debug replay-diff`, `debug test-run`, `debug replays`, and `debug clean`.
---

# meticulous debug

Commands for setting up self-contained debug workspaces that bundle replay data, session recordings, diffs, analysis artifacts, and AI agent context files. The workspace is designed to be opened in an AI coding tool (Claude Code, Cursor, etc.) for assisted debugging.

Workspaces are created under `~/.meticulous/debug-sessions/` by default.

## debug replay-diff

```bash
meticulous debug replay-diff <replayDiffId> [options]
```

**Purpose:** Debug a specific replay diff. Downloads the head and base replays, session data, diff results, and PR diff, then generates a workspace with analysis artifacts and AI context.

**Required arguments:**

| Argument | Type | Description |
|----------|------|-------------|
| `replayDiffId` | string (positional) | The replay diff ID to debug |

**Options:**

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `--apiToken` | string | — | Meticulous API token; prompts OAuth login if omitted |
| `--sessionId` | string | — | Override the session ID for this replay diff |
| `--workspaceName` | string | timestamp | Custom name for the debug workspace directory |
| `--screenshot` | string | — | Screenshot filename to focus analysis on |

**Example:**
```bash
meticulous debug replay-diff rd_abc123
meticulous debug replay-diff rd_abc123 --screenshot="checkout-page.png"
```

---

## debug test-run

```bash
meticulous debug test-run <testRunId> [options]
```

**Purpose:** Debug all diffs in a test run. Fetches replay diffs for the test run, prioritizing diffs with visual changes, and downloads up to `--maxDiffs` of them along with their replays and session data.

**Required arguments:**

| Argument | Type | Description |
|----------|------|-------------|
| `testRunId` | string (positional) | The test run ID to debug |

**Options:**

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `--apiToken` | string | — | Meticulous API token; prompts OAuth login if omitted |
| `--maxDiffs` | number | `5` | Maximum number of replay diffs to download |
| `--workspaceName` | string | timestamp | Custom name for the debug workspace directory |
| `--screenshot` | string | — | Screenshot filename to focus analysis on |

**Example:**
```bash
meticulous debug test-run tr_xyz789
meticulous debug test-run tr_xyz789 --maxDiffs=10
```

---

## debug replays

```bash
meticulous debug replays <replayIds..> [options]
```

**Purpose:** Debug one or more replays by ID. Downloads each replay's archive and session data, then creates a workspace. Useful when you have specific replay IDs rather than a diff or test run.

**Required arguments:**

| Argument | Type | Description |
|----------|------|-------------|
| `replayIds` | string[] (positional, variadic) | One or more replay IDs to debug |

**Options:**

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `--apiToken` | string | — | Meticulous API token; prompts OAuth login if omitted |
| `--sessionId` | string | — | Override the session ID |
| `--workspaceName` | string | timestamp | Custom name for the debug workspace directory |
| `--screenshot` | string | — | Screenshot filename to focus analysis on |

**Example:**
```bash
meticulous debug replays rpl_aaa rpl_bbb
meticulous debug replays rpl_aaa --sessionId=ses_override
```

---

## debug clean

```bash
meticulous debug clean [--all]
```

**Purpose:** Clean up debug workspaces. By default, opens an interactive menu to select which workspaces to delete. Use `--all` for non-interactive deletion (e.g. from scripts or AI agents).

**Options:**

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `--all` | boolean | `false` | Delete all workspaces without prompting (useful for non-interactive environments) |

**Interactive mode (default):** Lists all workspaces with their sizes, then offers three choices:
1. Delete all workspaces (with confirmation)
2. Select individual workspaces to delete
3. Cancel

**Non-interactive mode (`--all`):** Lists all workspaces and deletes them immediately without prompting.

**Example:**
```bash
# Interactive cleanup
meticulous debug clean

# Non-interactive — delete everything
meticulous debug clean --all
```

---

## Workspace Contents

Each debug workspace contains:

| Directory | Contents |
|-----------|----------|
| `debug-data/` | Replay archives, session recordings, diff results, filtered logs, and analysis artifacts |
| `.claude/` | AI agent context files (CLAUDE.md, rules, skills, hooks) for assisted debugging |
| `project-repo/` | A git worktree of your codebase at the relevant commit (only if run from within a git repository) |

The workspace path is copied to the clipboard on creation. Open it in your preferred tool:

```bash
# Claude Code
cd "<workspace-path>" && claude

# Cursor
cursor "<workspace-path>"
```

## Git Worktree

When run from within a git repository, the command detects the repository root, fetches the relevant commit SHA (from the test run or replay), and creates a `git worktree` at `project-repo/` inside the workspace. This is automatically cleaned up by `debug clean`.
