---
name: meticulous-cli-auth
description: Meticulous CLI authentication commands. Use when logging in, checking who is authenticated, or logging out of the Meticulous CLI. Covers `meticulous auth whoami` and `meticulous auth logout`.
---

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

## auth logout

```bash
meticulous auth logout [--dryRun]
```

**Purpose:** Clear all stored OAuth tokens from disk, effectively logging out.

**Effects:**
- Deletes the cached OAuth token file used by the CLI
- Subsequent commands that require authentication will prompt for login again

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `--dryRun` | boolean | `false` | Print what would be cleared without actually deleting tokens |

**Example:**
```bash
meticulous auth logout
# Logged out successfully.

meticulous auth logout --dryRun
# Dry run: would clear stored OAuth tokens
```
