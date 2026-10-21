---
name: meticulous-update
description: Check how Meticulous is connected (the hosted MCP server, the CLI, or both), that it's signed in and up to date, and that the Meticulous skills are current — installing or updating what's missing. Invoked at the start of every other Meticulous skill, since the CLI and skills are under active development with frequent changes and improvements.
user-invocable: false
---

# Check and update the Meticulous setup

Agents connect to Meticulous through the hosted [MCP server](https://app.meticulous.ai/api/mcp) or the `meticulous` CLI (`@alwaysmeticulous/cli`), which offer the same operations, and the skills (this document set) work with either. The CLI and the skills are under active development, so run this skill once at the start of any Meticulous workflow.

This only needs to run **once per conversation**. If you've already run it earlier in this conversation, skip it.

Full setup instructions — connecting the MCP server, installing the CLI, and installing these skills — live at [app.meticulous.ai/docs/agents/setup](https://app.meticulous.ai/docs/agents/setup).

## How to handle the install/update commands

This skill normally runs as a sub-step of another Meticulous skill. The install/update commands below are security-sensitive — they install packages and reach the network — so treat them as **best-effort and non-blocking**:

- You generally can't tell in advance whether a command is whitelisted. If it's whitelisted it runs silently; if not, attempting it surfaces a permission prompt. Either outcome is fine — let that be how you find out.
- **If a command needs permission you don't have** (a prompt appears, or the user declines it), treat that as the signal to _recommend rather than force_. Tell the user it's recommended to do XYZ — e.g. "It's recommended to update the Meticulous CLI by running `npm install --global @alwaysmeticulous/cli@latest`" — and move on. A declined prompt is **not** a failure; it just means "do it later."
- **If the user doesn't want to run it now, continue anyway.** Do not stop the workflow; carry on with the remaining steps and then return to the calling skill. The only hard requirement is a way to reach Meticulous at all (Step 1) — if neither the MCP server nor the CLI is available, no Meticulous skill can proceed, so stop there.

The read-only checks (`meticulous --version`, `npm view …`, `meticulous auth whoami`, and the MCP `whoami`/`get_project` tools) are safe to run directly.

## Step 1 — Find out how Meticulous is connected

- **MCP** — check whether Meticulous MCP tools (such as `whoami` and `get_test_run_diffs`) are available. They usually come from the Meticulous plugin, or from a connector the user or their organization added. If they are, prefer them.
- **CLI** — run `meticulous --version`. If that isn't found, try `npx @alwaysmeticulous/cli --version`, and if that works, call the CLI that way wherever a skill says `meticulous`.

Then:

- **MCP available** — the server is always on the latest version, so there's nothing to update: go to Step 3. Exception: a calling skill that runs commands only the CLI has (local replays and simulations, as in `meticulous-iterative-dev` and `meticulous-simulate-and-diff`) needs the CLI as well, so also follow the CLI path below for it.
- **CLI available** — continue with Step 2.
- **Neither** — suggest the user either add the MCP server (on Claude Code, Codex and Cursor, the Meticulous plugin connects it and adds these skills in one go) or install the CLI (`npm install --global @alwaysmeticulous/cli@latest`), ask which they'd prefer, and point them to the [setup docs](https://app.meticulous.ai/docs/agents/setup). Stop until one of them is available.

## Step 2 — Update the CLI if outdated

```bash
npm view @alwaysmeticulous/cli version
```

If the installed version already matches the latest, skip to Step 3. Otherwise, update according to how the CLI is installed (best-effort, see the note above):

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

## Step 3 — Check authentication and project selection

### Over MCP

Call the `whoami` tool. If the tools fail because the server isn't authenticated, the user has to sign in through their client — in Claude Code, type `/mcp`, select **meticulous** and choose **Authenticate**; in Codex, run `codex mcp login meticulous`; elsewhere, through the client's MCP settings. If Claude Code fails the sign-in with "Issuer mismatch in authorization response (RFC 9207)", it is reusing a stale cached authorization-server URL — tell the user to choose **Clear authentication** (not Re-authenticate) for the Meticulous server in `/mcp`, then authenticate again.

Then call `get_project` to see which project tool calls resolve to. If none is set, or it's clearly the wrong one for this repo, list the options with `list_projects` and pin one with `set_project` — ask the user which one to use unless it's unambiguous. A connection authenticated by a project API token is pinned to its project, so there's nothing to select.

### Over the CLI

```bash
meticulous auth whoami
```

Add `--json` if you'd rather branch on structured output — it prints `authenticatedVia` and `selectedProject`.

There are four outcomes:

#### (a) Signed in via OAuth, with a project selected

```
Authenticated via: OAuth
Logged in as: Jane Smith (jane@example.com)
Organizations: acme-corp (member)
Selected project: acme-corp/Web App
```

Nothing to do — continue to Step 4.

#### (b) Authenticated by an API token or injected credentials

```
Authenticated via: project API token (METICULOUS_API_TOKEN environment variable)
Pinned project: acme-corp/Web App
```

Also seen as `project API token (~/.meticulous/config.json)`, `test-run API token`, or `credentials injected at request time` (agent platforms that attach a bearer credential to outbound requests to `app.meticulous.ai`). These credentials are scoped to a single project, which is already pinned, so there is nothing to select — do **not** run `meticulous auth set-project`: with an API token it errors out (the token is bound to one project), and with injected credentials there is likewise nothing to choose. Continue to Step 4.

#### (c) "Not logged in"

The command exits with:

> Not logged in. Run `meticulous auth login`, or set METICULOUS_API_TOKEN. In terminals without a browser, use `meticulous auth login --non-interactive`.

Sign-in is browser SSO, so a human always has to complete it in a browser; which login command to use depends on where you're running:

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

Both forms skip the interactive project picker, so add `--project "Organization/Project"` when you already know which project to use — that logs in and pins the project in one step. Without `--project`, an account with access to several projects lands in case (d) below, so re-run `meticulous auth whoami` once login completes and handle whichever case it reports before continuing to Step 4.

In a CI-like environment with no human available at all, the alternative is an API token: set `METICULOUS_API_TOKEN` (or pass `--apiToken`) instead of logging in.

#### (d) Signed in via OAuth, but no default project

`whoami` succeeds and additionally logs (on stderr):

> No default project set. Run `meticulous auth set-project` to choose one.

Project-scoped commands can't resolve a project in this state. It happens for accounts with access to several projects — including right after a `--non-interactive` or `--device` login, which skips the picker. List the options, then pin one:

```bash
meticulous auth list-projects   # one "organization/project" slug per line
meticulous auth set-project --project "Organization/Project"
```

Ask the user which one to use unless it's unambiguous (e.g. only one project is listed, or the repo clearly corresponds to one of them). Alternatively, ask the user to run `meticulous auth set-project` themselves — without `--project` it shows an interactive picker.

The selection is saved to the account rather than the machine, so it also applies to the MCP server and to the user's other machines. `meticulous auth get-project` prints the currently resolved project on its own.

## Step 4 — Update the installed Meticulous skills

How to update the skills depends on where they came from — look at where this skill's own file lives:

- **From a plugin** — the plugin brings both the skills and the MCP server, and updates them together. `npx skills` won't touch these, and running it would install a second copy alongside, so don't.
  - **Claude Code** — the skills are namespaced as `meticulous:<skill-name>`. Run `claude plugin update meticulous@meticulous`, and tell the user the update takes effect after they restart Claude Code. Also recommend turning on auto-update (`/plugin` → **Marketplaces** → **meticulous** → **Enable auto-update**), so this isn't needed again. If `claude plugin list` reports the plugin's scope as `managed`, it's pinned by their organization: don't update it, just mention it.
  - **Codex** — run `codex plugin marketplace upgrade meticulous`; the update takes effect in the next Codex session.
  - **Cursor** — there's no command to run. If the user installed it, recommend they update it from Cursor's plugin settings; if their team installed it for everyone, the team's marketplace keeps it current.

- **Installed into the repository with `npx skills`** — skill files under `.claude/skills/`, `.cursor/skills/` or the equivalent for the agent, with a `skills-lock.json` at the repository root. Re-running the install command updates them and adds any new skills:

  ```bash
  npx skills add alwaysmeticulous/skills --skill "*" --agent claude-code --agent codex --agent cursor -y
  ```

  If `skills-lock.json` is committed (`git ls-files skills-lock.json` prints it), the skills are shared by everyone working in the repository, and updating them changes tracked files. Don't do that as a side effect of another task: recommend the user update them in a change of its own, unless the repository already updates them on a schedule (e.g. a GitHub workflow that runs the command above). If it isn't committed, they're the user's own copy — update them directly.

If the applicable command isn't whitelisted, recommend the user run it themselves. Either way — whether it ran, or the user declined — proceed with the calling skill.
