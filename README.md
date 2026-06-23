# agent-skills

Yasuhisa Yoshida's personal agent skills for Claude Code, Codex, and other coding agents.

## Overview

このリポジトリは、Claude Code、Codex、その他のコーディングエージェントで利用できる agent skills（スキル）を管理する個人用リポジトリです。各スキルは `skills/<skill-name>/SKILL.md` に配置され、エージェント側のスキル読み込み機構から利用することを想定しています。

## Installation

公開リポジトリからインストールする場合は、`npx skills add` を使います：

```bash
npx skills add syou6162/agent-skills
```

ローカルで確認・編集する場合は、このリポジトリをクローンして `skills/` 配下の `SKILL.md` を直接参照してください。エージェント固有の導入方法がある場合は、そのエージェント側の手順に従ってください。

## Repository Structure

このリポジトリはエージェントスキル集として構成されており、以下のディレクトリ構造を持ちます：

```
agent-skills/
├── skills/
│   ├── codex-review/
│   │   └── SKILL.md                         # スキル定義
│   ├── codex-plan-review/
│   │   └── SKILL.md                         # スキル定義
│   ├── execute-plan/
│   │   └── SKILL.md                         # スキル定義
│   ├── gha-sha-reference/
│   │   └── SKILL.md                         # スキル定義
│   ├── breaking-down-linear-issues/
│   │   └── SKILL.md                         # スキル定義
│   ├── devin-handoff-to-cursor/
│   │   └── SKILL.md                         # スキル定義
│   ├── orchestrating-cursor-cloud-agent/
│   │   └── SKILL.md                         # スキル定義
│   ├── planning-guardrails/
│   │   └── SKILL.md                         # スキル定義
│   ├── reading-notion/
│   │   └── SKILL.md                         # スキル定義
│   ├── requesting-gcloud-bq-auth/
│   │   └── SKILL.md                         # スキル定義
│   ├── updating-pr-title-and-description/
│   │   └── SKILL.md                         # スキル定義
│   └── writing-dev-diary/
│       └── SKILL.md                         # スキル定義
└── README.md
```

- **`skills/`**: スキル定義を配置するディレクトリ
- **`skills/<skill-name>/SKILL.md`**: 各スキルの本体。必要に応じて `references/` などの補助ファイルを同居させる

## Available Skills

スキルは、対応するエージェントが状況に応じて参照・適用する拡張指示です。自動適用されるものもあれば、ユーザーが明示的に呼び出す想定のものもあります。

### codex-review
Codex CLIを使ってコードの変更を客観的にレビューし、指摘が収束するまで繰り返すスキル。planファイルと開発日誌（コンテキストにある場合）を参照し、計画に沿った実装になっているかを確認します。レビュー結果をタスクリスト化して対応し、毎回新規セッションでバイアスなくレビューします。

### codex-plan-review
Codex CLIを使ってplanファイルを客観的にレビューし、指摘が収束するまで繰り返すスキル。planファイルの実現可能性・技術的妥当性・抜け漏れを確認し、レビュー結果をタスクリスト化して対応します。毎回新規セッションでバイアスなくレビューし、コードベースを網羅的に読んで指摘漏れを防ぎます。

### execute-plan
planモードまたは同等の計画フェーズで作成した計画に基づいて実装を開始するスキル（ユーザーが明示的に呼び出す形式）。planファイルを読み取り、タスクリストを自動作成し、各タスクを順次実行してタスクごとにコミットを作成します。最後にcodex-reviewを実行して品質を確保します。

### gha-sha-reference
ユーザーがGitHub Actionsのタグ参照をSHA参照に変換するよう要求したときに自動的に発動するスキル。セキュリティのベストプラクティスに従い、`uses:`フィールドのタグ参照（例: `@v4`）を不変なSHA参照（例: `@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2`）に変換します。GitHub APIを使用してコミットSHAを自動取得し、サプライチェーン攻撃のリスクを軽減します。

### breaking-down-linear-issues
Linearに大きなタスクを複数のIssueに分割して作成するスキル。階層構造を避け、Issue間の依存関係を`blockedBy`リレーションで表現します。ユーザーが承認するまでIssueの作成を実行せず、分割内容と依存関係を確認してから作成します。

### planning-guardrails
Plan modeや計画作成・プラン作成の依頼時に自動的に発動するスキル。参考情報（URL/過去PR/ファイルパス）とユーザー発言（要約＋生ログ）を必須化し、テストがある場合はTDD前提と正常系/異常系テストケース記載を強制します。他のエージェントでは「計画フェーズ」のガードレールとして利用できます。

### reading-notion
NotionページやドキュメントをキーワードまたはURLで検索・取得し、プロパティとブロック内容を読み込んで要約・説明するスキル。Notion MCP Serverは直接使用するとコンテキストを圧迫するため、[mcptools](https://github.com/f/mcptools)を利用してNotion APIをラップしています。Notion URLが会話に登場した時に自動的にページを取得して内容を説明し、キーワード検索時には検索結果から選択したページの内容を説明します。

### requesting-gcloud-bq-auth
gcloudやbqコマンド実行時に認証エラー（`Reauthentication required`や`Your browser has been opened to visit: ...accounts.google.com...`）を検出したときに自動的に発動するスキル。エージェントが勝手に認証コマンドを実行せず、ユーザーに認証を依頼します。ブラウザ操作が必要な認証フローのため、ユーザーによる手動認証を促し、認証完了後に作業を再開します。

### devin-handoff-to-cursor
Devin CLIからcloud Devinにハンドオフされた際に自動的に発動するスキル。受け取ったタスクを自分で実装せず、orchestrating-cursor-cloud-agentスキルの手順に従ってCursor Cloud Agentにオーケストレーションとして委託します。ハンドオフコンテキスト（リポジトリ、ブランチ、未コミットの変更、会話コンテキスト）を整理してCursorへのプロンプトを構築し、監視・レビュー・フォローアップを行います。

### orchestrating-cursor-cloud-agent
Cursor Cloud Agentにタスクを委託するよう依頼された際に自動的に発動するスキル。エージェント作成、SSEストリーム監視（`stream`）、実行キャンセル（`cancel`）、ステータスポーリング（フォールバック）、Pull Request状態確認、Linear紐付け、レビュー・フォローアップを行います。コーディング作業はCursorに任せ、オーケストレーターは判断・監視・指示に特化します。

### updating-pr-title-and-description
PR作成・更新時に自動的に発動するスキル。Pull Requestのタイトルと説明文を自動生成・更新します。差分やコミットメッセージを分析し、適切な説明文を作成します。説明文は日本語で記載され、`.github/PULL_REQUEST_TEMPLATE.md`がある場合はテンプレートに沿った形で生成されます。

### writing-dev-diary
「開発日誌更新」「開発日誌作って」の言及時に自動的に発動するスキル。esa-llm-scoped-guardを使って開発日誌を新規作成・更新します。トリガーによって動作を分岐し、「作って」の場合は直接新規作成、「更新」+URLの場合は指定記事を更新、「更新」のみの場合は検索して関連記事を更新（なければ新規作成）します。

## License

MIT License - see [LICENSE](LICENSE) file for details.
