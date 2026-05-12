---
name: meticulous-cli-update
description: Check whether the Meticulous CLI (@alwaysmeticulous/cli) is installed and up to date, and install/update it if not. Run at the start of any Meticulous workflow, since the CLI is in active beta with frequent breaking changes.
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

## Step 3 — Update if outdated

If the installed version already matches the latest, you're done.

Otherwise, update according to how the CLI is installed:

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

## Step 4 — Update the installed Meticulous skills

The skills themselves are also under active development. Update them to the latest version:

```bash
npx skills update --project
```

## Step 5 — Verify

Re-run `meticulous --version` and confirm it matches the latest. Then proceed with the calling skill.
