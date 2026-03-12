---
name: meticulous-simulate-and-diff
description: Run a Meticulous session simulation against a live URL and, if a base replay ID is provided, analyze any visual differences found — including pixel diffs and HTML diffs. Use when you need to check whether a code change has introduced visual regressions for a specific session.
---

# Simulate a session and analyze diffs

This skill covers running a single simulation and interpreting the results. For the `simulate` command's full option reference see the `meticulous-cli-simulate` skill.

## Prerequisites

- A `sessionId` to replay
- An `appUrl` (local dev server, or leave blank to use the original recorded URL)
- Optionally: a `baseReplayId` — the ID of a prior replay to diff screenshots against. Without this, screenshots are stored but not compared.

If you don't have a `baseReplayId`, you can find one from a downloaded test run:

```bash
meticulous download test-run --apiToken=$METICULOUS_API_TOKEN
# Then inspect ~/.meticulous/test-runs/<testRunId>/coverage.json
# or check the testCases[].replayId fields
```

## Step 1 — Run the simulation

```bash
meticulous simulate \
  --sessionId=<sessionId> \
  --appUrl=<url> \
  --baseReplayId=<baseReplayId> \
  --takeSnapshots
```

If you don't have a base replay to compare against, omit `--baseReplayId`. Screenshots will still be stored locally.

Capture the full stdout. Key things to look for:

```
# Per-screenshot diff outcomes (one line each):
0.412% pixel mismatch for screenshot screenshot-1234.png (threshold is 0.100%) => FAIL!
0.000% pixel mismatch for screenshot screenshot-5678.png (threshold is 0.100%) => PASS

# Final summary block:
=======
View simulation at: https://app.meticulous.ai/projects/<org>/<project>/simulations/<headReplayId>
View comparison with base: https://app.meticulous.ai/projects/<org>/<project>/simulations/<baseReplayId>/compare-to/<headReplayId>
=======
```

**If there are no `FAIL!` lines:** the session is visually identical to the base — stop here and report no regressions.

## Step 2 — Extract the head replay ID and locate the replay directory

From the `View simulation at:` URL, extract the `<headReplayId>` (the last path segment).

To find the local replay directory created by this run:

```bash
ls -lt ~/.meticulous/replays/ | head -5
```

The most recently created entry will be the head replay's directory (named with a timestamp, e.g. `2024-01-15T12-30-45.123Z-abc123/`). Note this path — it's referred to below as `<replayDir>`.

## Step 3 — Identify which screenshots diffed

```bash
ls ~/.meticulous/replays/<replayDir>/diffs/<baseReplayId>/
```

Each `.png` file here corresponds to a screenshot where a visual difference was detected. The pixel diff image highlights changed pixels in color. There are also `thumb_` prefixed thumbnail versions.

Note the filenames — they match the screenshot identifiers (e.g. `screenshot-after-event-42.png`).

## Step 4 — Analyze the HTML diff for each diffed screenshot

Each screenshot has a corresponding metadata file containing a full HTML snapshot of the page taken just before the screenshot was captured. These files are already on disk:

- **Head metadata:** `~/.meticulous/replays/<replayDir>/screenshots/<screenshotFilename>.metadata.json`
- **Base metadata:** `~/.meticulous/replays/<baseReplayId>/screenshots/<screenshotFilename>.metadata.json`

The base metadata is permanently cached when the simulation downloads the base replay, so no additional download is needed.

Read both `.metadata.json` files. The relevant field is `before.dom` — a full HTML string of the page state before the screenshot was taken.

Diff the two `before.dom` strings to understand what changed in the DOM. Focus on:
- Tags that were added or removed
- Attribute changes (especially `class`, `aria-*`, `data-*`, `style`)
- Text content changes

The `before.dom` also includes `before.routeData.url` which tells you which page/route this screenshot was taken on.

### Quick summary already computed

The simulation also computes a compact summary of the HTML change during diffing:

- `changedSectionsClassNames` — the CSS class names present in the DOM regions that changed (e.g. `["btn", "btn-primary", "text-red-500"]`). These are stored in the diff result JSON files under `<replayDir>/`.
- `mismatchFraction` — what proportion of pixels changed (visible in stdout per-screenshot lines)

If `changedSectionsClassNames` is empty but there is a pixel diff, this usually means the change is purely visual (e.g. a color or size change on an element with no class name) rather than a structural DOM change.

## Step 5 — Summarize the findings

The key output from this skill is a **high-level human-readable description of what visually changed and why**. Use the pixel diff counts, route URLs, changed class names, and HTML diffs gathered above to answer: *what did the user experience change, and which part of the UI is responsible?*

Present this in whatever format fits the current context (conversational answer, structured report, input to a calling workflow, etc.). Useful signals to draw on:

- Which routes were affected
- Which CSS classes appeared in the changed DOM regions (these usually map directly to components)
- Whether changes were structural (DOM additions/removals) or purely visual (pixel shift with no HTML diff)
- Whether the same change appears across multiple screenshots (suggesting a shared component changed) vs. isolated to one screenshot

The comparison URL logged to stdout is always worth surfacing, as it lets a human quickly verify the diff visually:
`https://app.meticulous.ai/.../simulations/<baseReplayId>/compare-to/<headReplayId>`

## Notes

- All files under `~/.meticulous/replays/<replayDir>/` persist after the run and can be freely re-read.
- The base replay screenshots and metadata are cached at `~/.meticulous/replays/<baseReplayId>/` and will not be re-downloaded on subsequent runs with the same base.
- The pixel diff images at `<replayDir>/diffs/<baseReplayId>/` can be opened directly and are useful for visually inspecting what changed.
- If `--baseReplayId` is omitted, no diff analysis is possible. Screenshots are still stored locally and can be compared manually by running again with `--baseReplayId` set to the new replay's ID from the first run.
