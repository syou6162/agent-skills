---
name: orchestrating-cursor-cloud-agent
description: Cursor Cloud Agentにタスクを委託するよう依頼された際に発動。エージェント作成、SSEストリーム監視（stream）、実行キャンセル（cancel）、ステータスポーリング（フォールバック）、Pull Request状態確認、Linear紐付け、レビュー・フォローアップを行う。
compatibility: Requires cursor-agent-cli, CURSOR_CLOUD_AGENT_API_KEY environment variable, gh CLI, and Linear MCP server (https://mcp.linear.app/mcp)
---

# Cursor Cloud Agent オーケストレーション

あなた自身がオーケストレーターとしてCursor Cloud Agentにタスクを委託し、監視・レビュー・フォローアップを行います。コーディング作業はCursorに任せ、あなたは判断・監視・指示に特化します。

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

プロンプトは `-prompt` フラグに加えて、標準入力（パイプ）からも渡せます。優先順位は `-prompt` フラグ > 標準入力 > エラーです。標準入力はパイプ経由の場合のみ有効で、TTY（対話端末）では読み取りません。

```bash
echo "<タスク指示>" | cursor-agent-cli create \
  -repo "https://github.com/<owner>/<repo>" \
  -branch "<開始ブランチ>"
```

長いプロンプトやシェルのクオート問題を避けたい場合は、ファイルに書いてパイプで渡すと安全です。

```bash
cat prompt.txt | cursor-agent-cli create \
  -repo "https://github.com/<owner>/<repo>" \
  -branch "<開始ブランチ>"
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

以下の場合に、実行中のrunをキャンセルします。

- Cursorが応答しなくなった場合
- オーケストレーターであるあなたが、Cursorの作業が間違った方向に進んでいると判断した場合
- 作業中のCursorに追加で指示を出したい場合

```bash
cursor-agent-cli cancel <agent_id> <run_id>
```

<important>

- キャンセルは不可逆。会話を続ける場合は `cursor-agent-cli run` で新しいrunを作成する。
- 既に終了した実行のキャンセルは `409` エラーになる。
- **RUNNING中に追加指示を送ることはできない**（`cursor-agent-cli run <agent_id> -prompt "..."` は `409 agent_busy` になる）。追加指示や方向修正が必要な場合は、まず `cancel` してから `run` で新しい指示を送ること。

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

### 5. Pull Requestレビュー

Pull Requestのコードレビューは、**PRブランチを手元にチェックアウトしてから**行います。差分だけでなく、呼び出し元・関連ファイル・既存の設計との整合性まで踏み込んで確認します。

1. デフォルトブランチ名を取得する：

```bash
git symbolic-ref refs/remotes/origin/HEAD --short | cut -d/ -f2
```

2. リモートのデフォルトブランチを最新化する（ローカル `main` のチェックアウトや pull は不要）：

```bash
git fetch origin <デフォルトブランチ名>
```

3. PRブランチを手元にチェックアウトする：

```bash
gh pr checkout <pr-url>
```

4. 差分を確認する（基準は `origin/<デフォルトブランチ名>`）：

```bash
git diff origin/<デフォルトブランチ名>...HEAD
git log origin/<デフォルトブランチ名>..HEAD
```

5. 差分だけでなく、変更箇所の周辺コード・呼び出し元・関連テスト・既存の類似実装も読んでレビューする。

<important>

- **レビュー前に必ず `gh pr checkout` でPRブランチを手元に持ってくること**。
- **差分の基準は `origin/<デフォルトブランチ名>` を使うこと**。ローカルの `<デフォルトブランチ名>` は古いことが多く、PRの実際の差分と一致しない。
- `git fetch origin <デフォルトブランチ名>` でリモート追跡ブランチを更新すれば十分。レビューのためにローカル `main` をチェックアウトして `git pull` する必要はない。

</important>

### 6. Linear 紐付け

Cursor が Pull Request を作成したら、関連する Linear issue に Pull Request を紐付けます。

1. Linear MCP の `get_issue` または `search_issues` で該当 issue を特定
2. 既存のリンクを取得（`get_issue` で `links` フィールドを確認）
3. `update_issue` で Pull Request URL をリソースリンクとして追記

既存のリンクを上書きしないよう、追記する形で更新してください。

### 7. 追加プロンプト（修正指示・方向修正）

Pull Request確認やレビューで修正が必要な場合、または作業中のCursorの方向を修正したい場合に、追加プロンプトを投入します。

```bash
cursor-agent-cli run <agent_id> -prompt "<修正指示>"
```

`create` と同様、`run` でも `-prompt` フラグに加えて標準入力（パイプ）からプロンプトを渡せます。優先順位は `-prompt` フラグ > 標準入力 > エラーです。

```bash
cat fix_instructions.md | cursor-agent-cli run <agent_id>
```

長い修正指示はファイルに書いてパイプで渡すことを推奨します。シェルのクオート問題を回避でき、具体的なコード例を含めやすくなります。

<important>

- 前の実行が `RUNNING` 中に `cursor-agent-cli run` を実行すると `409 agent_busy` になる。完了済み（`FINISHED` / `CANCELLED` 等）でないと新しいrunは作成できない。
- **作業中に方向修正・追加指示が必要な場合**: `cancel` → `run` の順で実行する。
  ```bash
  # 1. 現在の実行をキャンセル
  cursor-agent-cli cancel <agent_id> <run_id>
  # 2. 新しい指示で再実行
  cursor-agent-cli run <agent_id> -prompt "<修正後の指示>"
  ```
- 修正プロンプトは抽象的な説明ではなく、具体的なコード例を含めること。

</important>

### 8. マージ完了後の Linear ステータス更新

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

## 重要な注意事項

### すべきこと

<important>

- `cursor-agent-cli create` は自動的に `autoCreatePR: true` を送信する
- エージェント作成直後に `agent.url` をユーザーに提示する
- リアルタイム監視は `cursor-agent-cli stream` で行う（推奨）。フォールバックとして `cursor-agent-cli status --watch` も利用可
- Cursorが応答しなくなったら `cursor-agent-cli cancel` で強制終了する
- `FINISHED` 後は必ず `gh pr view` でPull Request状態を確認する
- Pull Requestレビュー前に `git fetch origin <デフォルトブランチ名>` し、`gh pr checkout` でPRブランチを手元に持ってきてからレビューする。差分は `origin/<デフォルトブランチ名>` を基準にする（ローカル `main` は古くて実際のPR差分と一致しないことがある）
- Linear 紐付けは既存リンクを上書きせず追記する
- 修正プロンプトは具体的なコード例を含める
- Linear issue を Done にするのは、Pull Request がマージされた後のみ（`gh pr view` で `state` が `MERGED` を確認してから）
- レビューと Copilot レビュー対応を同じ PR で行う場合、先に自身のレビューで構造的な問題を直してから Copilot コメントを処理する（構造的修正で Copilot 指摘が自然解消される場合がある）

</important>

### してはいけないこと

<important>

- **オーケストレーター自身がPRブランチにコードをコミット・pushしないこと**。Cursorが後追いで修正を行い、変更が競合してぐちゃぐちゃになるため。修正が必要な場合は必ず `cursor-agent-cli run` 経由でCursorに指示すること。レビューで問題を発見した場合も、自分で直すのではなくCursorへの修正プロンプトとして投入する。なお、PRコメントやラベル付け等のメタ操作はこの制限の対象外。
- `autoCreatePR` を後から変更しようとしない（`cursor-agent-cli create` は自動的に `true` を送信する）
- Pull Requestの状態を仮定で報告しない
- `RUNNING` 中に追加プロンプトを投げない（`409 agent_busy` になる）。方向修正・追加指示が必要な場合は `cancel` してから `run` すること
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

## 参考リンク

- [cursor-agent-cli](https://github.com/syou6162/cursor-agent-cli)
- [Cursor Cloud Agent API](https://api.cursor.com/v1)
- [Linear MCP Server](https://linear.app/docs/mcp)
