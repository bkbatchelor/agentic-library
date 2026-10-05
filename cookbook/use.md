# Use an Item from the Library

## Context
Pull a skill, agent, prompt, or MCP server from the catalog into the local environment. If already installed locally, overwrite with the latest from the source (refresh).

## Input
The user provides a skill name or description.

## Steps

### 1. Sync the Library Repo
Pull the latest catalog before reading:
```bash
cd <LIBRARY_SKILL_DIR>
git pull
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
  - If found, recursively run the `use` workflow for that dependency first
  - If not found, warn the user: "Dependency <ref> not found in agentic-library catalog"
- Process all dependencies before the requested item

### 4. Determine Target Directory
- Read `default_dirs` from `library.yaml`
- If user said "global" or "globally" → use the `global` path
- If user specified a custom path → use that path
- Otherwise → use the `project` path
- Select the correct section based on type (skills/agents/prompts/mcp)

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
If the type is `skill`, link the installed copy into Claude Code's skills folder so the harness can find it. `.agents/skills/` stays the one real install location; `.claude/skills/<name>` is only a symlink to it. Skip this step for agents, prompts, and MCP servers.

| Install scope | Real copy | Symlink |
|---|---|---|
| Project (default) | `.agents/skills/<name>/` | `.claude/skills/<name>` → `../../.agents/skills/<name>` |
| Global | `~/.agents/skills/<name>/` | `~/.claude/skills/<name>` → `~/.agents/skills/<name>` |
| Custom path | `<custom>/<name>/` | None — tell the user how to link it manually |

- Pick the link folder: `.claude/skills/` (project) or `~/.claude/skills/` (global)
- **Custom path** → do not create a link. Tell the user Claude Code will not see the skill until they link it, e.g. `ln -sfn <absolute_custom_path>/<name> ~/.claude/skills/<name>`
- **Never overwrite a real folder.** If `<link_dir>/<name>` exists and is not a symlink, warn the user that a non-symlink folder is already there and skip the link
- Otherwise create the folder and create or refresh the link with `ln -sfn` (idempotent, so re-running `use` never duplicates it). Use a relative target for project installs so the link survives moving or cloning the repo:
  ```bash
  # Project
  mkdir -p .claude/skills
  if [ -e .claude/skills/<name> ] && [ ! -L .claude/skills/<name> ]; then
    echo "WARN: .claude/skills/<name> is a real folder; not linking"
  else
    ln -sfn ../../.agents/skills/<name> .claude/skills/<name>
  fi

  # Global
  mkdir -p ~/.claude/skills
  if [ -e ~/.claude/skills/<name> ] && [ ! -L ~/.claude/skills/<name> ]; then
    echo "WARN: ~/.claude/skills/<name> is a real folder; not linking"
  else
    ln -sfn ~/.agents/skills/<name> ~/.claude/skills/<name>
  fi
  ```
- Verify the link resolves to a folder containing `SKILL.md`:
  ```bash
  test -f <link_dir>/<name>/SKILL.md && echo "linked"
  ```
- Report both the install path and the link path. Tell the user to **restart Claude Code** so it lists the skill as `/<name>`

### 8. Register MCP with the Harness
If the type is `mcp`, register the server with the active agent harness so it is actually loaded. Claude Code is the default harness:
- Read the installed `mcp.json` — it holds a single server definition in Claude Code's format (`{"type": "stdio", "command": "...", "args": [...], "env": {...}}` or `{"type": "http", "url": "..."}`)
- **Global install** → merge it into `~/.claude.json` under `mcpServers.<name>`, preserving existing keys
- **Project install** → merge it into `./.mcp.json` under `mcpServers.<name>` (create the file as `{"mcpServers": {}}` if it doesn't exist), preserving existing keys
- If the user targets opencode, translate the server (`stdio` → `{"type": "local", "command": [<command>, ...<args>], "environment": <env>}`; `http`/`sse` → `{"type": "remote", "url": <url>, "headers": <headers>}`) and merge it under `mcp.<name>` in `~/.config/opencode/opencode.json` (global) or `./opencode.json` (project)
- For any other harness, skip the merge and tell the user how to register the server manually
- Tell the user to **restart the harness** for the new MCP server to load

### 9. Confirm
Tell the user:
- What was installed and where
- For skills, the `.claude/skills/<name>` symlink path (or why it was skipped) and to restart Claude Code
- Any dependencies that were also installed
- If this was a refresh (overwrite), mention that
- For MCP entries, that the server was registered with the harness (and to restart)
