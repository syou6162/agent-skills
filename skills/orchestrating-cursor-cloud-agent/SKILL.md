---
name: orchestrating-cursor-cloud-agent
description: Cursor Cloud Agentにタスクを委託するよう依頼された際に発動。エージェント作成、SSEストリーム監視（stream）、実行キャンセル（cancel）、ステータスポーリング（フォールバック）、Pull Request状態確認、Linear紐付け、レビュー・フォローアップを行う。
compatibility: Requires cursor-agent-cli, CURSOR_CLOUD_AGENT_API_KEY environment variable, gh CLI, and Linear MCP server (https://mcp.linear.app/mcp)
---

# Cursor Cloud Agent オーケストレーション

DevinがオーケストレーターとしてCursor Cloud Agentにタスクを委託し、監視・レビュー・フォローアップを行います。コーディング作業はCursorに任せ、Devinは判断・監視・指示に特化します。

## 使用タイミング

<trigger>

以下の場合にこのスキルを発動してください：

- ユーザーが「Cursorに投げて」「Cursor Cloud Agentに任せて」「Cursorに委託して」と言及したとき
- ユーザーがCursor Cloud Agentを使ったタスク実行を依頼したとき

</trigger>

## 前提条件

- 環境変数 `CURSOR_CLOUD_AGENT_API_KEY` が設定されていること（Basic認証に使用）
- `gh` CLI がインストール・認証済みであること（Pull Request状態確認に使用）
- Linear MCP server (`https://mcp.linear.app/mcp`) が設定・認証済みであること
- `cursor-agent-cli` がインストール済みであること（`go install github.com/syou6162/cursor-agent-cli@latest`）

## 実行手順

<procedure>

### 1. タスク内容の確認

ユーザーと会話して、Cursor Cloud Agentに委託するタスク内容を固めます。

確認すべき項目：

- ターゲットリポジトリのURL
- 開始ブランチ（未指定の場合は `main`）
- タスクの概要・目的
- 関連するLinear issue（あれば）
- 特別な制約や注意点

### 2. エージェント作成

Cursor Cloud Agent APIでエージェントを作成します。

```bash
cursor-agent-cli create \
  -repo "https://github.com/<owner>/<repo>" \
  -branch "<開始ブランチ>" \
  -prompt "<タスク指示>"
```

<important>

- `cursor-agent-cli create` は自動的に `autoCreatePR: true` を送信する。後から変更できないため、CLI の挙動に依存してよい。
- レスポンスの `agent.url` を必ずユーザーに提示すること。API経由で作成したエージェントはWeb UIの一覧に表示されないことがあるため、直リンクが必要。

</important>

レスポンスから `agent.id` と `run.id` を抽出し、後続のステップで使用します。

### 3. リアルタイムストリーム監視（推奨）

SSEストリームに接続し、エージェントの実行をリアルタイムで監視します。

```bash
cursor-agent-cli stream <agent_id> <run_id>
```

- NDJSON（1行1JSON）でイベントを出力
- 主要イベント: `status`（状態変更）、`assistant`（テキストデルタ）、`thinking`（思考デルタ）、`tool_call`（ファイル読み書き・シェル実行等）、`result`（最終結果）、`done`（終了）
- **ポーリングよりも推奨**: Cursorが「今何をやっているか」が分かるため、オーケストレーターとしての判断精度が上がる
- 終了コード: `done` イベント受信時は 0、`error` イベント受信時は 2
- `result` イベントに最終返答テキスト・`git` 情報（Pull Request URL含む）が含まれる

### 3b. ステータスポーリング（フォールバック）

ストリームが使えない場合のフォールバック手段です。

```bash
cursor-agent-cli status <agent_id> <run_id> --watch
```

- `cursor-agent-cli status` はデフォルトで15秒間隔でポーリングする。`--watch` を付けると、終了状態に到達するまで自動的にポーリングを続行する。
- **終了条件**: `status` が `FINISHED` / `ERROR` / `CANCELLED` / `EXPIRED` のいずれか
- `FINISHED` 時に `result` フィールドに最終返答テキスト、`git.branches[0].prUrl` にPull RequestのURLが含まれる

### 3c. 実行のキャンセル

Cursorが応答しなくなった場合に、実行中のrunを強制終了します。

```bash
cursor-agent-cli cancel <agent_id> <run_id>
```

<important>

- キャンセルは不可逆。会話を続ける場合は `cursor-agent-cli run` で新しいrunを作成する。
- 既に終了した実行のキャンセルは `409` エラーになる。

</important>

### 4. Pull Request状態確認

`FINISHED` 後、必ず `gh pr view` でPull Requestの実際の状態を確認します。

```bash
gh pr view <pr-url> --json state,isDraft,mergeStateStatus,statusCheckRollup
```

確認項目：

- Pull Request が draft かどうか
- CI の状態
- マージ可能かどうか（`mergeStateStatus` が `CLEAN` など）
- Pull Request がマージ済みかどうか（`state` が `MERGED`）

<important>

- **Pull Requestの状態は仮定で報告しないこと**。必ず `gh pr view` の結果に基づいて報告すること。
- draft の場合は、ユーザーに「Pull Requestはdraft状態です。draft を外すと Copilot review もトリガーされます」と伝えること。
- **マージ可能（`mergeStateStatus` が `CLEAN` など）とマージ済み（`state` が `MERGED`）は別物**。マージ可能なだけでは Linear issue を Done にしないこと。

</important>

### 5. Linear 紐付け

Cursor が Pull Request を作成したら、関連する Linear issue に Pull Request を紐付けます。

1. Linear MCP の `get_issue` または `search_issues` で該当 issue を特定
2. 既存のリンクを取得（`get_issue` で `links` フィールドを確認）
3. `update_issue` で Pull Request URL をリソースリンクとして追記

既存のリンクを上書きしないよう、追記する形で更新してください。

### 6. 追加プロンプト（修正指示）

Pull Request確認やレビューで修正が必要な場合、追加プロンプトを投入します。

```bash
cursor-agent-cli run <agent_id> -prompt "<修正指示>"
```

<important>

- 前の実行が `RUNNING` 中に送ると `409 agent_busy`。必ず完了を待ってから投入すること。
- 修正プロンプトは抽象的な説明ではなく、具体的なコード例を含めること。

</important>

### 7. マージ完了後の Linear ステータス更新

Pull Request が**実際にマージされた後**にのみ、Linear issue のステータスを Done に変更します。

<important>

- **マージ可能な状態では Done にしないこと**。CI が通り、`mergeStateStatus` が `CLEAN` であっても、Pull Request がまだ open のままなら Done にしてはいけない。
- **Done にする条件**: `gh pr view` で `state` が `MERGED` であることを確認してから更新する。
- マージ前の報告では「マージ可能です」「CI は通っています」などと伝え、Done への更新はマージ完了を待つ。

</important>

1. `gh pr view <pr-url> --json state` で `state` が `MERGED` であることを確認
2. `list_issue_statuses` でチーム内のステータス一覧を取得し、Done に相当するステータス ID を特定
3. `update_issue` の `stateId` で該当ステータス ID を指定して更新

</procedure>

## オーケストレーションパターン

### パターンA: タスク委託 → Pull Request確認

1. ユーザーと会話してタスク内容を固める
2. Cursorにエージェント作成APIで投げる
3. セッションURLをユーザーに提示
4. `cursor-agent-cli stream` でリアルタイム監視（フォールバック: `status --watch`）
5. `gh pr view` でPull Request状態を確認
6. Pull RequestのURL・状態を報告
7. Linear に Pull Request 紐付け

### パターンB: CI失敗時の修正

1. `gh pr view` / `gh pr checks` でCI確認
2. 失敗内容を読み取り
3. 修正プロンプトを `cursor-agent-cli run <agent_id> -prompt "<修正指示>"` で投入
4. CIパスまで繰り返す

### パターンC: Copilot review 対応

1. `gh pr view` でPull Requestの状態を確認
2. draft の場合はユーザーに draft を外すよう報告（Copilot review 依頼のトリガー）
3. `gh pr view` でレビューコメントを検知
4. 修正が必要なものを修正プロンプトとしてCursorに投げる

### パターンD: Devin自身がレビュアー

1. `gh pr checkout <pr-url>` で該当PRのブランチを手元にチェックアウトする
2. `git diff --merge-base <base-branch>` でPRの差分を手元で確認し、以下の観点で徹底的にレビューする：
   - **意図との整合性**: ユーザーの元のタスク指示・目的に対して、変更内容が意図通りになっているか
   - **作業の漏れ**: タスクで要求された内容に対して、未実装・未対応の箇所がないか
   - **余計な変更**: タスクの範囲外の不要な変更や、意図しない副作用を含む変更がないか
   - **テスト網羅性**: プロダクションコードのエラーパス・分岐が全てテストされているか
   - **コード一貫性**: リポジトリ全体の命名規則・コードスタイル・構造パターンとの整合性
   - **設計上の問題**: 不要な複雑さ、責務の混在、抽象化レベルの不統一
3. 問題を発見したら、具体的なコード例を含む修正プロンプトを `cursor-agent-cli run <agent_id> -prompt "<修正指示>"` で投入
4. 修正 run は1つずつ完了を待ってから次を送る
5. 修正完了後に CI green を確認

### パターン D → C の逐次実行

Devin レビュー（パターンD）と Copilot レビュー対応（パターンC）を同じ Pull Request で行う場合：

1. まず Devin 自身がレビューして修正指示 → CI green 確認
2. 次に Copilot コメントを処理（対応 → resolve）
3. 最終的に CI green を確認してユーザーに報告
4. Pull Request がマージされたら Linear issue を Done に更新（マージ前は Done にしない）

この順序にする理由：Devin レビューで構造的な問題を先に直しておくと、Copilot の指摘の一部が自然に解消される場合がある。

## 重要な注意事項

### すべきこと

<important>

- `cursor-agent-cli create` は自動的に `autoCreatePR: true` を送信する
- エージェント作成直後に `agent.url` をユーザーに提示する
- リアルタイム監視は `cursor-agent-cli stream` で行う（推奨）。フォールバックとして `cursor-agent-cli status --watch` も利用可
- Cursorが応答しなくなったら `cursor-agent-cli cancel` で強制終了する
- `FINISHED` 後は必ず `gh pr view` でPull Request状態を確認する
- Linear 紐付けは既存リンクを上書きせず追記する
- 修正プロンプトは具体的なコード例を含める
- Linear issue を Done にするのは、Pull Request がマージされた後のみ（`gh pr view` で `state` が `MERGED` を確認してから）

</important>

### してはいけないこと

<important>

- `autoCreatePR` を後から変更しようとしない（`cursor-agent-cli create` は自動的に `true` を送信する）
- Pull Requestの状態を仮定で報告しない
- `RUNNING` 中に追加プロンプトを投げない（`409 agent_busy` になる）
- 複数の修正を同時にCursorに投げない（1つずつ完了を待つ）
- マージ可能な状態で Linear issue を Done にしない（マージされて初めて Done にする）

</important>

### 既知の制約

<important>

- `autoCreatePR` は作成時のみ設定可能（`cursor-agent-cli create` は自動的に `true` を送信する）
- CursorのPull Requestはデフォルトdraft になることが多いが、必ず `gh pr view` で実際の状態を確認する
- Copilot reviewの依頼はAPI/bot経由では不可。人間がGitHub UI上で行う
- API経由で作成したエージェントはWeb UIの一覧に出ないことがある。直リンクをユーザーに提示する
- 1エージェント1実行。前の実行が終わってから次を投げる

</important>

## 実装例

<examples>

### 例1: エージェント作成とポーリング

<example>

**エージェント作成：**

```bash
cursor-agent-cli create \
  -repo "https://github.com/foo/bar" \
  -branch "main" \
  -prompt "foo/bar リポジトリの README.md を更新し、セットアップ手順を追加してください。"
```

**レスポンスからの抽出：**

- `agent.url`: `https://cursor.com/agents/<id>`
- `agent.id`: `<id>`
- `run.id`: `<run_id>`

**ユーザーへの提示：**

> Cursor Cloud Agent セッションを作成しました: `https://cursor.com/agents/<id>`

**リアルタイム監視（推奨）：**

```bash
cursor-agent-cli stream <id> <run_id>
```

`done` イベントを受信するまでリアルタイムでNDJSON出力される。`result` イベントに最終結果とPull Request URLが含まれる。

**フォールバック（ストリームが使えない場合）：**

```bash
cursor-agent-cli status <id> <run_id> --watch
```

`status` が `FINISHED` になるまで `cursor-agent-cli status --watch` でポーリングする。

**キャンセル（応答しなくなった場合）：**

```bash
cursor-agent-cli cancel <id> <run_id>
```

</example>

### 例2: Pull Request状態確認

<example>

```bash
gh pr view https://github.com/foo/bar/pull/123 --json state,isDraft,mergeStateStatus,statusCheckRollup
```

**draft の場合の報告例：**

> Pull Request は作成されました: https://github.com/foo/bar/pull/123
> 現在 draft 状態です。draft を外すと Copilot review もトリガーされます。

</example>

</examples>

## 参考リンク

- [cursor-agent-cli](https://github.com/syou6162/cursor-agent-cli)
- [Cursor Cloud Agent API](https://api.cursor.com/v1)
- [Linear MCP Server](https://linear.app/docs/mcp)
