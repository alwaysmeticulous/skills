---
name: meticulous-test
description: Run a Meticulous test run after implementing a frontend change, then hand off to the `meticulous-review` skill to classify each visual change as intended or unintended. Uploads the build once — the same build can be re-triggered against different bases without rebuilding — and it works with uncommitted changes. Use when implementing a feature autonomously end-to-end before creating a PR.
user-invocable: true
---

To test a frontend change using Meticulous, follow the workflow below step by step, using the CLI commands as described.

> Before starting, run the `meticulous-cli-update` skill to ensure the Meticulous CLI is up to date — unless it has already run earlier in this conversation, in which case skip it.

If you are already given a test run id, skip to Step 4.

## Step 1 -- Build the frontend

1. Find out what build artefact Meticulous expects by checking `.github/workflows/` for one of the following steps:
    - `uses: alwaysmeticulous/report-diffs-action/upload-assets@v1` — build assets
    - `uses: alwaysmeticulous/report-diffs-action/upload-container@v1` — docker image
2. Build the frontend following the same instructions as used in the GitHub workflow.

## Step 2 -- Upload the build

Register the build as a reusable deployment with `agent upload-build`. This uploads the artefact and prints a `deploymentId` to stdout — it does **not** trigger a run yet.

```
# assets
meticulous agent upload-build --appDirectory <path-to-build>

# container
meticulous agent upload-build --localImageTag <image-tag>
```

- `--appDirectory` points to the build output directory (e.g. a `dist/` subfolder); `--localImageTag` is the local Docker image tag. The build mode is auto-detected.
- The build's commit defaults to the local git HEAD. If the working tree is dirty, it is captured as an **ephemeral commit** (printed as `commitSha (local, ephemeral due to dirty working tree): …`) — see the uncommitted-changes note below.
- Untracked files are rejected (they can't be captured) — `git add` them first.
- Capture the `deploymentId` from stdout (pass `--verbose` to also see progress on stderr).

_MCP alternative: there's no 1:1 `upload-build` tool — instead call `request_asset_upload` (or `request_container_upload` for a container), PUT the zip / push the image yourself, then `register_asset_build` / `register_container_build` with the returned `uploadId` to get `deploymentId`. Unlike the CLI, MCP never infers `commitSha` — pass it explicitly (`git stash create` gives a commit SHA for a dirty tree without touching HEAD, but still excludes untracked files)._

## Step 3 -- Trigger a test run

Trigger a run for the deployment, comparing against a base. Run it from the repo directory to infer both the base (merge-base with the origin default branch) and the git diff automatically:

```
meticulous agent trigger-test-run --deploymentId <deploymentId>
```

- A base is **required**. It's auto-inferred from the current directory, or pass `--baseSha <sha>` (and optionally `--gitDiffOutput`) to set it explicitly.
- Omit `--deploymentId` to use the most recent deployment already uploaded for the local HEAD commit instead — this requires a clean working tree (no uncommitted changes).
- The command **blocks until the run finishes** by default and prints the `testRunId` to stdout; the final status is `Failure` when visual differences were detected (a normal completed verdict, not an error). Pass `--dontWaitForTestRunToComplete` to return as soon as the run is triggered.
- **One build, many bases:** the same `deploymentId` can be re-triggered against different bases — just run `agent trigger-test-run` again with a different `--baseSha`. No rebuild or re-upload needed.

Note the `testRunId` from the output.

_MCP alternative: `trigger_test_run`, passing the `deploymentId` from Step 2. Two differences from the CLI: it **never infers `baseSha`/`gitDiffOutput`** — compute them locally (e.g. `git merge-base origin/main HEAD`) and pass them explicitly — and it **always returns immediately** without waiting for the run to finish, regardless of `--dontWaitForTestRunToComplete`; poll `get_test_run_diffs_counts` or `get_test_run_diffs` to know when it's done._

## Step 4 -- Review the visual changes

Follow the `meticulous-review` skill, passing the `testRunId` from Step 3. It fetches the diff summary, inspects representative screenshots / DOM diffs / timelines, and produces a final report classifying each visual change as intended or unintended.

> **Uncommitted changes:** if you built/tested with a dirty working tree, the run is recorded against an ephemeral commit that is not your HEAD (and isn't pushed). Commands that resolve a run from the local checkout — `meticulous-review` / `agent test-run-diffs` run with no `--testRunId` — won't find it by commit, so **always pass the explicit `--testRunId`** to the review step in that case.
