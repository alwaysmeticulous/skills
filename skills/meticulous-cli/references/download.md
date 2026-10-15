# meticulous download

Commands for downloading Meticulous artifacts to the local `dataDir` (default `~/.meticulous`). All commands are idempotent — if the artifact already exists locally it will not be re-fetched.

**Output:** stdout carries only the path(s) the data was written to — or, with `--json`, an object naming them. Progress logs go to stderr and only appear with `--verbose`.

## download session

```bash
# CLI
meticulous download session --sessionId=<id> [--format=<format>] [--json] [--apiToken=<token>]

# MCP (returns the same structured data inline instead of writing it to disk)
get_session_data(sessionId="<id>")
```

**Purpose:** Download a recorded session's data from Meticulous. `download replay`/`download test-run` below have no MCP equivalent.

**Options:**

| Option        | Type                   | Default                | Description                                                                                      |
| ------------- | ---------------------- | ---------------------- | ------------------------------------------------------------------------------------------------ |
| `--sessionId` | string                 | —                      | ID of the session to download (required)                                                         |
| `--format`    | `json` \| `multi-file` | `json`                 | `json` downloads the original single JSON file. `multi-file` writes a structured directory tree. |
| `--outputDir` | string                 | `.meticulous/sessions` | Output directory for multi-file format                                                           |
| `--apiToken`  | string                 | —                      | Meticulous API token; otherwise use the default auth chain (see `auth whoami`)                   |

**JSON format** (default) downloads into `dataDir/sessions/` and prints both paths (`--json`: `{ sessionId, metadataPath, dataPath }`):

- `<sessionId>.json` — session metadata (recording info, timestamps, URL)
- `<sessionId>_data.json` — the full session data used for replay

**Multi-file format** writes a structured directory tree and prints the output directory (`--json`: `{ sessionId, path }`). See the `meticulous-use-session-data` skill for details on the output structure.

**Examples:**

```bash
# Download raw session data as JSON (default)
meticulous download session --sessionId=abc123

# Download in multi-file structured format
meticulous download session --sessionId=abc123 --format=multi-file

# Download to a custom output directory
meticulous download session --sessionId=abc123 --format=multi-file --outputDir=./test-data
```

---

## download replay

```bash
meticulous download replay --replayId=<id> [--json] [--apiToken=<token>]
```

**Purpose:** Download a replay's metadata and full archive (screenshots, diffs, timeline, console logs, coverage) from Meticulous. Also generates human-readable log files from the raw NDJSON log.

**What gets downloaded (into `dataDir/replays/<replayId>/`):**

- Replay metadata JSON
- Full replay archive (screenshots, assets, logs)

**Generated files (post-download):**

- `logs.concise.txt` — Human-readable logs with virtual timestamps and trace IDs
- `logs.deterministic.txt` — Like `logs.concise.txt` but with non-deterministic values stripped (event IDs replaced, `[non-deterministic]` entries removed). Suitable for diffing between replay runs.

**Options:**

| Option       | Type   | Required | Description                                                                    |
| ------------ | ------ | -------- | ------------------------------------------------------------------------------ |
| `--replayId` | string | yes      | ID of the replay to download                                                   |
| `--apiToken` | string | no       | Meticulous API token; otherwise use the default auth chain (see `auth whoami`) |

**Example:**

```bash
meticulous download replay --replayId=rpl_xyz
# /Users/you/.meticulous/replays/rpl_xyz
```

---

## download test-run

```bash
meticulous download test-run [--testRunId=<id|latest> | --prNumber=<n> | --commitSha=<sha>] [--project=<project>] [--scope=<scope>] [--json] [--apiToken=<token>]
```

**Purpose:** Download data for a Meticulous test run. Primarily used to inspect code coverage data. The run is resolved like the `agent` commands: `--testRunId`, else `--prNumber`, else `--commitSha`, else the current git HEAD. Pass `--testRunId=latest` for the project's latest main-branch run.

**What gets downloaded (into `dataDir/test-runs/<testRunId>/`):**

- Coverage data and other artifacts within the requested `--scope`

**Options:**

| Option                           | Type    | Default         | Description                                                                                                                                                                              |
| -------------------------------- | ------- | --------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `--testRunId`                    | string  | —               | ID of the test run to download, or `latest` for the project's latest main-branch run                                                                                                     |
| `--prNumber`                     | number  | —               | Download the latest test run for this pull request's head commit                                                                                                                         |
| `--commitSha`                    | string  | git HEAD        | Download the latest test run for this commit                                                                                                                                             |
| `--project`                      | string  | default project | Project to look the run up in (id, `org/proj`, or `proj`); with `--testRunId`, the run must belong to it                                                                                 |
| `--scope`                        | string  | `coverage-only` | What to download: `coverage-only`, `app-container-logs`, `coverage-pr-only`, `coverage-by-replay-pr-only` or `everything`. A later download of another scope adds only the missing files |
| `--dontWaitForTestRunToComplete` | boolean | `false`         | Exit without downloading if the run is still in progress, instead of waiting for it                                                                                                      |
| `--apiToken`                     | string  | —               | Meticulous API token; otherwise use the default auth chain (see `auth whoami`)                                                                                                           |

The pull-request scopes need read access to the project's source code; without it the command fails rather than downloading nothing.

With `--json`, prints `{ testRunId, path }` (or `{ testRunId, status }` for a run skipped by `--dontWaitForTestRunToComplete`).

**Example:**

```bash
# Download coverage for the test run of the current git HEAD
meticulous download test-run

# Download coverage for the project's latest main-branch run
meticulous download test-run --testRunId=latest

# ...of a project other than your default one
meticulous download test-run --testRunId=latest --project=my-org/my-project

# Download coverage for a specific test run or pull request
meticulous download test-run --testRunId=tr_abc
meticulous download test-run --prNumber=123
# /Users/you/.meticulous/test-runs/tr_abc
```
