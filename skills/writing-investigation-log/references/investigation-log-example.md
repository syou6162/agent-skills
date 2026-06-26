# 公開ソースだけを使った調査ログ記入例

> このファイルは記入粒度の例です。公開GitHubリポジトリとBigQuery public datasetだけを使います。esa、Notion、社内URLは例に含めません。

## 問い

- GitHubの公開PRで、依存関係更新の根拠をどの粒度で残すかを確認する。
- BigQuery public datasetを使う調査で、コマンド、クエリ、結果、限界をどう分けて残すかを確認する。

## 現時点の結論

- `cli/cli` の公開PRでは、`actions/setup-go` を `v6.4.0` から `v6.5.0` に更新していることを、PR本文と差分から確認できる。
- BigQuery public datasetを使う場合は、クエリ本文だけでなく、実行コマンド、BigQuery CLIの出力、そこから言えること、言えないことを分けて書く。
- dry run のログでは処理予定バイト数を確認し、実クエリのログでは `hamlet` における単語出現数の上位5件を確認できる。

## 根拠

| 種別 | URL / パス | 確認したこと |
|---|---|---|
| GitHub PR | https://github.com/cli/cli/pull/13740 | PR本文に `actions/setup-go` を `6.4.0` から `6.5.0` に更新する旨が書かれている。差分対象はGitHub Actions workflowの複数ファイルである。 |
| GitHub file | https://github.com/cli/cli/blob/da11fa052947a509d5aff22a7f272b8966ada443/.github/workflows/go.yml#L23-L24 | `Set up Go` のstepで `actions/setup-go@924ae3a1cded613372ab5595356fb5720e22ba16 # v6.5.0` が使われている。 |
| official docs | https://cloud.google.com/bigquery/public-data | BigQuery public datasetsを調査例に使う場合の参照先。公開データセットを使う例として扱う。 |

## 実行したコマンド

```bash
gh pr view 13740 --repo cli/cli --json number,title,url,files
gh pr diff 13740 --repo cli/cli --patch
bq query --dry_run --use_legacy_sql=false 'SELECT corpus, word, word_count FROM `bigquery-public-data.samples.shakespeare` WHERE corpus = "hamlet" ORDER BY word_count DESC LIMIT 5'
bq query --format=prettyjson --use_legacy_sql=false 'SELECT corpus, word, word_count FROM `bigquery-public-data.samples.shakespeare` WHERE corpus = "hamlet" ORDER BY word_count DESC LIMIT 5'
```

## コマンド結果

```text
gh pr view:
- PR #13740 のタイトルは chore(deps): bump actions/setup-go from 6.4.0 to 6.5.0。
- 変更対象は .github/workflows/bump-go.yml, codeql.yml, deployment.yml, go.yml, govulncheck.yml, lint.yml。

gh pr diff:
- 各workflowで actions/setup-go の参照SHAとコメント上のバージョンが v6.4.0 から v6.5.0 に更新されている。

bq query --dry_run:
- Query successfully validated.
- 実行すると 5,114,816 bytes を処理する見込みである。

bq query:
- `hamlet` における `word_count` 上位5件をprettyjsonで取得した。
```

## 実行したクエリ

```sql
SELECT corpus, word, word_count
FROM `bigquery-public-data.samples.shakespeare`
WHERE corpus = "hamlet"
ORDER BY word_count DESC
LIMIT 5;
```

## クエリ結果

```text
[
  {
    "corpus": "hamlet",
    "word": "the",
    "word_count": "995"
  },
  {
    "corpus": "hamlet",
    "word": "and",
    "word_count": "706"
  },
  {
    "corpus": "hamlet",
    "word": "to",
    "word_count": "635"
  },
  {
    "corpus": "hamlet",
    "word": "of",
    "word_count": "630"
  },
  {
    "corpus": "hamlet",
    "word": "I",
    "word_count": "546"
  }
]
```

## 結果から言えること

- GitHub PR #13740 は、公開PR本文と差分から `actions/setup-go` のバージョン更新PRだと確認できる。
- GitHub file permalinkを見ると、少なくとも `.github/workflows/go.yml` では `actions/setup-go` が `v6.5.0` のSHAに更新されている。
- BigQueryのdry runにより、このクエリは実行前に 5,114,816 bytes を処理する見込みだと確認できる。
- BigQueryの実クエリにより、`hamlet` では `the`, `and`, `to`, `of`, `I` が `word_count` 上位5件であることを確認できる。

## 結果からは言えないこと

- PR #13740 がmerge可能か、CIが通っているか、レビュー済みかは、この調査だけでは言えない。
- このクエリだけでは、`hamlet` 以外のcorpusにおける上位単語は言えない。
- `word_count` がどの前処理や正規化に基づく値かは、このクエリ結果だけでは言えない。

## 後続エージェント向けメモ

- 公開リポジトリの例では、PR URLだけでなく、該当commitのfile permalinkも残す。
- BigQueryの例では、クエリ本文と実行コマンドを分けて残す。
- dry run の処理予定バイト数と実データ取得結果を分けて残す。
- 調査結果の説明は日本語で書く。コマンド、クエリ、URLは原文のままでよい。
