# meticulous agent

Read, analysis, and run-triggering commands designed for AI coding agents. They resolve git context (commit SHA, base, diff) automatically from the local repository, and default to machine-readable output.

All commands are also exposed as tools on the hosted **MCP server** (`https://app.meticulous.ai/api/mcp`) — an MCP-enabled client can call the `get_…` tool directly instead of shelling out. Each read tool takes broadly the same arguments and returns the same data as the CLI command's `--json` output, with differences inherent to a hosted endpoint with no access to your local repo or filesystem: **`commitSha`/`baseSha`/`gitDiffOutput` are never inferred — always compute and pass them explicitly** (e.g. `git rev-parse HEAD`, `git merge-base origin/main HEAD`), there are no output-format flags, and — for `image-files` — you get signed URLs rather than files downloaded to disk. The **MCP tool** column below gives the mapping.

`upload-build`/`trigger-test-run` (the mutating commands) map to MCP tools too, but not 1:1 — the CLI's single `agent upload-build` call is split into a **request → (upload) → register** pair on MCP (`request_asset_upload`/`request_container_upload` then `register_asset_build`/`register_container_build`), and `trigger_test_run` **does not wait for the run to finish** (unlike the CLI, which blocks by default) — poll `get_test_run_diffs_counts` or `get_test_run_diffs` to follow it. See the [`meticulous-test`](../../meticulous-test/SKILL.md) skill for the CLI workflow; use `mcp-server.ts`/the in-app MCP docs for the exact MCP tool call sequence.

## Common options

Accepted by every `agent` command (in addition to the [global options](../SKILL.md#global-options)):

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `--apiToken` | string | — | Meticulous API token; otherwise use the default auth chain (see `auth whoami`) |
| `--json` | boolean | `false` | Emit JSON on stdout instead of the default TSV/plain-text format |
| `--verbose` | boolean | `false` | Print additional progress logs on stderr |

Commands that resolve a test run from a commit (`test-run-for-commit`, `test-run-diffs`, `js-coverage`, `trigger-test-run`) also accept `--project <id | org/name | name>` — a one-off override of your default project for that call only (it does not change the stored default; see [`auth`](auth.md)).

## Command → MCP tool overview

| Command | Purpose | MCP tool |
|---------|---------|----------|
| `test-run-for-commit` | Look up the latest test run for a commit | `get_test_run_for_commit` |
| `test-run-diffs` | List the screenshot diffs of a test run | `get_test_run_diffs` |
| `test-run-diffs --counts` | Aggregate diff/review totals only | `get_test_run_diffs_counts` |
| `image-urls` | Signed URLs for a screenshot diff's images | `get_image_urls` |
| `image-files` | Download a screenshot diff's images to disk | *(none — use `get_image_urls`)* |
| `dom-diff` | DOM diff for a screenshot diff | `get_dom_diff` |
| `timeline-diff` | Timeline event diffs for a replay diff | `get_timeline_diff` |
| `js-coverage --testRunId` | Per-file JS coverage for a test run | `get_test_run_js_coverage` |
| `js-coverage --latestForProject` | Per-file JS coverage for a project's latest successful run | `get_project_js_coverage` |
| `js-coverage --replayId` | Per-file JS coverage for a replay | `get_replay_js_coverage` |
| `js-coverage-diff` | Per-file JS coverage diff for a replay diff | `get_replay_diff_js_coverage_diff` |
| `sessions` | List a project's recently recorded sessions | `get_sessions` |
| `upload-build` | Upload a build, register a deployment | `request_asset_upload` + `register_asset_build` (assets), or `request_container_upload` + `register_container_build` (container) |
| `trigger-test-run` | Trigger a run against a deployment | `trigger_test_run` (returns immediately — does not wait for completion) |
| `submit-feedback` | Submit free-form feedback about Meticulous | `submit_feedback` |

For full, always-current option lists, run `meticulous schema agent <command>`.

---

## agent test-run-for-commit

```bash
meticulous agent test-run-for-commit [--commitSha=<sha>] [--project=<project>]
```

**Purpose:** Look up the latest test run for a commit (defaults to the current git HEAD) and output the `testRunId`.

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `--commitSha` | string | current git HEAD | Commit to look up the run for |
| `--dontWaitForTestRunToComplete` | boolean | `false` | Report an in-progress run and exit immediately instead of waiting |

**MCP tool:** `get_test_run_for_commit`.

## agent test-run-diffs

```bash
meticulous agent test-run-diffs [--testRunId=<id> | --commitSha=<sha>] [options]
```

**Purpose:** List the screenshot diffs for a test run — by default a selected, priority-ordered subset of representative visual differences. Outputs a TSV table (`replayDiffId`, `screenshotName`, `index`, `outcome`, `mismatchFraction`, plus requested columns). See the [`meticulous-review`](../../meticulous-review/SKILL.md) skill for the full workflow and column semantics.

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `--testRunId` | string | — | Target run explicitly (else resolved from `--commitSha`, else git HEAD) |
| `--commitSha` | string | current git HEAD | Resolve the latest run for this commit |
| `--includeAllDiffs` | boolean | `false` | Return every difference, not just the selected subset; adds an `isSelected` column |
| `--onlyUnreviewed` | boolean | `false` | Only diffs still awaiting review (implies `--includeAllDiffs`) |
| `--includeReviewDecisions` | boolean | `false` | Add a `decision` column (accepted/rejected/ignored/unreviewed) |
| `--includeReplayIds` | boolean | `false` | Add `baseReplayId` / `headReplayId` columns |
| `--includeDomDiffIds` | boolean | `false` | Add a `domDiffIds` column (one ID per distinct structural DOM change) |
| `--orderByReplayDiffs` | boolean | `false` | Order by replay diff instead of global priority |
| `--counts` | boolean | `false` | Print aggregate totals only (replays, differences, review-decision breakdown); cannot be combined with the list/filter flags |
| `--dontWaitForTestRunToComplete` | boolean | `false` | Report an in-progress run and exit immediately instead of waiting |

**MCP tools:** `get_test_run_diffs` (the list); `get_test_run_diffs_counts` (the `--counts` totals).

## agent image-urls / agent image-files

```bash
meticulous agent image-urls  --replayDiffId=<id> --screenshotName=<name>
meticulous agent image-files --replayDiffId=<id> --screenshotName=<name>
```

**Purpose:** Get the images of a screenshot diff. `image-urls` prints the outcome plus a signed URL per image (`before` / `after` / `diffImage`); `image-files` downloads them under `~/.meticulous/agent-images/` and prints the local paths instead.

| Option | Type | Description |
|--------|------|-------------|
| `--replayDiffId` | string | Replay diff the screenshot belongs to (required) |
| `--screenshotName` | string | Screenshot name (required) |

**MCP tool:** `get_image_urls` (mirrors `image-urls`). There is no download-to-disk tool, so an MCP client fetches the URLs to view the images.

## agent dom-diff

```bash
meticulous agent dom-diff --replayDiffId=<id> --screenshotName=<name> [--context=<N|full>]
```

**Purpose:** Unified-diff-style DOM diff for a screenshot diff, one hunk per change.

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `--replayDiffId` | string | — | Replay diff (required) |
| `--screenshotName` | string | — | Screenshot name (required) |
| `--context` | number \| `full` | `3` | Context lines around each hunk (`0` for none, `full` for a single full-file diff) |

**MCP tool:** `get_dom_diff`.

## agent timeline-diff

```bash
meticulous agent timeline-diff --replayDiffId=<id>
```

**Purpose:** Timeline event diffs for a replay diff. Outputs a TSV table (`diff`, `timeMs`, `event`, `description`). Useful for diagnosing why a screenshot diff occurred (failed requests, redirects, timing).

| Option | Type | Description |
|--------|------|-------------|
| `--replayDiffId` | string | Replay diff (required) |

**MCP tool:** `get_timeline_diff` (its `diff` field carries the raw status enum rather than the TSV symbol).

## agent js-coverage

```bash
meticulous agent js-coverage [--testRunId=<id> | --commitSha=<sha> | --replayId=<id> | --latestForProject] [options]
```

**Purpose:** Per-file JavaScript coverage for a whole test run, a single replay, a combined set of runs, or a project's latest successful run. Outputs a TSV table keyed on `repoFilePath` plus the requested columns.

| Option | Type | Description |
|--------|------|-------------|
| `--testRunId` / `--commitSha` | string | Coverage for a test run (defaults to the current git HEAD) |
| `--latestForProject` | boolean | Coverage for the project's preferred latest successful test run (the same run the webapp's project coverage view uses); mutually exclusive with the other run-selector options |
| `--replayId` | string | Coverage for a single replay |
| `--screenshotName` | string | Restrict to a single screenshot of the replay |
| `--headPlusTestRunIds` / `--testRunIds` | string | Comma-separated run IDs to union coverage across (same project + commit) |
| `--globFilter` | string | Only include files matching the glob |
| `--includeAllFiles` | boolean | Include files with no coverage too |
| `--prDiffOnly` | boolean | Restrict to files changed in the PR (test-run queries only) |
| `--includeExecutableRanges` / `--includeUncoveredRanges` / `--includeCoveragePercentage` | boolean | Add richer per-file coverage columns |
| `--dontWaitForTestRunToComplete` | boolean | Report an in-progress run and exit immediately instead of waiting |

**MCP tools:** `get_test_run_js_coverage` (with `--testRunId`); `get_project_js_coverage` (with `--latestForProject`); `get_replay_js_coverage` (with `--replayId`).

## agent js-coverage-diff

```bash
meticulous agent js-coverage-diff --replayDiffId=<id> [--screenshotName=<name>] [--globFilter=<glob>]
```

**Purpose:** Per-file JS coverage diff for a replay diff. Outputs a TSV table (`repoFilePath`, `status`, `baseRanges`, `headRanges`).

**MCP tool:** `get_replay_diff_js_coverage_diff`.

## agent sessions

```bash
meticulous agent sessions [options]
```

**Purpose:** List a project's most recently created sessions, newest first (default: 100). Useful for finding the ID of a session just recorded. Outputs a TSV table (`id`, `createdAt`, `recordedAt`, `recordedBy`, `status`, plus requested columns).

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `--project` | string | default project | One-off override (id, `org/proj`, or `proj`) |
| `--createdSince` / `--createdUntil` | string | — | ISO-8601 date/time bounds on creation time |
| `--recordedSince` / `--recordedUntil` | string | — | ISO-8601 date/time bounds on recording time |
| `--recordedBy` | string | — | Filter by the user who recorded the session |
| `--excludeSyntheticSessions` | boolean | `false` | Drop synthetic sessions (also drops the `status` column) |
| `--visitedUrlFilter` | string | — | Glob over visited URLs (only `*` is a wildcard), e.g. `*/checkout*` |
| `--includeStartUrl` / `--includeAbandonedReason` | boolean | `false` | Add extra columns |
| `--limit` | number | `100` | 1–1000 |
| `--offset` | number | `0` | — |

**MCP tool:** `get_sessions`.

---

## agent upload-build

```bash
meticulous agent upload-build --appDirectory=<path>     # static assets
meticulous agent upload-build --localImageTag=<tag>     # container image
```

**Purpose:** Upload a build and register a reusable deployment **without** triggering a run. Outputs the `deploymentId`. The commit defaults to the local git HEAD (a dirty working tree is captured as an ephemeral commit; untracked files are rejected). See the [`meticulous-test`](../../meticulous-test/SKILL.md) skill for the full workflow.

| Option | Type | Description |
|--------|------|-------------|
| `--appDirectory` | string | Build output directory (static-assets mode) |
| `--appZip` | string | Zipped build, as an alternative to `--appDirectory` |
| `--localImageTag` | string | Local Docker image tag (container mode) |
| `--containerPort` / `--containerEnv` / `--containerHealthCheckEndpoint` | — | Container runtime configuration |
| `--rewrites` | string | Static-asset rewrite rules |
| `--commitSha` | string | Override the commit the build is registered against |
| `--dryRun` | boolean | Print what would be uploaded without doing it |

**MCP tool:** no 1:1 equivalent — MCP splits this into `request_asset_upload`/`request_container_upload` (get an upload URL or registry credentials) → upload the zip/image yourself → `register_asset_build`/`register_container_build` (verify the upload and return `deploymentId`). The `commitSha` is never inferred on MCP — always pass your local commit explicitly (e.g. `git stash create` for a dirty tree, since untracked files are still excluded).

## agent trigger-test-run

```bash
meticulous agent trigger-test-run [--deploymentId=<id>] [--baseSha=<sha>] [options]
```

**Purpose:** Trigger a test run against a deployment from `agent upload-build`, comparing against a base. Outputs the `testRunId`. A base is required (auto-inferred from the repo, or set via `--baseSha`). Omit `--deploymentId` to reuse the most recent deployment for the local HEAD commit (requires a clean working tree). See the [`meticulous-test`](../../meticulous-test/SKILL.md) skill.

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `--deploymentId` | string | latest for HEAD | Deployment to run against |
| `--commitSha` | string | current git HEAD | Resolve the most recent deployment for this commit |
| `--baseSha` | string | inferred merge-base | Base commit to compare against |
| `--gitDiffOutput` | string | inferred | Explicit git diff, paired with `--baseSha` |
| `--sessionIds` | string | project golden set | Comma-separated session IDs to replay for both base and head |
| `--maxDurationSeconds` | number | — | Cap the run's duration |
| `--dontWaitForTestRunToComplete` | boolean | `false` | Return as soon as the run is triggered |
| `--dryRun` | boolean | `false` | Print what would be triggered without doing it |

**MCP tool:** `trigger_test_run`, taking a `deploymentId` from `register_asset_build`/`register_container_build`. Unlike the CLI, it **returns immediately without waiting** for the run to finish (poll `get_test_run_diffs_counts`/`get_test_run_diffs`), and `baseSha`/`gitDiffOutput` are never inferred — compute them locally (e.g. `git merge-base origin/main HEAD`) and pass them explicitly.

## agent submit-feedback

```bash
meticulous agent submit-feedback --message="<one or two sentences>" [options]
```

**Purpose:** Submit free-form feedback about Meticulous to the Meticulous team — e.g. whether it helped catch or debug a problem, what was confusing, or what information would have made your task easier. Outputs the `feedbackId`.

| Option | Type | Description |
|--------|------|-------------|
| `--message` | string | **Required.** The feedback itself: one or two sentences on whether Meticulous helped, what was missing or confusing, and what would have made the task easier |
| `--outcome` | string | `helped`, `neutral`, or `hindered` |
| `--testRunId` | string | The test run the feedback relates to, if any |
| `--skill` | string | The agentic skill or workflow being followed, e.g. `meticulous-review` |
| `--agentName` | string | The agent product submitting the feedback, e.g. `claude-code` |
| `--agentModel` | string | The underlying model, e.g. `claude-sonnet-5` |
| `--project` | string | One-off project override for this call |

**MCP tool:** `submit_feedback`, taking the same arguments.
