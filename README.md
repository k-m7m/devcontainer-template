# 並列開発用 devcontainer テンプレート

特定の言語やフレームワークに依存しない devcontainer です。`git worktree` で作業ディレクトリを複数用意し、それぞれで Claude Code を動かして並列に開発できます。

## 必要なもの

- Docker（Docker Desktop など）
- VS Code と Dev Containers 拡張機能（または devcontainer CLI）
- ホストでも git を使う場合は **git 2.48 以上**（理由は「注意点」を参照）

## セットアップ

1. `.devcontainer/` と `.gitignore` を自分のリポジトリにコピーする
2. VS Code で「Dev Containers: Reopen in Container」を実行する
3. コンテナ内のターミナルで `claude` を実行してログインする（初回のみ）

ログイン情報は Docker の volume `claude-code-config` に保存されます。コンテナを作り直しても消えず、このテンプレートを使う全プロジェクトで共有されます。

## 並列開発の流れ

```sh
# worktree を作る（.worktrees/ 配下に置く）
git worktree add .worktrees/feat-a -b feat-a
git worktree add .worktrees/feat-b -b feat-b

# ターミナルを分けて、それぞれで Claude Code を起動する
cd .worktrees/feat-a && claude
cd .worktrees/feat-b && claude

# 一覧
git worktree list

# 作業が終わったら片付ける
git worktree remove .worktrees/feat-a
git branch -d feat-a
```

worktree は自動で**相対パス**で登録されます（コンテナ内で `worktree.useRelativePaths=true` を設定済み）。コンテナ内とホストでリポジトリのパスが違っても、worktree のリンクは壊れません。`.worktrees/` は `.gitignore` で除外しています。

## ホストの Claude Code 設定の引き継ぎ

コンテナを起動するたびに、ホストの `~/.claude` から次のものを自動でコピーします。

- `CLAUDE.md`
- `settings.json`
- `commands/`
- `skills/`
- `agents/`

補足:

- ホストにないものはスキップします。`~/.claude` 自体がなくても、コンテナは問題なく起動します
- 会話履歴（`history.jsonl`、`projects/` など）はコピーしません
- `~/.claude` の外を指すシンボリックリンク（例: `~/.agents/skills/...`）は、実体としてコピーします
- コピー処理は、使い捨ての `alpine` コンテナで行います。初回だけイメージ（約 4MB）をダウンロードします

## 言語やツールを追加する

`devcontainer.json` の `features` に追加します。一覧は [containers.dev/features](https://containers.dev/features) にあります。

```jsonc
"features": {
  "ghcr.io/devcontainers/features/python:1": {},
  "ghcr.io/devcontainers/features/go:1": {}
}
```

## 注意点

- **ホストの git が 2.48 未満だと、リポジトリを開けなくなります。** 相対パスの worktree を作ると、リポジトリの設定に `extensions.relativeWorktrees` が追加されるためです
- **コンテナ内で `settings.json` などを編集しても、次の起動時にホストの内容で上書きされます。** 設定の変更はホスト側で行ってください
- **plugins は引き継ぎません。** インストール先が絶対パスで記録されているためです。必要ならコンテナ内で `/plugin install` してください
- **hooks や statusline に OS 専用のコマンドがあると、コンテナ内では動きません**
- **Windows で環境変数 `HOME` と `USERPROFILE` の両方が設定されていると、ホームディレクトリのパスが壊れます。** その場合は `HOME` を外してください
- **ネットワーク制限（ファイアウォール）は入れていません。** `--dangerously-skip-permissions` で使う場合は、Anthropic の[参考実装](https://github.com/anthropics/claude-code/tree/main/.devcontainer)の `init-firewall.sh` を参考に追加してください
