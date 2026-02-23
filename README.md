# SelfHostedTornadoAction

GitHub Issue を起点に `@mizchi/tornado` を実行し、Windows Self-Hosted Runner 上で実装とレビューを自動化するサンプルです。

## 主要ファイル

- ワークフロー: `.github/workflows/tornado-issue-driven.yml`
- Issue テンプレート: `.github/ISSUE_TEMPLATE/tornado-task.md`
- 仕様書: `tornado-issue-driven-spec.md`

※ ルートの `tornado-task.md` と `tornado-issue-driven.yml` は参照用です。

## 最短セットアップ

1. ラベル作成
- `tornado`
- `tornado-failed`（任意）

2. Secrets 設定
- `ANTHROPIC_API_KEY`
- `OPENAI_API_KEY`

3. Variables 設定（任意）
- `UNITY_EDITOR_PATH`（既定: `C:\Program Files\Unity\Hub\Editor`）

4. Self-Hosted Runner
- ラベル: `self-hosted`, `Windows`
- `gh` CLI が利用できること
- Node.js が利用できること（ワークフローで `setup-node` も実行）

5. 動作確認
- Issue を `tornado-task` テンプレートで作成
- `tornado` ラベルを付与
- Actions が起動し、Issue に開始/完了コメントが投稿されることを確認

## 補足

- 文字コードは UTF-8 を前提です（`.editorconfig` で固定）。
- ワークフロー検証として `.github/workflows/actionlint.yml` を追加しています。
