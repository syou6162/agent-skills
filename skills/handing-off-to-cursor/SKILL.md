---
name: handing-off-to-cursor
description: Devin CLIから /handoff でDevin Cloudにタスクを引き継ぐ際に発動。ハンドオフ先のDevin Cloudに対して、タスクの実行をCursor Cloud Agentに委譲するよう指示を含める。
compatibility: Requires Devin CLI with /handoff command
---

# Devin Handoff → Cursor 委譲

## 使用タイミング

<trigger>

- Devin CLI で `/handoff` を実行してDevin Cloudにタスクを引き継ぐとき

</trigger>

## 委譲の流れ（二段階）

1. **Devin CLI → Devin Cloud**: `/handoff` でタスク・コンテキストを引き継ぐ（通常のhandoff）
2. **Devin Cloud → Cursor Cloud Agent**: Devin Cloud が `orchestrating-cursor-cloud-agent` ナレッジに従い、コーディング作業を Cursor に委託する

## やること

通常の `/handoff` に加えて、ハンドオフ先の Devin Cloud に対して以下を伝えること：

- タスクの実装は自分（Devin Cloud）で行わず、Cursor Cloud Agent に委譲すること
- 委譲の手順は `orchestrating-cursor-cloud-agent` ナレッジに従うこと
