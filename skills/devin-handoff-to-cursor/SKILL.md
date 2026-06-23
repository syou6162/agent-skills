---
name: devin-handoff-to-cursor
description: Devin CLIからDevin Cloudにハンドオフされた際に発動。受け取ったタスクを自分で実装せず、Cursor Cloud Agentにオーケストレーションとして委託する。orchestrating-cursor-cloud-agentスキルの手順に従い、監視・レビュー・フォローアップを行う。
compatibility: Requires Devin Cloud session (handoff from Devin CLI), cursor-agent-cli, CURSOR_CLOUD_AGENT_API_KEY environment variable, gh CLI, and Linear MCP server (https://mcp.linear.app/mcp)
---

# Devin Handoff → Cursor オーケストレーション

Devin CLI から `/handoff` でクラウドセッションに引き継がれたタスクを、自分で実装せず Cursor Cloud Agent に委託してオーケストレーションを行います。

## 使用タイミング

<trigger>

以下の場合にこのスキルを発動してください：

- Devin CLI から `/handoff` コマンドでクラウドセッションを作成するとき
- セッションプロンプトにハンドオフ由来の前提情報（リポジトリ、ブランチ、未コミットの変更、会話コンテキスト）が含まれているとき
- ユーザーが「Devin に引き継いで」「handoff して」と言及し、タスクがコーディング作業を含むとき

</trigger>

## 背景と役割分担

<context>

- **Devin Cloud（オーケストレーター）**: 実行コストが高い。自分でコードを書かず、タスク設計・委託・監視・レビュー・修正指示に徹する
- **Cursor Cloud Agent（コーディング担当）**: 安価で高速だが、判断力は限定的。コーディング作業を担当する
- **この分担の目的**: ハンドオフされたタスクを低コストかつ高品質に完遂する

</context>

## 前提条件

- Devin Cloud セッションであること（Devin CLI からのハンドオフ）
- 環境変数 `CURSOR_CLOUD_AGENT_API_KEY` が設定されていること
- `cursor-agent-cli` がインストール済みであること
- `gh` CLI がインストール・認証済みであること
- Linear MCP server が設定・認証済みであること（Linear issue がある場合）

## 実行手順

<procedure>

### 1. ハンドオフコンテキストの整理

ハンドオフで渡された前提情報を整理します。

確認すべき項目：

- **リポジトリURL**: ハンドオフ元の `git remote` から引き継がれたリポジトリ
- **ブランチ**: 引き継がれたブランチ名（デフォルトブランチか作業ブランチか）
- **未コミットの変更**: `git diff HEAD` として渡された差分（あれば）
- **会話コンテキスト**: Devin CLI での会話で得られた情報（確認したファイル、根本原因の仮説、部分的な修正など）
- **タスク内容**: ユーザーが `/handoff` で指定したタスクの説明

### 2. Cursor へのプロンプト構築

ハンドオフコンテキストを元に、Cursor Cloud Agent に渡すプロンプトを構築します。

<important>

プロンプトには以下をすべて含めること：

- ハンドオフ元で得られた前提情報（根本原因の仮説、確認済みファイル、部分的な修正内容など）
- 未コミットの変更がある場合、その差分を適用するか新たに実装するかの判断と指示
- タスクの具体的なゴール
- CI 確認指示（`orchestrating-cursor-cloud-agent` スキルに記載の定型文）

</important>

### 3. orchestrating-cursor-cloud-agent スキルに従って実行

以降の手順は `orchestrating-cursor-cloud-agent` スキルの実行手順に従います：

1. **エージェント作成** — `cursor-agent-cli create` でタスクを委託
2. **リアルタイムストリーム監視** — `cursor-agent-cli stream` で進捗を監視
3. **Pull Request 状態確認** — `gh pr view` で PR の実際の状態を確認
4. **Pull Request レビュー** — PR ブランチをチェックアウトしてコードレビュー
5. **Linear 紐付け** — 関連する Linear issue があれば PR を紐付け
6. **追加プロンプト** — 修正が必要な場合は `cursor-agent-cli run` で修正指示
7. **マージ完了後の Linear ステータス更新** — マージ後に Done に変更

<important>

- `orchestrating-cursor-cloud-agent` スキルの「してはいけないこと」を厳守すること。特に、オーケストレーター自身が PR ブランチにコードをコミット・push しないこと。
- 未コミットの変更が渡された場合は、その内容を Cursor へのプロンプトに含めて Cursor 側で適用させること（自分で apply しない）。

</important>

### 4. ユーザーへの報告

タスク完了後、以下をユーザーに報告します：

- Pull Request の URL と状態（draft / open / merged）
- CI の状態
- レビュー結果のサマリ
- Linear issue の紐付け状況（該当する場合）

</procedure>

## 重要な注意事項

### Devin Cloud がやらないこと（ハンドオフオーケストレーション時）

<important>

- `edit` / `write` / `MultiEdit` によるソースコード編集
- `git checkout -b` によるブランチ作成
- 直接のコード変更・PR 作成

修正が必要な場合は必ず `cursor-agent-cli run` 経由で Cursor に指示すること。

</important>

### Devin Cloud がやってよいこと

- `gh pr checkout` / `git checkout` でレビュー用に既存ブランチをチェックアウト
- `git diff` / `git log` / `read` / `grep` などの読み取り操作
- `cursor-agent-cli` 経由で Cursor にプロンプト・修正指示を送る
- `git_view_pr` / `git_pr_checks` / `git_take_over_pr` などの PR 確認操作

## 参考リンク

- [orchestrating-cursor-cloud-agent スキル](../orchestrating-cursor-cloud-agent/SKILL.md)
- [Devin Handoff ドキュメント](https://docs.devin.ai/ja/work-with-devin/devin-handoff)
- [Devin Handoff プラグイン](https://github.com/club-cog/devin-handoff)
- [cursor-agent-cli](https://github.com/syou6162/cursor-agent-cli)
