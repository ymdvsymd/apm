# APM — 設定ガイド

**生成日:** 2026-05-21
**対象バージョン:** apm-cli v0.14.1 / apm.yml schema v0.10.0

APM の挙動は次の 4 レイヤーで決まる:

1. 環境変数（最優先）
2. プロジェクトの `apm.yml` / `apm.lock.yaml` / `apm-policy.yml`
3. ユーザーグローバル設定 `~/.apm/config.json`
4. CLI のハードコード既定値

このページは各レイヤーで設定できる項目を網羅する。

---

## 設定ファイルの一覧

| スコープ | パス | 形式 | 用途 |
|---------|------|------|------|
| プロジェクト | `<repo>/apm.yml` | YAML | マニフェスト（コミット対象） |
| プロジェクト | `<repo>/apm.lock.yaml` | YAML | ロックファイル（コミット対象） |
| プロジェクト or org | `<path>/apm-policy.yml` | YAML | ポリシー（コミット対象、tighten-only 継承） |
| ユーザー | `~/.apm/config.json` | JSON | CLI のグローバル設定 |
| ユーザー | `~/.apm/` | ディレクトリ | グローバル install / キャッシュ |
| ユーザー（MCP） | クライアントごとに違うパス | JSON | MCP server 設定（[後述](#mcp-クライアント別-config-ファイル)） |

---

## apm.yml（マニフェスト）

`apm.yml` はプロジェクトのルートに置き、依存と target を宣言する。

### 主要フィールド

| キー | 型 | 必須 | 既定 | 説明 |
|------|----|----|-----|------|
| `name` | string | 必須 | — | プロジェクト名 |
| `version` | string | 必須 | — | semver |
| `description` | string | 任意 | "" | 説明 |
| `author` | string | 任意 | — | 作者 |
| `license` | string | 任意 | — | ライセンス |
| `target` | string \| list \| `all` | 任意 | 自動検出 | 対象エージェントクライアント |
| `includes` | string \| list | 任意 | `auto` | コンパイル対象パス。`auto` で `.apm/` 配下を全自動検出 |
| `dependencies.apm` | list | 任意 | `[]` | APM パッケージ依存 |
| `dependencies.mcp` | list | 任意 | `[]` | MCP server 依存 |
| `scripts` | mapping | 任意 | `{}` | `apm run` で実行する名前付きスクリプト |
| `marketplace` | mapping | 任意 | — | Marketplace authoring 用（[後述](#marketplace-authoring-用フィールド)） |
| `policy` | mapping | 任意 | — | プロジェクト側ポリシーオーバーライド |

### target の指定

`target` フィールドはコンパイル先のエージェントクライアントを決める。

```yaml
target: copilot                                    # 単一
target: [copilot, claude, cursor]                  # リスト
target: all                                        # 全部
target: "copilot,claude"                           # カンマ区切り文字列
```

利用可能なターゲット: `copilot`, `claude`, `cursor`, `opencode`, `codex`, `gemini`, `windsurf`, `agent-skills`, `copilot-cowork`（実験的）, `copilot-app`（実験的）。

`target` を省略した場合、APM は次の順序で検出する:

1. `apm.yml` の `target:` 明示指定
2. プロジェクトに存在するルートディレクトリ（`.github/` / `.claude/` 等）
3. フォールバックで `copilot`

### dependencies.apm の書き方

```yaml
dependencies:
  apm:
    # shorthand
    - microsoft/apm-sample-package
    # 版指定
    - microsoft/apm-sample-package#v1.0.0
    # 個別の primitive
    - github/awesome-copilot/agents/api-architect.agent.md
    # marketplace
    - my-plugin@my-marketplace
    # object 形式（詳細指定）
    - name: vercel-labs/agent-skills
      version: ">=1.0.0 <2.0.0"
      ref: "abc123sha"
      subdir: skills/deploy-to-vercel
    # ローカルパス
    - path: ./local-package
    # HTTPS 直
    - https://github.com/owner/repo
    # SSH 直
    - git@github.com:owner/repo.git
    # Azure DevOps
    - https://dev.azure.com/myorg/myproject/_git/myrepo
    # Virtual mono-repo（同一リポの兄弟 subdir）
    - git: parent
      subdir: skills/another-skill
```

### dependencies.mcp の書き方

```yaml
dependencies:
  mcp:
    - name: io.github.github/github-mcp-server
      transport: http
    - name: my-org/jira-mcp
      transport: stdio
      command: ["python", "-m", "jira_mcp"]
      env:
        JIRA_URL: "${JIRA_URL}"
```

各 transport の意味:

| Transport | 用途 |
|-----------|------|
| `stdio` | 標準入出力（プロセス起動） |
| `http` | HTTP/HTTPS（推奨） |
| `sse` | Server-Sent Events |
| `streamable-http` | Streamable HTTP |

### scripts の書き方

```yaml
scripts:
  start: ".apm/prompts/welcome.prompt.md"
  release-notes: ".apm/prompts/release.prompt.md"
  review:
    prompt: ".apm/prompts/review.prompt.md"
    params:
      max_tokens: 4000
      model: claude-sonnet-4
```

`apm run release-notes --param version=1.2.0` のように呼び出す。

### marketplace authoring 用フィールド

`apm marketplace init` で書かれる。

```yaml
marketplace:
  name: awesome-prompts
  owner: my-org
  packages:
    - name: release-notes
      source: https://github.com/my-org/release-notes-plugin
      version: ">=1.0.0 <2.0.0"
      tag_pattern: "v*.*.*"
      tags: [release, docs]
```

---

## apm.lock.yaml（ロックファイル）

`apm install` が解決済み依存をピンする。コミット対象。手で編集しない。

### 主要フィールド

| キー | 説明 |
|------|------|
| `lockfile_version` | ロックファイルのスキーマ版 |
| `generated_at` | 生成日時 |
| `dependencies` | 解決済み依存のリスト |

各エントリの主要フィールド:

| キー | 説明 |
|------|------|
| `name` | パッケージ名 |
| `source` | 解決元 URL |
| `ref` | 解決された具象 ref（SHA） |
| `version` | 一致した semver |
| `hash` | コンテンツハッシュ（既定 sha256） |
| `hash_algorithm` | `sha256` / `sha384` / `sha512` |
| `transitive_deps` | 推移依存のリスト |

### Frozen インストール

```bash
apm install --frozen
```

`apm.lock.yaml` に書かれた ref と hash と完全一致でなければ install を失敗させる。CI 推奨。

---

## apm-policy.yml（ポリシー）

セキュリティチームが組織横断のルールを書くファイル。階層継承は **enterprise → org → repo** の tighten-only。下位は厳しくしかできない。

### 主要フィールド

| キー | 型 | 説明 |
|------|----|------|
| `name` | string | ポリシー名 |
| `version` | string | semver |
| `enforcement.level` | `warn` \| `block` | 違反時の動作 |
| `enforcement.bypass_label` | string | PR ラベルで例外承認するためのラベル名 |
| `dependencies.allowed_sources` | list | 許可するソース（glob 可） |
| `dependencies.denied_sources` | list | 明示拒否するソース |
| `mcp.allowed_servers` | list | 許可する MCP server 名 |
| `mcp.denied_servers` | list | 拒否する MCP server 名 |
| `primitives.allowed_types` | list | 許可する primitive 種別 |
| `policy.fetch_failure_default` | `allow` \| `deny` | 解決失敗時の挙動 |
| `policy.hash` | string | 自身の hash（改ざん検出用） |
| `policy.hash_algorithm` | string | `sha256` / `sha384` / `sha512` |

### 例

```yaml
name: my-org-policy
version: 1.0.0
enforcement:
  level: warn
  bypass_label: policy-bypass-approved
dependencies:
  allowed_sources:
    - github.com/my-org/*
    - github.com/microsoft/apm-sample-package
  denied_sources:
    - github.com/untrusted/*
mcp:
  allowed_servers:
    - io.github.github/github-mcp-server
primitives:
  allowed_types:
    - skills
    - instructions
    - agents
```

### 配置場所と発見

- 組織 org-level repo（テンプレートリポジトリ等）に置く
- プロジェクトの `apm install` / `apm audit` は自動的にポリシーを発見する
- 階層継承で複数のポリシーソースをマージする

---

## ~/.apm/config.json（グローバル設定）

`apm config` で読み書きする CLI 設定。

### 主要キー

| キー | 型 | 既定 | 説明 |
|------|----|-----|------|
| `auto-integrate` | bool | `true` | パッケージ取得後に自動でターゲット統合するか |
| `temp-dir` | path | OS 既定 | 一時ディレクトリ |
| `copilot-cowork-skills-dir` | path | OneDrive 動的解決 | Copilot Cowork skills の配置ルート |
| `experimental.*` | bool | `false` | 実験機能フラグ（`apm experimental` で操作） |

### 操作

```bash
apm config                              # 全設定を表示
apm config get auto-integrate           # 値を取得
apm config set auto-integrate false     # 値を設定
apm config unset auto-integrate         # 値を削除
```

---

## 環境変数

APM は次の環境変数を解釈する。`os.environ` / `os.getenv` で参照されている主要な変数を分類して掲載する。

### キャッシュ

| 変数 | 用途 | 既定 |
|------|------|------|
| `APM_NO_CACHE` | `1` でキャッシュ機構全体を無効化 | unset |
| `APM_CACHE_DIR` | キャッシュディレクトリ | `~/.cache/apm/` |
| `APM_BROAD_FETCH_DEPTH` | git fetch の depth | 1 |

### 一時ファイル

| 変数 | 用途 |
|------|------|
| `APM_TEMP_DIR` | 一時ディレクトリ。`apm config set temp-dir` と等価 |

### 振る舞い

| 変数 | 用途 |
|------|------|
| `APM_VERBOSE` | `1` で全コマンドで verbose 出力 |
| `APM_DEBUG` | `1` でデバッグログを出す |
| `APM_E2E_TESTS` | `1` で E2E テストを有効化（開発用） |
| `APM_RUN_INTEGRATION_TESTS` | `1` で integration テストを有効化（開発用） |
| `APM_RUN_INFERENCE_TESTS` | `1` で実モデル呼び出しテストを有効化（開発用） |
| `APM_TEST_ADO_BEARER` | `1` で `az` CLI 認証を ADO テストに使う（開発用） |

### Resolver

| 変数 | 用途 |
|------|------|
| `APM_RESOLVE_PARALLEL` | 解決の並列度 |
| `APM_TIERED_RESOLVER` | tiered resolver の有効/無効 |
| `APM_LEGACY_SKILL_PATHS` | レガシーな skill パスを許可 |

### ポリシー

| 変数 | 用途 |
|------|------|
| `APM_POLICY_DISABLE` | `1` でポリシー強制を全面無効化（ローカル緊急回避用） |

### MCP

| 変数 | 用途 |
|------|------|
| `MCP_REGISTRY_URL` | カスタム MCP registry の URL |
| `MCP_REGISTRY_ALLOW_HTTP` | `1` で http の registry を許可（既定は https のみ） |

### Application config

| 変数 | 用途 |
|------|------|
| `APM_COPILOT_APP_DB` | Copilot Desktop App の SQLite DB パス |
| `APM_COPILOT_COWORK_SKILLS_DIR` | Copilot Cowork skills ディレクトリの明示指定 |
| `CLAUDE_CONFIG_DIR` | Claude Code のユーザ設定ディレクトリ |

### 認証

| 変数 | 用途 |
|------|------|
| `GITHUB_HOST` | GitHub Enterprise のホスト名 |
| `GITHUB_APM_PAT` | APM 専用 GitHub PAT |
| `GITHUB_TOKEN` | 汎用 GitHub token（`gh auth token` も自動使用される） |
| `ADO_APM_PAT` | Azure DevOps PAT |
| `GITLAB_TOKEN` | GitLab token |

`gh` CLI でログインしていれば、APM は zero-config で token を借りる。

### CI 検出

| 変数 | 検出ベンダー |
|------|------------|
| `CI` | 汎用 |
| `GITHUB_ACTIONS` | GitHub Actions |
| `TRAVIS` | Travis CI |
| `JENKINS_URL` | Jenkins |
| `BUILDKITE` | Buildkite |

CI が検出されると非対話モードに切り替わり、`--yes` を要求する。

---

## ターゲット指定

優先順位は次の通り:

1. CLI フラグ `--target` / `-t` （最優先）
2. `apm.yml` の `target:` フィールド
3. プロジェクト内のルートディレクトリ存在検出
4. フォールバック `[copilot]`

エイリアスとして `vscode` → `copilot`、`agents` → `copilot` が解決される。

---

## MCP クライアント別 config ファイル

`apm install --mcp` が書き込む先:

| クライアント | 設定ファイル | スコープ | env 解決 |
|------------|------------|--------|---------|
| Copilot CLI | `~/.copilot/mcp-config.json` | user | **runtime** `${VAR}` 解決（disk には placeholder のまま） |
| Claude Code | `.mcp.json`（project）→ `~/.claude.json`（user fallback） | both | install 時に環境変数を解決 |
| Cursor | `.cursor/mcp.json` | project only | install 時 |
| Windsurf | `.windsurf/mcp_config.json` | project | install 時 |

Copilot のみ runtime substitution に対応しているため、secret を disk に書き出さずに済む。

---

## 認証

### GitHub

優先順:
1. `gh auth token`（`gh` CLI でログイン済みの場合）
2. `GITHUB_APM_PAT`
3. `GITHUB_TOKEN`

GitHub Enterprise (GHES) は `GITHUB_HOST=github.example.com` で対応。

### Azure DevOps

- `ADO_APM_PAT` を環境変数に設定
- `APM_TEST_ADO_BEARER=1` を立てれば `az` CLI の bearer token を使う

### GitLab

- `GITLAB_TOKEN` を環境変数に設定

---

## 設定例集

### 個人開発者向け（Copilot のみ）

```yaml
name: my-project
version: 0.1.0
target: copilot
dependencies:
  apm:
    - microsoft/apm-sample-package
  mcp: []
scripts:
  start: .apm/prompts/welcome.prompt.md
```

### マルチクライアント

```yaml
name: my-context
version: 1.0.0
target: [copilot, claude, cursor, codex]
dependencies:
  apm:
    - vercel-labs/agent-skills
    - github/awesome-copilot/plugins/context-engineering
  mcp:
    - name: io.github.github/github-mcp-server
      transport: http
```

### Plugin プロジェクト

```yaml
name: release-notes-plugin
version: 1.2.0
target: [copilot, claude, cursor]
plugin: true
dependencies:
  apm: []
  mcp: []
includes: auto
```

### Marketplace authoring

```yaml
name: my-marketplace
version: 0.1.0
marketplace:
  name: awesome-prompts
  owner: my-org
  versioning_strategy: tag_pattern
  packages:
    - name: release-notes
      source: https://github.com/my-org/release-notes-plugin
      version: ">=1.0.0 <2.0.0"
      tag_pattern: "v*.*.*"
```

---

## 関連ドキュメント

- 各コマンドでの設定の参照方法は [COMMANDS.md](./COMMANDS.md)
- 認証エラーや invalid YAML の対処は [TROUBLESHOOTING.md](./TROUBLESHOOTING.md)
