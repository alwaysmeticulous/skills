---
name: test-with-meticulous
description: Test and debug frontend changes using Meticulous. All frontend changes should be checked this way.
user_invocable: true
---

To test and debug frontend changes using Meticulous, follow the workflow below step by step, using the CLI commands as described.

## Step 1 -- Build, commit, push, and upload

If you are given a test run id, then skip this step.

1. If possible, find out what build artefact Meticulous expects by checking `.github/workflows/` for one of the following steps:
    - `uses: alwaysmeticulous/report-diffs-action/upload-assets@v1` - build assets
    - `uses: alwaysmeticulous/report-diffs-action/upload-container@v1` - docker image
2. Build the frontend.
3. Upload the build artefact and trigger a Meticulous test run:

```
# upload-assets
npx @alwaysmeticulous/cli upload-assets-and-execute-test-run-in-cloud --appDirectory <path-to-build> --repoDirectory <path-to-repo> --waitForTestRunToComplete

# upload-container
npx @alwaysmeticulous/cli ci upload-container --localImageTag <image-tag> --repoDirectory <path-to-repo> --waitForTestRunToComplete
```

`--appDirectory` must point to the build output directory (e.g., `dist/` subfolder), whereas `--localImageTag` must point to the local Docker image tag. The `--repoDirectory` must point to the root of the git repository (e.g., `.`). The `--waitForTestRunToComplete` flag is required.

The command will finish with status `Failure` if visual differences have been detected.

## Step 2 -- Get the replay diff summary for a test run

```
npx @alwaysmeticulous/cli agent test-run-diffs --testRunId <testRunId>
```

**Output format:** TSV on stdout, metadata on stderr.

stdout columns:

```
replayDiffId	screenshotName	index	total	outcome	mismatch	domDiffIds
```

Example output:

```
CqctwLpPC7	after-event-0	1	5	diff	0.00234	1;3
RRMGQft7PD	after-event-174	3	8	diff	0.01050	1;2
CLkCJ8WLrJ	after-event-8	2	4	diff	0.00100	none
Ct8HwmJNzM	end-state	5	5	flake	0.00010	n/a
```

Each row represents a screenshot where a visual pixel difference was detected between the base (before) and head (after) replay. Rows with `outcome=diff` are confirmed visual differences; other outcomes (`flake`, `error`, `warning`, `missing-base`, `missing-head`) are informational.

- `outcome`: `diff` (visual pixel difference), `flake`, `error`, `warning`, `missing-base`, `missing-head`
- `mismatch` (0-1, 5 decimal places) is the pixel mismatch fraction
- `domDiffIds` is a semicolon-separated ordered list of diff IDs, one per independent DOM change in the screenshot. Each ID groups structurally identical DOM changes across screenshots (same ID = same structural change). Example: `1;3` means two independent DOM changes with IDs 1 and 3. Special values: `none` means no DOM changes were found (either computed successfully with no differences, or a matching screenshot where no diff is expected) -- the visual difference is purely pixel-level (e.g. anti-aliasing, rendering differences), so you must inspect the screenshot images (Step 4) to understand the change. `n/a` means the DOM diff is not applicable or could not be computed (e.g. missing base or head screenshot, error/warning outcomes, or the metadata was unavailable).

stderr shows: total counts, unique diff counts, and timing breakdown. Proceed to Step 3 for rows with `outcome=diff`.

**Efficiency tip:** Use `domDiffIds` to quickly identify repeated changes. If many screenshots share the same ID (e.g. `1` appears in `1;3` and `1;2` and `1`), they share that DOM change -- you only need to investigate one representative per unique ID.

## Step 3 -- Inspect the DOM diff for a changed screenshot

To view a specific diff by index (corresponding to the position in the `domDiffIds` list from Step 2):

```
npx @alwaysmeticulous/cli agent dom-diff --replayDiffId <replayDiffId> --screenshotName <screenshotName> --index <0-based index>
```

To view all diffs for a screenshot (omit `--index`):

```
npx @alwaysmeticulous/cli agent dom-diff --replayDiffId <replayDiffId> --screenshotName <screenshotName>
```

Optional: pass `--context <N|full>` to control how many context lines surround each hunk (default 3). Use `--context 0` for no context, or `--context full` for a single unified diff with full file context (requires `--index` to be omitted).

**Output format:** Unified diff (`+`/`-` format) with leading indentation stripped.

With `--index`, outputs only the requested diff block. Without `--index`, outputs all diff blocks separated by `[diff 0]`, `[diff 1]`, etc. headers. Example:

```
[diff 0]
 <span class="text-zinc-400">#7687</span>
-<span class="min-w-0 flex-1 truncate transition-colors">Use divergence-aware comparison</span>
+<span class="min-w-0 flex-1 truncate transition-colors" data-tooltip-id=":r1h:">Use divergence-aware comparison</span>
[diff 1]
 <span class="inline-flex items-center rounded-lg bg-zinc-800">Temporal Workflow</span></a>
+<a href="/projects/Foo/Bar/test-runs/abc123"><span class="inline-flex items-center rounded-lg bg-zinc-800">Original: abc123</span></a>
```

The `--index` value maps to the position in the `domDiffIds` list from Step 2. For example, if `domDiffIds` is `1;3`, then `--index=0` shows the diff for ID 1 and `--index=1` shows the diff for ID 3.

Use this to determine **what exactly changed** in the DOM:

- Expected change (e.g. matching a code change you made) -> confirmed, continue.
- Unexpected change -> potential regression; investigate further with Step 4 or 5.

## Step 4 -- Get screenshot image URLs (optional, for visual confirmation)

If `domDiffIds` is `none` (no DOM diff detected), the DOM diff is ambiguous, or you want to visually confirm what changed:

```
npx @alwaysmeticulous/cli agent image-urls --replayDiffId <replayDiffId> --screenshotName <screenshotName>
```

**Output format:**

```
outcome: <outcome>
screenshot: <url>          # present for missing-base/missing-head
before: <url>              # present for diff/no-diff
after: <url>
diffImage: <url>
```

Open the `before`, `after`, and `diffImage` URLs in a browser to visually inspect the change.

## Step 5 -- Get the replay timeline (optional, for diagnosing unexpected diffs)

If a diff is unexpected and the DOM doesn't make it obvious why it happened:

```
npx @alwaysmeticulous/cli agent timeline-diff --replayDiffId <replayDiffId>
```

**Output format:** TSV on stdout, replay IDs on stderr.

stdout columns:

```
diff	timeMs	event	description
```

- `diff` column: ` ` (identical), `-` (removed), `+` (added), `!` (changed)
- `event` types: `user`, `screenshot`, `network`, `console`, `debug`, `urlChange`, `error`, `fatalError`, etc.
- `description`: concise one-line summary of the event

Look for anomalies such as failed network requests, unexpected redirects, or timing-related differences that could explain a visual change.

## Decision guide

For each unique diff ID, classify it as **expected** or **unexpected**:

- **Expected**: The DOM change directly corresponds to a code change you made. Confirm and move on.
- **Unexpected**: The change is not explained by your code changes. Flag it and investigate further.

For unexpected diffs:

1. Use the timeline (Step 5) to check for failed network requests, redirects, or other anomalies that could explain the diff.
2. If you can confidently explain the cause (e.g. a flaky timestamp, a non-deterministic element), note the explanation.
3. If the diff looks like a bug you introduced, attempt to fix it, then re-run the test.
4. If you cannot explain or fix it, flag it to the user.

## Final report

After investigating all diffs and attempting fixes for any fixable issues, produce a summary:

1. **Expected diffs**: List of diff IDs confirmed as intentional changes (brief description each).
2. **Unexpected diffs** (if any): For each, include:
   - The diff ID and a representative `replayDiffId` / `screenshotName`
   - What the diff shows (e.g. "new badge element added", "layout shift in header")
   - Your best assessment of the cause, if known (e.g. "likely caused by failed API request", "no explanation found")

## Workflow summary

Start with Step 2 to get an overview. Use `domDiffIds` to identify shared changes across screenshots. For each unique diff ID, pick a representative screenshot and inspect the specific diff with `--index` (Step 3). Use Steps 4-5 only when needed. Attempt to fix any unexpected diffs that appear to be regressions from your changes, then produce the final report.
