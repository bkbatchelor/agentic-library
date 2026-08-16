# Add and Install Examples

Worked examples for the `/library install`, `/library use`, `/library add`, `/library remove`, and `/library push` commands. They show the full flow from user request to result. For the step-by-step procedures, read [cookbook/use.md](use.md), [cookbook/add.md](add.md), [cookbook/install.md](install.md), [cookbook/remove.md](remove.md), and [cookbook/push.md](push.md) first — these examples follow them.

Each add example covers: type detection, source validation, dependency parsing, and the exact YAML entry written to `library.yaml`. Each install example covers: prerequisites, fork status, cloning, and variable setup. Each use example covers: dependency resolution, target directory selection, fetching from source, and (for MCP) harness registration. Each remove example covers: syncing, confirmation, dependency checks, and (for MCP) harness unregistration. Each push example covers: locating the local copy, conflict checking, staging only relevant changes, and asking permission before pushing.

## Table of Contents

- [Install on a New Device](#install-on-a-new-device)
- [Use an Item from the Catalog](#use-an-item-from-the-catalog)
- [Add a Skill](#add-a-skill)
- [Add an Agent](#add-an-agent)
- [Add a Prompt](#add-a-prompt)
- [Add an MCP Server](#add-an-mcp-server)
- [Combining Types in One Request](#combining-types-in-one-request)
- [Remove an Entry from the Catalog](#remove-an-entry-from-the-catalog)
- [Push Changes to the Source](#push-changes-to-the-source)

## Install on a New Device

### Example: First install from the template repo

**User says:**
> Install Agentic Library on my new machine

**Steps:**
1. Prerequisites: `git --version` succeeds, global skills directory `~/.agents/skills/` exists (or is created)
2. Fork status: user is on the template repo (hasn't forked) — instruct them to create a private fork on GitHub, then update the remote:
   ```bash
   cd ~/.agents/skills/library
   git remote set-url origin <fork_url>
   git remote -v
   ```
3. Clone to the global skills directory:
   ```bash
   mkdir -p ~/.agents/skills/library
   cd ~/.agents/skills/library
   git clone <fork_url> .
   ```
4. Update the `## Variables` section in `SKILL.md`:
   - `LIBRARY_REPO_URL` → the user's fork URL
   - `LIBRARY_YAML_PATH` → confirm path
   - `LIBRARY_SKILL_DIR` → confirm path

**Result:**
- `SKILL.md` and `library.yaml` exist at `~/.agents/skills/library/`
- The `/library` command is now available
- `/library list` shows the catalog (empty by default)
- `/library add` to start adding skills, agents, prompts, and MCP servers

### Example: Install on another device (already forked)

**User says:**
> Set up the library on my Mac mini

**Steps:**
1. Prerequisites: `git --version` succeeds, `~/.agents/skills/` exists (or is created)
2. Fork status: already forked — the remote already points to their fork, skip the remote update
3. Clone to the global skills directory:
   ```bash
   mkdir -p ~/.agents/skills/library
   cd ~/.agents/skills/library
   git clone <fork_url> .
   ```
4. Update the `## Variables` section in `SKILL.md` (same as the first install example)

**Result:**
- `SKILL.md` and `library.yaml` exist at `~/.agents/skills/library/`
- `/library list` shows the catalog, `/library use <name>` pulls entries on demand

## Use an Item from the Catalog

### Example: Pull a skill with a dependency

**User says:**
> Use the diagram-kroki skill

**Steps:**
1. Library repo synced: `git pull`
2. Entry found: `diagram-kroki` in `library.skills`
3. Dependencies resolved: `requires: [skill:firecrawl]` — `firecrawl` found in the catalog, so the `use` workflow runs for it first
4. Target directory: default → `~/.agents/skills/` for skills
5. Fetched from GitHub source: temp clone of the repo, copied `skills/diagram-kroki/` → `~/.agents/skills/diagram-kroki/`, temp dir cleaned up
6. Verified: `~/.agents/skills/diagram-kroki/SKILL.md` exists

**Result:**
- Installed `~/.agents/skills/diagram-kroki/` (SKILL.md)
- Dependency `firecrawl` was installed first
- If `diagram-kroki` was already installed, its local copy is overwritten with the latest from source (refresh)

### Example: Use an MCP server globally (registers with the harness)

**User says:**
> Use the github MCP server globally

**Steps:**
1. Library repo synced: `git pull`
2. Entry found: `github` in `library.mcp`
3. Dependencies: none
4. Target directory: "globally" → `~/.agents/mcp/` for MCP
5. Fetched from GitHub source: temp clone, copied `mcp/github/` → `~/.agents/mcp/github/`, temp dir cleaned up
6. Verified: `~/.agents/mcp/github/mcp.json` exists
7. Registered with the harness: the installed `mcp.json` (`{"type": "local", "command": [...]}`) was merged into `~/.config/opencode/opencode.json` under `mcp.github`, preserving existing keys

**Result:**
- Installed at `~/.agents/mcp/github/`
- Server registered with opencode — restart the harness for it to load

## Add a Skill

### Example: Local skill with a dependency

**User says:**
> Add the firecrawl skill from `/Users/me/projects/tools/skills/firecrawl/SKILL.md`

**Steps:**
1. Type detected: `skill` (source path contains `SKILL.md`)
2. Source validated: local path exists
3. Dependencies: none found in the frontmatter

**YAML added to `library.skills` (kept alphabetically sorted):**

```yaml
- name: firecrawl
  description: Scrape, crawl, and search websites using Firecrawl CLI
  source: /Users/me/projects/tools/skills/firecrawl/SKILL.md
```

### Example: Remote skill with dependencies

**User says:**
> Add diagram-kroki from `https://github.com/someones-org/private-skills/blob/main/skills/diagram-kroki/SKILL.md`. It needs firecrawl.

**Steps:**
1. Type detected: `skill`
2. Source validated: well-formed GitHub browser URL
3. Dependencies parsed: `skill:firecrawl` — verified it already exists in `library.skills`

**YAML added to `library.skills`:**

```yaml
- name: diagram-kroki
  description: Generate diagrams via Kroki HTTP API supporting 28+ languages
  source: https://github.com/someones-org/private-skills/blob/main/skills/diagram-kroki/SKILL.md
  requires: [skill:firecrawl]
```

## Add an Agent

### Example: Agent from a local AGENT.md

**User says:**
> Add my video-processor agent from `/Users/me/projects/tools/agents/video-processor/AGENT.md`

**Steps:**
1. Type detected: `agent` (source path contains `AGENT.md`)
2. Source validated: local path exists
3. Dependencies: none

**YAML added to `library.agents`:**

```yaml
- name: video-processor
  description: Processes video files with ffmpeg and whisper transcription
  source: /Users/me/projects/tools/agents/video-processor/AGENT.md
```

### Example: Agent from a private GitHub repo

**User says:**
> Add the code-reviewer agent from `https://github.com/someones-org/agent-configs/blob/main/agents/code-reviewer/AGENT.md`

**Steps:**
1. Type detected: `agent`
2. Source validated: well-formed GitHub browser URL (private repos use SSH or `GITHUB_TOKEN`)
3. Dependencies: none

**YAML added to `library.agents`:**

```yaml
- name: code-reviewer
  description: Reviews code for quality, security, and performance
  source: https://github.com/someones-org/agent-configs/blob/main/agents/code-reviewer/AGENT.md
```

## Add a Prompt

### Example: Prompt from a local file

**User says:**
> Add a caption-style prompt from `/Users/me/projects/content/prompts/caption-style.md`

**Steps:**
1. Type detected: `prompt` (user said "prompt")
2. Source validated: local path exists
3. Dependencies: none

**YAML added to `library.prompts`:**

```yaml
- name: caption-style
  description: Style guide for generating video captions
  source: /Users/me/projects/content/prompts/caption-style.md
```

### Example: Prompt from a GitHub URL

**User says:**
> Add the team commit-message prompt from `https://github.com/someones-org/team-prompts/blob/main/prompts/commit-message.md`

**Steps:**
1. Type detected: `prompt`
2. Source validated: well-formed GitHub browser URL
3. Dependencies: none

**YAML added to `library.prompts`:**

```yaml
- name: commit-message
  description: Standardized commit message format for all projects
  source: https://github.com/someones-org/team-prompts/blob/main/prompts/commit-message.md
```

## Add an MCP Server

MCP entries are validated more strictly than the other types: `mcp.json` must be a well-formed MCP server object in the harness's native format (opencode: `type` is required, `command` is an array of strings).

### Example: Local MCP server

**User says:**
> Add the playwright MCP server from `/Users/me/projects/tools/mcp/playwright/mcp.json`

**Steps:**
1. Type detected: `mcp` (source path contains `mcp.json`)
2. Source validated: local path exists and `mcp.json` parses as a valid server object
3. Dependencies: none

**The source `mcp.json`:**

```json
{
  "type": "local",
  "command": ["npx", "@playwright/mcp@latest"]
}
```

**YAML added to `library.mcp`:**

```yaml
- name: playwright
  description: Browser automation via the Playwright MCP server
  source: /Users/me/projects/tools/mcp/playwright/mcp.json
```

### Example: Remote MCP server

**User says:**
> Add the github MCP server from `https://github.com/someones-org/mcp-servers/blob/main/mcp/github/mcp.json`

**Steps:**
1. Type detected: `mcp`
2. Source validated: well-formed GitHub browser URL
3. Dependencies: none

**YAML added to `library.mcp`:**

```yaml
- name: github
  description: GitHub API access via the GitHub MCP server
  source: https://github.com/someones-org/mcp-servers/blob/main/mcp/github/mcp.json
```

## Combining Types in One Request

**User says:**
> Add the green-screen-captions skill from `https://raw.githubusercontent.com/someones-org/video-tools/main/skills/green-screen-captions/SKILL.md`. It depends on my video-processor agent and the caption-style prompt.

**Steps:**
1. Type detected: `skill`
2. Source validated: well-formed GitHub raw URL
3. Dependencies parsed: `agent:video-processor`, `prompt:caption-style` — both already exist in the catalog (see the agent and prompt examples above)

**YAML added to `library.skills`:**

```yaml
- name: green-screen-captions
  description: Generate and burn AI-powered captions onto green screen videos
  source: https://raw.githubusercontent.com/someones-org/video-tools/main/skills/green-screen-captions/SKILL.md
  requires: [agent:video-processor, prompt:caption-style]
```

> **Note:** If a dependency is not already in the catalog, add it to `library.yaml` first (recursively checking its dependencies), then add the entry that references it.

## Remove an Entry from the Catalog

> **Note:** Only owners and maintainers with direct push access can run `remove` directly on the catalog repo. Contributors must submit a pull request / merge request instead.

### Example: Remove a skill and its local copy

**User says:**
> Remove the diagram-kroki skill

**Steps:**
1. Library repo synced: `git pull` before modifying
2. Entry found: `diagram-kroki` in `library.skills`, type `skill`
3. Confirmed with user: "Remove diagram-kroki from the library catalog?" and "Also delete the local copy at `~/.agents/skills/diagram-kroki`?"
4. Dependency check: no other entries list `skill:diagram-kroki` in `requires` — safe to remove
5. Entry removed from `library.skills`
6. Local copy deleted (user confirmed): `rm -rf ~/.agents/skills/diagram-kroki`

**YAML before:**

```yaml
- name: diagram-kroki
  description: Generate diagrams via Kroki HTTP API supporting 28+ languages
  source: https://github.com/someones-org/private-skills/blob/main/skills/diagram-kroki/SKILL.md
  requires: [skill:firecrawl]
```

**YAML after:** the `diagram-kroki` entry is gone from `library.skills`; `firecrawl` (its dependency) remains.

**Result:**
- Entry removed from the catalog
- Local copy deleted
- Change committed (`library: removed skill diagram-kroki`) and pushed after asking for permission

### Example: Remove an MCP server and unregister it from the harness

**User says:**
> Remove the github MCP server

**Steps:**
1. Library repo synced: `git pull`
2. Entry found: `github` in `library.mcp`, type `mcp`
3. Confirmed with user
4. Dependency check: no other entries reference `mcp:github`
5. Entry removed from `library.mcp`
6. Unregistered from the harness: deleted the `mcp.github` key from `~/.config/opencode/opencode.json` (it was a global install)
7. User restarts the harness for the removal to take effect

**Result:**
- Entry removed from the catalog
- Server unregistered from the harness (restart required to take effect)

## Push Changes to the Source

> **Note:** Only owners and maintainers with direct write access can push directly to the source repository. Contributors must submit a pull request / merge request against the source.

### Example: Push a locally improved skill to its GitHub source

**User says:**
> I improved the firecrawl skill locally. Push it back to the source.

**Steps:**
1. Entry found: `firecrawl` in `library.skills`
2. Local copy located: `~/.agents/skills/firecrawl/` (only one copy — no need to ask which one)
3. Conflict check: temp-cloned the source repo and compared `skills/firecrawl/` — the remote has no changes that aren't in the local copy, so no conflict
4. Applied the changes to the temp clone:
   ```bash
   tmp_dir=$(mktemp -d)
   git clone --depth 1 --branch main <clone_url> "$tmp_dir"
   rm -rf "$tmp_dir/skills/firecrawl"
   cp -R ~/.agents/skills/firecrawl/ "$tmp_dir/skills/firecrawl/"
   ```
5. Staged only the relevant path and committed:
   ```bash
   cd "$tmp_dir"
   git add skills/firecrawl
   git commit -m "library: updated firecrawl improved retry handling"
   ```
6. Permission asked: commit summary, files changed, and destination repo/branch shown to the user before pushing
7. Pushed (with user permission) and cleaned up the temp dir

**Result:**
- Changes pushed to `someones-org/private-skills@main`
- Commit message used: `library: updated firecrawl improved retry handling`

### Example: Push to a local path source

**User says:**
> Push the video-processor agent changes back to `/Users/me/projects/tools/agents/video-processor/AGENT.md`

**Steps:**
1. Entry found: `video-processor` in `library.agents`
2. Local copy located: `~/.agents/agents/video-processor/AGENT.md` (only one copy)
3. Conflict check: source at `/Users/me/projects/tools/agents/video-processor/` compared — source unchanged since last pull, no conflict
4. Overwrote the source:
   ```bash
   cp -R ~/.agents/agents/video-processor/ /Users/me/projects/tools/agents/video-processor/
   ```
5. Overwrite confirmed with the user

**Result:**
- Local copy pushed to `/Users/me/projects/tools/agents/video-processor/AGENT.md`
