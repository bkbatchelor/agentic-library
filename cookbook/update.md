# Update Installed Items to the Newest Version

## Context
Pull the newest upstream version of one or all installed items (typically skills added from other GitHub repos, but any catalog entry works). `sync` only reports drift; `update` applies it.

## Input
- `update <name>` — update one installed entry
- `update --all` — update every installed entry that is outdated or unknown
- Optional `-g` / `-p` to pick the scope, same rules as `use` (global by default; if the item is installed in only one scope, use that one)

## Steps

### 1. Check the Catalog
```bash
cd <LIBRARY_SKILL_DIR>
[ -f library.yaml ] || { cp library.example.yaml library.yaml && chmod 444 library.yaml; }
```
Read `library.yaml`, find the entry (or all entries for `--all`), and confirm each is installed. If a named entry is not installed, tell the user to run `/agentic-library use <name>` instead.

### 2. Find What Is Outdated
Run the comparison from `cookbook/sync.md` step 4 for the selected entries. Skip entries that are `up to date` (report them as such). Continue with `outdated` and `unknown`.

### 3. Fetch the Newest Version
For each entry to update, clone the source into a temporary directory (same parsing, `--depth 1 --branch <branch>` and SSH fallback as `cookbook/use.md`). Local-path sources are copied directly with no clone.
```bash
tmp_dir=$(mktemp -d)
git clone --depth 1 --branch <branch> https://github.com/<org>/<repo>.git "$tmp_dir"
new_commit=$(git -C "$tmp_dir" rev-parse HEAD)
```
Confirm the file referenced by `source` still exists in the clone. If it was moved or deleted upstream, report `failed: source file no longer exists upstream` and leave the installed copy alone.

### 4. Show What Changed
Compare the installed copy with the new upstream directory and summarize it for the user:
```bash
diff -rq <target_directory>/<name> "$tmp_dir/<parent_path>" --exclude=.git
```
List added, removed, and modified files (use `diff -u` on `SKILL.md` if the user wants detail). If the metadata has an old `commit` and the clone is not shallow-limited from it, commit subjects may be shown with `git -C "$tmp_dir" log --oneline <old_commit>..HEAD` — skip silently if the old commit is not reachable.

### 5. Detect Local Edits
If the installed copy differs from what was originally pulled (fetch the recorded `commit` with `git fetch --depth 1 origin <old_commit>` in the clone and diff against it), the user has local edits. Warn clearly and ask before overwriting, offering to: overwrite, skip, or keep a backup copy at `<target_directory>/<name>.bak`. If the old commit cannot be fetched, say that local edits cannot be detected and ask for confirmation only when the diff in step 4 is non-trivial. For `--all`, ask per item that has local edits.

### 6. Apply the Update
Replace the installed copy exactly as `use` does (remove, recreate, copy, strip `.git`), then rewrite the metadata with the new commit:
```bash
rm -rf <target_directory>/<name>
mkdir -p <target_directory>/<name>
cp -R "$tmp_dir/<parent_path>/." <target_directory>/<name>/
rm -rf <target_directory>/<name>/.git
echo "{\"commit\": \"$new_commit\", \"repo\": \"<org>/<repo>\", \"branch\": \"<branch>\", \"path\": \"<parent_path>\"}" > <target_directory>/.<name>.agentic-metadata.json
rm -rf "$tmp_dir"
```
- Agents and prompts are single files: copy just the file, as in `use`.
- Skills: re-verify the `.claude/skills/<name>` symlink as in `use` step 7 (never replace a real folder).
- MCP: only the files are refreshed; the harness registration is unchanged.

### 7. Resolve Dependencies
If an updated entry has `requires`, check each dependency is installed; install missing ones with the `use` steps. Process dependencies first.

### 8. Report Results
```
## Update Complete

| Type | Name | Status |
|------|------|--------|
| skill | grill-me | updated (a1b2c3d → e4f5a6b) |
| skill | adhd | already up to date |
| skill | other | skipped (local edits) |
| skill | broken | failed: <reason> |

Updated: X   Current: Y   Skipped: Z   Failed: W
```
Remind the user to **restart Claude Code** if any skill changed.
