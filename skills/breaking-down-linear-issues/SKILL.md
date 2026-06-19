---
name: breaking-down-linear-issues
description: Linearに大きなタスクを複数のIssueに分割して作成する際に使用。階層構造を避け、Issue間の依存関係はblockedByリレーションで表現する。
compatibility: Requires Linear GraphQL API or Linear MCP server (https://mcp.linear.dev)
---

# Linearに大きなタスクを分割してIssueを作成する

ユーザーが大きなタスクをLinearにIssueとして登録したい場合、タスクを分割して複数のIssueを作成してください。Issue間の関係は、**階層構造ではなく`blockedBy`リレーション**で表現します。

## 制約

<important>

- 各Issueは独立した同等のフラットな単位として作成する。sub-issueや親子関係は作らない。
- Issue間の依存関係は、必ず `blockedBy` リレーションで表現する。
- <procedure>タグの手順を一つずつ順番に実行する。
- ユーザーが承認するまで、Issueの作成を実行しない。

</important>

## プロンプトの扱い

<important>

- 呼び出し時のプロンプトに特に明確な指示がされていない場合は、<procedure>タグの実行手順通りに進めてください
- プロンプトに特定の意図（例：「3つのIssueに分割して」「Aを先にやってからBを作成して」など）が加えられている場合は、その意図をタスク分解と依存関係の設定に反映してください
- ただし、**<procedure>タグの実行手順は一切変更せず**、記載された手順に従って実行してください

</important>

## 実行手順

<procedure>

以下の手順で、大きなタスクを分割してLinearにIssueを作成してください。

1. **入力の整理**

   ユーザーから以下の情報を確認してください。情報が不足している場合は、ユーザーに確認を取ってください。

   - タスクの概要（タイトル・目的）
   - 分割したい作業単位
   - 各作業の依存関係（どのIssueが他のIssueをブロックするか）
   - Linearのチーム（Team）またはプロジェクト（Project）
   - ラベル、担当者、期限などの属性（任意）

2. **タスクの分解**

   ユーザーから得た情報に基づき、タスクを意味のある単位に分割してください。

   - 各Issueは独立して完了可能な単位にすること
   - 1つのIssueに詰め込みすぎないこと
   - 分割したIssue間の依存関係を明確にすること

3. **作成内容の確認**

   ユーザーに以下の内容を提示し、承認を得てから作成を実行してください。

   - 作成するIssueの一覧（タイトルと説明）
   - 各Issue間の依存関係（`blockedBy` で表現）
   - チーム/プロジェクト、ラベル、担当者などの設定

4. **Issueの作成**

   承認を得たら、Linear GraphQL APIまたはLinear MCP serverを使ってIssueを作成してください。

   ```bash
   # Linear MCP serverを利用する場合の例（mcp_call_toolで実行）
   # Linear issue create のスキーマに従って引数を指定する
   ```

   - 各Issueを個別に作成する
   - 作成後に各IssueのID（identifierまたはUUID）を記録する

5. **blockedByリレーションの設定**

   作成したIssue間で依存関係がある場合は、**必ず`blockedBy`リレーション**で設定してください。

   - 後に実行すべきIssueを先に作成されたIssueでブロックする
   - `blockedBy` の方向に注意する

6. **結果の報告**

   ユーザーに以下を報告してください。

   - 作成したIssueのタイトルとURL
   - 設定した`blockedBy`の一覧
   - 次に行うべき作業（あれば）

</procedure>

## 依存関係の表現

<decision-criteria name="dependency-expression">

| 状況 | 表現方法 | 理由 |
|------|----------|------|
| Issue A が Issue B の完了を待っている | Issue B を Issue A の `blockedBy` に設定 | B が完了するまで A が進めないという関係を表現 |
| 複数の Issue が同じ Issue をブロックしている | 複数の `blockedBy` リレーションを設定 | 複数の依存関係を並列で表現 |
| 特に依存関係がない | `blockedBy` を設定しない | 依存関係がない場合は無理に作らない |

</decision-criteria>

## 例

<example name="split-issue">

### 入力

「ユーザー認証機能を作りたい。大きなタスクなので分割してIssueを作成してほしい。」

### 分割結果

1. ログイン画面のUI作成
2. 認証APIの実装
3. 認証APIのテスト作成
4. フロントエンドと認証APIの連携

### 依存関係

- 認証APIのテスト作成は、認証APIの実装をブロックする
- フロントエンドと認証APIの連携は、ログイン画面のUI作成と認証APIの実装をブロックする

### 実行後の状態

- 認証APIの実装 → 認証APIのテスト作成 (`blockedBy`)
- ログイン画面のUI作成 → フロントエンドと認証APIの連携 (`blockedBy`)
- 認証APIの実装 → フロントエンドと認証APIの連携 (`blockedBy`)

</example>
