---
name: meticulous-cli-ci
description: Meticulous CLI CI/CD commands for running visual regression tests in cloud or locally, uploading build artifacts or Docker containers, preparing base test runs, and managing secure tunnels. Covers `meticulous ci run-with-tunnel`, `run-local`, `prepare`, `upload-assets`, `upload-container`, and `start-tunnel`.
---

# meticulous ci

Commands for running Meticulous visual regression tests in CI/CD pipelines.

## ci upload-assets

```bash
meticulous ci upload-assets --appDirectory=<path> [options]
# or
meticulous ci upload-assets --appZip=<path> [options]
```

**Purpose:** Upload static build artifacts (HTML/JS/CSS) to Meticulous and trigger a cloud test run against them. No tunnel required — Meticulous serves the assets itself.

**Required options (one of):**

| Option | Type | Description |
|--------|------|-------------|
| `--appDirectory` | string | Directory of static assets to upload |
| `--appZip` | string | Zip file of static assets to upload |

**Optional options:**

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `--apiToken` | string | — | Meticulous API token |
| `--commitSha` | string | auto-detected | Git commit SHA |
| `--baseSha` | string | auto-detected | Base commit SHA; auto-detected via merge-base with `origin/main` if `--repoDirectory` is set |
| `--repoDirectory` | string | — | Path to git repo for auto-detecting `--commitSha` and base SHA |
| `--rewrites` | string | `"[]"` | JSON array of URL rewrite rules (`[{"source":"**","destination":"/index.html"}]`). Defaults to a catch-all SPA rewrite if empty. |
| `--waitForBase` | boolean | `true` | Wait for a base test run to be created before triggering |
| `--waitForTestRunToComplete` | boolean | `false` | Block until the triggered test run finishes; implies `--waitForBase` |
| `--dryRun` | boolean | `false` | Print what would happen without uploading |

**Note:** If `--baseSha` equals `--commitSha`, the command exits early with "nothing to test".

---

## ci upload-container

```bash
meticulous ci upload-container --localImageTag=<tag> [options]
```

**Purpose:** Push a local Docker image to Meticulous' registry and trigger a cloud test run that spins up that container. Use when your app requires a server-side process (not just static assets).

**Required options:**

| Option | Type | Description |
|--------|------|-------------|
| `--localImageTag` | string | Local Docker image tag to upload (e.g., `myapp:latest` or image SHA) |

**Optional options:**

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `--apiToken` | string | — | Meticulous API token |
| `--commitSha` | string | auto-detected | Git commit SHA |
| `--baseSha` | string | auto-detected | Base commit SHA |
| `--repoDirectory` | string | — | Path to git repo for auto-detecting commit SHAs |
| `--containerPort` | number | — | Port to expose the container on |
| `--containerEnv` | array | — | Environment variables in `NAME=VALUE` format (repeatable) |
| `--waitForBase` | boolean | `true` | Wait for a base test run before triggering |
| `--waitForTestRunToComplete` | boolean | `false` | Block until test run finishes; implies `--waitForBase` |
| `--dryRun` | boolean | `false` | Print what would happen without pushing |

---

## ci run-with-tunnel

```bash
meticulous ci run-with-tunnel --appUrl=<url> [options]
```

**Purpose:** Open a secure tunnel from Meticulous' cloud infrastructure to a locally running app, dispatch a cloud test run against that tunnel, then poll until the run completes.

This is the primary CI command when you want Meticulous' cloud workers to execute replays but your app is running locally (e.g., in GitHub Actions).

**Required options:**

| Option | Type | Description |
|--------|------|-------------|
| `--apiToken` | string | Meticulous API token (or `METICULOUS_API_TOKEN` env var) |
| `--appUrl` | string | URL of the locally running app to test |

**Optional options:**

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `--commitSha` | string | auto-detected | Git commit SHA being tested |
| `--keepTunnelOpenSec` | number | `0` | Keep tunnel open N seconds after tests complete (useful for debugging) |
| `--secureTunnelHost` | string | — | Override the tunnel server host |
| `--allowInvalidCert` | boolean | `false` | Accept invalid TLS certificates from the local server |
| `--proxyAllUrls` | boolean | `false` | Proxy all URLs through tunnel, not just the app URL |
| `--rewriteHostnameToAppUrl` | boolean | `false` | Rewrite request hostnames to the app URL |
| `--http2Connections` | number | CPU count | Number of HTTP/2 connections for multiplexing |
| `--silenceTunnelWorker` | boolean | `false` | Suppress tunnel worker process logs |
| `--companionAssetsFolder` | string | — | Folder to serve companion assets from (must be paired with `--companionAssetsRegex`) |
| `--companionAssetsZip` | string | — | Zip file of companion assets (mutually exclusive with `--companionAssetsFolder`) |
| `--companionAssetsRegex` | string | — | Regex to match companion asset URLs |
| `--hadPreparedForTests` | boolean | `false` | Set to `true` if you ran `meticulous ci prepare` before this command |
| `--triggerScript` | string | — | Path to script that generates a base test run on the base commit if none exists; called with the base commit SHA |
| `--postComment` | boolean | `false` | Post a PR comment even if comments are disabled for the project |
| `--redactPassword` | boolean | `false` | Redact the tunnel password from log output |
| `--dryRun` | boolean | `false` | Print what would happen without starting the tunnel or test run |

**Base test run behaviour:** If `--hadPreparedForTests` or `--triggerScript` is set, the command waits up to 30 minutes for a base test run to become available before proceeding. This ensures diffs can be computed.

---

## ci run-local

```bash
meticulous ci run-local --appUrl=<url> [options]
```

**Purpose:** Run all test cases locally using Puppeteer (no cloud). Tests execute in parallel by default (two per CPU). Exits with code 1 if any test fails.

**Required options:**

| Option | Type | Description |
|--------|------|-------------|
| `--appUrl` | string | URL to test against |

**Optional options:**

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `--apiToken` | string | — | Meticulous API token |
| `--commitSha` | string | auto-detected | Git commit SHA |
| `--baseCommitSha` | string | — | Base commit to compare against for tests without an explicit `baseReplayId` |
| `--baseTestRunId` | string | — | Specific test run ID to compare visual snapshots against |
| `--githubSummary` | boolean | `false` | Write a GitHub Actions step summary |
| `--noParallelize` | boolean | `false` | Run tests sequentially instead of in parallel |
| `--parallelTasks` | number | `2 × CPU` | Maximum concurrent replay tasks |
| `--maxRetriesOnFailure` | number | `0` | Re-run visual-diff failures; mark as flake if the retry screenshot differs |
| `--rerunTestsNTimes` | number | `0` | Re-run all tests N extra times; mark as flake if any run differs |
| `--testsFile` | string | auto-search | Path to `meticulous.json` test list; auto-searches up directory tree if omitted |
| `--dryRun` | boolean | `false` | Print what would happen without running |

Also accepts all [replay execution options](#shared-replay-options) and [screenshot diff options](#shared-screenshot-diff-options).

---

## ci prepare

```bash
meticulous ci prepare --triggerScript=<path> [options]
```

**Purpose:** Ensure a base test run exists for the current PR before running tests. Checks if a base test run already exists for the merge-base commit. If not and the head commit is part of a PR, executes the `--triggerScript` with the base commit SHA to generate one.

This command is intended to be run before `ci run-with-tunnel` so the cloud workers have a base to compare against.

**Required options:**

| Option | Type | Description |
|--------|------|-------------|
| `--apiToken` | string | Meticulous API token |
| `--triggerScript` | string | Path to script that triggers a test run on a given commit (called with the commit SHA as an argument) |

**Optional options:**

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `--headCommit` | string | auto-detected | The head commit SHA to prepare for |
| `--dryRun` | boolean | `false` | Print what would happen without executing the trigger script |

**Note:** Only runs the trigger script if the commit is associated with a pull request. This prevents triggering an infinite chain of runs back through Git history. Sets `METICULOUS_DISABLE_RECURSIVE_TRIGGER=true` in the child process environment to prevent the trigger script from recursing.

---

## ci start-tunnel

```bash
meticulous ci start-tunnel --port=<port> [options]
```

**Purpose:** Low-level command that exposes a local HTTP service through Meticulous' secure tunnel infrastructure. The tunnel stays open until the process receives `SIGINT`. Intended for debugging and advanced use cases.

**Note:** This command is only visible in help output for Meticulous super-users (`IS_METICULOUS_SUPER_USER=true`).

**Required options:**

| Option | Alias | Type | Description |
|--------|-------|------|-------------|
| `--port` | `-p` | number | Local port to expose |
| `--apiToken` | — | string | Meticulous API token |

**Optional options:**

| Option | Alias | Type | Default | Description |
|--------|-------|------|---------|-------------|
| `--host` | `-h` | string | — | Upstream tunnel server host |
| `--subdomain` | `-s` | string | — | Request a specific subdomain |
| `--localHost` | `-l` | string | `localhost` | Tunnel traffic to this host instead of localhost |
| `--localHttps` | — | boolean | `false` | Tunnel to a local HTTPS server |
| `--localCert` | — | string | — | Path to TLS certificate PEM file |
| `--localKey` | — | string | — | Path to TLS certificate key file |
| `--localCa` | — | string | — | Path to CA file for self-signed certs |
| `--allowInvalidCert` | — | boolean | `false` | Skip TLS certificate verification |
| `--proxyAllUrls` | — | boolean | `false` | Proxy all URLs, not just the local host |
| `--rewriteHostnameToAppUrl` | — | boolean | `false` | Rewrite request hostnames to the app URL |
| `--enableDnsCache` | — | boolean | `false` | Cache DNS lookups (recommended for non-localhost targets) |
| `--printRequests` | — | boolean | `false` | Log each incoming request |
| `--http2Connections` | — | number | CPU count | Number of HTTP/2 connections |
| `--silenceTunnelWorker` | — | boolean | `false` | Suppress tunnel worker logs |
| `--dryRun` | — | boolean | `false` | Print what would happen without opening the tunnel |

---

## Shared Replay Options

`ci run-local` and `ci run-with-tunnel` (via cloud) share these browser execution options:

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `--headless` | boolean | `false` | Run browser in headless mode |
| `--devTools` | boolean | `false` | Open Chrome DevTools |
| `--bypassCSP` | boolean | `false` | Bypass Content Security Policy |
| `--shiftTime` | boolean | — | Shift clock to match recording time |
| `--networkStubbing` | boolean | `true` | Stub network requests using recorded responses |
| `--skipPauses` | boolean | `false` | Fast-forward through delays |
| `--moveBeforeMouseEvent` | boolean | — | Simulate mouse movement before click events |
| `--disableRemoteFonts` | boolean | `false` | Block remote font loads |
| `--noSandbox` | boolean | `false` | Pass `--no-sandbox` to Chromium |
| `--maxDurationMs` | number | none | Abort replay after this many milliseconds |
| `--maxEventCount` | number | none | Abort replay after this many events |
| `--essentialFeaturesOnly` | boolean | `false` | Disable non-essential features (e.g., video recording) to reduce noise |
| `--enableCssCoverage` | boolean | `false` | Collect CSS coverage data during replay |

## Shared Screenshot Diff Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `--diffThreshold` | number | `0` | Max acceptable proportion of changed pixels (0–1) |
| `--diffPixelThreshold` | number | `0.1` | Per-pixel color difference tolerance (0–1) |
| `--storyboard` | boolean | `false` | Capture a storyboard of screenshots throughout replay (requires `--takeSnapshots`) |
