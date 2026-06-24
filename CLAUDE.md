# CLAUDE.md

This file provides guidance to coding agents working with code in this repository.

## プロジェクト概要

このリポジトリは、Claude Code、Codex、その他のコーディングエージェントで利用できるスキル集です。実行可能コードではなく、各エージェントに対する指示書（マークダウンファイル）を管理しています。

## アーキテクチャ

### スキル構成
- `skills/codex-review/SKILL.md` - Codex CLIを使った客観的コードレビュー（planファイルと開発日誌を参照、レビュー結果をタスクリスト化、指摘が収束するまで繰り返す、自動発動）
- `skills/codex-plan-review/SKILL.md` - Codex CLIを使ったplanファイルレビュー（実現可能性・妥当性を評価し、レビュー結果をタスクリスト化、指摘が収束するまで繰り返す、自動発動）
- `skills/execute-plan/SKILL.md` - planモードの計画に基づいて実装を開始するスキル（ユーザー明示呼び出し）。planファイル読み取り→タスクリスト作成→タスク順次実行→コミット→codex-reviewを自動化
- `skills/gha-sha-reference/SKILL.md` - GitHub ActionsのタグをSHA参照に変換するスキル（自動発動）
- `skills/breaking-down-linear-issues/SKILL.md` - Linearに大きなタスクを複数のIssueに分割して作成し、Issue間の依存関係を`blockedBy`で表現するスキル（自動発動）
- `skills/devin-handoff-to-cursor/SKILL.md` - Devin CLIから /handoff でDevin Cloudにタスクを引き継ぐ際、Devin Cloudに対してCursor Cloud Agentへの委譲を指示するスキル（自動発動）
- `skills/orchestrating-cursor-cloud-agent/SKILL.md` - Cursor Cloud Agentにタスクを委託し、SSEストリーム監視（`stream`）・キャンセル（`cancel`）・ポーリング（フォールバック）・レビュー・フォローアップを行うスキル（自動発動）
- `skills/planning-guardrails/SKILL.md` - Plan modeの計画作成で必須セクションを漏らさないためのガードレールを提供するスキル（自動発動）
- `skills/reading-notion/SKILL.md` - NotionページやドキュメントをキーワードまたはURLで検索・取得し、内容を要約・説明するスキル（自動発動）
- `skills/requesting-gcloud-bq-auth/SKILL.md` - gcloud/bqコマンドの認証エラーを検出してユーザーに認証を依頼するスキル（自動発動）
- `skills/updating-pr-title-and-description/SKILL.md` - PRタイトル・説明文の自動生成・更新スキル（自動発動）
- `skills/writing-dev-diary/SKILL.md` - esa-llm-scoped-guardで開発日誌を新規作成・更新するスキル（自動発動）

## 開発コマンド

### スキルの追加
新しいスキルを追加する際：

1. `skills/skill-name/` ディレクトリを作成
2. `skills/skill-name/SKILL.md` ファイルを作成
3. YAMLフロントマターを追加（name, description）
   - `name`: 小文字・ハイフン区切り（最大64文字）
   - `description`: いつ使うか + 何をするか（最大1024文字）
4. `README.md` の Available Skills セクションを更新
5. `CLAUDE.md` のスキル構成セクションを更新
6. コミット＆プッシュ

注：スキルは対応するエージェントが自動適用する場合もあれば、ユーザーが明示的に呼び出す場合もある

### テスト・検証
```bash
# GitHubリポジトリからスキルを追加
npx skills add syou6162/agent-skills
```

## 重要な設計原則

### 公式ドキュメント参照
スキル実装やエージェント固有記法の修正が必要になった場合は、**必ず該当する公式ドキュメントを参照すること**：

- [スキル](https://docs.claude.com/en/docs/claude-code/skills)
### XMLタグ構造化の実験（パイロット）

Claudeのシステムプロンプトおよび公式ドキュメントでは、XMLタグによるプロンプト構造化が推奨されています。このリポジトリでは、複数のスキルでXMLタグ化を利用しています。

**標準XMLタグ語彙**:
```markdown
<important>          - 禁止事項や絶対に守るべきルール
<trigger>            - スキルの起動条件（いつ使うか）
<procedure>          - 重要な実行手順
<context>            - コンテキスト情報の定義（後続の手順で参照される）
<example>            - 実行例
<examples>           - 複数の例を含むコンテナ（個別の<example>をネスト）
<decision-criteria>  - 判断基準（表形式など）
```

**タグ定義と参照の使い分け**:
- **定義**: `<tag>...</tag>` (開始タグと終了タグで内容を囲む)
- **参照**: テキスト中で「<tag>タグで定義した...」のように言及（終了タグなし、バッククォートなし）
- 例: `<context>`で定義 → 「以下は<context>タグで定義した手順に従って...」で参照

**標準語彙への追加ルール**:
標準語彙に新しいタグを追加する場合は、以下の条件をすべて満たすこと:
1. プロジェクト全体（複数のファイル）で共通的に使われる
2. 明確な意味を持つタグ名（抽象的すぎない、具体的すぎない）
3. 既存の標準タグでは表現できない構造である
4. レビューで妥当性を確認後、CLAUDE.mdに追加する

**各ファイル内でのタグ使用ルール**:
各ファイル内で独自のタグを使用する場合（標準語彙に含まれないタグ）:
1. そのファイル内で明確な意味を持ち、分かりやすくなる
2. 複数箇所で参照される情報の定義である
3. 標準語彙に追加する必要はない（そのファイル専用で問題ない）
- 単発の構造化には標準タグまたはMarkdown見出しを使用すること

**XMLタグ使用の原則**:
- ネストは1-2階層まで（過度な入れ子を避ける）
- 意味のあるタグ名を使用（目的が明確）
- Markdown見出しとXMLタグを混在させる場合、見出しはタグの外に配置
- **GitHubレンダリング対応**: XMLタグとMarkdownコンテンツ（コードブロック、箇条書き、表など）の間には空行を入れる
  - 開始タグの直後と終了タグの直前に空行が必要
  - 例: `<important>` + 空行 + 箇条書き + 空行 + `</important>`
  - これによりGitHubのMarkdownパーサーが正しくレンダリングする

**ロールバック方法**:
効果が不明確な場合や問題が発生した場合は `git revert` で元に戻すことができます。

**今後の展開**:
- パイロット結果が良好であれば、他のスキルにも適用を検討
- タグ語彙は既にCLAUDE.mdに標準化済み

### 外部依存関係
スキルごとに以下のツールを使用：
- **GitHub CLI (`gh`)** - Pull Request操作・GitHub URL状態確認（updating-pr-title-and-description, writing-dev-diary, orchestrating-cursor-cloud-agent）
- **Cursor Cloud Agent API (`https://api.cursor.com/v1`)** - エージェント作成・追加プロンプト・ステータスポーリング（orchestrating-cursor-cloud-agent スキル）
- **cursor-agent-cli (`https://github.com/syou6162/cursor-agent-cli`)** - Cursor Cloud Agent API 呼び出し（orchestrating-cursor-cloud-agent スキル）
- **Linear MCP server (`https://mcp.linear.app/mcp`)** - Linear issue の紐付け・ステータス更新（orchestrating-cursor-cloud-agent スキル）、Linear への issue 作成・依存関係設定（breaking-down-linear-issues スキル）
- **git-sequential-stage** - semantic-committing（スキル）で使用する専用ツール
- **copilot CLI / Codex CLI (`copilot --model gpt-5.3-codex`, `codex review`)** - コードレビュー（codex-review, codex-plan-review スキル）
- **esa-llm-scoped-guard** - 開発日誌の作成・更新・取得（writing-dev-diary, codex-review, codex-plan-review スキル）
- **notion-to-md** - Notion URLをMarkdownに変換（reading-notion スキルのURL直接モード）
- **mcptools + @notionhq/notion-mcp-server (`mcptools call API-post-search npx -y @notionhq/notion-mcp-server`)** - Notionキーワード検索（reading-notion スキルの検索モード）
- **jq** - Notion検索結果のJSON整形（reading-notion スキルの検索モード）
- **Node.js / npx** - @notionhq/notion-mcp-serverの実行（reading-notion スキルの検索モード）

### ファイル更新時の注意
- READMEと各スキルの説明の整合性を保つ
- 各スキルの説明は実際の機能と正確に一致させる
