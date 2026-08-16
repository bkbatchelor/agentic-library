# Remove an Entry from the Library

## Context
The user wants to remove a skill, agent, prompt, or MCP server from the library catalog and optionally delete the local copy.

## Permissions & Requirements
> **Note:** Only owners and maintainers of the Git repository are allowed to execute `remove` commands directly on the catalog repository. Contributors without direct write access must submit a pull request / merge request.

## Input
The user provides a skill name or description.

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
- "Remove **<name>** from the library catalog?"
- If installed locally, also ask: "Also delete the local copy at `<path>`?"

### 4. Remove from library.yaml
- Remove the entry from the appropriate section (`library.skills`, `library.agents`, `library.prompts`, or `library.mcp`)
- If other entries depend on this one (via `requires`), warn the user before proceeding

### 5. Delete Local Copy (if requested)
If the user confirmed local deletion:
- Check the default directory for the type (from `default_dirs`)
- Check the global directory
- Remove the directory or file:
  ```bash
  rm -rf <target_directory>/<name>
  ```

### 6. Unregister MCP from the Harness
If the type is `mcp`, also remove the server from the active agent harness config:
- Delete the `mcp.<name>` key from `~/.config/opencode/opencode.json` (if it was a global install)
- Delete the `mcp.<name>` key from `./opencode.json` (if it was a project install)
- If the harness is not opencode, skip this step and tell the user to remove the server manually
- Tell the user to **restart the harness** for the removal to take effect

### 7. Commit and Push (or Submit Merge Request)
If you are an owner or maintainer with direct push permissions:
```bash
cd <LIBRARY_SKILL_DIR>
git add library.yaml
git commit -m "library: removed <type> <name>"
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
- Whether the local copy was also deleted
- For MCP entries, that the server was unregistered from the harness (and to restart)
- If other entries depended on it, remind them to update those entries
