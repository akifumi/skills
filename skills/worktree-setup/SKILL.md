---
name: worktree-setup
description: Set up parallel Git worktree-based development environments with shared config files via symlinks.
argument-hint: [repo-path] [num-environments]
allowed-tools: Bash(cd *), Bash(git rev-parse *), Bash(git worktree *), Bash(git show-ref *), Bash(mkdir *), Bash(cp *), Bash(ln *), Bash(ls *), Bash(find *), AskUserQuestion
---

# Worktree Setup Skill

Set up Git worktree-based parallel development environments under `$HOME/.claude/worktrees/<org>/<repo>/`. Shared config files (e.g. `.env`) are managed via symlinks through a `shared_config/` directory, so editing one file updates all worktrees at once.

## Directory Structure

```
$HOME/.claude/worktrees/<org>/<repo>/
├── shared_config/           # Shared config files (source of truth)
│   ├── .env
│   ├── .node-version
│   └── ...
├── working-a/               # worktree 1
│   ├── .env -> ../shared_config/.env
│   └── ...
├── working-b/               # worktree 2
│   └── ...
└── working-c/               # worktree 3
    └── ...
```

## Steps

### 1. Identify the target repository

Use the repo path from arguments if provided. Otherwise, ask with `AskUserQuestion`.

```bash
cd <repo-path>
git rev-parse --show-toplevel
```

Derive `<org>/<repo>` from the repository root path:
- Parent directory name of the repo root → `<org>`
- Directory name of the repo root → `<repo>`

Example: `/Users/user/Works/acme/my-app` → `acme/my-app`

### 2. Confirm number of environments

Use the count from arguments if provided. Otherwise, ask with `AskUserQuestion`:

- **Question**: "How many parallel environments do you want?"
- **Options**:
  1. "2 (working-a, working-b)"
  2. "3 (working-a, working-b, working-c)"
  3. "5 (working-a through working-e)"

If Other is selected, accept a numeric input.

Environment names follow alphabetical order: `working-a`, `working-b`, ... `working-z`.

### 3. Create worktrees

```bash
REPO_ROOT=$(cd <repo-path> && git rev-parse --show-toplevel)
ORG=$(basename "$(dirname "$REPO_ROOT")")
REPO=$(basename "$REPO_ROOT")
WORKTREE_BASE="$HOME/.claude/worktrees/${ORG}/${REPO}"

mkdir -p "$WORKTREE_BASE"
mkdir -p "$WORKTREE_BASE/shared_config"

cd "$REPO_ROOT"

# Create worktrees for the requested count (e.g. 3)
git worktree add "$WORKTREE_BASE/working-a" -b worktree/working-a
git worktree add "$WORKTREE_BASE/working-b" -b worktree/working-b
git worktree add "$WORKTREE_BASE/working-c" -b worktree/working-c

git worktree list
```

**Note:** Branch names use the `worktree/` prefix. Skip and warn if a branch or worktree already exists.

### 4. Set up shared config files

Ask with `AskUserQuestion` which files to share:

- **Question**: "Which config files do you want to share? (e.g. .env, .node-version, .go-version)"
- **Options**:
  1. "Auto-detect" — Scan the repo for .env, .*-version, etc.
  2. "Specify manually" — Enter file paths manually
  3. "Skip" — Proceed without shared config

#### Auto-detect

Search from the repository root:
```bash
cd "$REPO_ROOT"
find . -maxdepth 3 -name ".env" -o -name ".env.local" -o -name ".env.development" | head -20
find . -maxdepth 1 -name ".*-version" -o -name ".tool-versions" | head -10
find . -maxdepth 3 -name "local.properties" | head -10
```

Present the detected files to the user and confirm which ones to share.

#### Manual specification

Ask with `AskUserQuestion` for file paths (comma-separated or multiple inputs).

#### Copy files and create symlinks

For each shared file:

```bash
# 1. Copy from the main repo to shared_config
# Root-level file:
cp "$REPO_ROOT/.env" "$WORKTREE_BASE/shared_config/.env"

# Subdirectory file (preserve directory structure):
mkdir -p "$WORKTREE_BASE/shared_config/path/to/"
cp "$REPO_ROOT/path/to/file" "$WORKTREE_BASE/shared_config/path/to/file"

# 2. Create symlinks from each worktree to shared_config
for name in working-a working-b working-c; do
  WORKTREE_PATH="$WORKTREE_BASE/$name"
  
  # Root-level file
  ln -sf "../shared_config/.env" "$WORKTREE_PATH/.env"
  
  # Subdirectory file (e.g. path/to/file)
  mkdir -p "$WORKTREE_PATH/path/to/"
  ln -sf "../../../shared_config/path/to/file" "$WORKTREE_PATH/path/to/file"
done
```

**Relative path rules:**
- Root level → `../shared_config/<file>`
- 1 level deep → `../../shared_config/<dir>/<file>`
- 2 levels deep → `../../../shared_config/<dir1>/<dir2>/<file>`
- General: `../` repeated for the file depth + `shared_config/` + relative file path

### 5. Report completion

Verify and report:

```bash
git worktree list

for name in working-a working-b working-c; do
  echo "=== $name ==="
  find "$WORKTREE_BASE/$name" -type l -exec ls -la {} \;
done
```

Report to the user:

1. **Created environments** — paths and branch names
2. **Shared config files** — list of files in shared_config
3. **Usage** — how to use each environment

```
## Created Environments

| Environment | Path | Branch |
|-------------|------|--------|
| working-a | $HOME/.claude/worktrees/<org>/<repo>/working-a | worktree/working-a |
| working-b | $HOME/.claude/worktrees/<org>/<repo>/working-b | worktree/working-b |
| working-c | $HOME/.claude/worktrees/<org>/<repo>/working-c | worktree/working-c |

## Usage

Launch Claude Code in each environment:
  cd $HOME/.claude/worktrees/<org>/<repo>/working-a && claude
```

## Usage Examples

```bash
# Interactive setup
/worktree-setup

# Specify number of environments
/worktree-setup 3
```

## Notes

- If a worktree or branch with the same name already exists, skip and warn
- Files in shared_config are copies from the original repo (not symlinks), so changes in the original repo are not auto-reflected
- To remove a worktree: `git worktree remove <path>`
- To remove all: remove each worktree, then delete `$HOME/.claude/worktrees/<org>/<repo>/`
