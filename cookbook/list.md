# List Available Items

## Context
Show the full agentic-library catalog with install status.

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

### 3. Check Install Status
For each entry:
- Determine the type and corresponding project/global directories from `default_dirs`
- Check if a directory matching the entry name exists in the **project** directory
- Check if a directory matching the entry name exists in the **global** directory
- Search recursively for name matches
- Mark as: `installed (project)`, `installed (global)`, or `not installed`

### 4. Display Results

Format the output as a table grouped by type:

```
## Skills
| Name | Description | Source | Status |
|------|-------------|--------|--------|
| skill-name | skill-description | /local/path/... | installed (project) |
| other-skill | other-description | github.com/... | not installed |

## Agents
| Name | Description | Source | Status |
|------|-------------|--------|--------|
| agent-name | agent-description | /local/path/... | installed (global) |

## Prompts
| Name | Description | Source | Status |
|------|-------------|--------|--------|
| prompt-name | prompt-description | github.com/... | not installed |

## MCP Servers
| Name | Description | Source | Status |
|------|-------------|--------|--------|
| mcp-name | mcp-description | github.com/... | installed (global) |
```

If a section is empty, show: `No <type> in catalog.`

### 5. Summary
At the bottom, show:
- Total entries in catalog
- Total installed locally
- Total not installed
