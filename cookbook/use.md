# Use an Item from the Library

## Context
Pull a skill, agent, prompt, or MCP server from the catalog into the local environment. If already installed locally, overwrite with the latest from the source (refresh).

## Input
The user provides a skill name or description, optionally followed by a scope flag:
```bash
/agentic-library use <name>             # no flag → global install (default)
/agentic-library use <name> --global    # or -g → global install
/agentic-library use <name> --project   # or -p → project install
```

## Steps

### 1. Check the Catalog
Make sure the local catalog exists (see *The Catalog File* in `SKILL.md`). No `git pull` is needed — `library.yaml` is local to this machine:
```bash
cd <LIBRARY_SKILL_DIR>
[ -f library.yaml ] || { cp library.example.yaml library.yaml && chmod 444 library.yaml; }
```

### 2. Find the Entry
- Read `library.yaml`
- Search across `agentic-library.skills`, `agentic-library.agents`, `agentic-library.prompts`, and `agentic-library.mcp`
- Match by name (exact) or description (fuzzy/keyword match)
- If multiple matches, show them and ask the user to pick one
- If no match, tell the user and suggest `/agentic-library search`

### 3. Resolve Dependencies
If the entry has a `requires` field:
- For each typed reference (`skill:name`, `agent:name`, `prompt:name`, `mcp:name`):
  - Look it up in `library.yaml`
  - If found, recursively run the `use` workflow for that dependency first, using the **same scope** as the parent item (resolved in step 4)
  - If not found, warn the user: "Dependency <ref> not found in agentic-library catalog"
- Process all dependencies before the requested item

### 4. Determine Target Directory
Resolve the scope **before** installing anything (including dependencies):
- Parse scope flags anywhere after the item name: `--global` / `-g` and `--project` / `-p`
- Natural-language scope maps to the same flags: "global", "globally", "for all projects" → `--global`; "project", "this project only", "locally" → `--project`
- If both global and project are requested → report the conflict (`Conflicting scope: use either --global or --project, not both`) and **stop without installing anything**
- Scope resolution:
  - `--project` / `-p` → **project** scope
  - `--global` / `-g` → **global** scope
  - No flag → **global** scope (the default)
- Read `default_dirs` from `library.yaml` and select the `global` or `project` path for the item's type (skills/agents/prompts/mcp)
- If the user specified a custom path → use that path instead; it overrides both flags (for MCP, the custom path only changes where the files go — registration still follows the resolved scope)
- Use the resolved scope for every dependency installed in step 3, and for the symlink (step 7) and MCP registration (step 8)

### 5. Fetch from Source

**If source is a local path** (starts with `/` or `~`):
- Resolve `~` to the home directory
- Get the parent directory of the referenced file
- For skills and mcp: copy the entire parent directory to the target:
  ```bash
  cp -R <parent_directory>/ <target_directory>/<name>/
  ```
- For agents: copy just the agent file to the target:
  ```bash
  cp <agent_file> <target_directory>/<agent_name>.md
  ```
- For prompts: copy just the prompt file to the target:
  ```bash
  cp <prompt_file> <target_directory>/<prompt_name>.md
  ```
- If the agent or prompt is nested in a subdirectory under the `agents/` or `commands/` directories, copy the subdirectory to the target as well, creating the subdir if it doesn't exist. This is useful because it keeps the agents or commands grouped together.

**If source is a GitHub URL**:
- Parse the URL to extract: `org`, `repo`, `branch`, `file_path`
  - Browser URL pattern: `https://github.com/<org>/<repo>/blob/<branch>/<path>`
  - Raw URL pattern: `https://raw.githubusercontent.com/<org>/<repo>/<branch>/<path>`
- Determine the clone URL: `https://github.com/<org>/<repo>.git`
- Determine the parent directory path within the repo (everything before the filename)
- Clone into a temporary directory:
  ```bash
  tmp_dir=$(mktemp -d)
  git clone --depth 1 --branch <branch> https://github.com/<org>/<repo>.git "$tmp_dir"
  ```
- Copy the parent directory of the file to the target:
  ```bash
  cp -R "$tmp_dir/<parent_path>/" <target_directory>/<name>/
  ```
**Save Installation Metadata**:
- For GitHub URLs, capture the commit hash of the pulled version:
  ```bash
  commit_hash=$(git -C "$tmp_dir" rev-parse HEAD)
  echo "{\"commit\": \"$commit_hash\"}" > <target_directory>/.<name>.agentic-metadata.json
  ```
- For Local Paths, capture the last modified timestamp of the source and save it similarly.

- Clean up:
  ```bash
  rm -rf "$tmp_dir"
  ```

**If clone fails (private repo)**, try SSH:
  ```bash
  git clone --depth 1 --branch <branch> git@github.com:<org>/<repo>.git "$tmp_dir"
  ```

### 6. Verify Installation
- Confirm the target directory exists
- Confirm the main file (SKILL.md, AGENT.md, prompt file, or mcp.json) exists in it
- Report success with the installed path

### 7. Link Skill into Claude Code
If the type is `skill`, link the installed copy into the Claude Code skills folder for the scope resolved in step 4 so the harness can find it. `.agents/skills/` stays the one real install location; `.claude/skills/<name>` is only a symlink to it. Skip this step for agents, prompts, and MCP servers.

| Install scope | Real copy | Symlink |
|---|---|---|
| Global (default, `-g`) | `~/.agents/skills/<name>/` | `~/.claude/skills/<name>` → `~/.agents/skills/<name>` |
| Project (`-p`) | `.agents/skills/<name>/` | `.claude/skills/<name>` → `../../.agents/skills/<name>` |
| Custom path | `<custom>/<name>/` | None — tell the user how to link it manually |

- Pick the link folder: `~/.claude/skills/` (global) or `.claude/skills/` (project)
- **Custom path** → do not create a link. Tell the user Claude Code will not see the skill until they link it, e.g. `ln -sfn <absolute_custom_path>/<name> ~/.claude/skills/<name>`
- **Never overwrite a real folder.** If `<link_dir>/<name>` exists and is not a symlink, warn the user that a non-symlink folder is already there and skip the link
- Otherwise create the folder and create or refresh the link with `ln -sfn` (idempotent, so re-running `use` never duplicates it). Use a relative target for project installs so the link survives moving or cloning the repo:
  ```bash
  # Global (default)
  mkdir -p ~/.claude/skills
  if [ -e ~/.claude/skills/<name> ] && [ ! -L ~/.claude/skills/<name> ]; then
    echo "WARN: ~/.claude/skills/<name> is a real folder; not linking"
  else
    ln -sfn ~/.agents/skills/<name> ~/.claude/skills/<name>
  fi

  # Project (--project / -p)
  mkdir -p .claude/skills
  if [ -e .claude/skills/<name> ] && [ ! -L .claude/skills/<name> ]; then
    echo "WARN: .claude/skills/<name> is a real folder; not linking"
  else
    ln -sfn ../../.agents/skills/<name> .claude/skills/<name>
  fi
  ```
- Verify the link resolves to a folder containing `SKILL.md`:
  ```bash
  test -f <link_dir>/<name>/SKILL.md && echo "linked"
  ```
- Report both the install path and the link path. Tell the user to **restart Claude Code** so it lists the skill as `/<name>`

### 8. Register MCP with the Harness
If the type is `mcp`, register the server with the active agent harness so it is actually loaded, using the scope resolved in step 4. Claude Code is the default harness:
- Read the installed `mcp.json` — it holds a single server definition in Claude Code's format (`{"type": "stdio", "command": "...", "args": [...], "env": {...}}` or `{"type": "http", "url": "..."}`)
- **Global install** (default) → merge it into `~/.claude.json` under `mcpServers.<name>`, preserving existing keys
- **Project install** (`--project` / `-p`) → merge it into `./.mcp.json` under `mcpServers.<name>` (create the file as `{"mcpServers": {}}` if it doesn't exist), preserving existing keys
- For any other harness, skip the merge and tell the user how to register the server manually
- Tell the user to **restart the harness** for the new MCP server to load

### 9. Confirm
Tell the user:
- What was installed, which scope was used, and where — e.g. `Installed <name> globally → ~/.agents/skills/<name>` or `Installed <name> for this project → .agents/skills/<name>`
- For skills, the `.claude/skills/<name>` symlink path (or why it was skipped) and to restart Claude Code
- Any dependencies that were also installed
- If this was a refresh (overwrite), mention that
- For MCP entries, that the server was registered with the harness (and to restart)
