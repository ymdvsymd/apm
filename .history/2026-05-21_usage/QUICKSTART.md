# APM — クイックスタート

**生成日:** 2026-05-21
**対象バージョン:** apm-cli v0.14.1
**所要時間:** 約 5–10 分

ここではインストールから最初の `apm install` まで、最短経路を案内する。GitHub Copilot を使う場合の例で進める。他のエージェント（Claude Code / Cursor / Codex / Gemini / Windsurf / OpenCode）でもコマンドは同じで、ターゲット名だけ差し替えれば動く。

---

## 前提条件

| 要件 | 確認コマンド |
|------|-------------|
| Python 3.10 以上（pip 経由でインストールする場合） | `python3 --version` |
| git | `git --version` |
| 任意の AI エージェントクライアント（VS Code / Claude Code / Cursor 等） | — |
| GitHub アカウント（公開パッケージのみ使うなら token は不要） | — |

OS は macOS / Linux / Windows x86_64 が公式サポート対象。

---

## Step 1: インストール

任意の 1 つを選ぶ。再現性が必要な CI では pip を推奨。

### macOS / Linux

```bash
# ワンライナー（公式インストーラ）
curl -sSL https://aka.ms/apm-unix | sh

# Homebrew
brew install microsoft/apm/apm

# pip
pip install apm-cli
```

### Windows (PowerShell)

```powershell
# 公式インストーラ
irm https://aka.ms/apm-windows | iex

# Scoop
scoop bucket add apm https://github.com/microsoft/scoop-apm
scoop install apm

# pip
pip install apm-cli
```

### 動作確認

```bash
apm --version
```

`apm-cli v0.14.1` のようにバージョンが表示されれば成功。

---

## Step 2: プロジェクトを初期化する

新規ディレクトリを作って `apm init` する。対象エージェント（target）はあとから変えられるので、ここでは Copilot を選ぶ。

```bash
mkdir my-agent-context && cd my-agent-context
apm init --yes --target copilot
```

生成される主なファイル:

```
my-agent-context/
├── apm.yml          # マニフェスト（コミット対象）
├── .gitignore
└── .github/         # コンパイル出力先
```

`apm.yml` を見ると次のような骨組みになっている:

```yaml
name: my-agent-context
version: 0.1.0
description: ...
target: copilot
dependencies:
  apm: []
  mcp: []
scripts: {}
```

> **複数ターゲットを使う場合**: `apm init --target copilot,claude,cursor` のようにカンマ区切りで指定する。`--target all` で全部入りになる。

---

## Step 3: 認証（パブリックパッケージのみなら省略可）

公開リポジトリのパッケージだけを使うなら不要。プライベートリポジトリにアクセスする場合は以下のいずれか:

```bash
# GitHub: gh CLI が入っていれば自動使用される
gh auth login

# あるいは環境変数を設定
export GITHUB_APM_PAT="ghp_..."

# Azure DevOps の場合
export ADO_APM_PAT="..."
```

詳細は [CONFIGURATION.md#認証](./CONFIGURATION.md#認証) を参照。

---

## Step 4: サンプルパッケージをインストールする

公式サンプルを入れて挙動を確認する。

```bash
apm install microsoft/apm-sample-package
```

実行結果（抜粋）:

- `apm_modules/microsoft/apm-sample-package/` にパッケージが取得される
- `.github/instructions/` などターゲット別のディレクトリに skills / instructions が配置される
- `apm.yml` の `dependencies.apm:` にエントリが追加される
- `apm.lock.yaml` が生成・更新される（コミット対象）

`--dry-run` を付ければ何が起こるかを事前に確認できる:

```bash
apm install microsoft/apm-sample-package --dry-run
```

---

## Step 5: コンパイルしてエージェントに読み込ませる

Copilot ターゲットなら、`.github/copilot-instructions.md` の生成までやって初めて VS Code の Copilot が読む。

```bash
apm compile -t copilot
```

VS Code でこのリポジトリを開き直すと、Copilot が `.github/copilot-instructions.md` を自動的にコンテキストに取り込む。Claude Code を使っているなら `CLAUDE.md` が、Gemini なら `GEMINI.md` が生成される（`-t claude` / `-t gemini`）。

複数ターゲットを一度にコンパイルする場合:

```bash
apm compile -t copilot,claude,cursor
```

---

## Step 6: 配置を確認する

何が入ったかを見るコマンドが 3 つある。

```bash
# どのターゲットがアクティブか
apm targets

# インストール済みパッケージ一覧
apm view

# 個別パッケージの中身
apm view microsoft/apm-sample-package
```

`apm targets --json` で機械可読出力にもできる。

---

## Step 7: 監査する

セキュリティチェック（隠し Unicode、lockfile 整合性、drift）を回す。

```bash
apm audit
```

CI で使う場合は `--ci` を付けると drift / lockfile inconsistency で exit 1 になる。

```bash
apm audit --ci --format sarif --output apm-audit.sarif
```

---

## ここまで来たら

| 次にやりたいこと | 進むページ |
|---------------|----------|
| MCP サーバを追加したい | [USE-CASES.md#mcp-サーバを追加する](./USE-CASES.md#シナリオ-4-mcp-サーバを-1-コマンドで全クライアントに追加する) |
| 自前の plugin を作って配布したい | [USE-CASES.md#plugin-を作って-marketplace-に発行する](./USE-CASES.md#シナリオ-5-plugin-を作って-marketplace-に発行する) |
| 組織ポリシーを敷きたい | [USE-CASES.md#組織ポリシーを段階導入する](./USE-CASES.md#シナリオ-6-組織ポリシーを段階導入する) |
| 全コマンドを把握したい | [COMMANDS.md](./COMMANDS.md) |
| 設定のチューニングをしたい | [CONFIGURATION.md](./CONFIGURATION.md) |
| 詰まったとき | [TROUBLESHOOTING.md](./TROUBLESHOOTING.md) |
