---
name: pr-analysis
description: Analyze PR source code changes and correlate with screenshot diffs. Only use inside a workspace created by `npx @alwaysmeticulous/cli agent debug` — requires debug-data/pr-diff.txt to be present.
---

# PR Analysis

When `debug-data/pr-diff.txt` is present in the workspace, analyze the source code changes and correlate
them with the screenshot diffs.

1. Read `debug-data/pr-diff.txt` to understand what code changed
2. Read the diff summaries in `debug-data/diffs/*.summary.json` to see which screenshots differ
3. For each screenshot that differs, identify which code changes are most likely responsible

Provide a structured analysis:

- Which files were modified and what the key changes are
- Which code changes are most likely to affect visual output (CSS, layout, component rendering)
- For each differing screenshot, the most likely code change that caused it
- Whether the visual changes appear intentional (matching the code intent) or unintentional
