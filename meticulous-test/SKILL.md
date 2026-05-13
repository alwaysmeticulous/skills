---
name: meticulous-test
description: Run after implementing any frontend change to verify its visual impact. Triggers a Meticulous test run, then hands off to the `meticulous-review` skill to classify each visual change as intended or unintended. Use this before marking a frontend task as complete.
user_invocable: true
---

To test a frontend change using Meticulous, follow the workflow below step by step, using the CLI commands as described.

> Before starting, run the `meticulous-cli-update` skill to ensure the Meticulous CLI is up to date.

## Step 1 -- Trigger Meticulous test run

If you are already given a test run id, then skip this step.

1. If possible, find out what build artefact Meticulous expects by checking `.github/workflows/` for one of the following steps:
    - `uses: alwaysmeticulous/report-diffs-action/upload-assets@v1` - build assets
    - `uses: alwaysmeticulous/report-diffs-action/upload-container@v1` - docker image
2. Build the frontend following the same instructions as used in the GitHub workflow.
3. Upload the build artefact and trigger a Meticulous test run:

```
# upload-assets
meticulous ci upload-assets --waitForTestRunToComplete --repoDirectory <path-to-repo> --appDirectory <path-to-build>

# upload-container
meticulous ci upload-container --waitForTestRunToComplete --repoDirectory <path-to-repo> --localImageTag <image-tag>
```

The `--repoDirectory` must point to the root of the git repository (e.g., `.`). The `--waitForTestRunToComplete` flag is required. `--appDirectory` must point to the build output directory (e.g., `dist/` subfolder), whereas `--localImageTag` must point to the local Docker image tag. 

The command will finish with status `Failure` if visual differences have been detected. Note the `testRunId` from the output.

## Step 2 -- Review the visual changes

Follow the `meticulous-review` skill, passing the `testRunId` from Step 1. It fetches the diff summary, inspects representative screenshots / DOM diffs / timelines, and produces a final report classifying each visual change as intended or unintended.
