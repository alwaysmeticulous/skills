# meticulous project

Commands for inspecting the Meticulous project you're authenticated against.

## project show

```bash
meticulous project show [--apiToken=<token>]
```

**Purpose:** Print the project configuration for your current credentials. Useful for confirming which organization and project you're authenticated against before running tests.

**Options:**

| Option | Type | Required | Description |
|--------|------|----------|-------------|
| `--apiToken` | string | no | Meticulous API token; falls back to the `METICULOUS_API_TOKEN` env var, or prompts OAuth login if neither is set. |

**Output:** Logs the full project object, including organization name, project name, and associated configuration.

**Exit behaviour:** Exits with code 1 if the project cannot be retrieved (invalid or missing credentials).

**Example:**
```bash
meticulous project show
# { name: 'my-app', organization: { name: 'acme-corp' }, ... }
```
