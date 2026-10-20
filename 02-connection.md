# 接続方式

## 1. 結論

- プライベートサブネットにある RDS / Aurora へは「VPC 接続」でつなぐ（Enterprise 限定）
- 接続先は、RDS ならリードレプリカ、Aurora ならリーダーエンドポイントにする（本番の書き込みと負荷を分ける）
- DB ユーザーは読み取り専用にし、SPICE 用と直接クエリ用の 2 つに分ける
- 認証情報は Secrets Manager に置く（設定は API / CLI）。ユーザー名とパスワードを直接入れる方法もある
- TLS は有効のままにする

## 2. 対応データソース（本件に関係するもの）

| 対象                    | 可否 | 前提・注意                                                                 |
| ----------------------- | ---- | -------------------------------------------------------------------------- |
| RDS / Aurora MySQL      | 可   | MySQL 5.7 以上（8.0 以上を推奨）。TLS 1.2 を使うには 5.7.28 以上が要る     |
| RDS / Aurora PostgreSQL | 可   | PostgreSQL 9.3.1 以上                                                      |
| Redshift                | 可   | VPC 接続を使える                                                           |
| Athena                  | 可   | VPC 接続は使えない。IAM ロールで Athena と S3 へのアクセスを許可する       |
| オンプレミスの DB       | 可   | VPC 接続の先に Direct Connect / VPN でルートを通す                         |

- 新しいエンジンバージョン（MySQL 8.4、PostgreSQL 17 以降など）への対応は公式に明記がない（要検証）
- RDS / Aurora への IAM データベース認証は公式に記載がない。IAM 認証のパラメータがあるのは Redshift だけ

## 3. ネットワーク方式の比較

略語: SG = セキュリティグループ、ENI = ネットワークインターフェース、NACL = ネットワーク ACL（サブネット単位の通信許可）

| 方式               | 仕組み                                                         | 本件                       |
| ------------------ | -------------------------------------------------------------- | -------------------------- |
| パブリック         | DB を公開し、SG で Quick Sight の送信元 IP を許可する          | 不採用（DB を公開できない）|
| VPC 接続           | Quick Sight の ENI を自社 VPC のサブネットに作る               | 採用                       |
| PrivateLink        | 利用者がインターネットを通らずに Quick Sight の画面や API に入る仕組み | DB 接続には使えない |
| Direct Connect/VPN | VPC 接続のサブネットからオンプレミスへルートを通す             | オンプレミスの DB の場合   |

- 東京リージョンの Quick Sight から DB へ向かう送信元 IP は `13.113.244.32/27`（パブリック方式で使う）
- VPC 接続は Quick Sight と同じリージョンの VPC にしか作れない

## 4. VPC 接続の作り方

### 4.1 前提条件

- サブネット: 異なる AZ のサブネットを 2 つ以上（API では 2〜15 個）
- 名前解決: DB のホスト名が解決できること（RDS / Aurora のエンドポイントは追加設定なしで解決できる）
- 経路: VPC 接続のサブネットから DB までルートがあること。NACL を使う場合は双方向を許可する
- 各サブネットに ENI 用の空き IP が要る（必要数は公式に記載なし）

### 4.2 実行ロール（Quick Sight が ENI を作るための IAM ロール）

信頼ポリシー:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Service": "quicksight.amazonaws.com"},
    "Action": "sts:AssumeRole"
  }]
}
```

許可ポリシー（最小）:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": [
      "ec2:CreateNetworkInterface",
      "ec2:DeleteNetworkInterface",
      "ec2:ModifyNetworkInterfaceAttribute",
      "ec2:DescribeSubnets",
      "ec2:DescribeSecurityGroups"
    ],
    "Resource": "*"
  }]
}
```

- VPC 接続を作る管理者に要る権限
  - `quicksight:CreateVPCConnection` などの VPC 接続系アクション（List / Describe / Update / Delete を含む）
  - `ec2:DescribeSubnets`、`ec2:DescribeVpcs`、`ec2:DescribeSecurityGroups`
  - `iam:ListRoles`、`iam:PassRole`（実行ロールの ARN に限定する）

### 4.3 セキュリティグループ

Quick Sight の ENI 専用の SG を新しく作る。次の設定なら、新旧どちらの仕様でも通信できる。

| SG                | 方向     | プロトコル・ポート    | 相手                |
| ----------------- | -------- | --------------------- | ------------------- |
| Quick Sight ENI 用 | inbound  | TCP 0-65535           | DB の SG            |
| Quick Sight ENI 用 | outbound | TCP 3306 または 5432  | DB の SG            |
| DB 用             | inbound  | TCP 3306 または 5432  | Quick Sight ENI 用 SG |

- 2023-04-27 より前に作った VPC 接続では、ENI の SG が stateful でない。そのため戻りの通信用に全ポートの inbound 許可が要る
- それ以降に作った接続の挙動は公式に明記がない（要検証）
- ENI 用 SG で全ポートの outbound を許可するのは非推奨

### 4.4 コンソールでの作成手順

1. プロフィールメニュー → Manage Quick → Manage VPC connections → Add VPC connection
2. 次の項目を入れて保存する

| 項目                   | 入力内容                                                                  |
| ---------------------- | ------------------------------------------------------------------------- |
| VPC connection name    | 任意の名前                                                                |
| VPC ID                 | DB のある VPC（作成後は変更できない）                                     |
| Execution role         | 4.2 の実行ロール（信頼ポリシーが正しいロールだけが一覧に出る）            |
| Subnets                | 異なる AZ のサブネットを 2 つ以上                                         |
| Security group IDs     | 4.3 の Quick Sight ENI 用 SG                                              |
| DNS resolver endpoints | 任意。プライベート DNS でしか引けないホスト名を使う場合だけ指定する       |

- 保存すると各サブネットに ENI が自動で作られる
- 状態は AVAILABLE / PARTIALLY_AVAILABLE / UNAVAILABLE。ENI が 1 つ消えても PARTIALLY_AVAILABLE で使い続けられる

### 4.5 CLI での作成

```json
{
  "AwsAccountId": "111122223333",
  "VPCConnectionId": "vpcconn-prod-db",
  "Name": "prod-db-vpc-connection",
  "SubnetIds": ["subnet-0aaa1111", "subnet-0bbb2222"],
  "SecurityGroupIds": ["sg-0qs00000"],
  "RoleArn": "arn:aws:iam::111122223333:role/qs-vpc-connection-role"
}
```

```bash
aws quicksight create-vpc-connection --region ap-northeast-1 --cli-input-json file://vpcconn.json
aws quicksight describe-vpc-connection --region ap-northeast-1 --aws-account-id 111122223333 --vpc-connection-id vpcconn-prod-db
```

## 5. DB 側の準備

### 5.1 DB ユーザーの方針

| DB ユーザー    | 用途                   | 権限                                         | 管理する人 |
| -------------- | ---------------------- | -------------------------------------------- | ---------- |
| `qs_spice_ro`  | SPICE 取り込み         | 問い合わせ用ビューへの SELECT だけ           | 管理者     |
| `qs_direct_ro` | 直接クエリ             | 問い合わせ用ビューへの SELECT だけ           | 管理者     |

- 2 つに分ける理由: DB の監査ログで、取り込みと直接クエリを見分けられる。直接クエリ側だけを止めることもできる
- Quick Sight は、データソースに設定した 1 組の認証情報を全利用者で共有する。DB 側では個人を区別できない
- テーブル情報を読むため、次のシステムテーブルを参照できる必要がある
  - MySQL 系: `INFORMATION_SCHEMA.STATISTICS`、`INFORMATION_SCHEMA.TABLES`（通常は権限を持つ表の行だけが自動で見える）
  - PostgreSQL 系: `pg_stats`、`pg_class`、`pg_namespace`（通常は既定で参照できる）

### 5.2 SQL の例

MySQL / Aurora MySQL:

```sql
CREATE USER 'qs_direct_ro'@'%' IDENTIFIED BY '<password>' REQUIRE SSL;
GRANT SELECT ON appdb.v_inq_contract TO 'qs_direct_ro'@'%';
GRANT SELECT ON appdb.v_inq_payment  TO 'qs_direct_ro'@'%';
```

PostgreSQL / Aurora PostgreSQL:

```sql
CREATE ROLE qs_direct_ro LOGIN PASSWORD '<password>';
GRANT CONNECT ON DATABASE appdb TO qs_direct_ro;
GRANT USAGE ON SCHEMA inquiry TO qs_direct_ro;
GRANT SELECT ON inquiry.v_inq_contract, inquiry.v_inq_payment TO qs_direct_ro;
ALTER ROLE qs_direct_ro SET statement_timeout = '120s';
```

- 120 秒でタイムアウトしても、DB 側でクエリが走り続けることがある。DB 側でも文のタイムアウトを設定しておく
- PostgreSQL で TLS を強制するには、パラメータグループで `rds.force_ssl = 1` にする
- MySQL でホストを `'user'@'host'` で絞る場合は、VPC 接続のサブネットの範囲に合わせる

## 6. データソースの作成

### 6.1 コンソールでの手順

1. Data → Create → New dataset
2. Aurora / MySQL / PostgreSQL のカードを選ぶ（手動入力）
3. 次の項目を入れ、Validate connection で確認してから Create data source を押す

| 項目              | 入力内容                                                  |
| ----------------- | --------------------------------------------------------- |
| Data source name  | 例: `prod-db-direct`（SPICE 用と直接クエリ用で分ける）     |
| Connection type   | 4 章で作った VPC 接続の名前                               |
| Database server   | RDS / Aurora のエンドポイント（ポート番号は付けない）     |
| Port              | 3306 / 5432                                               |
| Database name     | 接続する DB 名                                            |
| UserName/Password | 5 章の DB ユーザー                                        |
| Enable SSL        | オン（既定）                                              |

- RDS の自動検出カードもあるが、VPC 接続を選べるかは Redshift でしか確かめられていない。手動のカードを使う（要検証）
- 作成後、テーブルを選ぶ画面で SPICE か直接クエリかを選ぶ（「データセット」ページ）

### 6.2 Secrets Manager を使う場合

1. Manage Quick → Security & permissions → AWS リソースへのアクセス（Manage）→ AWS Secrets Manager → Select secrets
2. Quick Sight に読ませるシークレットを選んで Finish を押す（IAM ロール `aws-quicksight-secretsmanager-role-v0` が自動で作られる）
3. シークレットの値は `{"username":"qs_direct_ro","password":"..."}` の JSON にする（この 2 項目が必須）
4. CLI の `create-data-source` / `update-data-source` で `Credentials.SecretArn` を指定する
5. シークレットを KMS のカスタマー管理キーで暗号化している場合は、上記ロールに `kms:Decrypt` を付ける

- 注意: データソースを画面から編集すると、シークレットの指定が外れる。`update-data-source` で戻す
- API を呼ぶ人の IAM に `secretsmanager:GetSecretValue`（対象シークレットに限定）が要る

### 6.3 CLI での作成例（Aurora MySQL、VPC 接続、Secrets Manager）

```json
{
  "AwsAccountId": "111122223333",
  "DataSourceId": "prod-db-direct",
  "Name": "prod-db-direct",
  "Type": "AURORA",
  "DataSourceParameters": {"AuroraParameters": {
    "Host": "mycluster.cluster-ro-xxxx.ap-northeast-1.rds.amazonaws.com",
    "Port": 3306, "Database": "appdb"}},
  "Credentials": {"SecretArn": "arn:aws:secretsmanager:ap-northeast-1:111122223333:secret:qs/direct-ro-AbCdEf"},
  "VpcConnectionProperties": {"VpcConnectionArn": "<create-vpc-connection の応答の Arn>"},
  "SslProperties": {"DisableSsl": false},
  "Permissions": [{
    "Principal": "arn:aws:quicksight:ap-northeast-1:111122223333:group/default/dev-authors",
    "Actions": ["quicksight:DescribeDataSource", "quicksight:DescribeDataSourcePermissions",
                "quicksight:PassDataSource"]
  }]
}
```

```bash
aws quicksight create-data-source --region ap-northeast-1 --cli-input-json file://ds.json
```

- `Type` は `AURORA`（Aurora MySQL）、`AURORA_POSTGRESQL`、`MYSQL`、`POSTGRESQL` など
- パラメータの入れ物はエンジンごとに違う（`AuroraParameters`、`AuroraPostgreSqlParameters`、`MySqlParameters`、`PostgreSqlParameters`）

## 7. 直接クエリの制約

| 項目                       | 値                                           |
| -------------------------- | -------------------------------------------- |
| ビジュアルのタイムアウト   | 120 秒                                       |
| データセットのプレビュー   | 45 秒                                        |
| 結果の列数                 | 2,000 列まで。列名は 127 文字まで            |
| 行数                       | Quick Sight 側の上限なし                     |
| 同時接続数                 | 公式の値なし（要検証）                       |

- 画面を開く、フィルターを変える、といった操作のたびに DB へ SQL が飛ぶ。利用者が増えると DB の負荷が増える

## 8. よくある接続エラー

| 症状                          | 主な原因                                         | 対処                                                   |
| ----------------------------- | ------------------------------------------------ | ------------------------------------------------------ |
| `UNKNOWN_HOST`                | ホスト名の誤り、名前解決できない                 | エンドポイントを見直す。必要なら DNS resolver を設定   |
| `TIMEOUT`                     | SG、NACL、ルート、ポートの不備                   | 4.3 の SG を確認。同じサブネットの EC2 から疎通を試す  |
| `ACCESS_DENIED`               | 認証情報、ホスト制限、システムテーブルの権限     | 5 章の権限、`pg_hba.conf` を確認                       |
| `ENGINE_VERSION_NOT_SUPPORTED`| 対応外のバージョン                               | 2 章の最小バージョンを確認                             |
| SSL エラー（MySQL）           | 5.7.28 未満で TLS 1.2 だけを許可している         | 5.7.28 以上に上げる                                    |
| VPC 接続が UNAVAILABLE        | 実行ロールの権限か信頼ポリシーの不足             | 4.2 を確認                                             |
| ENI が CREATION_FAILURE       | 実行ロールの権限不足                             | ロールを直して作り直す                                 |
| シークレットが使えない        | 未選択、JSON の形式違い、`kms:Decrypt` 不足      | 6.2 を確認                                             |

## 9. 出典

確認日はいずれも 2026-10-20。「第三者情報」「コミュニティ情報」と書いたもの以外は AWS 公式。

- 2 章 対応データソース
  - 対応データソースとバージョン: https://docs.aws.amazon.com/quicksight/latest/user/supported-data-sources.html
  - VPC 接続に対応するデータソース: https://docs.aws.amazon.com/quicksight/latest/user/vpc-connection-supported-data-sources.html
  - Athena: https://docs.aws.amazon.com/quicksight/latest/user/athena.html
  - Redshift: https://docs.aws.amazon.com/quicksight/latest/user/enabling-access-redshift.html
  - IAM 認証のパラメータ（CreateDataSource）: https://docs.aws.amazon.com/quicksight/latest/APIReference/API_CreateDataSource.html
- 3 章 ネットワーク方式の比較
  - リージョン別の送信元 IP: https://docs.aws.amazon.com/quicksight/latest/user/regions.html
  - VPC 接続の概要: https://docs.aws.amazon.com/quicksight/latest/user/working-with-aws-vpc.html
  - パブリック接続の条件: https://docs.aws.amazon.com/quicksight/latest/user/configure-access.html
  - PrivateLink（VPC エンドポイント）: https://docs.aws.amazon.com/quicksight/latest/developerguide/vpc-interface-endpoints.html
  - Direct Connect / VPN へのルート: https://docs.aws.amazon.com/quicksight/latest/user/vpc-route-table.html
  - 同じリージョンの VPC だけ: https://docs.aws.amazon.com/quicksight/latest/user/vpc-finding-setup-information.html
- 4 章 VPC 接続の作り方
  - コンソールでの作成、実行ロール、ENI の状態: https://docs.aws.amazon.com/quicksight/latest/user/vpc-creating-a-connection-in-quicksight-console.html
  - CLI での作成: https://docs.aws.amazon.com/quicksight/latest/user/vpc-creating-a-connection-in-quicksight-cli.html
  - CreateVPCConnection の API: https://docs.aws.amazon.com/quicksight/latest/APIReference/API_CreateVPCConnection.html
  - サブネット: https://docs.aws.amazon.com/quicksight/latest/user/vpc-subnets.html
  - ENI: https://docs.aws.amazon.com/quicksight/latest/user/vpc-qeni.html
  - セキュリティグループ: https://docs.aws.amazon.com/quicksight/latest/user/vpc-security-groups.html
  - セキュリティグループの設定例: https://docs.aws.amazon.com/quicksight/latest/user/vpc-sample-rules.html
  - DNS リゾルバー: https://docs.aws.amazon.com/quicksight/latest/user/vpc-route-53.html
- 5 章 DB 側の準備
  - DB ユーザーに要る権限: https://docs.aws.amazon.com/quicksight/latest/user/required-permissions.html
  - ホスト制限と pg_hba.conf: https://docs.aws.amazon.com/quicksight/latest/user/configure-access.html
  - 出典なし（一般的な RDS / DB の設定）: リードレプリカへの接続、DB 側の文タイムアウト、`rds.force_ssl`、MySQL の information_schema の見え方
- 6 章 データソースの作成
  - データベースからのデータセット作成: https://docs.aws.amazon.com/quicksight/latest/user/create-a-database-data-set.html
  - VPC 接続を使うデータソース: https://docs.aws.amazon.com/quicksight/latest/user/vpc-creating-a-quicksight-data-source-profile.html
  - データソースの作成: https://docs.aws.amazon.com/quicksight/latest/user/create-a-data-source.html
  - RDS の自動検出: https://docs.aws.amazon.com/quicksight/latest/user/autodiscover-aws-data-sources.html
  - Secrets Manager 連携: https://docs.aws.amazon.com/quicksight/latest/user/secrets-manager-integration.html
  - CreateDataSource の API: https://docs.aws.amazon.com/quicksight/latest/APIReference/API_CreateDataSource.html
- 7 章 直接クエリの制約
  - タイムアウトと列数: https://docs.aws.amazon.com/quicksight/latest/user/data-source-limits.html
  - サービスクォータ: https://docs.aws.amazon.com/general/latest/gr/quicksight.html
  - DB への負荷（AWS ブログ）: https://aws.amazon.com/blogs/business-intelligence/best-practices-for-amazon-quicksight-spice-and-direct-query-mode/
- 8 章 よくある接続エラー
  - エラーの種類: https://docs.aws.amazon.com/quicksight/latest/APIReference/API_DataSourceErrorInfo.html
  - MySQL の TLS エラー: https://docs.aws.amazon.com/quicksight/latest/user/troubleshoot-connect-mysql.html
