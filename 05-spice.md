# SPICE

## 1. 結論

- ビジネス部門向けの参照は SPICE を基本にする。DB に負荷がかかるのは取り込みのときだけで、DB が止まっていても閲覧できる
- データの新しさは「最後に取り込みが成功した時刻」で決まる。画面に取り込み時刻を出す
- 取り込みは 1 日 1 回の全件更新とし、DB の負荷が低い時間帯（例: 毎日 3:00）に実行する。画面のデータは最大で約 1 日前のものになる
- 容量の自動購入はオフにし、CloudWatch で使用量を監視する

## 2. SPICE と直接クエリの比較

| 観点         | SPICE                                          | 直接クエリ                                 |
| ------------ | ---------------------------------------------- | ------------------------------------------ |
| 表示速度     | 速い（インメモリ）                             | DB 次第。ビジュアルは 120 秒でタイムアウト |
| データの鮮度 | 最後に取り込んだ時点                           | 表示した時点                               |
| DB への負荷  | 取り込み時だけ                                 | 画面を操作するたびに発生                   |
| 費用         | 容量課金（閲覧回数では増えない）               | DB 側の増強費用                            |
| 規模の上限   | Enterprise は 20 億行または 2TB                | Quick Sight 側の上限なし                   |
| DB 停止中    | 閲覧できる                                     | 閲覧できない                               |
| DB の監査ログ | 取り込みの SQL だけが残る（閲覧は残らない）   | 表示ごとの SQL が残る                      |

片方でしか使えない機能:

- SPICE だけ: 異なるデータソース間の結合、Append / Aggregate / Pivot / Unpivot、取り込みスケジュール、増分更新
- 直接クエリだけ: データセットパラメータをカスタム SQL に埋め込む機能

## 3. 容量と料金

### 3.1 容量の考え方

- 容量は「アカウント × リージョン」単位で、同じリージョンの全ユーザーが共有する
- 同梱の容量は、Author（と Author Pro）1 人につき 10GB。Reader には付かない
- 同梱の容量はサインアップ時に選んだリージョンにだけ付く。他のリージョンは購入しない限り 0
- 追加は $0.38/GB/月。月の途中でも増減できる

### 3.2 購入・解放・自動購入（管理者だけが操作できる）

| 操作       | 画面                                                                         |
| ---------- | ---------------------------------------------------------------------------- |
| 購入       | Manage Quick → SPICE capacity → Purchase more capacity → GB 数を入力         |
| 解放       | SPICE capacity → Release unused purchased capacity（未使用分だけ解放できる） |
| 自動購入   | SPICE capacity → Auto-purchase capacity のトグル                             |

- 自動購入は、コンソールで作ったアカウントでは既定でオン。オフにすると、容量を超えたときに取り込みが失敗する
- 自動購入は増やすだけで、自動では減らない
- CLI: `aws quicksight update-spice-capacity-configuration --aws-account-id 111122223333 --purchase-mode MANUAL`

### 3.3 容量の見積もり（公式の計算式）

```text
1 データセットのサイズ(byte) =
  (数値のセル数 x 12) + (日付のセル数 x 12) + 文字列セルごとの (24 + UTF-8 のバイト数) の合計
```

- 例: 500 万行、数値・日付 8 列、文字列 6 列（平均 20 バイト）
  - 500 万 x (8 x 12 + 6 x (24 + 20)) = 500 万 x 360 = 約 1.8GB
- 加工後（除外した列を除き、計算列を含めた）の形で計算する
- 全サービスの取り込み量を合計して、必要な容量を確保する。料金の概算（「概要と推奨構成」ページ）では 100GB を想定した
- 取り込み後の実際の使用量は次で確認する
  - データセット単位: `describe-data-set` の `ConsumedSpiceCapacityInBytes`
  - アカウント単位: CloudWatch の `SPICECapacityConsumedInMB`

## 4. データセットの上限

| 項目                 | 値                                                       |
| -------------------- | -------------------------------------------------------- |
| 行数・サイズ         | Enterprise: 20 億行または 2TB / Standard: 2,500 万行または 25GB |
| 列                   | 2,000 列。列名は 127 文字                                |
| 文字列               | 2,047 文字（新しいデータ準備画面では 65,534 文字）       |
| 数値                 | 19 桁を超えると、その行は取り込みで落ちる                |

## 5. 更新（リフレッシュ）

### 5.1 更新の種類

| 種類       | 動き                                                             | 条件                      |
| ---------- | ---------------------------------------------------------------- | ------------------------- |
| 全件更新   | 全データを取り込み直す。完了するまでは古いデータを表示し、完了すると新しいデータに置き換わる（古いデータは残らない） | なし |
| 増分更新   | 日付列で指定した期間（Window size）の行だけを取り直して入れ替える | Enterprise、SQL 系のソース |

- 増分更新の設定: データセット → Refresh → Refresh now → Refresh type: Incremental refresh → Configure
  - Date column（更新日時の列）と Window size（時間・日・週）を指定する
- 期間より古い行の変更や削除は反映されない。週次などの全件更新と組み合わせる
- カスタム SQL では DB 側で期間の条件を最適化できず、全件更新より遅くなることがある

### 5.2 スケジュール

| エディション・種類       | 選べる頻度                                              |
| ------------------------ | ------------------------------------------------------- |
| Standard（全件）         | 日次、週次、月次                                        |
| Enterprise（全件）       | 毎時、日次、週次、月次                                  |
| Enterprise（増分）       | 15 分、30 分、毎時、日次、週次、月次                    |

- 1 データセットにスケジュールは 5 つまで
- 15 分、30 分、毎時のスケジュールは 1 データセットに 1 つだけで、他のスケジュールと併用できない
- 実行は指定時刻から 10 分以内に始まる

### 5.3 回数の上限

- API（`CreateIngestion`）による取り込みは、24 時間で Enterprise 32 回、Standard 8 回（調整不可）
- 上限がデータセット単位かアカウント単位かは公式に明記がない（コミュニティではデータセット単位とされる。要検証）
- 同時に走れる取り込みの数は公式の値がない。DB の負荷を考え、データセットごとに時刻をずらす

### 5.4 手動更新と CLI

- 画面: Data → Datasets → 対象のデータセット → Refresh → Refresh now

```bash
aws quicksight create-ingestion --region ap-northeast-1 --aws-account-id 111122223333 \
  --data-set-id <DATASET_ID> --ingestion-id manual-20261020-0900 --ingestion-type FULL_REFRESH
aws quicksight describe-ingestion --region ap-northeast-1 --aws-account-id 111122223333 \
  --data-set-id <DATASET_ID> --ingestion-id manual-20261020-0900
```

- `ingestion-id` は毎回別の値にする（1〜128 文字の英数字、`-`、`_`）

スケジュールの作成（毎日 3 時に全件更新）:

```bash
aws quicksight create-refresh-schedule --region ap-northeast-1 --aws-account-id 111122223333 \
  --data-set-id <DATASET_ID> --schedule '{"ScheduleId":"daily-full","RefreshType":"FULL_REFRESH",
  "ScheduleFrequency":{"Interval":"DAILY","TimeOfTheDay":"03:00","Timezone":"Asia/Tokyo"}}'
```

- 毎時にするには `Interval` を `HOURLY`、増分の 15 分間隔は `MINUTE15` にする

### 5.5 失敗したときの通知と調査

- メール通知: データセット作成の最後の画面で Email owners when a refresh fails を選ぶ（通知先は所有者だけ）
- 履歴: データセットの Refresh タブの下部。開始時刻（UTC）、状態、所要時間、取り込んだ行数、落ちた行数が見られる
- 落ちた行の確認: 履歴の View error summary → Download error rows file

| 主なエラー                         | 原因                             | 対処                                   |
| ---------------------------------- | -------------------------------- | -------------------------------------- |
| `ACCOUNT_CAPACITY_LIMIT_EXCEEDED`  | 容量不足                         | 容量を購入するか、不要なデータを削る   |
| `SQL_SCHEMA_MISMATCH_ERROR`        | DB 側の列が変わった              | データセットを開いて保存し直す         |
| `CONNECTION_FAILURE` など          | VPC 接続、SG の誤り              | 「接続方式」ページの 8 章              |
| `PERMISSION_DENIED`                | DB ユーザーの権限、パスワード    | DB の権限とシークレットを確認          |
| `SOURCE_RESOURCE_LIMIT_EXCEEDED`   | DB の接続数などの上限            | 取り込み時刻をずらす                   |
| `INGESTION_SUPERSEDED`             | 次の取り込みが始まった           | 取り込み中にデータセットを編集しない   |

## 6. 暗号化

- 既定: SPICE のデータは AWS 所有のキーで暗号化されて保存される
- カスタマー管理キー（KMS）も使える（Enterprise。社内規程で求められる場合だけ）
  - 設定: Manage Quick → KMS keys → Manage → Select key → 既定キーにするチェックを入れる
  - キーはアカウントとリージョンが Quick Sight と同じ対称キー
  - 既存のデータセットを新しいキーで暗号化し直すには、全件更新を実行する
  - キーを無効にすると最大 15 分で読めなくなる。キーを完全に削除すると、そのデータは戻らない

## 7. 監視（CloudWatch、名前空間 `AWS/QuickSight`）

| メトリクス                     | 内容                     | 単位                 |
| ------------------------------ | ------------------------ | -------------------- |
| `IngestionErrorCount`          | 失敗した取り込みの数     | データセット / 全体  |
| `IngestionInvocationCount`     | 開始した取り込みの数     | データセット / 全体  |
| `IngestionLatency`             | 取り込みの所要時間（秒） | データセット / 全体  |
| `IngestionRowCount`            | 取り込んだ行数           | データセット / 全体  |
| `SPICECapacityConsumedInMB`    | 容量の使用量             | 全体                 |
| `SPICECapacityLimitInMB`       | 容量の上限               | 全体                 |

取り込み失敗のアラーム例:

```bash
aws cloudwatch put-metric-alarm --region ap-northeast-1 --alarm-name qs-spice-ingestion-failed \
  --namespace AWS/QuickSight --metric-name IngestionErrorCount \
  --statistic Sum --period 3600 --evaluation-periods 1 --threshold 1 \
  --comparison-operator GreaterThanOrEqualToThreshold --treat-missing-data notBreaching \
  --alarm-actions arn:aws:sns:ap-northeast-1:111122223333:ops-alert
```

- 容量のアラームは、`SPICECapacityConsumedInMB` に上限の 80% 前後の値を閾値にして作る

## 8. 運用設計の目安

- 想定: 取り込みは 1 日 1 回の全件更新（例: 毎日 3:00。5.4 のスケジュール作成例のとおり）
  - 画面のデータは最大で約 1 日前のもの。スケジュールの開始は指定時刻から最大 10 分遅れる
  - 1 日 1 回なら、`CreateIngestion` の回数上限（24 時間で 32 回）や、毎時・増分更新（Enterprise 限定）は関係しない
- 取り込みに失敗すると、前日のデータが表示され続ける。失敗の通知（5.5 のメール、7 章のアラーム）を必ず設定する
- 当日に更新されたデータは画面に出ない。緊急時に当日の値が要るときは、開発部門が直接クエリで確かめる運用にする
- 取り込みは DB に全件の SELECT を発行する。リードレプリカから取り込む
- 複数のデータセットの取り込み時刻を重ねない

1 日 1 回で足りなくなった場合の選択肢:

| 求められる新しさ                 | 更新の方法                                         |
| -------------------------------- | -------------------------------------------------- |
| 日中の変化も見たい               | 毎時の全件更新（Enterprise）                       |
| 大きく、更新日時の列が信頼できる | 15〜30 分の増分更新 + 週次の全件更新（Enterprise） |

- 増分更新に切り替える場合は、日付列に DB 側でインデックスを張る

## 9. 出典

確認日は 2026-10-20 または 2026-10-21。「第三者情報」「コミュニティ情報」と書いたもの以外は AWS 公式。

- 2 章 SPICE と直接クエリの比較
  - SPICE の概要: https://docs.aws.amazon.com/quicksight/latest/user/spice.html
  - 使い分けのベストプラクティス（AWS ブログ）: https://aws.amazon.com/blogs/business-intelligence/best-practices-for-amazon-quicksight-spice-and-direct-query-mode/
  - SPICE 専用の機能: https://docs.aws.amazon.com/quick/latest/userguide/spice-only-features.html
  - 結合と SPICE: https://docs.aws.amazon.com/quick/latest/userguide/joining-data.html
- 3 章 容量と料金
  - 容量の管理（購入、解放、自動購入）: https://docs.aws.amazon.com/quicksight/latest/user/managing-spice-capacity.html
  - 料金: https://aws.amazon.com/quicksight/pricing/
  - 見積もりの計算式: https://docs.aws.amazon.com/quicksight/latest/user/spice.html
  - 自動購入の設定（API）: https://docs.aws.amazon.com/quicksight/latest/APIReference/API_UpdateSPICECapacityConfiguration.html
  - 使用量の確認（DescribeDataSet）: https://docs.aws.amazon.com/quicksight/latest/APIReference/API_DescribeDataSet.html
- 4 章 データセットの上限
  - 上限値: https://docs.aws.amazon.com/quicksight/latest/user/data-source-limits.html
  - 上限の拡大（2026-01）: https://aws.amazon.com/about-aws/whats-new/2026/01/amazon-quick-suite-launches-expanded-spice
  - 19 桁を超える数値: https://docs.aws.amazon.com/quicksight/latest/user/errors-spice-ingestion.html
- 5 章 更新
  - 更新とスケジュール: https://docs.aws.amazon.com/quicksight/latest/user/refreshing-imported-data.html
  - 更新の概要、失敗時のメール通知: https://docs.aws.amazon.com/quicksight/latest/user/refreshing-data.html
  - 全件更新中も古いデータを表示（AWS ブログ）: https://aws.amazon.com/blogs/business-intelligence/automate-spice-dataset-refreshes-using-amazon-quick-sight-apis/
  - 回数の上限: https://docs.aws.amazon.com/general/latest/gr/quicksight.html
  - 32 回の上限の単位（コミュニティ情報）: https://community.amazonquick.com/t/is-the-32-times-per-24-hours-createingestion-limit-applied-per-dataset-or-per-aws-account/51371
  - CreateIngestion: https://docs.aws.amazon.com/quicksight/latest/APIReference/API_CreateIngestion.html
  - スケジュールの頻度の値: https://docs.aws.amazon.com/quicksight/latest/APIReference/API_RefreshFrequency.html
  - create-refresh-schedule の CLI: https://docs.aws.amazon.com/cli/latest/reference/quicksight/create-refresh-schedule.html
  - 取り込みの履歴: https://docs.aws.amazon.com/quicksight/latest/user/view-history-of-spice-ingestion.html
  - 落ちた行の確認: https://docs.aws.amazon.com/quicksight/latest/user/troubleshooting-skipped-rows.html
  - 取り込みのエラー: https://docs.aws.amazon.com/quicksight/latest/user/errors-spice-ingestion.html
- 6 章 暗号化
  - 保存時の暗号化: https://docs.aws.amazon.com/quicksight/latest/user/data-encryption.html
  - カスタマー管理キー: https://docs.aws.amazon.com/quick/latest/userguide/customer-managed-keys.html
- 7 章 監視
  - CloudWatch メトリクス: https://docs.aws.amazon.com/quicksight/latest/user/cw-metrics.html
- 8 章 運用設計の目安
  - 出典なし（一般的な運用の指針）: 更新頻度の目安、リードレプリカからの取り込み、日付列のインデックス
