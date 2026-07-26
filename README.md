# quant-lab-data

`quant-lab-data` は、[`yu1Ro5/quant-lab`](https://github.com/yu1Ro5/quant-lab) が取得する  
USD/JPY の公開データ、公開アラート状態、定期実行ワークフローを所有するリポジトリです。  
アプリケーションコードはこのリポジトリへ複製せず、サブモジュールも使用しません。  
GitHub Actions が検証済みの `quant-lab` commit を完全な40文字のSHAでcheckoutして実行します。

> [!IMPORTANT]
> 現在固定している `quant-lab` SHA
> `133dafc5f329b027b18266ca9eba1376ffd20529` には、必要な `prepare` / `deliver`
> CLIと外部データパス対応が実装済みです。workflowはこのSHAへ固定し、インターフェース確認後に
> `QUANT_LAB_APP_INTERFACE_READY` を有効化しています。scheduleは手動移行確認が完了するまで
> コメントアウトしたままです。

## データ

### `data/usd_jpy.csv`

日次履歴です。`quant-lab` から既存の全履歴を移行しており、同じ `rate_date` は重複保存しません。

| 列 | 意味 |
| --- | --- |
| `fetched_at` | アプリケーションが取得を完了したUTC日時 |
| `rate_date` | API基準時刻をJSTへ変換した日付 |
| `rate` | bidとaskの仲値 |
| `bid` | 売値 |
| `ask` | 買値 |
| `spread` | `ask - bid` |
| `source_timestamp` | APIが返した相場のUTC日時 |
| `market_status` | `OPEN` または `CLOSE` |

旧3列形式の行は追加列が空のまま残り、元の値の意味を維持します。

### `data/usd_jpy_hourly.csv`

1時間単位の履歴です。

```csv
fetched_at,source_timestamp,bucket_start_utc,rate,bid,ask,spread,market_status
```

`source_timestamp` をUTCの時間単位へ切り下げた bucket ごとに、最初の成功値を1行だけ保存します。  
作業ツリーには現在の有効な `source_timestamp` から直近90日（境界を含む）を保持します。  
保持期間から外れて削除された行も通常のGit履歴には残り、履歴の書き換えは行いません。

### `data/alert_state.json`

schema version 1 の公開状態です。1時間前比と日次比のクールダウンを独立管理し、  
未送信の強いアラートを `pending_alert` に最新1件だけ保持します。Slack token、  
channel ID、ユーザー情報などの秘密情報は保存しません。

強いアラートは見逃しを避けるため at-least-once で扱います。Slack送信成功後の状態commitに  
失敗した場合、次回に重複送信される可能性があります。通常通知は再送しません。

## GitHub Actions

`.github/workflows/monitor.yml` は現在 `workflow_dispatch` のみ有効です。予定するscheduleは  
平日毎時17分（`17 * * * 1-5`、UTC基準）ですが、段階移行が完了するまでコメントアウトしています。  
同じ concurrency group と `cancel-in-progress: false` により、手動実行と将来のscheduleを直列化します。

処理順は次のとおりです。

1. `quant-lab-data` と固定SHAの `quant-lab` を別ディレクトリへcheckoutする。
2. `quant-lab` で `uv sync --locked` を実行する。
3. `prepare` がAPI取得、比較、3データファイルの更新、delivery envelope生成を行う。Slackは呼ばない。
   標準出力JSONは `RUNNER_TEMP` に保存し、`delivery_kind` を検証する。
4. 3データファイルだけを明示的にstageし、差分がある場合だけcommit/pushする。
5. データのpush成功後、`deliver` がenvelopeを読みSlackを最大1回呼ぶ。標準出力JSONの
   `state_commit_required` がbooleanであることを検証する。
6. 強いアラート送信成功で状態が変わった場合だけ、`alert_state.json` を再度commit/pushする。

最初のcommit/pushが失敗した場合はSlackへ進みません。delivery envelope とCLI結果JSONは  
`RUNNER_TEMP` に置かれ、commit対象になりません。

## 必要なアプリ側インターフェース

固定SHAを更新する前に `quant-lab` で、少なくとも次を実装して全 `unittest` を通してください。

- `QUANT_LAB_DATA_DIR` から日次CSV、時間別CSV、状態JSONのパスを一箇所で解決する。
- 1時間bucket、90日保持、45〜90分の比較候補、独立閾値、CLOSE時の抑制を実装する。
- schema version 1 の状態、3時間クールダウン、最新1件の強いアラート再送を実装する。
- `python main.py prepare --envelope PATH` を提供する。
  - Slack APIを呼ばない。
  - envelopeを `PATH` に作成する。
  - 標準出力JSONの `delivery_kind` は `normal` または `strong_alert` とする。
- `python main.py deliver --envelope PATH` を提供する。
  - envelopeだけを入力としてSlackを最大1回呼ぶ。
  - 強いアラート送信成功時だけ状態を更新する。
  - 標準出力JSONにbooleanの `state_commit_required` を含める。
  - `state_commit_required=true` の場合だけ送信後の状態commitを行う。
- 壊れた時間別CSVや状態JSONを自動初期化・上書きせず、強いアラートを安全側で抑制する。

workflow のCLI引数と出力契約を変更する場合は、`quant-lab` とこのREADME、  
`monitor.yml` を同じ移行作業内で整合させてください。

## Secrets と Variables

リポジトリの **Settings > Secrets and variables > Actions** で、人手により設定します。

必須Secrets:

| 名前 | 用途 |
| --- | --- |
| `SLACK_BOT_TOKEN` | Slack Bot token |
| `SLACK_CHANNEL` | 送信先channel IDまたは名前 |

任意Variables:

| 名前 | 未設定時 |
| --- | ---: |
| `USD_JPY_HOURLY_ALERT_THRESHOLD_PERCENT` | `0.3` |
| `USD_JPY_DAILY_ALERT_THRESHOLD_PERCENT` | `1.0` |

Secretsの値をログ、CSV、JSON、README、commit messageへ出力しないでください。

## `quant-lab` SHAの更新

1. `quant-lab` の対象commitで、外部APIとSlackをmockした `unittest`、`compileall`、
   既存CI相当の静的検証を実行する。
2. GitHub上に存在する完全な40文字SHAであることを確認する。
3. `monitor.yml` の `ref` をそのSHAへ変更する。
4. 必須インターフェースが揃った最初の更新では、同じPRで
   `QUANT_LAB_APP_INTERFACE_READY` を `"true"` に変更する。
5. 差分でSHAとアプリ側テスト結果をレビューする。SHA更新は自動化しない。

## 手動実行とscheduleの有効化

まず **Actions > USD/JPY Monitor > Run workflow** から手動実行します。  
固定SHAのインターフェース確認、prepare、データ永続化、deliverの順に成功することを確認します。

対応済みSHAへ更新した後は、テスト用Slack channelを使い、次を順に確認します。

1. 初回実行で時間別CSVが1行増え、日次CSVが同日重複しない。
2. 通常通知に1時間前比・日次比と各基準時刻が表示される。
3. 差分なし実行で空commitが作られず、Slack APIが最大1回だけ呼ばれる。
4. Slack失敗時もレートと強いアラートの `pending_alert` が先にpush済みである。
5. 次回にpendingが通常通知より優先され、成功後だけ状態commitが作られる。
6. `CLOSE`、3時間クールダウン、閾値復帰後の再超過をmockまたは安全な閾値で確認する。
7. 新workflowが安定してから旧 `quant-lab` workflowを無効化する。
8. 二重実行がないことを確認してから `monitor.yml` のscheduleコメントを外す。

旧workflowを先に止めず、新旧scheduleを同時に常用しないでください。

## 障害時の復旧

- **API/prepare失敗**: データcommitもSlack送信も行われません。原因を直して手動再実行します。
- **最初のcommit/push失敗**: Slackは送られません。リモート差分や権限を確認して再実行します。
- **通常通知のSlack失敗**: 通常通知は再送しません。保存済みデータを確認して次回実行を待ちます。
- **強いアラートのSlack失敗**: `pending_alert` とレートがリモートへ保存済みか確認し、次回実行で再送します。
- **送信後の状態commit失敗**: 次回の重複送信を許容します。見逃し防止を優先し、pendingを手で消しません。
- **壊れたCSV/JSON**: workflowを再実行せず、Git履歴から最後の正常版を特定して退避・レビュー後に復元します。
  アプリは壊れたファイルを自動上書きしてはいけません。

## ロールバック

1. `quant-lab-data` のscheduleをコメントアウトして無効化する。
2. 最新の3データファイルを別の安全な場所へ退避する。
3. 必要なデータを `quant-lab/data` へ戻す。
4. `QUANT_LAB_DATA_DIR` を未設定にしてローカル既定パスでテストする。
5. `quant-lab` の旧workflowを復元し、手動実行でデータ保存とSlackを確認する。
6. 確認後に旧workflowのscheduleだけを有効化する。

サブモジュールを使っていないため、Git構造を解除する作業はありません。
