---
name: using-meticulous
description: Guide to using Meticulous for visual regression testing of frontend changes. Use when making UI changes that need visual verification, running Meticulous tests, or investigating visual diffs.
---

# Using Meticulous

Meticulous records user sessions and replays them to catch visual regressions. These skills cover end-to-end workflows for testing frontend changes and investigating diffs.

## Available Skills

| Skill | When to use |
|-------|-------------|
| [test-with-meticulous](test-with-meticulous/SKILL.md) | Full testing workflow: build and upload artifacts, trigger a cloud test run, inspect DOM and pixel diffs, diagnose unexpected changes, and produce a final report. **Start here for any frontend change that needs visual regression testing.** |
| [simulate-and-diff](simulate-and-diff/SKILL.md) | Run a single session simulation against a live URL and analyze the resulting visual diffs (pixel + HTML). Use when you already have a `sessionId` and want to debug one specific session locally rather than running the full test suite. |

## Quick Decision Guide

- **"I changed frontend code and want to check for regressions"** → [test-with-meticulous](test-with-meticulous/SKILL.md)
- **"I have a specific session and want to replay it locally against my dev server"** → [simulate-and-diff](simulate-and-diff/SKILL.md)

## CLI Reference

These skills build on the `meticulous-cli` command reference skills. For full option details on any CLI command, see the corresponding `meticulous-cli-*` skill (auth, ci, download, project, record, simulate, schema).
