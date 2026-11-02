# Add a New Entry to the Library

## Context
Register a new skill, agent, prompt, or MCP server in the agentic-library catalog.

## Permissions & Requirements
> **Note:** `add` only edits your local `library.yaml`, which is gitignored and read-only. It needs no write access to the agentic-library repo and never commits or pushes.

## Input
The user provides: name, description, source, and optionally type and dependencies.

## Steps

### 1. Check the Catalog
Make sure the local catalog exists (see *The Catalog File* in `SKILL.md`):
```bash
cd <LIBRARY_SKILL_DIR>
[ -f library.yaml ] || { cp library.example.yaml library.yaml && chmod 444 library.yaml; }
```

### 2. Determine the Type
Figure out the type from the user's prompt or the source path:
- If the source path contains `SKILL.md` or user says "skill" → type is `skill`
- If the source path contains `AGENT.md` or user says "agent" → type is `agent`
- If user says "prompt" → type is `prompt`
- If the source path contains `mcp.json` or user says "mcp" → type is `mcp`
- If ambiguous, ask the user

### 3. Validate the Source
- **Local path**: Verify the file exists at the given path
- **GitHub URL**: Verify the URL is well-formed (matches browser or raw URL patterns), then verify the file really exists in the repo (see *External GitHub Repos* below)
- Confirm the source points to a specific file, not a directory

#### External GitHub Repos
The user may point at someone else's repo instead of a file: `owner/repo`, `https://github.com/<org>/<repo>`, or `https://github.com/<org>/<repo>/tree/<branch>/<path>`. Resolve it to one or more `SKILL.md` files first:
```bash
tmp_dir=$(mktemp -d)
git clone --depth 1 [--branch <branch>] https://github.com/<org>/<repo>.git "$tmp_dir"   # SSH fallback if it fails
branch=$(git -C "$tmp_dir" rev-parse --abbrev-ref HEAD)
# candidates: repo root, then <path>/ for a tree URL, otherwise any SKILL.md up to 3 levels deep
find "$tmp_dir" -maxdepth 3 -name SKILL.md -not -path '*/.git/*'
```
- One `SKILL.md` found → use it. Several → list them and ask which to add (multi-select). None → tell the user and stop.
- Read `name` and `description` from the `SKILL.md` frontmatter; use them unless the user gave their own.
- Build the source as the canonical browser URL so `use`, `update` and `push` work unchanged: `https://github.com/<org>/<repo>/blob/<branch>/<path>/SKILL.md` (for a root-level skill, `.../blob/<branch>/SKILL.md`).
- Clean up with `rm -rf "$tmp_dir"`. Do not install anything in this step.
- If a non-symlink folder with the same name already exists in `~/.claude/skills/` (or `.claude/skills/`), warn the user: `use` will install to `.agents/skills/` but will not replace that folder.
- **For mcp**: Confirm `mcp.json` is a single well-formed MCP server object in Claude Code's format (the value that goes under `mcpServers.<name>`):
  - `stdio` servers: `command` is a string, optional `args` is an array of strings, optional `env` is an object of strings (`type` may be omitted and defaults to `stdio`)
  - `http` / `sse` servers: `type` is `http` or `sse`, `url` is required, optional `headers` is an object of strings

### 4. Parse Dependencies
Detect dependencies by looking through the skill/agent/prompt/mcp files, format them as typed references:
- `skill:name`, `agent:name`, `prompt:name`, `mcp:name`
- Verify each dependency already exists in `library.yaml` if or warn the user
  - If they don't exist add them to `library.yaml` first. If those files have dependencies, add them recursively.
  - You can detect these sometimes by looking at the frontmatter, and then in the file content look for `/<prompt|agent|skill|mcp>:name` references. If you're not sure, ask the user the user if they have any dependencies.

### 5. Add the Entry to library.yaml
`library.yaml` is read-only. Unlock it, add the new entry under the correct section, then lock it again — always relock, even if the edit fails:
```bash
chmod u+w <LIBRARY_YAML_PATH>
# add the entry
chmod a-w <LIBRARY_YAML_PATH>
```

The entry:

```yaml
# Under agentic-library.skills, agentic-library.agents, agentic-library.prompts, or agentic-library.mcp
- name: <name>
  description: <description>
  source: <source>
  requires: [<typed:refs>]  # omit if no dependencies
```

**YAML formatting rules:**
- 2-space indentation
- List items use `- ` prefix
- Properties are indented under the list item
- Keep entries alphabetically sorted by name within each section
- For skills reference the `.../<skill-name>/SKILL.md` file,
- For agents reference the `.../<agent name>.md` file,
- For prompts reference the `.../<prompt name>.md` file (installed to `.agents/commands/`),
- For mcp reference the `.../<mcp-name>/mcp.json` file,
- Remember we'll be adding a absolute path or a github url (https or ssh)

### 6. Confirm
Tell the user the entry has been added to their local catalog and can be installed with `/agentic-library use <name>` (offer to run it now). Later, `/agentic-library sync` shows whether the upstream skill has changed and `/agentic-library update <name>` pulls the newest version. Do not commit or push `library.yaml` — it is gitignored and stays on this machine; to use the entry on another device, run `add` there.
