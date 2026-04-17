---
name: worktree-setup
description: 任意のgitリポジトリに対してworktreeベースの並列開発環境を構築。共通設定ファイルの共有もサポート。
argument-hint: [リポジトリパス] [環境数]
allowed-tools: Bash(cd *), Bash(git rev-parse *), Bash(git worktree *), Bash(git show-ref *), Bash(mkdir *), Bash(cp *), Bash(ln *), Bash(ls *), Bash(find *), AskUserQuestion
---

# Worktree Setup Skill

任意の git リポジトリに対して `$HOME/.claude/worktrees/<org>/<repo>/` 配下に git worktree ベースの並列開発環境を構築するスキルです。共通の設定ファイル（.env 等）をシンボリックリンクで共有できます。

## ディレクトリ構成

```
$HOME/.claude/worktrees/<org>/<repo>/
├── shared_config/           # 共有設定ファイル
│   ├── .env
│   ├── .node-version
│   └── ...（プロジェクト固有のファイル）
├── working-a/               # worktree 1
│   ├── .env -> ../shared_config/.env
│   └── ...
├── working-b/               # worktree 2
│   └── ...
└── working-c/               # worktree 3
    └── ...
```

## 実行手順

### 1. 対象リポジトリの特定

引数でリポジトリパスが指定されていればそれを使用。指定されていなければ `AskUserQuestion` でヒアリング。

```bash
# 指定されたパスが git リポジトリか確認
cd <リポジトリパス>
git rev-parse --show-toplevel
```

リポジトリのルートパスから `<org>/<repo>` を算出する:
- リポジトリルートの親ディレクトリ名 → `<org>`
- リポジトリルートのディレクトリ名 → `<repo>`

例: `/Users/user/Works/x-asia/kauche-app` → `x-asia/kauche-app`

### 2. 環境数の確認

引数で環境数が指定されていればそれを使用。指定されていなければ `AskUserQuestion` でヒアリング:

- **質問**: "いくつの並列開発環境を作成しますか？"
- **header**: "環境数"
- **options**:
  1. label: "2つ (working-a, working-b)"
  2. label: "3つ (working-a, working-b, working-c)"
  3. label: "5つ (working-a 〜 working-e)"

Other を選択した場合は数字を入力してもらう。

環境名は `working-a`, `working-b`, ... `working-z` のようにアルファベット順。

### 3. worktree の作成

```bash
REPO_ROOT=$(cd <リポジトリパス> && git rev-parse --show-toplevel)
ORG=$(basename "$(dirname "$REPO_ROOT")")
REPO=$(basename "$REPO_ROOT")
WORKTREE_BASE="$HOME/.claude/worktrees/${ORG}/${REPO}"

mkdir -p "$WORKTREE_BASE"
mkdir -p "$WORKTREE_BASE/shared_config"

cd "$REPO_ROOT"

# 環境数に応じてworktreeを作成（a, b, c, ...）
# 例: 3環境の場合
git worktree add "$WORKTREE_BASE/working-a" -b worktree/working-a
git worktree add "$WORKTREE_BASE/working-b" -b worktree/working-b
git worktree add "$WORKTREE_BASE/working-c" -b worktree/working-c

git worktree list
```

**注意:** ブランチ名は `worktree/working-a` のようにプレフィックスを付ける。既にブランチやworktreeが存在する場合はスキップして警告する。

### 4. 共有設定ファイルのセットアップ

`AskUserQuestion` で共有したいファイルをヒアリング:

- **質問**: "共有したい設定ファイルはありますか？（.env, .node-version, .go-version 等）"
- **header**: "共有ファイル"
- **options**:
  1. label: "自動検出", description: "リポジトリ内の .env, .*-version 等を自動検出します"
  2. label: "手動指定", description: "共有するファイルパスを手動で指定します"
  3. label: "スキップ", description: "共有設定ファイルなしで進めます"

#### 自動検出の場合

リポジトリルートで以下を探索:
```bash
cd "$REPO_ROOT"
# .env 系
find . -maxdepth 3 -name ".env" -o -name ".env.local" -o -name ".env.development" | head -20
# バージョン系
find . -maxdepth 1 -name ".*-version" -o -name ".tool-versions" | head -10
# その他（local.properties 等）
find . -maxdepth 3 -name "local.properties" | head -10
```

検出したファイルの一覧をユーザーに提示し、どれを共有するか確認する。

#### 手動指定の場合

`AskUserQuestion` でファイルパスを入力してもらう（カンマ区切りまたは複数回入力）。

#### ファイルのコピーとシンボリックリンク作成

共有対象の各ファイルについて:

```bash
# 1. メインリポジトリからshared_configにコピー
# ファイルがリポジトリルート直下の場合
cp "$REPO_ROOT/.env" "$WORKTREE_BASE/shared_config/.env"

# ファイルがサブディレクトリにある場合（ディレクトリ構造を維持）
mkdir -p "$WORKTREE_BASE/shared_config/path/to/"
cp "$REPO_ROOT/path/to/file" "$WORKTREE_BASE/shared_config/path/to/file"

# 2. 各worktreeからshared_configへのシンボリックリンクを作成
# ルート直下のファイルの場合: ../shared_config/<file>
# サブディレクトリの場合: 適切な深さの相対パス

for name in working-a working-b working-c; do
  WORKTREE_PATH="$WORKTREE_BASE/$name"
  
  # ルート直下のファイル
  ln -sf "../shared_config/.env" "$WORKTREE_PATH/.env"
  
  # サブディレクトリのファイル（例: path/to/file）
  mkdir -p "$WORKTREE_PATH/path/to/"
  # 相対パスを計算: ../../.. + /shared_config/path/to/file
  ln -sf "../../../shared_config/path/to/file" "$WORKTREE_PATH/path/to/file"
done
```

**相対パスの計算ルール:**
- worktree ルート直下 → `../shared_config/<file>`
- 1階層下 → `../../shared_config/<dir>/<file>`
- 2階層下 → `../../../shared_config/<dir1>/<dir2>/<file>`
- 一般化: ファイルの深さ分の `../` + `shared_config/` + ファイルの相対パス

### 5. セットアップ完了の報告

以下を確認・報告:

```bash
git worktree list

# 各worktreeのシンボリックリンクを確認
for name in working-a working-b working-c; do
  echo "=== $name ==="
  find "$WORKTREE_BASE/$name" -type l -exec ls -la {} \;
done
```

ユーザーに以下を報告:

1. **作成された環境一覧** - パスとブランチ名
2. **共有設定ファイル** - shared_config 内のファイル一覧
3. **使い方** - 各環境の利用方法

```
## 作成された環境

| 環境 | パス | ブランチ |
|------|------|---------|
| working-a | $HOME/.claude/worktrees/<org>/<repo>/working-a | worktree/working-a |
| working-b | $HOME/.claude/worktrees/<org>/<repo>/working-b | worktree/working-b |
| working-c | $HOME/.claude/worktrees/<org>/<repo>/working-c | worktree/working-c |

## 使い方

各環境で Claude Code を起動:
  cd $HOME/.claude/worktrees/<org>/<repo>/working-a && claude

または EnterWorktree で切り替え可能です。
```

## 使用例

```bash
# 対話的にセットアップ
/worktree-setup

# リポジトリパスを指定
/worktree-setup /Users/user/Works/x-asia/kauche-app

# リポジトリパスと環境数を指定
/worktree-setup /Users/user/Works/x-asia/kauche-app 5
```

## 注意事項

- 既に同名の worktree やブランチが存在する場合はスキップして警告
- shared_config のファイルはコピー（シンボリックリンクではない）なので、元リポジトリの変更は自動反映されない
- worktree を削除するには `git worktree remove <path>` を使用
- すべての worktree を削除するには各 worktree を remove した後、`$HOME/.claude/worktrees/<org>/<repo>/` を削除
