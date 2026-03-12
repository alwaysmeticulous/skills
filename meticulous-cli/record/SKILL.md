---
name: meticulous-cli-record
description: Meticulous CLI recording commands for capturing user sessions by opening a browser with the recorder snippet injected. Covers `meticulous record session` and `meticulous record login`.
---

# meticulous record

Commands for recording user sessions. These commands open a Chromium browser with the Meticulous recorder snippet injected. When the browser is closed the session is uploaded to Meticulous.

## record session

```bash
meticulous record session [options]
```

**Purpose:** Open a browser, inject the Meticulous recorder, and upload the recorded session when the browser is closed. Use this for recording normal application flows.

Each detected session is logged with a link to view it in the Meticulous dashboard:
```
Recording session: https://app.meticulous.ai/projects/<org>/<project>/sessions/<id>
```

**Options:**

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `--apiToken` | string | — | Meticulous API token; required to identify the project |
| `--commitSha` | string | auto-detected | Git commit SHA to associate with the session |
| `--incognito` | boolean | `true` | Use an incognito browser context (prevents cookie/storage bleed between recordings) |
| `--devTools` | boolean | `false` | Open Chrome DevTools in the recording window |
| `--bypassCSP` | boolean | `false` | Bypass Content Security Policy (danger: requests may reach production backends) |
| `--width` | number | — | Browser viewport width in pixels |
| `--height` | number | — | Browser viewport height in pixels |
| `--uploadIntervalMs` | number | — | Interval between session data uploads in milliseconds |
| `--captureHttpOnlyCookies` | boolean | `true` | Capture HTTP-only cookies in addition to regular cookies |
| `--trace` | boolean | `false` | Enable verbose debug logging |
| `--maxPayloadSize` | number | — | Maximum session payload size in bytes |

---

## record login

```bash
meticulous record login [options]
```

**Purpose:** Record a login flow session. Identical to `record session` except:

1. `--bypassCSP` defaults to `true` — required so the recorder snippet can intercept auth-related network requests.
2. **Sessions recorded with this command store credentials.** The resulting session will contain usernames, passwords, and auth tokens captured during the flow. Handle these sessions with care and avoid using them in shared environments.

Use this command to record a login sequence that can later be used to seed application state (via `--sessionIdForApplicationStorage` in `simulate`) before running other sessions.

**Options:** Same as `record session`, except:

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `--bypassCSP` | boolean | **`true`** | Bypass CSP (default flipped to `true` for this command) |
| `--commitSha` | — | — | Not available on `record login` (commit SHA is not recorded) |
| `--incognito` | — | — | Not available on `record login` |

All other `record session` options apply.

---

## Shared Recording Options

Both commands share these options (from `COMMON_RECORD_OPTIONS`):

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `--apiToken` | string | — | Meticulous API token |
| `--devTools` | boolean | `false` | Open Chrome DevTools |
| `--bypassCSP` | boolean | `false` (session) / `true` (login) | Bypass Content Security Policy |
| `--width` | number | — | Viewport width |
| `--height` | number | — | Viewport height |
| `--uploadIntervalMs` | number | — | Upload interval in ms |
| `--captureHttpOnlyCookies` | boolean | `true` | Capture HTTP-only cookies |
| `--trace` | boolean | `false` | Verbose debug logging |
| `--maxPayloadSize` | number | — | Max payload size in bytes |
