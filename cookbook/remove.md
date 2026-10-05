# Remove an Entry from the Library

## Context
The user wants to remove a skill, agent, prompt, or MCP server from the agentic-library catalog and optionally delete the local copy.

## Permissions & Requirements
> **Note:** Only owners and maintainers of the Git repository are allowed to execute `remove` commands directly on the catalog repository. Contributors without direct write access must submit a pull request / merge request.

## Input
The user provides a skill name or description, optionally followed by a scope flag that selects which local copy to delete:
```bash
/agentic-library remove <name>             # no flag → global copy (default)
/agentic-library remove <name> --global    # or -g → global copy
/agentic-library remove <name> --project   # or -p → project copy
```
Scope is resolved the same way as in [use.md](use.md) step 4: flags may appear anywhere after the name, natural language ("globally", "this project only") maps to the same flags, no flag means global, and passing both `--global` and `--project` is an error — report the conflict and stop without changing anything.

## Steps

### 1. Sync the Library Repo
Pull the latest catalog before modifying:
```bash
cd <LIBRARY_SKILL_DIR>
git pull
```

### 2. Find the Entry
- Read `library.yaml`
- Search across all sections for the matching entry
- Determine the type (skill, agent, prompt, or mcp)
- If no match, tell the user the item wasn't found in the catalog

### 3. Confirm with User
Show the entry details and ask:
- "Remove **<name>** from the agentic-library catalog?"
- If installed locally in the resolved scope, also ask: "Also delete the <scope> copy at `<path>`?"
- If it is only installed in the other scope, mention that and leave it alone unless the user re-runs with that scope's flag

### 4. Remove from library.yaml
- Remove the entry from the appropriate section (`agentic-library.skills`, `agentic-library.agents`, `agentic-library.prompts`, or `agentic-library.mcp`)
- If other entries depend on this one (via `requires`), warn the user before proceeding

### 5. Delete Local Copy (if requested)
If the user confirmed local deletion:
- Select the directory for the type and resolved scope from `default_dirs` (`global` by default, `project` with `--project` / `-p`)
- Remove the directory or file:
  ```bash
  rm -rf <target_directory>/<name>
  ```
- For skills, also delete the Claude Code symlink for the same scope (`~/.claude/skills/<name>` for global, `.claude/skills/<name>` for project), but **only if it is a symlink pointing into `.agents/skills/`**. Leave real folders and links pointing elsewhere alone, and tell the user:
  ```bash
  link=~/.claude/skills/<name>   # or .claude/skills/<name> for project scope
  if [ -L "$link" ] && case "$(readlink "$link")" in *.agents/skills/*) true;; *) false;; esac; then
    rm "$link"
  fi
  ```

### 6. Unregister MCP from the Harness
If the type is `mcp`, also remove the server from the harness config for the resolved scope:
- **Global** (default) → delete the `mcpServers.<name>` key from `~/.claude.json`
- **Project** → delete the `mcpServers.<name>` key from `./.mcp.json`
- For any other harness, skip this step and tell the user to remove the server manually
- Tell the user to **restart the harness** for the removal to take effect

### 7. Commit and Push (or Submit Merge Request)
If you are an owner or maintainer with direct push permissions:
```bash
cd <LIBRARY_SKILL_DIR>
git add library.yaml
git commit -m "agentic-library: removed <type> <name>"
```
**Ask for user permission before pushing:** Always show the commit message and target branch, then confirm with the user before running:
```bash
git push
```
If you do not have direct push permissions:
- Create a new branch, commit the `library.yaml` change, push the branch, and submit a pull request / merge request.

### 8. Confirm
Tell the user:
- The entry has been removed from the catalog
- Whether the local copy was also deleted, from which scope (and, for skills, its `.claude/skills/<name>` symlink)
- For MCP entries, that the server was unregistered from the harness (and to restart)
- If other entries depended on it, remind them to update those entries
