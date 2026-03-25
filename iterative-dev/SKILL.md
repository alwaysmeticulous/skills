---
name: iterative-frontend-dev
description: Iterative frontend development loop using Meticulous for per-step visual validation. Use when implementing a multi-step frontend change and want to catch visual regressions and unintended side effects at each step, before the final cloud test run.
user-invocable: true
---

# Iterative Frontend Development with Meticulous

Use this skill when implementing a multi-step frontend change. After each step, run a quick local visual check using Meticulous to catch regressions and unintended side effects early. After all steps are complete, run a full cloud test run to validate across all recorded sessions.

## Prerequisites

- Local dev server running (e.g. `npm run dev` or `pnpm dev`), serving the app at a known URL such as `http://localhost:3000`
- API token configured: `~/.meticulous/config.json` or `METICULOUS_API_TOKEN` environment variable
- Meticulous CLI available: `npx @alwaysmeticulous/cli` or `meticulous` if installed globally

---

## Per-step loop

Repeat the following for each step of your change.

### Step 1 — Implement the step's changes

Make your code changes for this step.

### Step 2 — Find relevant sessions

Run:

```bash
meticulous local relevant-sessions
```

If you have already committed previous steps and want to find sessions relevant only to this step's uncommitted changes, pass the SHA of the last commit:

```bash
meticulous local relevant-sessions --startingPointSha=<sha-of-last-commit>
```

For full option reference see the `meticulous-cli-local` skill. The key fields to extract from each session in the output:

- **Session ID** — pass as `--sessionId` when simulating
- **Base replay ID** — the replay of this session on the base branch; pass as `--baseReplayId` to diff against. May be absent if the session has never been replayed on the base branch.
- **Relevance** — `IsRelevant` / `IsRelevantBeta` means the session directly exercises changed code.

If no sessions are returned, the changed code is not covered by any recorded session. Proceed to Step 5 (commit) and rely on the final cloud run for coverage.

### Step 3 — Simulate 1–2 relevant sessions locally

Pick the most relevant session(s) — typically the one(s) with `IsRelevant` relevance that best describe the changed functionality. For full option reference see the `meticulous-cli-simulate` skill.

For each selected session, run:

```bash
meticulous simulate \
  --sessionId=<sessionId> \
  --appUrl=http://localhost:<port> \
  --baseReplayId=<baseReplayId> \
  --headless
```

`--headless` is required (agents cannot operate a visible browser).

If `baseReplayId` was not provided by `local relevant-sessions`, omit `--baseReplayId`. Screenshots will still be stored locally for visual inspection.

Note the head replay ID from the `View simulation at:` URL in the output (the last path segment) — you will need it in Step 4 if diffs are unexpected.

### Step 4 — Analyse the visual output

Follow the `meticulous-simulate-and-diff` skill to locate the screenshots and diff images and understand what changed.

Classify each visual difference as:

- **Expected** — a direct, intended consequence of this step's changes. Proceed.
- **Unexpected** — a visual change that was not a goal of this step (including unintended side effects of your code, even if explainable by it). Investigate.

Also check sessions that cover flows you didn't explicitly intend to change — `local relevant-sessions` may surface these, and they're worth a quick look for unintended side effects.

**If a diff is unexpected and the cause is clear:** fix the code and re-run the simulation from Step 3.

**If the cause is unclear:** create a self-contained AI-readable debug workspace (see the `meticulous-cli-debug` skill for full details):

```bash
meticulous debug replays <headReplayId> <baseReplayId>
```

Both IDs come from the simulation output: the head replay ID is the last path segment of the `View simulation at:` URL, and the base replay ID is the `--baseReplayId` you passed in Step 3. Open the generated workspace to diagnose the root cause, then fix the code and re-simulate.

### Step 5 — Commit

Once the step's visual output is correct, commit your changes:

```bash
git add -p
git commit -m "<concise description of this step>"
```

Committing after each step means the next iteration's `--startingPointSha` call computes the diff relative to this checkpoint, preventing prior steps' changes from inflating the set of relevant sessions.

---

## After all steps — full cloud test run

Once all steps are complete and committed, run a full cloud test run to validate across all recorded sessions (not just the 1–2 you simulated locally):

> Follow the `test-with-meticulous` skill.

The cloud run compares your branch against the base branch across the full golden set of sessions and reports any visual regressions.
