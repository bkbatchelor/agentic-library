---
name: agentic-library
description: Private skill distribution system. Use when the user wants to install, use, add, push, remove, sync, list, or search for skills, agents, prompts, or MCP servers from their private agentic-library catalog. Triggers on /agentic-library commands or mentions of agentic-library, skill distribution, agentic management, or MCP servers.
argument-hint: <command or prompt> <name or details>
---

# Agentic Library

A meta-skill for private-first distribution of agentics (skills, agents, prompts, and MCP servers) across agents, devices, and teams.

## Variables

> Update these after forking and cloning the agentic-library repo.

- **LIBRARY_REPO_URL**: `git@github.com:bkbatchelor/agentic-library.git`
- **LIBRARY_YAML_PATH**: `~/.agents/skills/agentic-library/library.yaml`
- **LIBRARY_SKILL_DIR**: `~/.agents/skills/agentic-library/`

## How It Works

Agentic Library is a catalog of references to your agentics. The `library.yaml` file points to where skills, agents, prompts, and MCP servers live (local filesystem or GitHub repos). Nothing is fetched until you ask for it.

**The `library.yaml` is a catalog, not a manifest.** Entries define what's *available* — not what gets installed. You pull specific items on demand with `/agentic-library use <name>`.

## Commands

> **Permissions Note:** Only owners and maintainers of the Git repository are allowed to execute `add`, `push`, or `remove` commands directly. All other contributors must submit a merge request (pull request) to propose changes to the catalog or source repositories.

| Command                     | Purpose                                  |
| --------------------------- | ---------------------------------------- |
| `/agentic-library install`          | First-time setup: fork, clone, configure |
| `/agentic-library add <details>`    | Register a new entry in the catalog (Owner/Maintainer or Merge Request) |
| `/agentic-library use <name>`       | Pull from source (install or refresh)    |
| `/agentic-library push <name>`      | Push local changes back to source (Owner/Maintainer or Merge Request) |
| `/agentic-library remove <name>`    | Remove from catalog and optionally local (Owner/Maintainer or Merge Request) |
| `/agentic-library list`             | Show full catalog with install status    |
| `/agentic-library sync`             | Re-pull all installed items from source   |
| `/agentic-library search <keyword>` | Find entries by keyword                  |

## Cookbook

Each command has a detailed step-by-step guide. **Read the relevant cookbook file before executing a command.**

| Command | Cookbook                                 | Use When                                                    |
| ------- | --------------------------------------- | ----------------------------------------------------------- |
| install | [cookbook/install.md](cookbook/install.md) | First-time setup on a new device                            |
| add     | [cookbook/add.md](cookbook/add.md)         | User wants to register a new skill/agent/prompt/mcp in catalog |
| add     | [cookbook/README.md](cookbook/README.md)   | Worked examples for adding items and installing the agentic-library |
| use     | [cookbook/use.md](cookbook/use.md)         | User wants to pull or refresh an item from the catalog       |
| push    | [cookbook/push.md](cookbook/push.md)       | User improved a skill locally and wants to update the source |
| remove  | [cookbook/remove.md](cookbook/remove.md)   | User wants to remove an entry from the catalog               |
| list    | [cookbook/list.md](cookbook/list.md)       | User wants to see what's available and what's installed      |
| sync    | [cookbook/sync.md](cookbook/sync.md)       | User wants to refresh all installed items at once            |
| search  | [cookbook/search.md](cookbook/search.md)   | User is looking for an item but doesn't know the exact name  |

**When a user invokes a `/agentic-library` command, read the matching cookbook file first, then execute the steps.**

## Source Format

The `source` field in `library.yaml` supports these formats (auto-detected):

- `/absolute/path/to/SKILL.md` — local filesystem
- `https://github.com/org/repo/blob/main/path/to/SKILL.md` — GitHub browser URL

Both GitHub URL formats are supported. Parse org, repo, branch, and file path from the URL structure. For private repos, use SSH or `GITHUB_TOKEN` for auth automatically.

**Important:** The source points to a specific file (SKILL.md, AGENT.md, prompt file, or mcp.json). We always pull the entire parent directory, not just the file.

## Source Parsing Rules

**Local paths** start with `/` or `~`:
- Use the path directly. Copy the parent directory of the referenced file.

**GitHub browser URLs** match `https://github.com/<org>/<repo>/blob/<branch>/<path>`:
- Parse: `org`, `repo`, `branch`, `file_path`
- Clone URL: `https://github.com/<org>/<repo>.git`
- File location within repo: `<path>`

**GitHub raw URLs** match `https://raw.githubusercontent.com/<org>/<repo>/<branch>/<path>`:
- Parse: `org`, `repo`, `branch`, `file_path`
- Clone URL: `https://github.com/<org>/<repo>.git`
- File location within repo: `<path>`


**Fetching (use):**
1. Clone the repo with `git clone --depth 1 <clone_url>` into a temporary directory
2. Navigate to the parent directory of the referenced file
3. Copy that entire directory to the target local directory
4. The temporary directory is cleaned up automatically

**Pushing (push):**
> **Note:** Only owners and maintainers with direct write access may push directly to the source repository. Non-maintainers must submit a pull request / merge request.
1. Clone the repo with `git clone --depth 1 <clone_url>` into a temporary directory
2. Overwrite the skill directory in the clone with the local version
3. Stage only the relevant changes: `git add <skill_directory_path>`
4. Commit with message: `agentic-library: updated <skill name> <what changed>`
5. **Ask for permission before pushing:** Always show the commit diff/summary and explicitly ask the user for permission before running `git push`.
6. Push to remote (or create a feature branch and submit a merge request if lacking write access)
7. The temporary directory is cleaned up automatically

## Typed Dependencies

The `requires` field uses typed references to avoid ambiguity:
- `skill:name` — references a skill in the agentic-library catalog
- `agent:name` — references an agent in the agentic-library catalog
- `prompt:name` — references a prompt in the agentic-library catalog
- `mcp:name` — references an MCP server in the agentic-library catalog

When resolving dependencies: look up each reference in `library.yaml`, fetch all dependencies first (recursively), then fetch the requested item.

## Target Directories

By default, items are installed to the **default** directory from `library.yaml`:

```yaml
default_dirs:
    skills:
        - default: .agents/skills/
        - global: ~/.agents/skills/
    agents:
        - default: .agents/agents/
        - global: ~/.agents/agents/
    prompts:
        - default: .agents/commands/
        - global: ~/.agents/commands/
    mcp:
        - default: .agents/mcp/
        - global: ~/.agents/mcp/
```

- If the user says "global" or "globally", use the `global` directory.
- If the user specifies a custom path, use that path.
- Otherwise, use the `default` directory.

## Harness Configuration Sync

MCP entries are special: they don't just get installed to disk — they must also be **registered with the active agent harness** so the servers are actually loaded. The agentic-library agent does this in the background as part of the normal workflow.

When you run `/agentic-library use <mcp-name>`, after copying the entry to `.agents/mcp/<name>/`:

1. Read the contents of `.agents/mcp/<name>/mcp.json`. The file holds the server definition in the harness's native format (for opencode: the object under `mcp.<name>` — `{"type": "local", "command": [...], ...}`).
2. **Global install** → merge the server into `~/.config/opencode/opencode.json` under `mcp.<name>`.
3. **Default (project) install** → merge the server into `./opencode.json` (create the file if it doesn't exist), preserving all existing keys.
4. If the harness is not opencode, skip the merge and tell the user how to register the server manually.
5. Tell the user to **restart the harness** for the new MCP server to load.

When you run `/agentic-library remove <mcp-name>`, also delete the `mcp.<name>` key from the same harness config file.

The `mcp.json` file uses the opencode MCP server shape directly so the merge is a 1:1 insert and opencode's strict config validation never rejects it.

## Library Repo Sync

The library skill itself lives in `<LIBRARY_SKILL_DIR>` as a cloned git repo. When running `add` or `remove` (which modifies `library.yaml`), always:
> **Note:** Only owners and maintainers of the agentic-library repo may push changes directly to `main`. Non-maintainers must create a branch and submit a merge request.
1. `git pull` in the agentic-library directory first to get latest
2. Make the changes
3. Stage and commit: `git add library.yaml && git commit`
4. **Ask for permission before pushing:** Always confirm with the user before executing `git push`.
5. `git push` (or submit a merge request if lacking direct push permissions)

This keeps the catalog in sync across devices.

## Example Filled Library File

```yaml
default_dirs:
  skills:
    - default: .agents/skills/
    - global: ~/.agents/skills/
  agents:
    - default: .agents/agents/
    - global: ~/.agents/agents/
  prompts:
    - default: .agents/commands/
    - global: ~/.agents/commands/
  mcp:
    - default: .agents/mcp/
    - global: ~/.agents/mcp/

agentic-library:
  skills:
    - name: firecrawl
      description: Scrape, crawl, and search websites using Firecrawl CLI
      source: /Users/me/projects/tools/skills/firecrawl/SKILL.md

    - name: meta-skill
      description: Creates new Agent Skills following best practices
      source: /Users/me/projects/tools/skills/meta-skill/SKILL.md

    - name: diagram-kroki
      description: Generate diagrams via Kroki HTTP API supporting 28+ languages
      source: https://github.com/someones-org/private-skills/blob/main/skills/diagram-kroki/SKILL.md
      requires: [skill:firecrawl]

    - name: green-screen-captions
      description: Generate and burn AI-powered captions onto green screen videos
      source: https://raw.githubusercontent.com/someones-org/video-tools/main/skills/green-screen-captions/SKILL.md
      requires: [agent:video-processor, prompt:caption-style]

  agents:
    - name: video-processor
      description: Processes video files with ffmpeg and whisper transcription
      source: /Users/me/projects/tools/agents/video-processor/AGENT.md

    - name: code-reviewer
      description: Reviews code for quality, security, and performance
      source: https://github.com/someones-org/agent-configs/blob/main/agents/code-reviewer/AGENT.md

  prompts:
    - name: caption-style
      description: Style guide for generating video captions
      source: /Users/me/projects/content/prompts/caption-style.md

    - name: commit-message
      description: Standardized commit message format for all projects
      source: https://github.com/someones-org/team-prompts/blob/main/prompts/commit-message.md

  mcp:
    - name: playwright
      description: Browser automation via the Playwright MCP server
      source: /Users/me/projects/tools/mcp/playwright/mcp.json

    - name: github
      description: GitHub API access via the GitHub MCP server
      source: https://github.com/someones-org/mcp-servers/blob/main/mcp/github/mcp.json
```
