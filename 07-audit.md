# 利用証跡

## 1. 結論

- 証跡の中心は CloudTrail。マルチリージョンの証跡（Trail）を作り、管理イベントを読み取り・書き込みの両方とも S3 に保存する
- Quick Sight の閲覧は `GetDashboard`、DB への問い合わせは `QueryDatabase` として記録される
- 画面で入力した検索値（アカウント識別子）は、Quick Sight のどのログにも残らない。次の方法で補う
  - 直接クエリ: DB の監査ログに、実際に発行された SQL（WHERE の値を含む）が残る
  - SPICE: DB に問い合わせないので、DB にも残らない。問い合わせ票などの運用で記録する
- 個人を特定できるよう、利用者ごとに個人の IAM ユーザーを使う
- CloudTrail Lake は 2026-05-31 から新規利用を受け付けていない。検索は S3 + Athena で組む

## 2. どの情報がどこに残るか

| 知りたいこと                 | CloudTrail                              | DB 監査ログ                        |
| ---------------------------- | --------------------------------------- | ---------------------------------- |
| 誰が                         | 残る（IAM ユーザー名）                  | 共有 DB ユーザー名だけ（個人は不明）|
| いつ                         | 残る                                    | 残る                               |
| どのダッシュボードを開いたか | 残る（`GetDashboard`）                  | 残らない                           |
| どのデータセットか           | 一部残る（関連するデータセット ID）     | 残らない（表名は SQL に出る）      |
| 検索したアカウント識別子     | 残らない                                | 直接クエリのときだけ SQL に残る    |
| 実行された SQL               | 残らない                                | 残る                               |
| ダウンロード                 | 一部記録されるという報告あり（要検証）  | 残らない                           |

## 3. CloudTrail に残る Quick Sight のイベント

### 3.1 記録の範囲

- API の呼び出しはすべて、コンソール操作は一部が記録される
- コンソール操作は `eventType` が `AwsServiceEvent` のイベントとして残る
- イベントソースは `quicksight.amazonaws.com`

### 3.2 主なイベント名（公式ページに載っているもの）

| 分類           | イベント名                                                              |
| -------------- | ----------------------------------------------------------------------- |
| ユーザー       | `CreateAccount`、`BatchCreateUser`、`DeleteUser`、`UpdateGroups`、`Unsubscribe` |
| ダッシュボード | `GetDashboard`、`CreateDashboard`、`UpdateDashboard`、`UpdateDashboardAccess`、`DeleteDashboard` |
| 分析           | `GetAnalysis`、`CreateAnalysis`、`UpdateAnalysis`、`UpdateAnalysisAccess`、`DeleteAnalysis` |
| データソース   | `CreateDataSource`、`UpdateDataSource`、`DeleteDataSource`              |
| データセット   | `CreateDataSet`、`UpdateDataSet`、`UpdateDatasetAccess`、`DeleteDataSet`、`Querydatabase` |

- `Querydatabase` の公式の説明は「データセット更新時にデータソースへ問い合わせる」。実際のログでは `QueryDatabase` と表記される例がある
  - 検索では大文字と小文字を区別しないよう `lower(eventname)` で比べる
- 公式の一覧は一部だけを載せたもの。全体は `eventSource = quicksight.amazonaws.com` で絞って洗い出す

### 3.3 GetDashboard のログの形（値はダミー）

```json
{
  "eventSource": "quicksight.amazonaws.com",
  "eventName": "GetDashboard",
  "eventType": "AwsServiceEvent",
  "readOnly": true,
  "eventTime": "2026-10-20T01:23:45Z",
  "userIdentity": {"type": "IAMUser", "userName": "po-yamada",
                   "arn": "arn:aws:iam::111122223333:user/po-yamada"},
  "serviceEventDetails": {
    "eventRequestDetails": {"dashboardId": "arn:aws:quicksight:ap-northeast-1:111122223333:dashboard/<id>"},
    "eventResponseDetails": {"dashboardDetails": {
      "dashboardName": "アカウント調査", "dataSetIdList": ["..."]}}
  }
}
```

- Quick Sight 固有の情報は `serviceEventDetails` に入る
- `readOnly` が true なので、証跡で「書き込みだけ」を記録する設定にすると閲覧が抜ける
- 直接クエリのダッシュボードで `QueryDatabase` だけが出て `GetDashboard` が出なかったという報告がある（コミュニティ情報。要検証）

## 4. サインインとのひも付け

- AWS コンソールへのサインインは `ConsoleLogin`（イベントソース `signin.amazonaws.com`）として記録される
  - 主な項目: `userIdentity`、`sourceIPAddress`、成否（`responseElements.ConsoleLogin`）、`additionalEventData.MFAUsed`
- Quick Sight 自体のサインインイベントはない
- IAM ユーザーの `ConsoleLogin` は、東京以外のリージョン（us-east-1 など）に記録されることがある。証跡はマルチリージョンにする
- ひも付けのキー

| 方式         | `ConsoleLogin` 側                   | Quick Sight のイベント側                       |
| ------------ | ----------------------------------- | ---------------------------------------------- |
| IAM ユーザー | `userIdentity.userName`             | `userIdentity.userName`                        |
| IAM ロール   | `principalId` のコロン以降（セッション名） | `principalId` のセッション名            |

- 同じユーザー名で「サインイン成功 → ダッシュボード閲覧」の時刻順に並べて突き合わせる

## 5. CloudTrail の設定

### 5.1 証跡の作成

```bash
aws cloudtrail create-trail --name qs-audit --s3-bucket-name <log-bucket> \
  --is-multi-region-trail --enable-log-file-validation --kms-key-id <KMS キーの ARN>
aws cloudtrail start-logging --name qs-audit
```

- CLI では、マルチリージョンと整合性検証の既定値が false。必ず指定する（コンソールで作る証跡は常にマルチリージョン）
- イベントセレクターを指定しなければ、管理イベントは読み取り・書き込みの両方が記録される
- 管理イベントの 1 つ目のコピーは無料
- ログが S3 に届くまで平均 5 分ほど

### 5.2 保存と改ざん対策

- ログ用の S3 バケットは専用にし、できれば別の AWS アカウントに置く
- 保存期間は S3 のライフサイクルで決める（公式の決まりはないので、社内の規程に合わせる）
- 整合性検証: 1 時間ごとにダイジェストファイルが届く。検証コマンドは次のとおり

```bash
aws cloudtrail validate-logs --trail-arn <証跡の ARN> --start-time 2026-10-01T00:00:00Z --verbose
```

### 5.3 イベント履歴との違い

| 項目     | イベント履歴（設定不要）     | 証跡（S3 に保存）            |
| -------- | ---------------------------- | ---------------------------- |
| 期間     | 過去 90 日                   | 自分で決める（無期限も可）   |
| 検索     | 1 リージョン、1 条件ずつ     | Athena で自由に検索          |
| 用途     | 一時的な調査                 | 監査の証跡                   |

## 6. Athena での検索

- テーブルは公式の DDL（パーティション射影つき）で作る。Athena ユーザーガイドの「CloudTrail ログのテーブル作成」を参照

誰がいつどのダッシュボードを開いたか:

```sql
SELECT eventtime,
  coalesce(useridentity.username, useridentity.principalid) AS who,
  json_extract_scalar(serviceeventdetails, '$.eventResponseDetails.dashboardDetails.dashboardName') AS dashboard
FROM cloudtrail_logs
WHERE eventsource = 'quicksight.amazonaws.com' AND eventname = 'GetDashboard'
  AND "timestamp" >= '2026/10/01'
ORDER BY eventtime DESC;
```

DB への問い合わせ（直接クエリ、取り込み）:

```sql
SELECT eventtime, useridentity.username, serviceeventdetails
FROM cloudtrail_logs
WHERE eventsource = 'quicksight.amazonaws.com' AND lower(eventname) = 'querydatabase'
  AND "timestamp" >= '2026/10/01';
```

コンソールへのサインイン:

```sql
SELECT eventtime, useridentity.username, sourceipaddress, additionaleventdata
FROM cloudtrail_logs
WHERE eventsource = 'signin.amazonaws.com' AND eventname = 'ConsoleLogin'
  AND json_extract_scalar(responseelements, '$.ConsoleLogin') = 'Success';
```

## 7. DB 側の監査ログ

| エンジン                  | 設定                                                                            | 出力先                         |
| ------------------------- | ------------------------------------------------------------------------------- | ------------------------------ |
| RDS for MySQL             | オプショングループに `MARIADB_AUDIT_PLUGIN`。`SERVER_AUDIT_EVENTS=CONNECT,QUERY` | CloudWatch Logs の `audit`     |
| Aurora MySQL              | クラスターパラメータ `server_audit_logging=ON`、`server_audit_events`           | CloudWatch Logs の `audit`     |
| RDS / Aurora PostgreSQL   | `shared_preload_libraries` に `pgaudit` → 再起動 → `CREATE EXTENSION pgaudit`   | CloudWatch Logs の `postgresql` |

- 対象を Quick Sight の DB ユーザーに絞る
  - MySQL 系: `SERVER_AUDIT_INCL_USERS`（Aurora は `server_audit_incl_users`）に `qs_spice_ro,qs_direct_ro`
  - PostgreSQL: `ALTER ROLE qs_direct_ro SET pgaudit.log TO 'read';`
- RDS for MySQL は、長い SQL が `SERVER_AUDIT_QUERY_LOG_LIMIT`（既定 1024 文字）で切れる
- CloudWatch Logs のロググループの保存期間は既定で無期限。期間を明示的に設定する
- DB のログに残るのは共有の DB ユーザー名。個人は CloudTrail の `QueryDatabase` と時刻で突き合わせる
- 「画面のフィルター値が SQL の WHERE に入る」ことは、導入時に監査ログで実際に確かめる（要検証）

## 8. アラート

| 検知したいこと                         | 方法                                                                             |
| -------------------------------------- | -------------------------------------------------------------------------------- |
| データソース・データセットの作成や変更 | EventBridge（`source: aws.quicksight`、例: `QuickSight DataSet Created`）         |
| ダッシュボードの共有範囲の変更         | EventBridge（`QuickSight Dashboard Permissions Updated`）                        |
| サインインの失敗                       | CloudWatch Logs のメトリクスフィルター（`ConsoleLogin` の失敗）                  |
| SPICE 取り込みの失敗                   | CloudWatch の `IngestionErrorCount`（「SPICE」ページ）                           |

- EventBridge の配信はベストエフォート（届く保証はない）。監査の正本は CloudTrail の証跡にする

## 9. 利用状況の見える化（任意）

- CloudWatch の `DashboardViewCount` でダッシュボードごとの閲覧数を見られる（誰が見たかは出ない）
- AWS 公式ブログ（2026-02）に、CloudTrail のログを Glue と Athena で整形し、Quick Sight で閲覧状況を見る構成例がある

## 10. 導入時に実機で確かめること

- サインイン → SPICE のダッシュボード閲覧 → 検索値の入力 → CSV 出力、の各操作でどのイベントが残るか
- 直接クエリのダッシュボードでも `GetDashboard` が必ず残るか
- `userIdentity` に個人の IAM ユーザー名が入るか
- Quick Sight のイベントがどのリージョンに記録されるか
- DB の監査ログに、入力したアカウント識別子が WHERE の値として残るか

## 11. 出典

確認日はいずれも 2026-10-20。「第三者情報」「コミュニティ情報」と書いたもの以外は AWS 公式。

- 1 章 結論
  - CloudTrail Lake の新規受付終了: https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-lake-service-availability-change.html
- 2 章 どの情報がどこに残るか
  - 検索値が記録されない（コミュニティ情報）: https://community.amazonquick.com/t/definition-of-a-eventname-in-cloudtrail-logs/22103
  - 同上（コミュニティ情報）: https://community.amazonquick.com/t/cloud-watch-information-avaliable/35917
  - ダウンロードの記録（コミュニティ情報）: https://community.amazonquick.com/t/in-aws-quick-sight-tool-how-to-create-audit-report-to-monitor-usage-of-published-dashboard/53208
  - SPICE は閲覧時に DB へ問い合わせない: https://docs.aws.amazon.com/quick/latest/userguide/spice.html
  - データソースの認証情報は 1 組: https://docs.aws.amazon.com/quick/latest/userguide/create-a-database-data-set.html
- 3 章 CloudTrail に残る Quick Sight のイベント
  - CloudTrail での記録（非 API イベントの一覧）: https://docs.aws.amazon.com/quicksight/latest/user/logging-non-api.html
  - ログと監視: https://docs.aws.amazon.com/quick/latest/userguide/incident-response-logging-and-monitoring.html
  - QueryDatabase の実例、GetDashboard の欠落（コミュニティ情報）: https://community.amazonquick.com/t/why-does-cloudtrail-show-querydatabase-events-for-quicksight-users-without-corresponding-getdashboard-events/46722
  - GetDashboard のログの形（第三者情報）: https://dev.classmethod.jp/articles/count-quicksight-access-with-athena-and-cloudtrail/
  - userIdentity の形（第三者情報）: https://dev.classmethod.jp/articles/quicksight-dashboard-usage-visualization-cloudtrail-athena/
- 4 章 サインインとのひも付け
  - コンソールのサインインイベント: https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-event-reference-aws-console-sign-in-events.html
  - userIdentity の項目: https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-event-reference-user-identity.html
  - Quick Sight のサインインイベントがない（コミュニティ情報）: https://community.amazonquick.com/t/event-name-to-capture-user-login-to-quick-sight/41294
- 5 章 CloudTrail の設定
  - create-trail の CLI: https://docs.aws.amazon.com/cli/latest/reference/cloudtrail/create-trail.html
  - 管理イベントの記録: https://docs.aws.amazon.com/awscloudtrail/latest/userguide/logging-management-events-with-cloudtrail.html
  - セキュリティのベストプラクティス: https://docs.aws.amazon.com/awscloudtrail/latest/userguide/best-practices-security.html
  - ログファイルの整合性検証: https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-log-file-validation-intro.html
  - 整合性検証の CLI: https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-log-file-validation-cli.html
  - ログが届くまでの時間: https://docs.aws.amazon.com/awscloudtrail/latest/userguide/get-and-view-cloudtrail-log-files.html
  - イベント履歴: https://docs.aws.amazon.com/awscloudtrail/latest/userguide/view-cloudtrail-events.html
  - 料金: https://aws.amazon.com/cloudtrail/pricing/
- 6 章 Athena での検索
  - CloudTrail のテーブル（パーティション射影）: https://docs.aws.amazon.com/athena/latest/ug/create-cloudtrail-table-partition-projection.html
- 7 章 DB 側の監査ログ
  - RDS for MySQL の監査プラグイン: https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Appendix.MySQL.Options.AuditPlugin.html
  - RDS for MySQL のログの CloudWatch 出力: https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_LogAccess.MySQLDB.PublishtoCloudWatchLogs.html
  - Aurora MySQL の監査: https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.Auditing.html
  - Aurora MySQL のログの CloudWatch 出力: https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.Integrating.CloudWatch.html
  - pgaudit の設定: https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Appendix.PostgreSQL.CommonDBATasks.pgaudit.basic-setup.html
  - pgaudit のユーザー別設定: https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Appendix.PostgreSQL.CommonDBATasks.pgaudit.exclude-user-db.html
  - Aurora PostgreSQL のログの CloudWatch 出力: https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraPostgreSQL.CloudWatch.html
- 8 章 アラート
  - Quick Sight の EventBridge イベント: https://docs.aws.amazon.com/eventbridge/latest/ref/events-ref-quicksight.html
  - サインイン失敗のアラーム: https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudwatch-alarms-for-cloudtrail.html
- 9 章 利用状況の見える化
  - CloudWatch メトリクス: https://docs.aws.amazon.com/quick/latest/userguide/monitoring-cloudwatch-metrics.html
  - 閲覧状況の見える化（AWS ブログ 2026-02）: https://aws.amazon.com/blogs/big-data/measure-the-adoption-of-your-amazon-quicksight-dashboards-and-view-your-bi-portfolio-in-a-single-pane-of-glass
