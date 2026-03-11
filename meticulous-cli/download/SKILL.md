---
name: meticulous-cli-download
description: Meticulous CLI download commands for fetching recorded sessions, replays, and test runs to the local data directory. Covers `meticulous download session`, `meticulous download replay`, and `meticulous download test-run`.
---

# meticulous download

Commands for downloading Meticulous artifacts to the local `dataDir` (default `~/.meticulous`). All commands are idempotent — if the artifact already exists locally it will not be re-fetched.

## download session

```bash
meticulous download session --sessionId=<id> [--apiToken=<token>]
```

**Purpose:** Download a recorded session's metadata and raw event data from Meticulous.

**What gets downloaded (into `dataDir/sessions/<sessionId>/`):**
- Session metadata JSON (recording info, timestamps, URL)
- Session data file (the full event log used for replay)

**Options:**

| Option | Type | Required | Description |
|--------|------|----------|-------------|
| `--sessionId` | string | yes | ID of the session to download |
| `--apiToken` | string | no | Meticulous API token; falls back to OAuth login if omitted |

**Example:**
```bash
meticulous download session --sessionId=abc123
# Downloaded session metadata to: /Users/you/.meticulous/sessions/abc123/metadata.json
# Downloaded session data to: /Users/you/.meticulous/sessions/abc123/session.json
```

---

## download replay

```bash
meticulous download replay --replayId=<id> [--apiToken=<token>]
```

**Purpose:** Download a replay's metadata and full archive (screenshots, network logs, console logs) from Meticulous. Also generates human-readable log files from the raw NDJSON log.

**What gets downloaded (into `dataDir/replays/<replayId>/`):**
- Replay metadata JSON
- Full replay archive (screenshots, assets, logs)

**Generated files (post-download):**
- `logs.concise.txt` — Human-readable logs with virtual timestamps and trace IDs
- `logs.deterministic.txt` — Like `logs.concise.txt` but with non-deterministic values stripped (event IDs replaced, `[non-deterministic]` entries removed). Suitable for diffing between replay runs.

**Options:**

| Option | Type | Required | Description |
|--------|------|----------|-------------|
| `--replayId` | string | yes | ID of the replay to download |
| `--apiToken` | string | no | Meticulous API token; falls back to OAuth login if omitted |

**Example:**
```bash
meticulous download replay --replayId=rpl_xyz
# Downloaded replay metadata to: /Users/you/.meticulous/replays/rpl_xyz/metadata.json
# Downloaded replay data to: /Users/you/.meticulous/replays/rpl_xyz/
```

---

## download test-run

```bash
meticulous download test-run [--testRunId=<id>] [--apiToken=<token>]
```

**Purpose:** Download data for a Meticulous test run (defaults to the most recent run). Primarily used to inspect code coverage data.

**What gets downloaded (into `dataDir/test-runs/<testRunId>/`):**
- Coverage data and other artifacts within the requested `--scope`

**Options:**

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `--testRunId` | string | `latest` | ID of the test run to download; omit to fetch the most recent run |
| `--apiToken` | string | — | Meticulous API token (no OAuth fallback — must be provided or set via env) |
| `--scope` | string | `coverage-only` | What to download: `coverage-only` or `app-container-logs` |

**Example:**
```bash
# Download coverage for the latest test run
meticulous download test-run --apiToken=$TOKEN

# Download coverage for a specific test run
meticulous download test-run --apiToken=$TOKEN --testRunId=tr_abc
# Downloaded test run data to: /Users/you/.meticulous/test-runs/tr_abc/
```
