---
name: meticulous-cli-update
description: Check whether the Meticulous CLI (@alwaysmeticulous/cli) is installed and up to date, and install/update it if not. Invoked at the start of every other Meticulous skill, since the CLI is under active development with frequent changes and improvements.
user-invocable: false
---

# Install or update the Meticulous CLI

The `@alwaysmeticulous/cli` package is under active development and ships frequent changes and improvements. Other Meticulous skills assume the `meticulous` command is on `PATH` and up to date, so run this skill once at the start of any Meticulous workflow.

This only needs to run **once per conversation**. If you've already run it earlier in this conversation, skip it — there's no need to re-check the version or re-update on every Meticulous skill invocation.

## How to handle the install/update commands

This skill normally runs as a sub-step of another Meticulous skill. The install/update commands below (Steps 1, 3, and 5) are security-sensitive — they install packages and reach the network — so treat them as **best-effort and non-blocking**:

- You generally can't tell in advance whether a command is whitelisted. If it's whitelisted it runs silently; if not, attempting it surfaces a permission prompt. Either outcome is fine — let that be how you find out.
- **If a command needs permission you don't have** (a prompt appears, or the user declines it), treat that as the signal to *recommend rather than force*. Tell the user it's recommended to do XYZ — e.g. "It's recommended to update the Meticulous CLI by running `npm install --global @alwaysmeticulous/cli@latest`" — and move on. A declined prompt is **not** a failure; it just means "do it later."
- **If the user doesn't want to run it now, continue anyway.** Do not stop the workflow; carry on with the remaining steps and then return to the calling skill. The only hard requirement is that the `meticulous` command exists at all (Step 1) — if it's genuinely not installed and the user declines to install it, no Meticulous skill can proceed, so stop there.

The read-only checks (`meticulous --version`, `npm view …`, `meticulous auth whoami`) are safe to run directly.

## Step 1 — Check the installed version

```bash
meticulous --version
```

**If the command is not found**, the CLI is not installed. Install it globally (best-effort, see the note above):

```bash
npm install --global @alwaysmeticulous/cli@latest
```

If installing isn't whitelisted, recommend the user run that command themselves. Because no Meticulous skill can run without the CLI, this is the one case where you should stop and wait if the user declines — there's nothing to continue with. Once installed, re-run `meticulous --version` to confirm it's on `PATH`, and skip to Step 4.

## Step 2 — Check the latest published version

```bash
npm view @alwaysmeticulous/cli version
```

## Step 3 — Update the CLI if outdated

If the installed version already matches the latest, skip to Step 4.

Otherwise, update according to how the CLI is installed (best-effort, see the note above — if updating isn't whitelisted, recommend the user run the appropriate command and continue regardless):

- **Globally installed** (typical — `which meticulous` resolves to a path outside the current project):
  ```bash
  npm install --global @alwaysmeticulous/cli@latest
  ```

- **Locally installed in the project** (`@alwaysmeticulous/cli` appears in the project's `package.json` and `which meticulous` resolves inside `node_modules/.bin`):
  ```bash
  npm install --save-dev @alwaysmeticulous/cli@latest
  # or, if the project uses pnpm:
  pnpm add --save-dev @alwaysmeticulous/cli@latest
  # or yarn:
  yarn add --dev @alwaysmeticulous/cli@latest
  ```

If the update ran, re-run `meticulous --version` and confirm it matches the latest before proceeding.

## Step 4 — Check authentication and project selection

Verify the user is authenticated with Meticulous and has a project selected:

```bash
meticulous auth whoami
```

**If it reports "No authentication found"**, sign-in is needed. Sign-in is browser SSO, so a human always has to complete it in a browser; which login command to use depends on where you're running:

- **On the user's own machine** (a browser there can reach this machine's localhost):

  ```bash
  meticulous auth login --non-interactive
  ```

  This prints a login URL and then waits on a local callback server, so run it in the background, then surface the printed URL to the user and ask them to open it and complete sign-in. Once they do, the command finishes and stores the token, and you can continue.

  Alternatively, ask the user to run `meticulous auth login` themselves — at their own terminal that opens the browser directly.

- **On a remote or sandboxed machine** (cloud agent, SSH session, container) where a browser on another device can't reach this machine's localhost:

  ```bash
  meticulous auth login --device
  ```

  This uses the OAuth device flow: it prints a URL and a short code instead of waiting on a local callback. Run it in the background, then surface the URL and code to the user and ask them to open the URL on any device and enter the code. Once confirmed, the command finishes and stores the token.

**If it reports "No project selected"** (this happens when the user belongs to multiple projects — including right after a non-interactive login, which skips the project picker), a project must be chosen before project-scoped commands work. Ask the user which organization/project to use (if you don't already know), then pin it non-interactively:

```bash
meticulous auth set-project --project "Organization/Project"
```

Alternatively, ask the user to run `meticulous auth set-project` themselves (it shows an interactive picker).

## Step 5 — Update the installed Meticulous skills

The skills themselves are also under active development. Update them to the latest version (best-effort, see the note above):

```bash
npx skills update --project
```

If this isn't whitelisted, recommend the user run it themselves. Either way — whether it ran, or the user declined — proceed with the calling skill.
