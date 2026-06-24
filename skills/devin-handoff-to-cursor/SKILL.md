---
name: devin-handoff-to-cursor
description: Devin CLIからDevin Cloudにハンドオフされた際に発動。受け取ったタスクの実行をCursor Cloud Agentに委譲する。
compatibility: Requires Devin Cloud session, cursor-agent-cli, CURSOR_CLOUD_AGENT_API_KEY environment variable
---

# Devin Handoff → Cursor オーケストレーション

Devin CLI から `/handoff` でクラウドセッションに引き継がれたタスクを受け取ったら、自分で実装せず `orchestrating-cursor-cloud-agent` スキルに従って Cursor Cloud Agent に実行を委譲してください。

## 使用タイミング

<trigger>

- Devin CLI から `/handoff` コマンドでクラウドセッションを作成するとき
- セッションプロンプトにハンドオフ由来の前提情報（リポジトリ、ブランチ、未コミットの変更、会話コンテキスト）が含まれているとき

</trigger>

## やること

1. ハンドオフで渡されたコンテキスト（タスク内容、リポジトリ、ブランチ、未コミットの変更、会話で得られた情報）を整理する
2. `orchestrating-cursor-cloud-agent` スキルの手順に従い、Cursor Cloud Agent にタスクを委託・監視・レビューする
