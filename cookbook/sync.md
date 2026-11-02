# Sync: Check Installed Items for Updates

## Context
Compare every locally installed skill, agent, prompt, and MCP server against its source and report which ones are outdated. `sync` is **read-only**: it never changes installed files or the catalog. Apply changes with `/agentic-library update`.

## Steps

### 1. Check the Catalog
Make sure the local catalog exists (see *The Catalog File* in `SKILL.md`). No `git pull` is needed — `library.yaml` is local to this machine:
```bash
cd <LIBRARY_SKILL_DIR>
[ -f library.yaml ] || { cp library.example.yaml library.yaml && chmod 444 library.yaml; }
```

### 2. Read the Catalog
- Read `library.yaml`
- Parse all entries from `agentic-library.skills`, `agentic-library.agents`, `agentic-library.prompts`, and `agentic-library.mcp`

### 3. Find All Installed Items
For each entry in the catalog:
- Determine the type (skill, agent, prompt, mcp) and corresponding directories from `default_dirs`
- Check if a directory or file matching the entry name exists in the **project** directory
- Check if a directory or file matching the entry name exists in the **global** directory
- Search recursively for name matches
- Collect every entry that is installed locally (either project or global), remembering the scope
- If nothing is installed, tell the user and exit

### 4. Compare Against Upstream
For each installed entry, read `<target_directory>/.<name>.agentic-metadata.json` and decide a status:

**GitHub source**:
- Parse `org`, `repo`, `branch` from the source URL (browser or raw pattern, see `SKILL.md`)
- Ask the remote for the newest commit without cloning:
  ```bash
  git ls-remote https://github.com/<org>/<repo>.git refs/heads/<branch>
  # SSH fallback: git@github.com:<org>/<repo>.git
  ```
- Compare the first column with the `commit` saved in the metadata:
  - Equal → `up to date`
  - Different → `outdated`
  - No metadata file or no `commit` (installed before metadata existed) → `unknown`
  - `git ls-remote` fails → `failed: <reason>`

**Local path source**:
- Compare the source file's modified time with the timestamp in the metadata → `up to date` or `outdated`; missing metadata → `unknown`. Show these as type `local`.

### 5. Report Results
Display a summary table. Do not modify anything.

```
## Sync Check

| Type | Name | Scope | Status |
|------|------|-------|--------|
| skill | grill-me | global | outdated |
| skill | adhd | global | up to date |
| agent | agent-name | project | unknown (no metadata) |
| mcp | mcp-name | global | failed: network error |

Up to date: X   Outdated: Y   Unknown: Z   Failed: W
```

End with next steps:
- `outdated` → `/agentic-library update <name>` (or `/agentic-library update --all`)
- `unknown` → `/agentic-library update <name>` also works; it re-pulls and records the metadata so later checks are accurate
