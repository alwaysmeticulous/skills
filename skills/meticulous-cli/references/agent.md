# meticulous agent

Read, analysis, and run-triggering commands designed for AI coding agents. They resolve git context (commit SHA, base, diff) automatically from the local repository, and default to machine-readable output.

The read/analysis commands are also exposed as tools on the hosted **MCP server** (`https://app.meticulous.ai/api/mcp`) — an MCP-enabled client can call the `get_…` tool directly instead of shelling out. Each tool takes broadly the same arguments and returns the same data as the CLI command's `--json` output, with minor differences inherent to a hosted endpoint (no git-inferred `commitSha`, no output-format flags, and — for `image-files` — URLs rather than files on disk). The **MCP tool** column below gives the mapping; write commands (`upload-build`, `trigger-test-run`) have no MCP equivalent.

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
| `js-coverage --replayId` | Per-file JS coverage for a replay | `get_replay_js_coverage` |
| `js-coverage-diff` | Per-file JS coverage diff for a replay diff | `get_replay_diff_js_coverage_diff` |
| `upload-build` | Upload a build, register a deployment | *(none — write op)* |
| `trigger-test-run` | Trigger a run against a deployment | *(none — write op)* |

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
meticulous agent js-coverage [--testRunId=<id> | --commitSha=<sha> | --replayId=<id>] [options]
```

**Purpose:** Per-file JavaScript coverage for a whole test run, a single replay, or a combined set of runs. Outputs a TSV table keyed on `repoFilePath` plus the requested columns.

| Option | Type | Description |
|--------|------|-------------|
| `--testRunId` / `--commitSha` | string | Coverage for a test run (defaults to the current git HEAD) |
| `--replayId` | string | Coverage for a single replay |
| `--screenshotName` | string | Restrict to a single screenshot of the replay |
| `--headPlusTestRunIds` / `--testRunIds` | string | Comma-separated run IDs to union coverage across (same project + commit) |
| `--globFilter` | string | Only include files matching the glob |
| `--includeAllFiles` | boolean | Include files with no coverage too |
| `--prDiffOnly` | boolean | Restrict to files changed in the PR (test-run queries only) |
| `--includeExecutableRanges` / `--includeUncoveredRanges` / `--includeCoveragePercentage` | boolean | Add richer per-file coverage columns |

**MCP tools:** `get_test_run_js_coverage` (with `--testRunId`); `get_replay_js_coverage` (with `--replayId`).

## agent js-coverage-diff

```bash
meticulous agent js-coverage-diff --replayDiffId=<id> [--screenshotName=<name>] [--globFilter=<glob>]
```

**Purpose:** Per-file JS coverage diff for a replay diff. Outputs a TSV table (`repoFilePath`, `status`, `baseRanges`, `headRanges`).

**MCP tool:** `get_replay_diff_js_coverage_diff`.

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

**MCP tool:** none (write operation).

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

**MCP tool:** none (write operation).
