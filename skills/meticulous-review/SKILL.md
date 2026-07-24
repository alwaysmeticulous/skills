---
name: meticulous-review
description: Analyze a completed Meticulous test run — fetch the diff summary, inspect representative screenshots, DOM diffs, and timelines. Resolves the test run from the local repo's current commit (the default), or from an explicit test-run ID or commit SHA. Use when asked to review Meticulous test results, or while reviewing or babysitting a pull/merge request to assess and fix a failing Meticulous Tests CI check.
user-invocable: true
---

To review a Meticulous test run, follow the workflow below step by step, using the CLI commands as described.

> Before starting, run the `meticulous-cli-update` skill to ensure the Meticulous CLI is up to date — unless it has already run earlier in this conversation, in which case skip it.

Every `meticulous agent …` command in this workflow has an equivalent tool on the hosted [Meticulous MCP server](https://app.meticulous.ai/api/mcp), named `get_…` (e.g. `agent test-run-diffs` → `get_test_run_diffs`). Each tool returns the same data as the CLI command's `--json` output. The steps below use the CLI commands and note the matching MCP tool.

## Assess visual frontend changes

Get an overview of the diffs, then visually inspect each one. By default the summary returns a pre-selected representative set — one screenshot per unique structural DOM change — so every returned row is worth inspecting. **The rows are returned in priority order (most representative / most significant first), so work through them top to bottom.** For each, always look at the screenshot images first (Step 2) — the diff image is the most informative way to understand what actually changed. Use the DOM diff (Step 3) for additional structural detail, and the timeline (Step 4) only when a diff is unexpected and not explained by the DOM or images.

**To conclude a PR is good, every returned diff must be checked and confirmed** — each one classified as either intended or explained (see the Decision guide). A PR is only safe to approve once there are no unexplained or unintended changes left. The final report should cover all significant visual changes: each deserves its own explanation.

### Step 1 -- Get the replay diff summary

Run from the local checkout to resolve the test run from the current commit's git HEAD — the common case when reviewing or babysitting a pull/merge request whose branch is checked out locally. First make sure HEAD matches the remote head CI ran on (e.g. `git pull`), or you may review a stale or missing run:

```
meticulous agent test-run-diffs
```

_MCP tool: `get_test_run_diffs`._

All non-result output goes to stderr (stdout carries only the diff table); pass `--verbose` to see the resolved commit and `testRunId`. To target a run explicitly instead, pass one of:

- `--testRunId <id>` — a 20+ character alphanumeric string (e.g. `aB3xK9LmN7QrStUvWxYz12`).
- `--commitSha <sha>` — the latest test run for that commit is used. For a pull/merge request, resolve its head commit SHA (e.g. via the hosting platform's CLI or API) and pass it here.

If the resolved run is still in progress, the command blocks until it finishes and then shows the diffs (waiting is the default); pass `--dontWaitForTestRunToComplete` to instead report the in-progress run and exit immediately. If no run is found for the commit, the run hasn't been triggered yet — wait and re-run, or ask the user.

**Output format:** TSV on stdout, metadata on stderr.

The output covers **visual differences only** — matching screenshots, known flakes, and screenshots downstream of a divergence are not included. By default it is further limited to the **selected** screenshots — a representative subset with one screenshot per unique structural DOM change. Rows are returned in **priority order** (most representative / most significant first) — inspect them in that order.

stdout columns:

```
replayDiffId	screenshotName	index	outcome	mismatchFraction
```

`index` is the row's global rank (the priority order above); with `--orderByReplayDiffs` it instead ranks rows grouped by replay diff.

Example output:

```
CqctwLpPC7	after-event-0	1	diff	0.00234
RRMGQft7PD	after-event-174	2	diff	0.01050
CLkCJ8WLrJ	after-event-8	4	diff	0.00100
```

Each row represents a screenshot compared between the base (before) and head (after) replay, and is a confirmed **visual difference** — every row has `outcome=diff`. Rows are in priority order.

- `outcome` is always `diff` — a visual pixel difference between base and head. Non-difference screenshots (matches, added/removed screenshots, screenshots downstream of a divergence) are not part of this summary; to look at one, pass its `screenshotName` to `agent image-files` / `agent dom-diff` (Steps 2-3), discovering the available names via `agent timeline-diff`.
- `mismatchFraction` (0-1, 5 decimal places) is the **pixel mismatch fraction** — the fraction of pixels that differ between the base and head screenshots (`0` = identical, higher = more of the image changed). A large `mismatchFraction` is a quick hint that a change is substantial, but always inspect the diff image regardless, since even a tiny fraction can be a meaningful change.

stderr shows: total counts, unique diff counts, and timing breakdown. Every returned row must be accounted for in the review — inspect each (Steps 2-3), and confirm it's intended or explained, before concluding the PR is good.

**Optional flags** (to widen the output beyond the default selected subset):

- `--includeDomDiffIds` — add a `domDiffIds` column: a semicolon-separated ordered list of diff IDs, one per independent DOM change in the screenshot. Each ID groups structurally identical DOM changes across screenshots (same ID = same structural change). Example: `1;3` means two independent DOM changes with IDs 1 and 3. Special values: `none` means no DOM changes were found (the visual difference is purely pixel-level, e.g. anti-aliasing — inspect the screenshot images to understand it); `error` means the DOM diff was attempted but failed (e.g. metadata unavailable). Combine with `--includeAllDiffs` to see how the selected subset covers the full set of unique diff IDs.
- `--includeAllDiffs` — return every difference rather than just the selected representative subset. Adds an `isSelected` column (`true`/`false`) marking which rows are in the selected subset.
- `--orderByReplayDiffs` — order rows grouped by replay diff (each session's screenshots read as a flow) instead of by global priority; `index` then ranks rows within that grouping. The output stays a flat row list (one row per screenshot) in both TSV and `--json` — this only changes the ordering and the `index` values.

### Step 2 -- Get screenshot images

For each representative screenshot:

```
meticulous agent image-files --replayDiffId <replayDiffId> --screenshotName <screenshotName>
```

This downloads the screenshot images to `~/.meticulous/agent-images/` and prints the local file paths.

_MCP tool: `get_image_urls` returns the same outcome and signed image URLs, fetch the URLs to view the images._

**Output format:**

```
outcome:	<outcome>
before:	<path>              # the base image; omitted when outcome is missing-base
after:	<path>               # the head image; omitted when outcome is missing-head
diffImage:	<path>           # present only for a diff outcome
```

Open the `before`, `after`, and `diffImage` files to visually inspect the change. The `diffImage` is usually the most informative — it highlights exactly which pixels changed. Always inspect the images to understand the actual visual impact of a change, even when the DOM diff is clear.

Alternative: use `image-urls` instead of `image-files` to get URLs to the images rather than downloading them locally.

### Step 3 -- Inspect the DOM diff (for structural detail)

```
meticulous agent dom-diff --replayDiffId <replayDiffId> --screenshotName <screenshotName>
```

_MCP tool: `get_dom_diff`._

Optional: pass `--context <N|full>` to control how many context lines surround each hunk (default 3). Use `--context 0` for no context, or `--context full` for a single unified diff with full file context.

**Output format:** Unified diff (`+`/`-` format) with leading indentation stripped. All diff blocks are separated by `[diff 0]`, `[diff 1]`, etc. headers. Example:

```
[diff 0]
 <span class="text-zinc-400">#7687</span>
-<span class="min-w-0 flex-1 truncate transition-colors">Use divergence-aware comparison</span>
+<span class="min-w-0 flex-1 truncate transition-colors" data-tooltip-id=":r1h:">Use divergence-aware comparison</span>
[diff 1]
 <span class="inline-flex items-center rounded-lg bg-zinc-800">Temporal Workflow</span></a>
+<a href="/projects/Foo/Bar/test-runs/abc123"><span class="inline-flex items-center rounded-lg bg-zinc-800">Original: abc123</span></a>
```

### Step 4 -- Get the replay timeline (optional, for diagnosing unexpected diffs)

If a diff is unexpected and the images/DOM don't make it obvious why it happened:

```
meticulous agent timeline-diff --replayDiffId <replayDiffId>
```

_MCP tool: `get_timeline_diff`._

**Output format:** TSV on stdout.

stdout columns:

```
diff	timeMs	event	description
```

- `diff` column: ` ` (identical), `-` (removed), `+` (added), `!` (changed)
- `event` types: `user`, `screenshot`, `network`, `console`, `debug`, `urlChange`, `error`, `fatalError`, etc.
- `description`: concise one-line summary of the event

Look for anomalies such as failed network requests, unexpected redirects, or timing-related differences that could explain a visual change.

### Decision guide

For each representative screenshot, classify the visual change as **intended** or **unintended** based on the diff image and DOM diff:

- **Intended**: The visual change is a desired outcome of the task you're working on. Confirm and move on.
- **Unintended**: The change was not a goal of the task. This includes both changes that are clearly unrelated to your code, and — often more importantly — side effects of your code changes that weren't meant to happen. A change being _explainable_ by your code does not make it _intended_; if the task didn't call for that visual change, it's unintended.

For unintended changes:

1. If the change is a side effect of your code, attempt to fix it so the code achieves the intended result without the unwanted visual change, then re-run the test.
2. Use the timeline (Step 4) to check for failed network requests, redirects, or other anomalies that could explain diffs unrelated to your code.
3. If you can confidently explain the cause (e.g. a flaky timestamp, a non-deterministic element), note the explanation.
4. If you cannot explain or fix it, flag it to the user.

### Final report

After investigating all diffs and attempting fixes for any fixable issues, produce a summary that covers **all significant visual changes**. The number of explanation points should be at least as many as the number of representative diffs you inspected — each visual change deserves its own explanation.

1. **Intended changes**: For each distinct visual change that is a desired outcome of the task, describe what changed visually (based on the diff image) and why it's intended.
2. **Unintended changes** (if any): For each, include:
   - A representative `replayDiffId` / `screenshotName`
   - What the visual change looks like (e.g. "new badge element added", "layout shift in header")
   - Whether it's a side effect of your code or unrelated, and your best assessment of the cause

The PR is only good when **every** returned diff has been checked and accounted for — each one either confirmed intended or explained. If any diff remains unintended or unexplained, the PR is not yet good: fix it, or surface it clearly to the user.

### Report feedback to Meticulous

As the last step, after delivering the final report, submit one brief feedback note to the Meticulous team: did Meticulous catch a real problem, was anything confusing or misleading, and what information would have made the review easier?

```bash
meticulous agent submit-feedback --message="<one or two sentences>" --outcome=<helped|neutral|hindered> --testRunId=<id> --skill=meticulous-review
```

_MCP tool: `submit_feedback`._
