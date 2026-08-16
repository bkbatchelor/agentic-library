# Add and Install Examples

Worked examples for the `/library add` and `/library install` commands. They show the full flow from user request to result. For the step-by-step procedures, read [cookbook/add.md](add.md) and [cookbook/install.md](install.md) first — these examples follow them.

Each add example covers: type detection, source validation, dependency parsing, and the exact YAML entry written to `library.yaml`. Each install example covers: prerequisites, fork status, cloning, and variable setup.

## Table of Contents

- [Install on a New Device](#install-on-a-new-device)
- [Add a Skill](#add-a-skill)
- [Add an Agent](#add-an-agent)
- [Add a Prompt](#add-a-prompt)
- [Add an MCP Server](#add-an-mcp-server)
- [Combining Types in One Request](#combining-types-in-one-request)

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
