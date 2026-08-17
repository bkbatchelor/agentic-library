# Push an Item to the Library Source

## Context
The user has improved a skill, agent, prompt, or MCP server locally and wants to push changes back to the source.

## Permissions & Requirements
> **Note:** Only owners and maintainers with direct write permissions are allowed to push changes directly to the source repository. Contributors without direct write access must submit a pull request / merge request to the source repository.

## Input
The user provides an item name or description.

## Steps

### 1. Find the Entry
- Read `library.yaml`
- Search across all sections for the matching entry
- If no match, tell the user the item wasn't found in the catalog

### 2. Locate the Local Copy
- Check the default directory for the type (from `default_dirs`)
- Check the global directory
- If found in multiple places, ask which one to push
- If not found locally, tell the user there's nothing to push

### 3. Check for Conflicts

**If source is a local path:**
- Compare the local installed copy with the source
- If the source has been modified since last pull, warn the user:
  "The source has changes that aren't in your local copy. Pushing will overwrite them. Continue?"

**If source is a GitHub URL:**
- Clone the repo to a temp directory (shallow):
  ```bash
  tmp_dir=$(mktemp -d)
  git clone --depth 1 --branch <branch> <clone_url> "$tmp_dir"
  ```
- Compare the skill directory in the clone with the local copy
- If they differ AND the remote has changes not in the local copy, warn about conflict
- Ask the user to resolve before continuing

### 4. Push to Source

**If source is a local path:**
- Copy the entire local directory to the source location, overwriting:
  ```bash
  cp -R <local_directory>/ <source_parent_directory>/
  ```
- Confirm the overwrite

**If source is a GitHub URL:**
- If we don't already have a tmp clone from step 3, clone now:
  ```bash
  tmp_dir=$(mktemp -d)
  git clone --depth 1 --branch <branch> <clone_url> "$tmp_dir"
  ```
- Remove the old skill directory in the clone:
  ```bash
  rm -rf "$tmp_dir/<skill_path_in_repo>"
  ```
- Copy the local version into the clone:
  ```bash
  cp -R <local_directory>/ "$tmp_dir/<skill_path_in_repo>/"
  ```
- Stage ONLY the relevant changes:
  ```bash
  cd "$tmp_dir"
  git add <skill_path_in_repo>
  ```
- Commit with the standard format:
  ```bash
  git commit -m "agentic-library: updated <name> <brief description of what changed>"
  ```
- **Ask for user permission before pushing:**
  Display the commit summary, files changed, and destination repository/branch. Ask the user for explicit permission to proceed with `git push`.
- Push (if owner/maintainer with write access and user permission granted):
  ```bash
  git push
  ```
- If you do not have direct push access:
  - Push to a new branch and open a pull request / merge request against the source repository.
- Clean up:
  ```bash
  rm -rf "$tmp_dir"
  ```

### 5. Confirm
Tell the user:
- What was pushed and where
- The commit message used
- If it was a local path push, confirm the overwrite
