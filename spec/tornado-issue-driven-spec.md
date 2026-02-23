# Tornado Issue-Driven CI/CD 仕様書

## 1. 概要

GitHub Issues を起点として、[`@mizchi/tornado`](https://github.com/mizchi/tornado) マルチエージェントオーケストレータを **Windows セルフホストランナー**上で実行し、AI エージェント（claude-code / codex）による自動コーディング・レビューを行うワークフロー。

**Unity プロジェクトの開発を前提**としており、ランナーには Unity Editor がインストールされていることを想定する。すべてのやり取りは GitHub Issue のコメントを通じて行われ、実行結果は Pull Request として自動生成される。

---

## 2. システム構成

```
┌──────────────┐     Issue作成/コメント      ┌──────────────────┐
│   開発者      │ ──────────────────────────▶ │   GitHub Issues   │
│  (Human)     │ ◀────────────────────────── │   (トリガー)       │
│              │     結果レポート(comment)     └────────┬─────────┘
└──────────────┘                                       │
                                                       ▼
                                              ┌──────────────────┐
                                              │  GitHub Actions   │
                                              │  (Workflow)       │
                                              └────────┬─────────┘
                                                       │
                                                       ▼
                                    ┌──────────────────────────────┐
                                    │  Windows Self-Hosted Runner   │
                                    │                               │
                                    │  ┌────────────┐ ┌──────────┐ │
                                    │  │ Unity Editor│ │ Node.js  │ │
                                    │  │ (installed) │ │ v22+     │ │
                                    │  └────────────┘ └──────────┘ │
                                    │                               │
                                    │  ┌─────────────────────────┐ │
                                    │  │  Tornado                 │ │
                                    │  │  ┌──────────┐┌────────┐ │ │
                                    │  │  │ dev      ││ review │ │ │
                                    │  │  │ (claude) ││ (codex)│ │ │
                                    │  │  └──────────┘└────────┘ │ │
                                    │  └─────────────────────────┘ │
                                    └──────────────┬───────────────┘
                                                   │
                                                   ▼
                                    ┌──────────────────────────────┐
                                    │  Pull Request (自動生成)      │
                                    └──────────────────────────────┘
```

---

## 3. 前提条件

### 3.1 セルフホストランナー（Windows）

| 要件 | 詳細 |
|------|------|
| OS | **Windows 10/11 または Windows Server 2019+** |
| Node.js | v22.0.0 以上（`actions/setup-node` で自動セットアップ） |
| gh CLI | インストール済み（https://cli.github.com） |
| Git | Git for Windows 2.x 以上 |
| PowerShell | pwsh (PowerShell 7+) — ワークフローのデフォルトシェル |
| Unity Editor | Unity Hub 経由でインストール済み |
| ネットワーク | Anthropic API / OpenAI API / npm registry への外部通信が可能 |

#### ランナーラベル

ランナーは以下の **2つのラベル** で登録すること:

```
self-hosted, Windows
```

#### Unity Editor のパス

デフォルトでは `C:\Program Files\Unity\Hub\Editor` を想定。異なる場合はリポジトリの **Variables** に `UNITY_EDITOR_PATH` を設定する。

### 3.2 GitHub Secrets

| Secret 名 | 用途 | 必須 |
|-----------|------|------|
| `ANTHROPIC_API_KEY` | Claude Code (dev agent) 用 | ✅ |
| `OPENAI_API_KEY` | Codex (review agent) 用 | ✅ |
| `GITHUB_TOKEN` | 自動提供（Issue/PR操作に使用） | 自動 |

### 3.3 GitHub Variables（任意）

| Variable 名 | 用途 | デフォルト値 |
|-------------|------|-------------|
| `UNITY_EDITOR_PATH` | Unity Editor のインストールパス | `C:\Program Files\Unity\Hub\Editor` |

### 3.4 リポジトリ設定

- ラベル `tornado` を事前に作成しておくこと
- ラベル `tornado-failed` を事前に作成しておくこと（任意・失敗時に自動付与）
- セルフホストランナーが `self-hosted` + `Windows` ラベルで登録されていること

---

## 4. ワークフロー詳細

### 4.1 トリガー条件

ワークフローは以下の 2 つのパターンで起動する。

#### パターン A: Issue 作成・ラベル付与時

1. 開発者が Issue を作成する
2. Issue に `tornado` ラベルを付与する
3. Issue 本文（body）がそのまま Tornado の plan ファイルとして使用される

> **注:** `labeled` イベントでは `github.event.label.name == 'tornado'` の場合のみ起動。`tornado-failed` 等の別ラベル付与では再トリガーしない。

#### パターン B: Issue コメント時（追加指示）

1. `tornado` ラベルが付いた Issue に `/tornado <追加指示>` とコメントする
2. 元の Issue body + 追加指示が合成された plan で Tornado が再実行される

### 4.2 Issue Body のフォーマット

Issue 本文は Tornado の plan ファイル（Markdown）として直接使用される。Unity プロジェクトを意識した推奨フォーマット:

```markdown
## タスク

プレイヤーキャラクターの移動システムを実装してください。

## 要件

- Rigidbody ベースの物理移動
- WASD キー入力対応
- カメラに対する相対移動
- 地面判定（Raycast）

## 対象ファイル

- Assets/Scripts/Player/PlayerMovement.cs（新規作成）
- Assets/Scripts/Player/GroundChecker.cs（新規作成）

## 制約

- Unity 2022.3 LTS を使用
- URP (Universal Render Pipeline) 環境
- 既存の InputSystem パッケージを利用すること
- テスト（EditMode / PlayMode）を含めること
```

### 4.3 実行フロー

```
 1. トリガー検知
    │
 2. Issue body / コメントの取得・パース（gh CLI 経由で安全に取得）
    │
 3. Issue に「実行開始」コメントを投稿
    │
 4. 作業ブランチ作成 or チェックアウト (tornado/issue-{number})
    │   - リモートに既存ブランチがあれば fetch して使用
    │   - なければ main から新規作成
    │
 5. Tornado 実行 (PowerShell)
    │   - dev agent: claude-code (コード生成)
    │   - review agent: codex (コードレビュー)
    │   - 終了コードを $LASTEXITCODE でキャプチャ
    │
 6. 一時ファイル削除 → 変更の検出・コミット・プッシュ
    │
 7. Pull Request の作成/更新
    │
 8. Issue に「実行完了」コメントを投稿
    │   - 成功/失敗ステータス
    │   - PR へのリンク
    │   - 実行ログの抜粋
    │
 9. (失敗時) tornado-failed ラベルを付与
    │
10. 一時ファイルのクリーンアップ
```

### 4.4 同時実行制御

- 同一 Issue に対する実行は同時に 1 つまで（`concurrency` グループで制御）
- 後続の実行はキューイングされる（`cancel-in-progress: false`）
- 異なる Issue の実行は並列に動作する

---

## 5. ブランチ戦略

| ブランチ名 | 用途 |
|-----------|------|
| `main` | メインブランチ（PR のマージ先） |
| `tornado/issue-{number}` | Issue ごとの作業ブランチ（自動作成） |

- 同一 Issue への再実行は同じブランチに追加コミットされる
- リモートに既存ブランチがあれば最新をフェッチして継続
- PR は初回実行時に作成され、以降は自動更新される

---

## 6. Issue を通じたインタラクション

### 6.1 Bot → 開発者（自動コメント）

| タイミング | 内容 |
|-----------|------|
| 実行開始時 | Run ID、トリガー種別、ブランチ名、Runner OS、Actions URL |
| 実行完了時 | 成功/失敗、PR リンク、実行ログ抜粋 |
| 失敗時 | 上記 + `tornado-failed` ラベル付与 |

### 6.2 開発者 → Bot（コマンド）

| コマンド | 動作 |
|---------|------|
| `/tornado` | Issue body の内容で再実行 |
| `/tornado <追加指示>` | Issue body + 追加指示で再実行 |

---

## 7. Agent 構成

Tornado はマルチエージェントで動作する。本ワークフローのデフォルト構成は以下の通り。

| ロール | Agent | 説明 |
|--------|-------|------|
| dev（開発） | `claude` (claude-code) | コード生成・実装を担当 |
| review（レビュー） | `codex` | コードレビュー・品質チェックを担当 |

`tornado.json` がリポジトリルートに存在する場合はその設定が優先される。

利用可能な Agent: `claude` / `claude-code`, `codex`, `api`, `mock`

---

## 8. ファイル配置

```
your-unity-project/
├── .github/
│   ├── ISSUE_TEMPLATE/
│   │   └── tornado-task.md              # Issue テンプレート（推奨）
│   └── workflows/
│       └── tornado-issue-driven.yml     # 本ワークフロー
├── Assets/                               # Unity Assets
├── Packages/
├── ProjectSettings/
├── tornado.json                          # (任意) Tornado 設定ファイル
└── ...
```

---

## 9. Windows / Unity 固有の考慮事項

### 9.1 シェル環境

- ワークフロー全体で **PowerShell 7+ (pwsh)** をデフォルトシェルとして使用
- `defaults.run.shell: pwsh` で統一
- Linux の `bash` コマンド（`sed`, `grep`, `tail` 等）は使用しない

### 9.2 パス区切り

- Windows のパス区切りは `\` だが、Git や Node.js は `/` も受け付ける
- 一時ファイルは `$env:RUNNER_TEMP`（Actions ランナーの一時ディレクトリ）に配置
- `/tmp` は使用しない

### 9.3 Unity プロジェクトとの親和性

- Tornado の dev agent (claude-code) は C# スクリプトの生成・編集を行う
- `.meta` ファイルの生成は Unity Editor が担当するため、Tornado 実行後に Unity を起動してアセットインポートが必要な場合がある
- 大規模なアセット（テクスチャ、モデル等）は Git LFS で管理することを推奨
- `Library/` フォルダは `.gitignore` に含めること

### 9.4 Unity ライセンス

- セルフホストランナー上の Unity Editor はあらかじめライセンス認証済みであること
- CI でのバッチモード実行が必要な場合は、Professional / Plus ライセンスまたは Personal のヘッドレス認証が必要

### 9.5 長いパスの問題

Windows の 260 文字パス制限に注意。以下を事前に設定しておくことを推奨:

```powershell
# Git の長いパス対応
git config --system core.longpaths true

# Windows レジストリ（管理者権限）
New-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\FileSystem" `
  -Name "LongPathsEnabled" -Value 1 -PropertyType DWORD -Force
```

---

## 10. セキュリティ設計

### 10.1 スクリプトインジェクション対策

Issue のタイトルや本文には任意の文字列が含まれるため、`${{ github.event.issue.title }}` 等の GitHub コンテキストをシェルスクリプトに直接展開すると、コマンドインジェクションの危険がある。

本ワークフローでの対策:

- Issue body / title は `gh issue view` コマンド経由で取得（直接展開しない）
- コメント本文は `gh api` 経由で取得
- GitHub コンテキストの直接展開は、安全な値（run_id, repository 等）のみに限定

### 10.2 API キー管理

- API キーは GitHub Secrets に格納し、ワークフローログには出力しない
- `GITHUB_TOKEN` は最小権限（`contents: write`, `issues: write`, `pull-requests: write`）で運用

### 10.3 運用上の注意

- セルフホストランナーは信頼できるネットワーク内に配置すること
- パブリックリポジトリでの運用時は、悪意あるプロンプトインジェクションに注意
- 外部コントリビューターからの Issue でも `tornado` ラベルが付けば実行されるため、**ラベル付与権限の管理が重要**
- Windows ランナー上では Tornado が生成するプロセスがアンチウイルスソフトに検知される場合がある。必要に応じて除外設定を行うこと

---

## 11. タイムアウトと制限

| 項目 | 値 |
|------|-----|
| ジョブタイムアウト | 60 分 |
| ログ出力上限（Issue コメント） | 末尾 100 行 / 3,000 文字 |
| 同時実行（同一 Issue） | 1 |

---

## 12. エラーハンドリング

| 状況 | 挙動 |
|------|------|
| Tornado 実行失敗 | Issue に失敗ログを投稿、`tornado-failed` ラベルを付与 |
| コード変更なし | Issue に「変更なし」と報告、PR は作成しない |
| API キー未設定 | Tornado 実行時にエラー、ログに記録 |
| ランナー未接続 | GitHub Actions がジョブをキュー待ちにする |
| リモートブランチ既存 | 最新をフェッチして継続 |
| 一時ファイル残存 | cleanup ステップで自動削除 |
| Windows パス長超過 | `core.longpaths` の設定を確認（セクション 9.5 参照） |

---

## 13. 運用例

### 例 1: Unity スクリプトの実装依頼

1. Issue を作成：タイトル「プレイヤー移動システムの実装」、本文に要件を記述
2. `tornado` ラベルを付与（テンプレート使用時は自動付与）
3. Bot が自動実行を開始、Issue にコメント
4. Tornado が C# スクリプト（PlayerMovement.cs 等）を生成
5. 完了後、PR `tornado/issue-42` が作成される
6. 開発者が PR をレビュー → Unity Editor でインポート確認 → マージ

### 例 2: 追加修正の依頼

1. 上記の Issue に `/tornado ジャンプ機能を追加してください。Space キーで発動。` とコメント
2. Bot が追加指示を含めて再実行
3. 同じ PR に追加コミットがプッシュされる

### 例 3: シェーダー修正

1. Issue を作成：タイトル「URP カスタムシェーダーのコンパイルエラー修正」
2. 本文にエラーログと対象シェーダーファイルのパスを記述
3. `tornado` ラベルを付与
4. Bot がシェーダーコードを分析・修正

---

## 14. トラブルシューティング

### ワークフローが起動しない

- Issue に `tornado` ラベルが付いているか確認
- `labeled` イベントの場合、付与されたラベルが `tornado` であること
- コメントトリガーの場合、コメントが `/tornado` で始まっているか確認
- PR のコメントではなく Issue のコメントであること

### ランナーがジョブを拾わない

- ランナーのラベルが `self-hosted` **と** `Windows` の両方を含んでいるか確認
- ランナーサービスが起動中か確認: `Get-Service actions.runner.*`
- GitHub Settings > Actions > Runners で接続状態を確認
- Windows Firewall が GitHub への接続をブロックしていないか確認

### Tornado が API エラーで失敗する

- `ANTHROPIC_API_KEY` / `OPENAI_API_KEY` が Secrets に登録されているか確認
- API キーの有効期限、レート制限を確認
- Windows ランナーからの外部通信がプロキシ/ファイアウォールで制限されていないか確認

### Node.js / npm 関連のエラー

- `actions/setup-node@v4` が正しく動作しているか確認
- セルフホストランナーの場合、Node.js のキャッシュが古い可能性がある
- `npm cache clean --force` を試す

### Git の文字化け・パスの問題

```powershell
# UTF-8 出力を強制
git config --global core.quotepath false
git config --global i18n.commitEncoding utf-8

# 長いパスを有効化
git config --global core.longpaths true
```

### Unity .meta ファイルの欠落

- Tornado が新規 C# ファイルを作成した場合、対応する `.meta` ファイルは存在しない
- PR をマージ後、Unity Editor で Asset Database のリフレッシュが必要
- 必要に応じてワークフローに Unity バッチモードのインポートステップを追加可能

---

## 15. カスタマイズ

### Agent の変更

ワークフロー YAML 内の `--dev` / `--review` オプションを変更:

```powershell
# 例: 両方 claude を使用
npx -y @mizchi/tornado $ExecPlan --dev=claude --review=claude

# 例: API モードを使用
npx -y @mizchi/tornado $ExecPlan --dev=api --review=api
```

### タイムアウトの変更

```yaml
timeout-minutes: 120  # 必要に応じて延長
```

### ラベルの変更

トリガーラベルを変更する場合は、ワークフロー YAML 内の以下を修正:

- `contains(github.event.issue.labels.*.name, 'tornado')`
- `github.event.label.name == 'tornado'`

### ベースブランチの変更

PR のマージ先を `main` 以外にする場合:

```yaml
# ステップ 7: ブランチ作成で origin/develop に変更
# ステップ 10: --base develop に変更
```

### Unity バッチモード実行の追加（オプション）

Tornado 実行後に Unity のコンパイルチェックを行う場合:

```yaml
- name: Unity compile check
  if: steps.commit.outputs.has_changes == 'true'
  run: |
    $UnityPath = Join-Path $env:UNITY_EDITOR_PATH "2022.3.XXf1\Editor\Unity.exe"
    & $UnityPath -batchmode -nographics -projectPath . -quit -logFile - 2>&1 | Tee-Object -FilePath compile.log
    if ($LASTEXITCODE -ne 0) {
      Write-Error "Unity compilation failed"
      exit 1
    }
```
