---
name: meticulous-review
description: Review a completed Meticulous test run — fetch the diff summary, inspect representative screenshots, DOM diffs, and timelines, then classify each visual change as intended or unintended. Use when you have a `testRunId`, a PR URL, or a PR number and need to assess what visually changed.
user-invocable: true
---

To review a Meticulous test run, follow the workflow below step by step, using the CLI commands as described.

## Prerequisites

- A `testRunId` for a completed Meticulous test run, **or** a PR URL / PR number. If you have neither, ask the user to provide one.

## Startup

Fire `meticulous-cli-update` as a **non-blocking background task** — do not wait for it before proceeding. It only needs to complete before you run any `meticulous agent` commands in Step 1. If it detects an actual update is needed, it will surface that before Step 1 runs.

Concurrently, proceed immediately to Step 0 (or Step 1 if a `testRunId` was provided directly).

## Step 0 — Resolve the testRunId from a PR

First, determine `{owner}`, `{repo}`, and `{pr_number}`. If only a bare PR number was given, resolve the repo:

```bash
gh repo view --json nameWithOwner --jq '.nameWithOwner'
```

Get the PR's head SHA:

```bash
gh api repos/{owner}/{repo}/pulls/{pr_number} --jq '.head.sha'
```

Then make both of these as **two separate parallel tool calls** (not shell `&`) — dispatch them in a single message and wait for both:

**A) Search issue comments** for a Meticulous bot comment:

```bash
gh api repos/{owner}/{repo}/issues/{pr_number}/comments | python3 -c "
import json, re, sys
for c in json.load(sys.stdin):
    if 'meticulous' in c.get('user',{}).get('login','').lower() or 'meticulous' in c.get('body','').lower():
        m = re.search(r'test-runs/([a-zA-Z0-9_-]+)', c.get('body',''))
        if m: print(m.group(1)); sys.exit(0)
sys.exit(1)
"
```

**B) Search check-runs** on the head commit for a Meticulous check with a `testRunId`:

```bash
gh api repos/{owner}/{repo}/commits/{head_sha}/check-runs | python3 -c "
import json, sys
for cr in json.load(sys.stdin).get('check_runs', []):
    if 'meticulous' in cr.get('name', '').lower() or 'meticulous' in (cr.get('app') or {}).get('name', '').lower():
        ext = cr.get('external_id', '')
        try:
            data = json.loads(ext)
            tid = (data.get('data') or {}).get('testRunId')
            if tid: print(tid); sys.exit(0)
            if (data.get('data') or {}).get('type') == 'not-yet-run':
                print('NOT_YET_RUN'); sys.exit(2)
        except Exception:
            pass
sys.exit(1)
"
```

**Resolution logic:**
- If either search finds a `testRunId`, use it and continue to Step 1.
- If the check-run search exits with code 2 (`NOT_YET_RUN`), stop and tell the user: "Meticulous has not run for this PR yet — no deployment has triggered it."
- If both exit with code 1, stop and tell the user: "No Meticulous test run found for this PR — the check may still be pending."

## Assess visual frontend changes

Get an overview of all diffs, then visually inspect representative screenshots to cover all changes. Follow these efficiency rules:

- **Batch image fetches in parallel** — after selecting all representatives from Step 1, dispatch all `image-files` calls in a single parallel batch, then read all diff images together. Do not fetch sequentially.
- **Start with the diff image only** — for each representative, read only the `diffImage` first. Only read `before`/`after` if the diff image alone is insufficient to understand the change.
- **Stop early once patterns are fully characterized** — once you have identified and verified all distinct diff patterns (same `domDiffIds` = same root cause), stop inspecting further representatives of already-understood patterns.
- **High-mismatch diffs** — a mismatch >50% typically indicates a large intentional UI change. Verify one representative but don't over-investigate unless something looks unexpected.

The `domDiffIds` from Step 1 tell you which screenshots share the same structural DOM change — pick one representative per unique ID. Use the DOM diff (Step 3) for structural detail, and the timeline (Step 4) only when a diff is unexpected and not explained by the DOM or images. The final report should cover all significant visual changes: each visual change deserves its own explanation.

### Step 1 -- Get the replay diff summary and select representatives

```
meticulous agent test-run-diffs --testRunId <testRunId>
```

**Output format:** TSV on stdout, metadata on stderr.

stdout columns:

```
replayDiffId	screenshotName	index	total	outcome	mismatch	domDiffIds
```

The output can be large (100s of rows for large test suites). **Do not read all rows** — pipe through this clustering script immediately to extract only the representatives you need:

```bash
meticulous agent test-run-diffs --testRunId <testRunId> 2>/dev/null | python3 -c "
import sys
seen_patterns = {}   # domDiffIds -> (replayDiffId, screenshotName, mismatch)
seen_screenshots = set()  # (replayDiffId, screenshotName) already claimed
for line in sys.stdin:
    parts = line.rstrip('\n').split('\t')
    if len(parts) < 7 or parts[4] != 'diff':
        continue
    replayDiffId, screenshotName, _, _, _, mismatch, domDiffIds = parts[:7]
    screenshot_key = (replayDiffId, screenshotName)
    if domDiffIds not in seen_patterns and screenshot_key not in seen_screenshots:
        seen_patterns[domDiffIds] = (replayDiffId, screenshotName, float(mismatch))
        seen_screenshots.add(screenshot_key)
for key, (rid, sname, m) in sorted(seen_patterns.items(), key=lambda x: -x[1][2])[:5]:
    print(f'{rid}\t{sname}\t{m:.5f}\t{key}')
sys.stderr.write(f'Total unique patterns: {len(seen_patterns)}\n')
"
```

stdout: up to 5 tab-separated representatives (replayDiffId, screenshotName, mismatch, domDiffIds), sorted by mismatch descending. Representatives are deduplicated on both `domDiffIds` pattern and `(replayDiffId, screenshotName)` pair — no duplicate image fetches.
stderr (informational, ignore): total unique pattern count. If the count exceeds 5, increase the `[:5]` cap in the script or re-run with a higher limit to avoid silently missing patterns.

- `mismatch` >0.5: likely a large intentional UI change — verify one representative, don't over-investigate
- `domDiffIds` = `none`: pixel-only diff — inspect image only, skip `dom-diff`
- `domDiffIds` = `n/a`/`error`: not applicable or failed

### Steps 2+3 -- Get images and DOM diffs in parallel

For each representative, dispatch `image-files` **and** `dom-diff` in the **same parallel batch** — they are independent. Send all calls for all representatives in a single message.

```
meticulous agent image-files --replayDiffId <replayDiffId> --screenshotName <screenshotName>
meticulous agent dom-diff --replayDiffId <replayDiffId> --screenshotName <screenshotName>
```

`image-files` downloads to `~/.meticulous/agent-images/` and prints local file paths (`before`, `after`, `diffImage`).

**Image reading strategy based on mismatch:**

- **mismatch > 0.90** — read `diffImage`, `before`, and `after` all at once. At this level the diff image is a near-solid red blob with no diagnostic value on its own; having before/after immediately avoids a second round-trip.
- **mismatch ≤ 0.90** — read `diffImage` only first. Only read `before`/`after` if the diff image alone is insufficient.

Skip `dom-diff` entirely if `domDiffIds` is `none`.

For `dom-diff`: output is unified diff format with `[diff N]` block headers. Pipe through `| head -100` for noisy diffs. Use `--context 0` for minimal context, `--context full` for complete file. Use `--index N` to view a single diff block by position.

Skip `dom-diff` entirely if `domDiffIds` is `none` (pixel-only diff).

Alternative for images: use `image-urls` instead of `image-files` to get URLs rather than downloading locally.

### Step 4 -- Get the replay timeline (optional, for diagnosing unexpected diffs)

If a diff is unexpected and the images/DOM don't make it obvious why it happened:

```
meticulous agent timeline-diff --replayDiffId <replayDiffId>
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
2. Use the timeline (Step 4) to check for failed network requests, redirects, or other anomalies that could explain diffs unrelated to your code.
3. If you can confidently explain the cause (e.g. a flaky timestamp, a non-deterministic element), note the explanation.
4. If you cannot explain or fix it, flag it to the user.

### Final report

Produce a concise summary covering all distinct visual change patterns. One explanation per unique pattern is sufficient — don't repeat the same explanation for multiple sessions sharing the same root cause.

1. **Intended changes**: what changed visually and why it's expected
2. **Unintended changes** (if any): `replayDiffId`/`screenshotName`, what it looks like, best assessment of cause
