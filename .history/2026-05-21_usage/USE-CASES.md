# APM — ユースケースガイド

**生成日:** 2026-05-21
**対象バージョン:** apm-cli v0.14.1

ここでは「こういうことがしたい」というユーザの視点から、APM の実用シナリオを 7 つ並べる。各シナリオは状況・手順・期待される結果・ヒントの 4 部構成。手順内のコマンドはコピペで実行可能。

---

## シナリオ一覧

| # | シナリオ | カテゴリ | 難易度 |
|---|---------|---------|--------|
| 1 | 公開パッケージから skills を取り込んで Copilot 用に固める | Getting Started | 初級 |
| 2 | チーム共通の指示書を package 化して社内 GitHub に配布する | Core Workflow | 中級 |
| 3 | 1 つの `apm.yml` を 7 つのエージェントクライアントに同時展開する | Core Workflow | 中級 |
| 4 | MCP サーバを 1 コマンドで全クライアントに追加する | Core Workflow | 中級 |
| 5 | Plugin を作って Marketplace に発行する | Advanced Usage | 上級 |
| 6 | 組織ポリシーを段階導入する (warn → block) | Administration | 上級 |
| 7 | air-gapped 環境にオフラインで配布する | Advanced Usage | 上級 |

---

## シナリオ 1: 公開パッケージから skills を取り込んで Copilot 用に固める

**カテゴリ:** Getting Started
**難易度:** 初級
**使うコマンド:** [`apm init`](./COMMANDS.md#apm-init), [`apm install`](./COMMANDS.md#apm-install), [`apm compile`](./COMMANDS.md#apm-compile), [`apm targets`](./COMMANDS.md#apm-targets)

### 状況

新しいプロジェクトを始めるにあたって、VS Code + GitHub Copilot にチーム標準の skills と指示書を読ませたい。手で `.github/copilot-instructions.md` を書き続けるのは限界。サードパーティの skill コレクション（例: `anthropics/skills`、`vercel-labs/agent-skills`）を取り込んで再現可能な形で運用したい。

### 手順

1. プロジェクトを初期化する。
   ```bash
   mkdir my-frontend && cd my-frontend
   apm init --yes --target copilot
   ```
   `apm.yml` と `.github/` ディレクトリの雛形ができる。

2. skill コレクションを丸ごとインストールする。
   ```bash
   apm install vercel-labs/agent-skills
   ```
   `apm_modules/vercel-labs/agent-skills/` に取得され、`.github/instructions/` や `.agents/skills/` に配置される。

3. 一部の skill だけが必要なら、絞り込んでインストールする。
   ```bash
   apm install vercel-labs/agent-skills --skill deploy-to-vercel
   ```
   `apm.yml` の `dependencies.apm:` に明示的にエントリが残るため、再現性は保たれる。

4. Copilot 用の集約ファイルを生成する。
   ```bash
   apm compile -t copilot
   ```
   `.github/copilot-instructions.md` が出来上がる。VS Code でこのリポを開き直せば自動で読み込まれる。

5. 結果を確認する。
   ```bash
   apm targets
   apm view vercel-labs/agent-skills
   ```

### 期待される結果

- `apm targets` の `copilot` 行が `STATUS: active` で表示される
- `.github/copilot-instructions.md` に skill 由来のセクションが含まれる
- `apm.lock.yaml` がコミット対象として追加されており、再 clone 時も同じ ref が解決される

### ヒント

- `apm install --dry-run` で適用前に diff を確認できる
- npm の `npx skills add` から来た人は `apm install vercel-labs/agent-skills` がドロップイン置き換え
- skills を消したくなったら `apm uninstall vercel-labs/agent-skills`

---

## シナリオ 2: チーム共通の指示書を package 化して社内 GitHub に配布する

**カテゴリ:** Core Workflow
**難易度:** 中級
**使うコマンド:** [`apm init`](./COMMANDS.md#apm-init), [`apm compile`](./COMMANDS.md#apm-compile), [`apm pack`](./COMMANDS.md#apm-pack--apm-unpack), [`apm install`](./COMMANDS.md#apm-install)

### 状況

社内で何度も書く「TypeScript で書くときの命名規約」「API レスポンスのエンベロープ規約」「ロギング規約」を 1 か所にまとめて、社内の他リポからも `apm install our-org/coding-standards` で取り込めるようにしたい。

### 手順

1. 共通指示書を入れる新リポを作って初期化する。
   ```bash
   mkdir coding-standards && cd coding-standards
   git init
   apm init --yes --target copilot,claude,cursor
   ```

2. 指示書を `.apm/instructions/` に置く（コンパイル元の primitive）。
   ```bash
   mkdir -p .apm/instructions
   ```
   `naming.instructions.md` などを書く。フロントマターに対象 target を絞れる:
   ```yaml
   ---
   name: typescript-naming
   targets: [copilot, claude]
   ---
   ```

3. ローカルで動作確認する。
   ```bash
   apm compile -t copilot,claude
   apm preview                # スクリプト実行前のプロンプトを確認
   ```

4. push して tag を切る。
   ```bash
   git push origin main
   git tag v1.0.0 && git push origin v1.0.0
   ```

5. 消費側リポで取り込む。
   ```bash
   cd ../other-repo
   apm install our-org/coding-standards#v1.0.0
   apm compile
   ```

6. （任意）bundle / plugin として配布する場合は pack する。
   ```bash
   cd ../coding-standards
   apm pack
   ```
   作られたアーカイブは `apm install ./coding-standards.tar.gz` で取り込める。

### 期待される結果

- 消費側リポの `.github/copilot-instructions.md` / `CLAUDE.md` / `.cursor/rules/*.mdc` に共通指示書が集約される
- `apm.lock.yaml` に commit SHA が記録され、いつ消費したかが追跡可能
- 上流の `coding-standards` を更新したら、消費側は `apm update` で取り込める

### ヒント

- 推移依存もサポートされるので、複数の coding-standards リポを 1 つの bundle で集約してもよい
- 社内が GitLab / Azure DevOps なら `--host gitlab.example.com` / `--host dev.azure.com` を `apm install` で指定するか、URL 形式で指定する

---

## シナリオ 3: 7 クライアントへ 1 コマンドで展開する

**カテゴリ:** Core Workflow
**難易度:** 中級
**使うコマンド:** [`apm init`](./COMMANDS.md#apm-init), [`apm install`](./COMMANDS.md#apm-install), [`apm compile`](./COMMANDS.md#apm-compile), [`apm targets`](./COMMANDS.md#apm-targets)

### 状況

チームメンバーが使うエージェントが Copilot / Claude Code / Cursor / Codex / Gemini と分散している。各人が同じプロジェクトコンテキストで作業できるよう、1 つの `apm.yml` で全クライアントへ同じ skills / agents / instructions を配置したい。

### 手順

1. すべてのターゲットを有効化して初期化する。
   ```bash
   apm init --yes --target all
   ```
   または既存プロジェクトに対しては `apm.yml` の `target:` フィールドを `all` か `[copilot, claude, cursor, codex, gemini, opencode, windsurf]` に書き換える。

2. 共通依存をインストールする。
   ```bash
   apm install microsoft/apm-sample-package
   ```
   各ターゲット固有のディレクトリ（`.github/` / `.claude/` / `.cursor/` / `.codex/` / `.gemini/` / `.opencode/` / `.windsurf/`）に同じ内容が配置される。

3. すべてのターゲットでコンパイルする。
   ```bash
   apm compile -t all
   ```
   - `.github/copilot-instructions.md`
   - `CLAUDE.md`
   - `GEMINI.md`
   - `AGENTS.md`（Codex / Cursor / Windsurf / OpenCode 用）

4. 検出と差分を確認する。
   ```bash
   apm targets
   ```

### 期待される結果

`apm targets` の出力例（抜粋）:

```
TARGET         STATUS    SOURCE        DEPLOY DIR
copilot        active    apm.yml       .github/
claude         active    apm.yml       .claude/
cursor         active    apm.yml       .cursor/
codex          active    apm.yml       .codex/
gemini         active    apm.yml       .gemini/
opencode       active    apm.yml       .opencode/
windsurf       active    apm.yml       .windsurf/
```

### ヒント

- ターゲットを一部だけにしたいときは `apm install --target copilot,claude` で都度オーバーライド可能
- 各 primitive は target-aware に書ける（frontmatter で `targets: [claude, cursor]` のように指定すると、コンパイル時にそのターゲットでのみ採用される）
- 不要になったクライアントは `apm.yml` の `target:` から外して `apm install --force` すれば deployed ファイルが消える

---

## シナリオ 4: MCP サーバを 1 コマンドで全クライアントに追加する

**カテゴリ:** Core Workflow
**難易度:** 中級
**使うコマンド:** [`apm install --mcp`](./COMMANDS.md#apm-install), [`apm mcp`](./COMMANDS.md#apm-mcp), [`apm targets`](./COMMANDS.md#apm-targets)

### 状況

GitHub MCP サーバ（`io.github.github/github-mcp-server`）を Copilot / Claude Code / Cursor / Windsurf すべてに登録したい。各クライアントの mcp config を手で書くのは煩雑なうえに、`${GITHUB_TOKEN}` のような環境変数プレースホルダの解決ルールが微妙に違う。

### 手順

1. MCP サーバを 1 つのコマンドでインストールする。
   ```bash
   apm install --mcp io.github.github/github-mcp-server --transport http
   ```
   APM が対応している MCP クライアント設定ファイルすべてに書き込む:
   - Copilot: `~/.copilot/mcp-config.json`（runtime substitution で `${VAR}` を実行時解決）
   - Claude: `.mcp.json` または `~/.claude.json`
   - Cursor: `.cursor/mcp.json`
   - Windsurf: `.windsurf/mcp_config.json`

2. ローカルレジストリやプライベートな MCP サーバを使いたい場合は `MCP_REGISTRY_URL` を設定する。
   ```bash
   export MCP_REGISTRY_URL="https://mcp.your-company.com/registry"
   apm install --mcp internal/jira-mcp --transport http
   ```

3. `apm.yml` の `mcp:` に永続的な依存として宣言する。
   ```yaml
   dependencies:
     mcp:
       - name: io.github.github/github-mcp-server
         transport: http
   ```

4. 認証トークンが必要なら、設定ファイルにベタ書きせず環境変数で解決させる（Copilot のみ自動解決対応。他は install 時に解決される）。
   ```bash
   export GITHUB_TOKEN=ghp_...
   apm install
   ```

### 期待される結果

- 各クライアントを起動すると MCP サーバが認識される
- VS Code Copilot では `${GITHUB_TOKEN}` が起動時に解決される（disk には placeholder のまま）
- `apm audit` で MCP 設定の drift も検出される

### ヒント

- 推移的 MCP サーバ（依存パッケージが MCP を要求するケース）には trust prompt が出る。CI では `--yes` で承認するか、`apm-policy.yml` で allowlist する
- `apm install` の `--transport` は MCP の transport（stdio / http / sse / streamable-http）であり、URL スキームではない
- Codex / Gemini CLI で MCP を使いたい場合は `apm runtime setup codex` / `apm runtime setup gemini` を先に実行する

---

## シナリオ 5: Plugin を作って Marketplace に発行する

**カテゴリ:** Advanced Usage
**難易度:** 上級
**使うコマンド:** [`apm plugin init`](./COMMANDS.md#apm-plugin), [`apm marketplace init`](./COMMANDS.md#apm-marketplace), [`apm marketplace doctor`](./COMMANDS.md#apm-marketplace-doctor), [`apm pack`](./COMMANDS.md#apm-pack--apm-unpack), [`apm marketplace publish`](./COMMANDS.md#apm-marketplace-publish)

### 状況

自社の「リリースノート生成 agent」を社内の他チームに使ってもらいたい。手段は 2 つあって、(a) 単純な APM パッケージとして公開する、(b) `plugin.json` 形式の plugin として Marketplace 経由で配信する。今回は (b) を選び、複数のターゲット（Copilot / Claude / Cursor）に対応した plugin として配布する。

### 手順

1. plugin プロジェクトをスカフォルドする。
   ```bash
   mkdir release-notes-plugin && cd release-notes-plugin
   apm plugin init --yes --target copilot,claude,cursor
   ```
   生成物: `plugin.json` + `apm.yml`。

2. プリミティブを書く。たとえば `.apm/agents/release-notes.agent.md` に agent 定義を作る。

3. ローカルで動作確認する。
   ```bash
   apm compile -t copilot,claude,cursor
   apm preview my-agent
   ```

4. 別リポに自分の Marketplace 用 manifest を作る。
   ```bash
   mkdir my-marketplace && cd my-marketplace
   apm marketplace init --owner my-org --name awesome-prompts
   ```
   `marketplace.json` が生成される。

5. Marketplace に plugin エントリを追加する。
   ```bash
   apm marketplace package add https://github.com/my-org/release-notes-plugin \
       --name release-notes \
       --version ">=1.0.0 <2.0.0" \
       --tags release,docs
   ```

6. 発行前の環境診断をする。
   ```bash
   apm marketplace doctor
   ```
   git / 認証 / 設定整合の 8 項目をチェック。

7. 配信先（consumer リポ）の一覧を `consumer-targets.yml` に書く:
   ```yaml
   targets:
     - repo: my-org/web-app
       branch: main
       path_in_repo: apm.yml
     - repo: my-org/api-service
       branch: main
       path_in_repo: apm.yml
   ```

8. 配信を計画 → 実行する。
   ```bash
   apm marketplace publish --targets consumer-targets.yml --dry-run
   apm marketplace publish --targets consumer-targets.yml --yes
   ```
   各 consumer リポに PR が出る。

9. 消費側は marketplace を登録すれば検索できる。
   ```bash
   cd ../consumer-repo
   apm marketplace add my-org/my-marketplace --name awesome-prompts
   apm search release-notes
   apm install release-notes@awesome-prompts
   ```

### 期待される結果

- consumer-targets.yml に列挙された各リポに PR が立つ（`--draft` で draft、`--no-pr` で直接コミット）
- `.apm/publish-state.json` に発行ジョブの状態が記録される
- 消費側で `apm search` が plugin を検出する

### ヒント

- バージョニング戦略は `--tag-pattern` と `--version` で制御する。lockstep / tag pattern / per-package の 3 パターンを選べる
- 配信前に `apm marketplace check` で URL 解決可能性を、`apm marketplace validate` で `marketplace.json` のスキーマを検証する
- `--parallel 4` で並列度をあげれば大規模な org でも高速に配信できる

---

## シナリオ 6: 組織ポリシーを段階導入する

**カテゴリ:** Administration
**難易度:** 上級
**使うコマンド:** [`apm policy status`](./COMMANDS.md#apm-policy), [`apm audit --ci`](./COMMANDS.md#apm-audit), [`apm install --no-policy`](./COMMANDS.md#apm-install)

### 状況

セキュリティチームから「全リポで使えるパッケージソースと MCP サーバを制限したい。ただし破壊的に block にすると現場が止まるので、最初は warn で運用したい」と要求された。`apm-policy.yml` で warn-first の段階導入を実現する。

### 手順

1. 組織用ポリシーファイルを書く。`apm-policy.yml` を組織ルート（org-level repo）に置く。
   ```yaml
   name: my-org-policy
   version: 1.0.0
   enforcement:
     level: warn          # 最初は warn
     bypass_label: policy-bypass-approved
   dependencies:
     allowed_sources:
       - github.com/my-org/*
       - github.com/microsoft/*
   mcp:
     allowed_servers:
       - io.github.github/github-mcp-server
   ```

2. 各リポでポリシー解決状態を確認する。
   ```bash
   apm policy status
   apm policy status --json   # CI/SIEM 連携用
   ```

3. CI に audit gate を仕込む。`.github/workflows/apm-audit.yml`:
   ```yaml
   - run: apm install --frozen
   - run: apm audit --ci --format sarif --output apm-audit.sarif
   - uses: github/codeql-action/upload-sarif@v3
     with:
       sarif_file: apm-audit.sarif
   ```

4. 各リポの開発者は通常通り `apm install` する。ポリシー違反があれば warning が出る。
   ```bash
   apm install --dry-run   # 何が違反になるかを事前確認
   ```

5. しばらく warning で慣らした後、ポリシーを `block` に切り替える。
   ```yaml
   enforcement:
     level: block
   ```

6. 例外を `bypass_label` で個別承認する仕組みも用意できる（PR ラベルベース）。
   ```bash
   apm install --no-policy   # 緊急時のローカル回避（CI ではブロック）
   ```

### 期待される結果

- 移行期間中は警告のみで導入を周知でき、リアル違反のスコープが分かる
- 切替後は禁止ソースからの install が `exit 1` で失敗する
- 階層継承（enterprise → org → repo）は tighten-only なので、下位リポが許可を緩めることはできない

### ヒント

- ポリシーは `apm audit --ci` で CI 側も強制できる。branch protection と組み合わせると有効
- `apm install --no-policy` はあくまでローカル開発の緊急回避用。CI で外すと意味がないので運用ルールで縛る
- 大規模 org への展開は段階的に: pilot リポ → 全体 warn → 個別 block → 全体 block

---

## シナリオ 7: air-gapped 環境にオフラインで配布する

**カテゴリ:** Advanced Usage
**難易度:** 上級
**使うコマンド:** [`apm pack`](./COMMANDS.md#apm-pack--apm-unpack), [`apm install`](./COMMANDS.md#apm-install) (ローカルパス), `APM_NO_CACHE`

### 状況

社内オンプレ環境はインターネット直結ができない。`apm install` で都度 GitHub にアクセスできないため、外で bundle を作って、内側に持ち込んで展開したい。

### 手順

1. インターネット接続環境でフルセットを `apm install` してロックする。
   ```bash
   cd my-context
   apm install
   git add apm.yml apm.lock.yaml apm_modules
   git commit -m "lock dependencies for air-gapped distribution"
   ```

2. 配布用 bundle を作る。
   ```bash
   apm pack
   ```
   `my-context-<version>.tar.gz` が生成される（lockfile のピン留めが埋め込まれる）。

3. air-gapped 環境に bundle を持ち込む。

4. ローカルファイルから直接 install する。
   ```bash
   cd target-project
   apm install ./my-context-1.0.0.tar.gz
   ```
   または、bundle を展開してから参照する:
   ```bash
   apm unpack ./my-context-1.0.0.tar.gz -o ./apm_modules/extracted
   ```

5. ネットワークが使えない環境で挙動を抑えたい場合は環境変数を設定する。
   ```bash
   export APM_NO_CACHE=1
   apm install ./my-context-1.0.0.tar.gz
   ```

6. 自前の Marketplace / Registry proxy を立てる選択肢もある（例: Artifactory）。
   ```bash
   export MCP_REGISTRY_URL=https://artifactory.internal/mcp
   apm install --mcp internal/server
   ```

### 期待される結果

- ネットワークアクセスなしで全プリミティブが配置される
- `apm.lock.yaml` に記録された SHA / hash と bundle 内容が一致することがインストール時に検証される
- ハッシュ不一致や drift があれば install が失敗する

### ヒント

- bundle 内には対象パッケージのソースが含まれるため、ライセンス確認をしてから配布する
- bundle のサイズが問題なら、必要な primitive だけを含めた slim な配布版を作るか、Registry proxy を立てて lazy fetch する
- `apm self-update` は air-gapped では使えないので、別経路で apm 本体を配布する

---

## カテゴリ別逆引き

| やりたいこと | 該当シナリオ |
|------------|------------|
| 動かしてみたい | [1](#シナリオ-1-公開パッケージから-skills-を取り込んで-copilot-用に固める) |
| 内製パッケージを社内で再利用したい | [2](#シナリオ-2-チーム共通の指示書を-package-化して社内-github-に配布する) |
| 複数のエージェントに同じコンテキストを配りたい | [3](#シナリオ-3-7-クライアントへ-1-コマンドで展開する) |
| MCP サーバを設定したい | [4](#シナリオ-4-mcp-サーバを-1-コマンドで全クライアントに追加する) |
| 自前 plugin を発行したい | [5](#シナリオ-5-plugin-を作って-marketplace-に発行する) |
| 組織にガバナンスを敷きたい | [6](#シナリオ-6-組織ポリシーを段階導入する) |
| インターネット不可の環境で使いたい | [7](#シナリオ-7-air-gapped-環境にオフラインで配布する) |

---

## 関連ドキュメント

- 個別コマンドの詳細は [COMMANDS.md](./COMMANDS.md)
- 設定キーや認証は [CONFIGURATION.md](./CONFIGURATION.md)
- エラー時の対処は [TROUBLESHOOTING.md](./TROUBLESHOOTING.md)
