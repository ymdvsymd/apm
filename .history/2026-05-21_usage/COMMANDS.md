# APM — コマンドリファレンス

**生成日:** 2026-05-21
**対象バージョン:** apm-cli v0.14.1

ここでは `apm` CLI が提供する全コマンドを網羅する。コマンドは大別して **依存管理**、**コンパイル/実行**、**Marketplace / Plugin**、**MCP**、**監査・ポリシー**、**設定・運用** の 6 グループに分類できる。

実装は `src/apm_cli/cli.py` と `src/apm_cli/commands/` 配下にある。

---

## コマンド一覧

| コマンド | グループ | 用途 |
|---------|---------|------|
| [`apm init`](#apm-init) | 依存管理 | プロジェクトを初期化する |
| [`apm install`](#apm-install) | 依存管理 | パッケージと MCP server をインストールする |
| [`apm update`](#apm-update) | 依存管理 | 依存を最新の合致 ref に更新する |
| [`apm uninstall`](#apm-uninstall) | 依存管理 | パッケージを削除する |
| [`apm prune`](#apm-prune) | 依存管理 | `apm.yml` にない孤立パッケージを掃除する |
| [`apm outdated`](#apm-outdated) | 依存管理 | 更新可能なパッケージを表示する |
| [`apm deps`](#apm-deps) | 依存管理 | 依存ツリーを可視化する |
| [`apm view`](#apm-view) | 依存管理 | パッケージのメタデータを表示する |
| [`apm list`](#apm-list) | 実行 | `apm.yml` のスクリプト一覧を表示する |
| [`apm run`](#apm-run) | 実行 | `apm.yml` のスクリプトを実行する |
| [`apm preview`](#apm-preview) | 実行 | プロンプトを実行せずに展開だけ確認する |
| [`apm compile`](#apm-compile) | コンパイル | エージェント別の集約ファイルを生成する |
| [`apm pack` / `unpack`](#apm-pack--apm-unpack) | コンパイル | bundle / plugin として配布形式化する |
| [`apm marketplace`](#apm-marketplace) | Marketplace | Marketplace の登録・参照・発行 |
| [`apm search`](#apm-search) | Marketplace | Marketplace 横断で plugin を検索する |
| [`apm plugin`](#apm-plugin) | Plugin | plugin プロジェクトをスカフォルドする |
| [`apm mcp`](#apm-mcp) | MCP | MCP server を発見・設定・インストールする |
| [`apm audit`](#apm-audit) | 監査 | 隠し Unicode と drift を検出する |
| [`apm policy`](#apm-policy) | ポリシー | ポリシーの解決状態を診断する |
| [`apm targets`](#apm-targets) | 設定 | 解決された対象エージェントを表示する |
| [`apm config`](#apm-config) | 設定 | グローバル設定を読み書きする |
| [`apm cache`](#apm-cache) | 運用 | ローカルキャッシュを操作する |
| [`apm experimental`](#apm-experimental) | 運用 | 実験的機能フラグを切り替える |
| [`apm runtime`](#apm-runtime) | 運用 | エージェント runtime をセットアップする |
| [`apm self-update`](#apm-self-update) | 運用 | apm CLI 自体を更新する |

`--version` で apm のバージョンを表示する。

---

## 共通オプション

ほぼ全てのコマンドで使える定形オプション:

| オプション | 説明 |
|----------|------|
| `--verbose` / `-v` | 詳細ログを出す |
| `--dry-run` | 実行せず計画だけ表示する（install / update / uninstall / prune / audit / publish 等） |
| `--yes` / `-y` | 対話プロンプトをスキップする |
| `--global` / `-g` | プロジェクトではなくユーザースコープ（`~/.apm/`）を対象にする |
| `--target` / `-t` | カンマ区切りでエージェントターゲットを指定する |
| `--json` | JSON 形式で出力する（対応コマンドのみ） |

---

## apm init

新規プロジェクトを初期化する。`apm.yml` と必要なディレクトリを作成し、対象エージェントを検出または指定する。

### 構文

```
apm init [PROJECT_NAME] [OPTIONS]
```

### オプション

| フラグ | 説明 | デフォルト |
|-------|------|-----------|
| `PROJECT_NAME` | プロジェクト名（位置引数） | カレントディレクトリ名から推定 |
| `--yes` / `-y` | 対話プロンプトをスキップ | `false` |
| `--target` / `-t` | カンマ区切りのターゲット（`copilot,claude,cursor,opencode,codex,gemini,windsurf` または `all`） | 自動検出 |
| `--verbose` / `-v` | 詳細出力 | `false` |
| `--plugin` | (非推奨) plugin プロジェクトとして初期化。代わりに [`apm plugin init`](#apm-plugin) を使う | — |
| `--marketplace` | (非推奨) marketplace authoring を初期化。代わりに [`apm marketplace init`](#apm-marketplace-init) | — |

### 使用例

```bash
# 既存リポジトリで対話的に初期化
apm init

# 非対話的に Copilot + Claude をターゲットに初期化
apm init my-project --yes --target copilot,claude

# すべてのターゲットを有効に
apm init --yes --target all
```

### 注意

- 既存の `apm.yml` がある場合は確認なしには上書きしない
- target の自動検出は各ルートディレクトリの存在 (`.github/` / `.claude/` 等) を見ている

---

## apm install

`apm.yml` に宣言された、または引数で指定されたパッケージを取得しデプロイする。MCP server もここで設定される。

### 構文

```
apm install [PACKAGES...] [OPTIONS]
```

### オプション

| フラグ | 説明 | デフォルト |
|-------|------|-----------|
| `PACKAGES` | インストールする 1 つ以上のパッケージ参照 | `apm.yml` の全依存 |
| `--update` | すべての依存を最新の合致 ref に更新 | `false` |
| `--dry-run` | 実行せず計画のみ表示 | `false` |
| `--verbose` / `-v` | パイプラインの詳細診断を出力 | `false` |
| `--force` | 既存の取得物を破棄して再インストール | `false` |
| `--global` / `-g` | ユーザースコープ (`~/.apm/`) にインストール | `false` |
| `--target` / `-t` | デプロイ先ターゲットをカンマ区切りで指定 | `apm.yml` の `target:` |
| `--no-policy` | ポリシー強制を無効化 | `false` |
| `--yes` / `-y` | 確認プロンプトをスキップ | `false` |
| `--mcp` | パッケージを MCP server として扱う | `false` |
| `--transport` | MCP transport (`stdio` / `http` / `sse` / `streamable-http`) | `stdio` |

### パッケージ参照の形式

| 形式 | 例 |
|------|-----|
| GitHub shorthand | `owner/repo` |
| 特定の primitive | `owner/repo/skills/<skill-name>` |
| 特定の primitive ファイル | `owner/repo/agents/<file>.agent.md` |
| バージョン付き | `owner/repo#v1.0.0` |
| ローカル bundle | `./my-bundle.tar.gz` |
| Marketplace plugin | `<plugin-name>@<marketplace-name>` |
| フル URL | `https://github.com/owner/repo` |

### 使用例

```bash
# apm.yml に従って全インストール
apm install

# 特定のパッケージを追加
apm install microsoft/apm-sample-package

# バージョンを指定
apm install microsoft/apm-sample-package#v1.0.0

# Marketplace plugin
apm install azure-cloud-development@awesome-copilot

# MCP server を直接インストール
apm install --mcp io.github.github/github-mcp-server --transport http

# 計画だけ確認
apm install --dry-run

# 既存をすべて入れ直す
apm install --force --yes
```

### 動作

1. 隠し Unicode と insecure 依存のスキャン
2. ポリシーチェック (`apm-policy.yml` があれば)
3. lockfile を考慮した依存解決
4. パッケージダウンロードと展開
5. プリミティブの各ターゲットへの統合（複製・フォーマット変換）
6. MCP 統合の更新（対応ターゲットの mcp config に書き込み）
7. lockfile の更新

### 終了コード

| Code | 意味 |
|------|------|
| 0 | 成功 |
| 1 | インストールエラーまたはポリシー違反 |
| 2 | スキーマエラー |

---

## apm update

依存を最新の互換 ref に更新する。計画フェーズで何が変わるかを表示し、承認後に install を実行する。

### 構文

```
apm update [OPTIONS]
```

### オプション

| フラグ | 説明 | デフォルト |
|-------|------|-----------|
| `--yes` / `-y` | 確認プロンプトをスキップ（CI 向け） | `false` |
| `--dry-run` | 計画を描画して終了 | `false` |
| `--verbose` / `-v` | 変化しない依存も計画に含めて詳細出力 | `false` |
| `--target` / `-t` | カンマ区切りターゲット | 全ターゲット |
| `--check` | (非推奨) `apm self-update --check` に転送 | — |

### 動作

1. 解決フェーズで「何が変わるか」を確定
2. 構造化計画を出力（updated / added / removed と ref / SHA の遷移）
3. `Apply these changes? [y/N]` を表示（デフォルト No、`--yes` でスキップ）
4. 承認されたら install パイプラインで適用

非 TTY 環境では `--yes` が必須。

### 使用例

```bash
# 対話的に更新
apm update

# CI で適用
apm update --yes

# 変化しない依存も全表示
apm update --verbose --dry-run
```

---

## apm uninstall

`apm.yml` からパッケージを削除し、`apm_modules/` および各ターゲットからデプロイされたファイルを取り除く。

### 構文

```
apm uninstall PACKAGE [PACKAGE...] [OPTIONS]
```

### オプション

| フラグ | 説明 |
|-------|------|
| `PACKAGE` | パッケージ名、または `name@marketplace` 形式 |
| `--dry-run` | 計画のみ表示 |
| `--verbose` / `-v` | 詳細出力 |
| `--global` / `-g` | ユーザースコープから削除 |

### 動作

1. パッケージが `apm.yml` に存在するか検証
2. 依存リストから除去
3. `apm_modules/` 配下のファイル削除
4. 推移的に未参照になった依存も削除
5. ターゲット別の deployed ファイルを削除
6. MCP 統合を解除
7. lockfile を更新

### 使用例

```bash
apm uninstall microsoft/apm-sample-package
apm uninstall my-plugin@official
apm uninstall pkg-a pkg-b --dry-run
```

---

## apm prune

`apm.yml` に宣言されていないが `apm_modules/` に残っている孤立パッケージを掃除する。

### 構文

```
apm prune [OPTIONS]
```

### オプション

| フラグ | 説明 |
|-------|------|
| `--dry-run` | 削除対象を表示するだけ |

---

## apm outdated

更新可能なパッケージを一覧表示する。

### 構文

```
apm outdated [OPTIONS]
```

### オプション

| フラグ | 説明 |
|-------|------|
| `--offline` | キャッシュ済み ref のみで判定 |
| `--include-prerelease` | プレリリース版も候補に含める |
| `--verbose` / `-v` | 詳細出力 |

### 終了コード

| Code | 意味 |
|------|------|
| 0 | すべて最新 |
| 1 | 更新可能なパッケージあり |

---

## apm deps

依存ツリーを可視化する。GitHub / GitLab / Azure DevOps / ローカルなど複数ソースをまたいだ推移依存も表示する。

### 構文

```
apm deps [OPTIONS]
```

詳細なサブコマンドや出力フォーマットは `apm deps --help` で確認できる。

---

## apm view

インストール済みパッケージのメタデータを表示する。`apm info` は非表示エイリアスとして残っている。

### 構文

```
apm view [PACKAGE] [OPTIONS]
```

### オプション

| フラグ | 説明 |
|-------|------|
| `PACKAGE` | 表示対象パッケージ名（省略で全パッケージ） |
| `--global` / `-g` | ユーザースコープを対象 |
| `--versions` | リモートのタグ / ブランチを GitHub API 経由で列挙 |

---

## apm list

`apm.yml` で定義されたスクリプトを一覧表示する。

### 構文

```
apm list
```

`default` スクリプト（通常は `start`）にはアイコンが付く。スクリプトがない場合は使い方の例パネルが出る。

---

## apm run

`apm.yml` の `scripts:` に定義されたエージェントスクリプトを実行する。

### 構文

```
apm run [SCRIPT_NAME] [OPTIONS]
```

### オプション

| フラグ | 説明 | デフォルト |
|-------|------|-----------|
| `SCRIPT_NAME` | 実行するスクリプト名 | `start` |
| `--param` / `-p` | `name=value` 形式のパラメータ（複数指定可能） | — |
| `--verbose` / `-v` | 実行詳細を表示 | `false` |

### 使用例

```bash
apm run hello
apm run my-prompt --param query="hello world" --param max_tokens=100
```

---

## apm preview

スクリプトを実行せずに、最終的にエージェントへ渡される展開済みプロンプトだけを表示する。

### 構文

```
apm preview [SCRIPT_NAME] [OPTIONS]
```

オプションは `apm run` と同じ。

---

## apm compile

複数のプリミティブを各ターゲット固有の集約ファイル（`AGENTS.md` / `CLAUDE.md` / `GEMINI.md` / `.github/copilot-instructions.md` 等）に統合する。

### 構文

```
apm compile [OPTIONS]
```

### 主なオプション

| フラグ | 説明 |
|-------|------|
| `--target` / `-t` | コンパイル先ターゲット（カンマ区切り） |
| `--watch` | ファイル変更を監視して再コンパイル |
| `--verbose` / `-v` | 詳細出力 |

### 主な出力

| ターゲット | 生成物 |
|----------|--------|
| `copilot` | `.github/copilot-instructions.md` + `AGENTS.md` |
| `claude` | `CLAUDE.md` + `.claude/rules/*` |
| `gemini` | `GEMINI.md` |
| `agents` (codex / windsurf 等) | `AGENTS.md` |

---

## apm pack / apm unpack

配布形式（bundle / plugin）にパッケージングする。`apm pack` はマニフェストとロックファイルを読み、`apm unpack` は受け取ったアーカイブを展開する。

### 構文

```
apm pack [OPTIONS]
apm unpack ARCHIVE [OPTIONS]
```

### 終了コード（pack）

| Code | 意味 |
|------|------|
| 0 | 成功 |
| 1 | ビルドエラー |
| 2 | スキーマエラー |
| 3 | バージョン整合エラー |
| 4 | 作業ツリーに drift がある |

---

## apm marketplace

Plugin marketplace を扱うコマンドグループ。Marketplace を「使う側」と「作る/発行する側」の両方をカバーする。

### サブコマンド一覧（コンシューマ側）

| サブコマンド | 用途 |
|-----------|------|
| `marketplace add REPO` | Marketplace を登録する |
| `marketplace list` | 登録済み Marketplace を表示 |
| `marketplace browse [NAME]` | Marketplace 内の plugin を閲覧 |
| `marketplace update [NAME]` | キャッシュを更新 |
| `marketplace remove NAME` | Marketplace 登録を解除 |
| `marketplace search EXPR` | Marketplace 横断で検索 |

### サブコマンド一覧（オーサ側）

| サブコマンド | 用途 |
|-----------|------|
| `marketplace init` | `apm.yml` に `marketplace:` ブロックをスカフォルド |
| `marketplace check` | 登録エントリの解決可能性を検証 |
| `marketplace doctor` | 発行環境の診断（git / 認証 / 設定整合） |
| `marketplace validate NAME` | `marketplace.json` のスキーマ検証 |
| `marketplace outdated` | Marketplace パッケージのうち更新可能なものを表示 |
| `marketplace publish` | コンシューマリポへ PR や直接コミットで配信 |

### サブコマンドグループ: `marketplace package`

`marketplace.json` 内の package エントリを操作する。

| サブコマンド | 用途 |
|-----------|------|
| `marketplace package add SOURCE` | パッケージエントリを追加 |
| `marketplace package set NAME` | パッケージエントリのフィールドを更新 |
| `marketplace package remove NAME` | パッケージエントリを削除 |

### apm marketplace add

```
apm marketplace add REPO [OPTIONS]
```

| フラグ | 説明 | デフォルト |
|-------|------|-----------|
| `--name` / `-n` | Marketplace の表示名 | リポジトリ名 |
| `--branch` / `-b` | ブランチ | `main` |
| `--host` | git host（非 GitHub の場合） | github.com |
| `--verbose` / `-v` | 詳細出力 | `false` |

### apm marketplace init

```
apm marketplace init [OPTIONS]
```

| フラグ | 説明 | デフォルト |
|-------|------|-----------|
| `--force` | 既存ブロックを上書き | `false` |
| `--no-gitignore-check` | `.gitignore` の検査を省略 | `false` |
| `--name` | Marketplace 名 | `my-marketplace` |
| `--owner` | GitHub オーナー | — |
| `--verbose` / `-v` | 詳細出力 | `false` |

### apm marketplace doctor

`git` / ネットワーク / 認証 token / `gh` CLI / Marketplace config / フォーマットカバレッジ / 重複名 / バージョン整合の 8 項目を検査する。

### apm marketplace publish

`consumer-targets.yml` に列挙したリポへ Marketplace 配信を行う。

```
apm marketplace publish [OPTIONS]
```

| フラグ | 説明 | デフォルト |
|-------|------|-----------|
| `--targets` | `consumer-targets.yml` のパス | カレントディレクトリの同名ファイル |
| `--dry-run` | 計画のみ表示 | `false` |
| `--no-pr` | PR ではなく直接コミット | `false` |
| `--draft` | Draft PR で出す | `false` |
| `--allow-downgrade` | バージョン降格を許可 | `false` |
| `--allow-ref-change` | 固定 ref の変更を許可 | `false` |
| `--parallel` | 並列度 | `4` |
| `--yes` / `-y` | 確認プロンプトをスキップ | `false` |

`.apm/publish-state.json` に状態を保存する。

### apm marketplace package add

```
apm marketplace package add SOURCE [OPTIONS]
```

| フラグ | 説明 | デフォルト |
|-------|------|-----------|
| `--name` | パッケージ名 | リポジトリ名から推定 |
| `--version` | semver 範囲（例: `>=1.0.0 <2.0.0`） | — |
| `--ref` | 具象 ref（SHA / tag / HEAD） | — |
| `--subdir` | リポジトリ内のサブディレクトリ | — |
| `--tag-pattern` | バージョン tag のマッチパターン（例: `v*.*.*`） | — |
| `--tags` | カンマ区切りの分類タグ | — |
| `--include-prerelease` | プレリリースを許可 | `false` |
| `--no-verify` | source の到達確認をスキップ | `false` |

`--version` と `--ref` は同時指定不可。

---

## apm search

Marketplace 横断のエイリアス検索（`apm marketplace search` と同じ）。

### 構文

```
apm search EXPRESSION [OPTIONS]
```

`EXPRESSION` は `QUERY` または `QUERY@MARKETPLACE` 形式。

---

## apm plugin

Plugin プロジェクトの初期化を扱う。

### apm plugin init

```
apm plugin init [PROJECT_NAME] [OPTIONS]
```

| フラグ | 説明 |
|-------|------|
| `PROJECT_NAME` | プラグインプロジェクト名 |
| `--yes` / `-y` | 対話スキップ |
| `--target` | カンマ区切りターゲット |
| `--verbose` / `-v` | 詳細出力 |

非推奨だった `apm init --plugin` の後継。`plugin.json` と最小限の `apm.yml` を生成する。

---

## apm mcp

MCP server の発見・設定・インストールを扱うコマンドグループ。

### apm mcp install

`apm install --mcp` のエイリアス。`MCP_REGISTRY_URL` 環境変数があれば自前のレジストリを参照する。

```
apm mcp install [REFERENCE] [OPTIONS]
```

オプションは `apm install` と同じ。

---

## apm audit

デプロイ済みのプロンプト・指示ファイルに対し、隠し Unicode / lockfile 整合性 / drift の検査を行う。

### 構文

```
apm audit [PACKAGE] [OPTIONS]
```

### オプション

| フラグ | 説明 | デフォルト |
|-------|------|-----------|
| `PACKAGE` | 検査対象パッケージ | 全 deployed ファイル |
| `--file` | 任意のファイル単体を検査 | — |
| `--strip` | 危険な Unicode を除去する | `false` |
| `--dry-run` | `--strip` の対象を表示するだけ | `false` |
| `--verbose` / `-v` | 詳細出力 | `false` |
| `--format` | 出力フォーマット（`text`/`json`/`sarif`/`markdown`） | `text` |
| `--output` / `-o` | 結果ファイル出力先 | stdout |
| `--ci` | lockfile 整合性のゲートを有効化（厳密モード） | `false` |
| `--policy` | カスタムポリシーソース | — |
| `--no-cache` | スキャン結果のキャッシュ無効化 | `false` |
| `--no-policy` | ポリシー強制を無視 | `false` |
| `--no-fail-fast` | 最初の critical で止めずに続行 | `false` |
| `--no-drift` | drift 検出を無効化 | `false` |

### 終了コード

| Code | 意味 |
|------|------|
| 0 | クリーン、または info のみ |
| 1 | critical な検出、または `--ci` 違反 |
| 2 | warning のみ |

### 使用例

```bash
apm audit
apm audit --ci --format sarif --output audit.sarif
apm audit --strip --dry-run    # 何が除去されるかを事前確認
apm audit my-package --verbose
```

---

## apm policy

ポリシー解決の状態を診断する。

### apm policy status

```
apm policy status [OPTIONS]
```

| フラグ | 説明 |
|-------|------|
| `--json` | JSON 出力（CI/SIEM 連携向け） |

ポリシーソースの発見、キャッシュ鮮度、継承チェーン、ルール総数を表示する。設定不正があっても常に exit 0 を返す（出力内で報告）。

---

## apm targets

解決された対象エージェントクライアントを一覧表示する。

### 構文

```
apm targets [OPTIONS]
```

| フラグ | 説明 |
|-------|------|
| `--json` | JSON 出力 |
| `--all` | JSON 出力時に `agent-skills` メタターゲットも含める |

`TARGET` / `STATUS` / `SOURCE` / `DEPLOY DIR` の 4 列でターゲットの解決結果を表示する。`source` は `apm.yml` の `target:` フィールドか、ディレクトリ存在検出か等を示す。

---

## apm config

CLI のグローバル設定を扱う。設定は `~/.apm/config.json` に保存される。

### サブコマンド

| サブコマンド | 用途 |
|-----------|------|
| `config` | 現在の設定を表示（サブコマンド未指定時） |
| `config set KEY VALUE` | 値を設定 |
| `config get [KEY]` | 値を取得（`KEY` 省略時は全件） |
| `config unset KEY` | 値を削除 |

### 主な設定キー

| キー | 型 | 説明 |
|------|----|------|
| `auto-integrate` | bool | パッケージ取得後に自動でターゲット統合を行うか |
| `temp-dir` | path | 一時ディレクトリ |
| `copilot-cowork-skills-dir` | path | Copilot Cowork skills の配置ルート（実験的） |

---

## apm cache

ローカルキャッシュ（git repo / checkout / HTTP）を操作する。

| サブコマンド | 用途 |
|-----------|------|
| `cache info` | キャッシュサイズと内訳を表示 |
| `cache clean` | 全キャッシュを削除（`--force` / `--yes` で確認スキップ） |
| `cache prune [--days N]` | N 日アクセスのないエントリを削除（デフォルト 30 日） |

---

## apm experimental

実験的機能フラグを扱う。

| サブコマンド | 用途 |
|-----------|------|
| `experimental list` | 利用可能な実験機能を一覧 |
| `experimental enable NAME` | 機能を有効化 |
| `experimental disable NAME` | 機能を無効化 |
| `experimental reset` | 全フラグをデフォルトに戻す |

フラグは `~/.apm/config.json` に永続化される。

---

## apm runtime

AI エージェント runtime を管理する（実験的）。

### サブコマンド

| サブコマンド | 用途 |
|-----------|------|
| `runtime setup RUNTIME_NAME` | 指定 runtime をインストール |
| `runtime list` | 利用可能 / インストール済み runtime を表示 |

### サポートする runtime

| 名前 | 起動コマンド | npm package |
|------|-------------|------------|
| `copilot` | `copilot` | `@github/copilot` |
| `codex` | `codex` | `@openai/codex@native` |
| `llm` | `llm` | (Python: Simon Willison's LLM) |
| `gemini` | `gemini` | `@google/gemini-cli` |

### apm runtime setup

```
apm runtime setup RUNTIME_NAME [OPTIONS]
```

| フラグ | 説明 |
|-------|------|
| `--version` | 特定バージョンを指定 |
| `--vanilla` | APM 独自のカスタマイズなし（ネイティブ既定値） |

---

## apm self-update

apm CLI バイナリ自体を最新版に更新する。

### 構文

```
apm self-update [OPTIONS]
```

| フラグ | 説明 |
|-------|------|
| `--check` | インストールせず更新可否のみ確認 |

Windows は PowerShell、Unix は shell の公式インストールスクリプトを使用する。PyInstaller の `LD_LIBRARY_PATH` 干渉を避けるための環境衛生処理が入っている。

### 終了コード

| Code | 意味 |
|------|------|
| 0 | 最新、または更新成功 |
| 1 | エラー、または `--check` で失敗 |
| 2 | 更新を拒否（対話モード） |

---

## 終了コードのまとめ

APM 全体で見ると次のような共通規約に従っている:

| Code | 一般的な意味 |
|------|----------|
| 0 | 成功 / 変更不要 / info-only |
| 1 | 一般的なエラー、ポリシー違反、ユーザーが拒否 |
| 2 | スキーマエラー、warning-only、検証エラー |
| 3 | バージョン整合エラー（`pack`） |
| 4 | 作業ツリー drift（`pack`） |

---

## 関連ドキュメント

- 各コマンドを「いつ・どう組み合わせるか」を見るには [USE-CASES.md](./USE-CASES.md)
- 設定キーや環境変数の詳細は [CONFIGURATION.md](./CONFIGURATION.md)
- エラー時の対処は [TROUBLESHOOTING.md](./TROUBLESHOOTING.md)
