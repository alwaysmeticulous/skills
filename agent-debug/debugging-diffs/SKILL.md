---
name: debugging-diffs
description: Investigate unexpected visual differences between head and base replays. Only use inside a workspace created by `npx @alwaysmeticulous/cli agent debug` — requires the debug-data/ directory to be present.
---

# Debugging Screenshot Diffs

Use this guide when investigating unexpected visual differences between head and base replays.

## Investigation Steps

### 1. Understand the Diffs

- Read the replay diff JSON in `debug-data/diffs/<id>.json`.
- Check `screenshotDiffResults` for which screenshots differ.
- Note the diff percentage and pixel count for each screenshot.

### 2. Correlate with Code Changes

- Check `commitSha` in `context.json` to identify your code changes.
- If `debug-data/customer-repo/` is available, use `git log` and `git diff` to see what changed.
- Focus on CSS changes, component rendering logic, and layout modifications.

### 3. Compare Logs at Screenshot Time

- Find the screenshot timestamps in `timeline.json`.
- Compare what events occurred before each screenshot in head vs base.
- Look for missing or extra events that could cause visual differences.

### 4. Check for Expected vs Unexpected Diffs

- **Expected**: Code changes that intentionally modify the UI (new features, style updates).
- **Unexpected**: Same code producing different visual output, or unrelated areas changing.
- Check if the diff is in a dynamic content area (timestamps, counters, user-specific data).

### 5. Examine Snapshotted Assets

- If `debug-data/replays/{head,base}/<replayId>/snapshotted-assets/` exists, compare JS/CSS between head and base.
- Look for changes in CSS that could cause layout shifts.
- Check for new or modified JavaScript that affects rendering.

### 6. Review Screenshot Assertions Config

- Check `screenshotAssertionsOptions` in the diff JSON for threshold settings.
- Some diffs may be within acceptable tolerance but still flagged.
