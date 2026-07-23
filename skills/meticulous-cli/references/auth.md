# meticulous auth

Authentication management for the Meticulous CLI. OAuth tokens are stored on disk and reused across sessions.

## auth whoami

```bash
meticulous auth whoami
```

**Purpose:** Display the currently authenticated user. If no valid token is stored, opens an OAuth browser login flow to authenticate.

**Output:** Logs name, email, admin status, and the list of organizations the user belongs to.

**Effects:**
- Reads the stored OAuth token
- If no token exists or it is expired (HTTP 403), prompts an interactive OAuth login
- Does not modify any stored state itself

**Example output:**
```
Logged in as: Jane Smith (jane@example.com)
Organizations:
  - acme-corp
```

**No options** beyond global flags.

---

## auth login

```bash
meticulous auth login
```

**Purpose:** Authenticate via browser SSO and store the resulting OAuth token on disk. If the account belongs to multiple projects, also prompts to pick a default project (saved to the account, so it's consistent across machines and available to the MCP server).

**Options:**

| Flag | Type | Default | Description |
| --- | --- | --- | --- |
| `--non-interactive` | boolean | `false` | Print the login URL and wait on a local callback server instead of opening a browser directly. Only works when a browser on the *same* machine can reach that callback — for a human at their own terminal, prefer the interactive `meticulous auth login` instead. |
| `--device` | boolean | `false` | Log in via the OAuth device flow: prints a URL and a short code that can be opened and confirmed from a browser on *any* device. Use this instead of `--non-interactive` when running on a remote or sandboxed machine (SSH session, container, cloud coding agent) where a browser on another device can't reach this machine's localhost. |
| `--project` | string | — | Organization/project to set as default, skipping the interactive picker (e.g. `"Organization/Project"`) |

**Effects:**
- Stores the OAuth token used by subsequent commands
- With `--project`, also sets the default project (see `auth set-project`)

---

## auth set-project

```bash
meticulous auth set-project
```

**Purpose:** Change the default project used by project-scoped commands (and the MCP server) without re-authenticating. Shows an interactive picker when run without `--project`.

**Options:**

| Flag | Type | Default | Description |
| --- | --- | --- | --- |
| `--project` | string | — | Organization/project to set as default, non-interactively (e.g. `"Organization/Project"`) |

**Effects:**
- Updates the default project saved to the account

---

## auth get-project

```bash
meticulous auth get-project
```

**Purpose:** Print the `organization/project` slug that project-scoped commands would currently resolve to (the account's default project, or the project pinned to an API token).

**No options** beyond global flags.

**Effects:**
- Read-only; exits non-zero if no default project is resolved

---

## auth list-projects

```bash
meticulous auth list-projects
```

**Purpose:** List the projects the authenticated account has access to. OAuth-only — if no token is stored and a TTY is present, this triggers an interactive OAuth login itself.

**Options:**

| Flag | Type | Default | Description |
| --- | --- | --- | --- |
| `--json` | boolean | `false` | Print `{id, name, organization: {name}}[]` instead of one `org/project` slug per line |

---

## auth logout

```bash
meticulous auth logout
```

**Purpose:** Clear all stored OAuth tokens from disk, effectively logging out.

**Effects:**
- Deletes the cached OAuth token file used by the CLI
- Subsequent commands that require authentication will prompt for login again

**Example:**
```bash
meticulous auth logout
# Logged out successfully.
```
