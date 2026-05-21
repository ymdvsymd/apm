# APM — トラブルシューティング

**生成日:** 2026-05-21
**対象バージョン:** apm-cli v0.14.1

ここではよく踏みやすい問題を症状→原因→対処の順で並べる。困ったら最初に `apm policy status`、`apm targets`、`apm cache info`、`apm audit` を流すと多くの場合切り分けが進む。

---

## よくある問題

### 1. `apm install` が認証エラーで失敗する

**現象:**

```
Error: Authentication failed for github.com
HTTP 401 / 403
```

**原因:**

- パッケージがプライベートリポジトリにある
- `GITHUB_TOKEN` / `GITHUB_APM_PAT` が未設定または期限切れ
- GitHub Enterprise の場合に `GITHUB_HOST` が未設定

**対処:**

1. `gh` CLI を使っているか確認する:
   ```bash
   gh auth status
   gh auth login        # 未ログインなら
   ```
   `gh` でログイン済みなら APM は token を自動取得する。

2. 環境変数で明示する:
   ```bash
   export GITHUB_APM_PAT="ghp_..."
   apm install
   ```

3. GitHub Enterprise を使う:
   ```bash
   export GITHUB_HOST="github.example.com"
   export GITHUB_APM_PAT="..."
   ```

4. Azure DevOps の場合は `ADO_APM_PAT` を、GitLab なら `GITLAB_TOKEN` を設定。

---

### 2. `apm install` がポリシー違反で止まる

**現象:**

```
Error: Policy violation
Source 'github.com/untrusted/repo' is not in allowed_sources
```

**原因:**

`apm-policy.yml` の `dependencies.allowed_sources` 制約に違反している。

**対処:**

1. どのポリシーが効いているか確認:
   ```bash
   apm policy status
   ```

2. 違反内容を詳細に確認:
   ```bash
   apm install --dry-run --verbose
   ```

3. 一時的に回避するには（ローカルのみ。CI では意味なし）:
   ```bash
   apm install --no-policy
   ```
   または `APM_POLICY_DISABLE=1`。

4. 正攻法は、(a) 許可ソースのミラーを使う、(b) `bypass_label` ラベルを PR に付けて例外承認、(c) ポリシー側に allowlist を追加してもらう。

---

### 3. `apm compile` が "no compiled output" 警告を出す

**現象:**

```
Warning: no compiled output for target 'copilot'
```

**原因:**

- `.apm/instructions/` などコンパイル元のディレクトリが空
- すべての primitive が他ターゲット向けにフィルタされている
- `includes:` が誤って空のリストになっている

**対処:**

1. `.apm/` 配下に primitive があるか確認:
   ```bash
   find .apm -name '*.md' -o -name '*.toml'
   ```

2. frontmatter の `targets:` をチェックし、対象ターゲットを含んでいるか確認:
   ```yaml
   ---
   targets: [copilot, claude]
   ---
   ```

3. `apm.yml` の `includes` を確認:
   ```yaml
   includes: auto    # .apm/ 配下を全自動検出
   ```

---

### 4. `apm audit` が "lockfile drift" を報告する

**現象:**

```
Critical: lockfile drift detected
deployed file <path> differs from lockfile content hash
```

**原因:**

- インストール後にユーザが deployed ファイルを手で編集した
- 上流の ref が動いた（branch を pin している場合）
- パッケージ提供者がタグを上書きした

**対処:**

1. drift の対象を `--verbose` で特定:
   ```bash
   apm audit --verbose
   ```

2. 意図的な編集なら、元の primitive を変更してから `apm install --force` でデプロイし直す:
   ```bash
   apm install --force
   ```

3. 上流変更を受け入れる場合は `apm update --yes` で lockfile を更新。

4. CI でブロックされた場合は `apm install --frozen` を実行して lockfile と一致しているかを確認する。

---

### 5. MCP サーバが各クライアントから見えない

**現象:**

`apm install --mcp` は成功するが、VS Code Copilot / Claude / Cursor から MCP server が認識されない。

**原因:**

- クライアントが起動済みで設定ファイルを再読み込みしていない
- 該当クライアントのルートディレクトリ（`.cursor/` 等）がプロジェクトに存在しない（Cursor はサイレントにスキップする）
- transport の指定が間違っている

**対処:**

1. どの設定ファイルが書き込まれたかを確認:
   ```bash
   apm targets
   ```

2. 該当ファイルを直接見る:
   ```bash
   cat ~/.copilot/mcp-config.json
   cat .mcp.json                     # Claude project scope
   cat ~/.claude.json                # Claude user scope
   cat .cursor/mcp.json
   cat .windsurf/mcp_config.json
   ```

3. クライアントを再起動。Copilot は runtime substitution に対応しているので `${GITHUB_TOKEN}` の解決は実行時に行われる。Claude/Cursor は install 時の値が固定される。

4. Cursor で見えない場合は `.cursor/` ディレクトリを先に作る:
   ```bash
   mkdir -p .cursor
   apm install --mcp <server>
   ```

---

### 6. SSL / TLS エラー（特に企業ネットワーク）

**現象:**

```
SSLError: certificate verify failed
```

**原因:**

企業プロキシが TLS をインターセプトしており、システムの信頼チェーンに企業 CA が入っていない。

**対処:**

1. システムの CA バンドルに企業 CA を追加する（推奨）。
2. Python の `requests` が使う CA バンドルを指定:
   ```bash
   export REQUESTS_CA_BUNDLE=/path/to/ca-bundle.crt
   export SSL_CERT_FILE=/path/to/ca-bundle.crt
   apm install
   ```
3. 検証を無効化するのは非推奨だが、開発時のみ:
   ```bash
   export PYTHONHTTPSVERIFY=0
   ```

---

### 7. Windows コンソールで文字化けする

**現象:**

```
Warning: Console is cp932 (Japanese), UTF-8 switch failed.
```

**原因:**

Windows のコンソールが UTF-8 (cp 65001) ではないため、APM の box-drawing 文字や絵文字が化ける。

**対処:**

1. APM は起動時に自動で UTF-8 への切替を試みる。失敗した場合は警告を出す。
2. 手動で切り替える:
   ```powershell
   chcp 65001
   ```
3. Windows Terminal や VS Code の terminal を使う。

---

### 8. `apm pack` が `exit 4` で失敗する（working-tree drift）

**現象:**

```
Error: working tree has uncommitted changes
exit code: 4
```

**原因:**

pack はクリーンな working tree が前提（再現性の担保）。コミットされていない変更がある。

**対処:**

```bash
git status
git add -A && git commit -m "..."
apm pack
```

または開発中の確認用には別ディレクトリで `apm install` してから pack する。

---

### 9. AppLocker / WDAC で Windows でインストールが阻まれる

**現象:**

```
Error: Cannot execute downloaded installer
AppLocker / WDAC policy blocks
```

**原因:**

Windows のアプリケーション制御ポリシーで未承認バイナリが実行できない。

**対処:**

1. APM のインストール先パス（`%LOCALAPPDATA%\Programs\apm\` など）を allow-list に追加してもらう。
2. pip 経由で入れる:
   ```powershell
   pip install apm-cli
   ```
3. ソースから入れる:
   ```powershell
   git clone https://github.com/microsoft/apm
   cd apm
   pip install -e .
   ```

---

### 10. `apm install` が遅い

**現象:**

数十パッケージの解決に数分〜数十分かかる。

**原因:**

- 並列度が低い
- キャッシュが効いていない
- 各 ref ごとに full clone している

**対処:**

1. 並列解決を有効にする:
   ```bash
   export APM_RESOLVE_PARALLEL=8
   export APM_TIERED_RESOLVER=1
   apm install
   ```
   tiered resolver（memory cache → API → rev-parse → clone）が大規模リポの解決時間を短縮する。

2. キャッシュを確認:
   ```bash
   apm cache info
   ```

3. キャッシュが汚れているかもしれない場合は prune:
   ```bash
   apm cache prune --days 30
   ```

---

## FAQ

### Q: `apm.yml` と `apm.lock.yaml` はどちらをコミットすべき？

A: **両方**。`apm.yml` は宣言、`apm.lock.yaml` は再現性のためのピン。`npm` の `package.json` と `package-lock.json` の関係と同じ。

### Q: `apm_modules/` は git にコミットすべき？

A: 通常は `.gitignore` に入れて除外する。`apm.lock.yaml` があれば誰でも同じものを再現できる。air-gapped 配布用に固める場合のみコミットする（[シナリオ 7](./USE-CASES.md#シナリオ-7-air-gapped-環境にオフラインで配布する)）。

### Q: `apm compile` を毎回手で実行するのが面倒。

A: `apm compile --watch` でファイル変更時に自動再コンパイル。または pre-commit hook や CI に組み込む。

### Q: ターゲットが自動検出されない。

A: `apm init` のときに対応するルートディレクトリ（`.github/` / `.claude/` 等）がなければ自動検出されない。`apm.yml` の `target:` フィールドを明示するか、`--target` で指定する。

### Q: ローカル開発中に試行錯誤したい。

A: ローカルパスを依存にできる。
```yaml
dependencies:
  apm:
    - path: ../local-package
```

### Q: バイナリ版から pip 版に切り替えたい。

A: バイナリを消してから `pip install apm-cli`。バージョンの混在に注意。

### Q: `apm self-update` が失敗する。

A: バイナリ版のみ対応。pip / Homebrew / Scoop 経由の場合は各 package manager で更新する。
```bash
pip install --upgrade apm-cli
brew upgrade apm
scoop update apm
```

### Q: Drift detection を切りたい。

A: `apm audit --no-drift` で個別の audit 実行のみ無効化できる。CI で恒久的に切るのは推奨されない（policy 違反の検出ができなくなる）。

---

## 診断チェックリスト

問題が解決しない場合の切り分け順序:

1. **環境の確認**
   ```bash
   apm --version
   python3 --version
   git --version
   ```

2. **ターゲット解決の確認**
   ```bash
   apm targets
   apm targets --json     # 機械可読
   ```

3. **設定の確認**
   ```bash
   apm config
   apm policy status
   ```

4. **依存関係の確認**
   ```bash
   apm view
   apm deps
   apm outdated
   ```

5. **キャッシュ状態**
   ```bash
   apm cache info
   ```

6. **詳細な install ログ**
   ```bash
   apm install --verbose --dry-run
   ```

7. **環境変数の見直し**
   ```bash
   env | grep APM_
   env | grep GITHUB_
   env | grep MCP_
   ```

8. **drift / unicode 検査**
   ```bash
   apm audit --verbose
   ```

---

## エスカレーション先

| 問題タイプ | 連絡先 |
|----------|--------|
| バグ報告 | <https://github.com/microsoft/apm/issues> |
| 質問 / 議論 | <https://github.com/microsoft/apm/discussions> |
| セキュリティ脆弱性 | `SECURITY.md` に従う（issue ではなく非公開チャネル） |

---

## 関連ドキュメント

- 個別コマンドの詳細は [COMMANDS.md](./COMMANDS.md)
- 設定の見直しは [CONFIGURATION.md](./CONFIGURATION.md)
- 典型的フローは [USE-CASES.md](./USE-CASES.md)
