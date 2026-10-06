---
name: meticulous-agent-swarm-review
description: Read a pull request's Agent swarm results, triage its failed and blocked test cases, and fix the regressions they found. Resolves the run from the local repo's current commit (the default), or from a pull request number, test run ID or Agent swarm run ID. Use when asked to check, review or fix Agent swarm results, or after launching a run with `meticulous ci agent-test`.
user-invocable: true
---

Agent swarm is a Meticulous-hosted agent that explores a pull request's build and reports a list of test cases, each `pass`, `fail`, `blocked` or `skipped`. Follow these steps using the CLI or MCP commands shown. See the `meticulous-cli` skill's [agent reference](../meticulous-cli/references/agent.md#agent-swarm-runs--swarm-run--swarm-case) for every option.

> Before starting, run the `meticulous-cli-update` skill to ensure the Meticulous CLI and skills are up to date — unless it has already run earlier in this conversation, in which case skip it.

## Step 1 -- Find the run and wait for it

```bash
# CLI (infers the commit from local git HEAD; --wait blocks until the run finishes)
meticulous agent swarm-run --wait --outcomes fail blocked

# MCP (git context is never inferred, and the tool does not wait)
get_agent_swarm_run(commitSha="<sha>", outcomes=["fail", "blocked"])
```

Make sure HEAD matches the commit the run tested (e.g. `git pull`), or pass `--prNumber` to get the pull request's latest run across all its commits. Over MCP, call again until `run.phase` is `succeeded` or `unsuccessful`; while it is `pending` or `running`, poll no more than every 30 seconds.

- **`unsuccessful`**: the run itself failed, timed out or was cancelled. Report `run.errorMessage` rather than triaging cases.
- **`notTestable`** is set: the agent decided the change can't be tested in a browser. Report its `reason` and stop.
- **`supersededByAgenticRunId`** is set: a newer run exists for this target. Look at that one instead, unless you were asked about this run.
- Read `takeaways` first: they are the agent's own summary of what matters.

## Step 2 -- Triage each failed and blocked case

For each case returned:

```bash
meticulous agent swarm-case --agenticRunId=<id> --caseIndex=<n> --json
get_agent_swarm_case(agenticRunId="<id>", caseIndex=<n>)
```

- **`fail`** has already been upheld by the failure checker, which ruled out agent mistakes. Treat it as a likely regression. `check` has its `rootCause`, suggested `fix`, `howToVerify` and `citations`.
- **`blocked` with `blockedBy: "application"`** means the app stopped the flow, e.g. a crash or a missing control. Treat it like a failure.
- **`blocked` with `blockedBy: "environment"`** means the test setup stopped it, e.g. a backend outage or failed login. Report it; don't change the code for it.
- Use `steps` (with `outcome` and `reason`) to see where the flow broke. `comparisons` say whether the base commit behaved the same way. A case where base and head both fail is probably not caused by this pull request.
- If `checkWithheld` is `"source_code_access"`, the caller can't read the project's code, so there is no check or fix prompt. Diagnose from the steps and your local checkout instead.

## Step 3 -- Fix and re-run

When a case has a `fixPrompt` (`meticulous agent swarm-case ... --fixPrompt` prints just the prompt), use it as the starting point, but **check every cited file and line against your branch first**: the checker read the commit it tested, which may be older than your checkout. Fix the root cause, not the symptom the agent hit.

Then commit, push, and launch a new run the way the project usually does (often a CI workflow running `meticulous ci agent-test`), and repeat from Step 1 on the new commit.

Summarise for the user: the run's link (`run.url`), each failure you fixed or left alone and why, and any environment problems they need to resolve.
