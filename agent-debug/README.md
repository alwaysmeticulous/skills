# `agent debug` Skills

Skills for AI-assisted investigation inside workspaces created by `npx @alwaysmeticulous/cli agent debug`.

`agent debug` creates a local workspace containing replay data, session recordings, screenshot diffs, and a worktree of your own repository, then launches an AI coding session to investigate visual regression issues.

These skills are automatically placed in `.claude/skills/` in every workspace the command creates.

## Skills

| Skill | Description |
| --- | --- |
| [`debugging-diffs`](debugging-diffs/SKILL.md) | Investigate unexpected screenshot diffs between head and base replays |
| [`debugging-flakes`](debugging-flakes/SKILL.md) | Investigate flaky or non-deterministic replay behavior |
| [`debugging-network`](debugging-network/SKILL.md) | Investigate network-related replay failures and divergences |
| [`debugging-timelines`](debugging-timelines/SKILL.md) | Investigate timeline divergence between head and base replays |
| [`pr-analysis`](pr-analysis/SKILL.md) | Analyze PR source changes and correlate with screenshot diffs |
