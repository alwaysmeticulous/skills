---
name: meticulous-cli-update
description: Check whether the Meticulous CLI (@alwaysmeticulous/cli) is installed and up to date, and install/update it if not. Invoked at the start of every other Meticulous skill, since the CLI is in active beta with frequent breaking changes.
user-invocable: false
---

# Install or update the Meticulous CLI

The `@alwaysmeticulous/cli` package is under active development and ships frequent breaking changes. Other Meticulous skills assume the `meticulous` command is on `PATH` and up to date, so run this skill once at the start of any Meticulous workflow.

## Steps 1–3 — Check and update the CLI

First check the cache:

```bash
python3 -c "
import json, datetime, subprocess, sys, os
try:
    cache = json.load(open(os.path.expanduser('~/.meticulous/cli-update-check.json')))
    checked = datetime.datetime.fromisoformat(cache['checkedAt'])
    now = datetime.datetime.now(datetime.timezone.utc)
    if checked.tzinfo is None:
        checked = checked.replace(tzinfo=datetime.timezone.utc)
    age = (now - checked).total_seconds()
    installed = subprocess.check_output(['meticulous', '--version']).decode().strip()
    if age < 3600 and cache.get('installedVersion') == installed and cache.get('installedVersion') == cache.get('latestVersion'):
        print('CACHE_HIT:' + installed)
        sys.exit(0)
except Exception:
    pass
sys.exit(1)
"
```

If the output starts with `CACHE_HIT:`, skip to Step 4 — the CLI is up to date.

Otherwise run Steps 1 and 2 **in parallel**:

**Step 1 — Check installed version:**
```bash
meticulous --version
```

If not found, install globally: `npm install --global @alwaysmeticulous/cli@latest`, then skip to Step 4.

**Step 2 — Check latest published version:**
```bash
npm view @alwaysmeticulous/cli version
```

**Step 3 — Update if outdated:**

If versions match, write cache and proceed to Step 4. Otherwise update:

- **Globally installed**: `npm install --global @alwaysmeticulous/cli@latest`
- **Locally installed**: `npm install --save-dev @alwaysmeticulous/cli@latest` (or pnpm/yarn equivalent)

Write the cache after any successful version check:

```bash
python3 -c "
import json, datetime, subprocess, os
v = subprocess.check_output(['meticulous', '--version']).decode().strip()
latest = subprocess.check_output(['npm', 'view', '@alwaysmeticulous/cli', 'version']).decode().strip()
os.makedirs(os.path.expanduser('~/.meticulous'), exist_ok=True)
with open(os.path.expanduser('~/.meticulous/cli-update-check.json'), 'w') as f:
    json.dump({'installedVersion': v, 'latestVersion': latest, 'checkedAt': datetime.datetime.now(datetime.timezone.utc).isoformat()}, f)
"
```

## Step 4 — Check authentication and project selection

This skill accepts an optional `repo` parameter (e.g. `acryldata/datahub-fork`) from the calling skill. Use it in Step 4b if provided.

### Step 4a — Check authentication

```bash
meticulous auth whoami
```

If the command succeeds, note the `Selected project:` line for Step 4b.

If the command reports "No authentication found", stop and tell the user to run `meticulous auth whoami` in their terminal (not via the agent) — it will open a browser to sign in. Then retry.

### Step 4b — Check project selection (only if `repo` was provided)

First check if the cache already has a verified project for this repo:

```bash
python3 -c "
import json, os, sys
try:
    cache = json.load(open(os.path.expanduser('~/.meticulous/cli-update-check.json')))
    mapping = cache.get('repoProjects', {})
    project = mapping.get('$REPO')
    if project:
        print(project); sys.exit(0)
except Exception:
    pass
sys.exit(1)
"
```

If a cached project is found, use it and skip to Step 5.

Otherwise run:

```bash
meticulous project show --rawJson '{}'
```

This returns the current project's linked GitHub repo (`repositoryData.owner`/`repositoryData.name`). Compare it to the `repo` passed by the calling skill.

If they match, cache the mapping and continue to Step 5:

```bash
python3 -c "
import json, os
path = os.path.expanduser('~/.meticulous/cli-update-check.json')
cache = json.load(open(path)) if os.path.exists(path) else {}
cache.setdefault('repoProjects', {})['$REPO'] = '$CURRENT_PROJECT'
json.dump(cache, open(path, 'w'))
"
```

If they don't match, fetch all available projects:

```bash
meticulous auth list-projects
```

If there are exactly 2 projects, auto-select the one that isn't currently selected. If more than 2, ask the user. Then switch:

```bash
meticulous auth set-project --project "<chosen project>"
```

Cache the new mapping using the same snippet above, then continue.

If no `repo` was provided, skip this step and continue.

## Step 5 — Update the installed Meticulous skills

Skip this step entirely — skill updates should be done manually by the user via `npx skills update --project` when they want to pull the latest versions. Running it automatically in the background on every invocation introduces supply-chain risk (unversioned `npx` pull) and can overwrite local skill customizations without warning.
