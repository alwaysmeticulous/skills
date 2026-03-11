---
name: meticulous-cli-project
description: Meticulous CLI project commands for inspecting the project configuration linked to an API token. Covers `meticulous project show`.
---

# meticulous project

Commands for inspecting the Meticulous project associated with an API token.

## project show

```bash
meticulous project show [--apiToken=<token>]
```

**Purpose:** Print the project configuration linked to the provided (or environment-configured) API token. Useful for confirming which organization and project an API token belongs to before running tests.

**Options:**

| Option | Type | Required | Description |
|--------|------|----------|-------------|
| `--apiToken` | string | no | Meticulous API token. Falls back to `METICULOUS_API_TOKEN` environment variable. |

**Output:** Logs the full project object, including organization name, project name, and associated configuration.

**Exit behaviour:** Exits with code 1 if the project cannot be retrieved (invalid or missing token).

**Example:**
```bash
meticulous project show --apiToken=$METICULOUS_API_TOKEN
# { name: 'my-app', organization: { name: 'acme-corp' }, ... }
```
