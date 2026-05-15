---
name: meticulous-cli-update
description: Check whether the Meticulous CLI (@alwaysmeticulous/cli) is installed and up to date, and install/update it if not. Invoked at the start of every other Meticulous skill, since the CLI is in active beta with frequent breaking changes.
user-invocable: false
---

# Install or update the Meticulous CLI

The `@alwaysmeticulous/cli` package is under active development and ships frequent breaking changes. Other Meticulous skills assume the `meticulous` command is on `PATH` and up to date, so run this skill once at the start of any Meticulous workflow.

## Step 1 — Check the installed version

```bash
meticulous --version
```

**If the command is not found**, the CLI is not installed. Install it globally:

```bash
npm install --global @alwaysmeticulous/cli@latest
```

Then re-run `meticulous --version` to confirm it's on `PATH`, and skip to Step 4.

## Step 2 — Check the latest published version

```bash
npm view @alwaysmeticulous/cli version
```

## Step 3 — Update the CLI if outdated

If the installed version already matches the latest, skip to Step 4.

Otherwise, update according to how the CLI is installed (requires network permission):

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

Re-run `meticulous --version` and confirm it matches the latest before proceeding.

## Step 4 — Check authentication and project selection

Verify the user is authenticated with Meticulous and has a project selected:

```bash
meticulous auth whoami
```

If the command reports that "No authentication found", stop and ask the user to run `meticulous auth whoami` themselves (it opens a browser to sign in). Do not attempt to run it on their behalf — it requires interactive sign-in.

If the command reports that "No project selected" (this happens when the user is a member of multiple projects), stop and ask the user to run `meticulous auth set-project` themselves to pick one. Do not attempt to run it on their behalf — it shows an interactive picker.

## Step 5 — Update the installed Meticulous skills

The skills themselves are also under active development. Update them to the latest version (requires network permission):

```bash
npx skills update --project
```

Then proceed with the calling skill.
