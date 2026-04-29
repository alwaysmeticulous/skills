---
name: test-with-meticulous
description: Run after implementing any frontend change to verify its visual impact. Triggers a Meticulous test run, then inspects screenshot diffs to classify each visual change as intended or unintended. Use this before marking a frontend task as complete.
user_invocable: true
---

To test and debug frontend changes using Meticulous, follow the workflow below step by step, using the CLI commands as described.

## Step 1 -- Trigger Meticulous test run

If you are already given a test run id, then skip this step.

1. If possible, find out what build artefact Meticulous expects by checking `.github/workflows/` for one of the following steps:
    - `uses: alwaysmeticulous/report-diffs-action/upload-assets@v1` - build assets
    - `uses: alwaysmeticulous/report-diffs-action/upload-container@v1` - docker image
2. Build the frontend.
3. Upload the build artefact and trigger a Meticulous test run:

```
# upload-assets
npx @alwaysmeticulous/cli ci upload-assets --appDirectory <path-to-build> --repoDirectory <path-to-repo> --waitForTestRunToComplete

# upload-container
npx @alwaysmeticulous/cli ci upload-container --localImageTag <image-tag> --repoDirectory <path-to-repo> --waitForTestRunToComplete
```

`--appDirectory` must point to the build output directory (e.g., `dist/` subfolder), whereas `--localImageTag` must point to the local Docker image tag. The `--repoDirectory` must point to the root of the git repository (e.g., `.`). The `--waitForTestRunToComplete` flag is required.

The command will finish with status `Failure` if visual differences have been detected.

## Step 2 -- Assess visual frontend changes

Get an overview of all diffs, then visually inspect representative screenshots to cover all changes. The `domDiffIds` from Step 2a tell you which screenshots share the same structural DOM change — pick one representative per unique diff ID to efficiently cover all changes. For each representative, always look at the screenshot images first (Step 2b) — the diff image is the most informative way to understand what actually changed. Use the DOM diff (Step 2c) for additional structural detail, and the timeline (Step 2d) only when a diff is unexpected and not explained by the DOM or images. The final report should cover all significant visual changes: each visual change deserves its own explanation.

### Step 2a -- Get the replay diff summary

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
Ct8HwmJNzM	end-state	5	5	flake	0.00010	error
Ab3xKLmN9Q	after-event-12	3	6	missing-base	0.00000	n/a
```

Each row represents a screenshot where a visual pixel difference was detected between the base (before) and head (after) replay. Rows with `outcome=diff` are confirmed visual differences; other outcomes (`flake`, `error`, `warning`, `missing-base`, `missing-head`) are informational.

- `outcome`: `diff` (visual pixel difference), `flake`, `error`, `warning`, `missing-base`, `missing-head`
- `mismatch` (0-1, 5 decimal places) is the pixel mismatch fraction
- `domDiffIds` is a semicolon-separated ordered list of diff IDs, one per independent DOM change in the screenshot. Each ID groups structurally identical DOM changes across screenshots (same ID = same structural change). Example: `1;3` means two independent DOM changes with IDs 1 and 3. Special values: `none` means no DOM changes were found (either computed successfully with no differences, or a matching screenshot where no diff is expected) -- the visual difference is purely pixel-level (e.g. anti-aliasing, rendering differences), so you must inspect the screenshot images to understand the change. `n/a` means the DOM diff is not applicable (e.g. error/warning outcomes where no comparison is possible). `error` means the DOM diff was attempted but failed (e.g. metadata unavailable or could not be retrieved).

stderr shows: total counts, unique diff counts, and timing breakdown. Proceed to Steps 2b-2c for rows with `outcome=diff`.

Use `domDiffIds` to identify which subset of diffs to inspect. Screenshots sharing the same ID contain the same structural DOM change — pick one representative per unique ID for efficient coverage.

### Step 2b -- Get screenshot images

For each representative screenshot:

```
npx @alwaysmeticulous/cli agent image-files --replayDiffId <replayDiffId> --screenshotName <screenshotName>
```

This downloads the screenshot images to `~/.meticulous/agent-images/` and prints the local file paths.

**Output format:**

```
outcome: <outcome>
screenshot: <path>          # present for missing-base/missing-head
before: <path>              # present for diff/no-diff
after: <path>
diffImage: <path>
```

Open the `before`, `after`, and `diffImage` files to visually inspect the change. The `diffImage` is usually the most informative — it highlights exactly which pixels changed. Always inspect the images to understand the actual visual impact of a change, even when the DOM diff is clear.

Alternative: use `image-urls` instead of `image-files` to get URLs to the images rather than downloading them locally.

### Step 2c -- Inspect the DOM diff (for structural detail)

```
npx @alwaysmeticulous/cli agent dom-diff --replayDiffId <replayDiffId> --screenshotName <screenshotName>
```

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

To view a single diff block, add `--index <0-based index>` (maps to the position in the `domDiffIds` list from Step 2a).

### Step 2d -- Get the replay timeline (optional, for diagnosing unexpected diffs)

If a diff is unexpected and the images/DOM don't make it obvious why it happened:

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

### Decision guide

For each representative screenshot, classify the visual change as **intended** or **unintended** based on the diff image and DOM diff:

- **Intended**: The visual change is a desired outcome of the task you're working on. Confirm and move on.
- **Unintended**: The change was not a goal of the task. This includes both changes that are clearly unrelated to your code, and — often more importantly — side effects of your code changes that weren't meant to happen. A change being _explainable_ by your code does not make it _intended_; if the task didn't call for that visual change, it's unintended.

For unintended changes:

1. If the change is a side effect of your code, attempt to fix it so the code achieves the intended result without the unwanted visual change, then re-run the test.
2. Use the timeline (Step 2d) to check for failed network requests, redirects, or other anomalies that could explain diffs unrelated to your code.
3. If you can confidently explain the cause (e.g. a flaky timestamp, a non-deterministic element), note the explanation.
4. If you cannot explain or fix it, flag it to the user.

### Final report

After investigating all diffs and attempting fixes for any fixable issues, produce a summary that covers **all significant visual changes**. The number of explanation points should be at least as many as the number of unique domDiffIds you inspected — each visual change deserves its own explanation.

1. **Intended changes**: For each distinct visual change that is a desired outcome of the task, describe what changed visually (based on the diff image) and why it's intended.
2. **Unintended changes** (if any): For each, include:
   - A representative `replayDiffId` / `screenshotName`
   - What the visual change looks like (e.g. "new badge element added", "layout shift in header")
   - Whether it's a side effect of your code or unrelated, and your best assessment of the cause
