# 未確認事項と参考 URL

## 1. 実機で確かめる項目

試用アカウントで確かめてから本番の設計に入れる。設計への影響が大きい順に並べた。

| #  | 項目                                                                 | 影響                         | 確かめ方                                      |
| -- | -------------------------------------------------------------------- | ---------------------------- | --------------------------------------------- |
| Q1 | サインアップした人が Admin Pro になるか                              | $250/月 の課金               | 作成直後に Manage users でロールを見る        |
| Q2 | 認証方式の表示名と API の値の対応                                    | 作り直しが要るかどうか       | `aws quicksight describe-account-subscription` |
| Q3 | 登録済みの IAM ユーザーに、Quick Sight 用の IAM 権限が別に要るか     | IAM ポリシーの設計           | 権限なしの IAM ユーザーで開いてみる           |
| Q4 | 直接クエリで、画面のフィルター値が SQL の WHERE に入るか             | DB 負荷、監査ログの中身      | DB の監査ログで SQL を見る                    |
| Q5 | 直接クエリのダッシュボードで `GetDashboard` が必ず記録されるか       | 証跡の抜け                   | 操作後に Athena で検索する                    |
| Q6 | Quick Sight のイベントが記録されるリージョン                         | Athena のテーブル設計        | 操作後に各リージョンのログを見る              |
| Q7 | パラメータの既定値を空にすると全件が表示されるか                     | 意図しない全件表示           | 既定値なしで公開して開く                      |
| Q8 | ダッシュボードの Viewer に、データセットの権限が要らないか           | 共有の設計                   | データセットを共有せずに Reader で開く        |
| Q9 | 2023-04-27 以降の VPC 接続で、ENI の SG が stateful か              | SG ルールを減らせるか       | inbound の全ポート許可を外して接続を試す      |
| Q10 | `CreateIngestion` の 32 回/24 時間がデータセット単位か               | 1 日 1 回なら影響しない（頻度を上げる場合だけ） | API で連続実行する                 |
| Q11 | 使う DB のエンジンバージョンに対応しているか                         | 接続できるかどうか           | Validate connection                           |
| Q12 | カスタム権限の `ShareDashboards` を DENY にした Author の挙動        | 共有作業の分担               | Author で共有を試す                           |
| Q13 | Quick Sight 独自のセッション時間の設定があるか（IAM ユーザーのコンソールは既定 12 時間。変える設定は公式に記載なし） | 放置端末の対策 | 管理画面と公式ドキュメントで確認 |
| Q14 | 「Quick Sight 以外を拒否」のポリシーの下で、Quick Sight の画面が問題なく使えるか | IAM ポリシーの設計 | 検証環境でビジネス部門の IAM ユーザーを作って試す |
| Q15 | 自動登録で作られるロールが想定どおりか（Reader / Author）            | 権限の誤り、課金             | 初回アクセス後に Manage users で確認          |
| Q16 | CLS の見せる相手をメンバーのいないグループにして、全員に禁止できるか（所有者を含む） | 個人情報の保護 | CLS を設定し、所有者と Reader で列が見えないか確かめる |
| Q17 | シークレットの自動ローテーション後も、取り込みと直接クエリが成功するか | 定期的な接続断             | ローテーションを手動で実行して試す            |
| Q18 | RLS のルールに IAM ユーザーを書く形式                                | 行の制限が効くか             | ユーザー名で書いて、Reader で開いて試す       |
| Q19 | Control Tower のガードレール（SCP）が Quick Sight の操作やリージョンを妨げないか | サインアップや VPC 接続ができない | 検証用アカウントで同じ SCP の下で試す |
| Q20 | Admin にも SPICE 10GB が付くか                                       | 容量の見積もり               | 管理画面の SPICE 容量を見る                   |

## 2. 公式情報どうしの食い違い

| 項目                           | 記述 A                                   | 記述 B                                 | 採用           |
| ------------------------------ | ---------------------------------------- | -------------------------------------- | -------------- |
| SPICE の 1 データセット上限    | 料金ページ: 10 億行 / 1TB                | ユーザーガイド: 20 億行 / 2TB          | B（2026-01 に拡大） |
| アカウント料金 $250 の条件     | 料金ページ本文: Pro ユーザー「または」Q&A | 同ページ脚注: Pro ユーザー「かつ」Q&A  | 両方を避ける   |
| 1 シートのビジュアル数         | API 管理の制限: 50                       | UI の説明: 30                          | 少ない 30 で設計 |
| 新規アカウントの推奨認証方式   | サインアップ画面: Password-based を推奨  | IAM フェデレーションの章: Identity Center を推奨 | 要件に合う前者 |
| 無料プロモーションの期限       | 2025-10 のブログ: 2026-04-30             | 現在の料金ページ: 2026-12-31           | 新しい後者     |

## 3. 調べたが公式に答えがなかったこと

- RDS / Aurora への IAM データベース認証（Redshift 以外は記載なし）
- 直接クエリの同時接続数の上限
- 表ビジュアルで表示できる行数の上限
- 同時に走れる SPICE 取り込みの数
- 1 アカウントに作れるダッシュボード数、データセット数

## 4. 出典

確認日は 2026-10-20 または 2026-10-21。「第三者情報」「コミュニティ情報」と書いたもの以外は AWS 公式。

製品・料金:

- 製品の概要: https://docs.aws.amazon.com/quick/latest/userguide/what-is.html
- 料金: https://aws.amazon.com/quicksight/pricing/
- サインアップ手順: https://docs.aws.amazon.com/quick/latest/userguide/signing-up.html
- 上限値: https://docs.aws.amazon.com/quick/latest/userguide/data-source-limits.html
- サービスクォータ: https://docs.aws.amazon.com/general/latest/gr/quicksight.html

接続方式:

- VPC 接続: https://docs.aws.amazon.com/quicksight/latest/user/working-with-aws-vpc.html
- VPC 接続のセキュリティグループ: https://docs.aws.amazon.com/quicksight/latest/user/vpc-security-groups.html
- Secrets Manager 連携: https://docs.aws.amazon.com/quicksight/latest/user/secrets-manager-integration.html
- DB ユーザーに要る権限: https://docs.aws.amazon.com/quicksight/latest/user/required-permissions.html

データセット・ビュー・SPICE:

- データセットの作成: https://docs.aws.amazon.com/quick/latest/userguide/create-a-database-data-set.html
- パラメータとコントロール: https://docs.aws.amazon.com/quick/latest/userguide/parameters-set-up.html
- URL でのパラメータ指定: https://docs.aws.amazon.com/quick/latest/userguide/parameters-in-a-url.html
- SPICE の更新: https://docs.aws.amazon.com/quick/latest/userguide/refreshing-imported-data.html
- SPICE の容量管理: https://docs.aws.amazon.com/quicksight/latest/user/managing-spice-capacity.html

権限:

- IAM ポリシーの例: https://docs.aws.amazon.com/quick/latest/userguide/iam-policy-examples.html
- カスタム権限: https://docs.aws.amazon.com/quick/latest/userguide/custom-permissions.html
- IAM ユーザーの管理: https://docs.aws.amazon.com/quick/latest/userguide/managing-user-access-qs-iam.html
- 認証方式の選び方（AWS 規範ガイダンス）: https://docs.aws.amazon.com/prescriptive-guidance/latest/quick-suite-access-approach/approach-overview.html

利用証跡:

- CloudTrail での記録: https://docs.aws.amazon.com/quicksight/latest/user/logging-non-api.html
- CloudTrail Lake の新規受付終了: https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-lake-service-availability-change.html
- Athena の CloudTrail テーブル: https://docs.aws.amazon.com/athena/latest/ug/create-cloudtrail-table-partition-projection.html
- Aurora MySQL の監査: https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.Auditing.html
- PostgreSQL の pgaudit: https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Appendix.PostgreSQL.CommonDBATasks.pgaudit.basic-setup.html

1・2 章（実機で確かめる項目、公式情報どうしの食い違い）の追加の出典:

- SPICE の上限拡大（2026-01）: https://aws.amazon.com/about-aws/whats-new/2026/01/amazon-quick-suite-launches-expanded-spice
- 分析の制限値（ビジュアル 50）: https://docs.aws.amazon.com/quick/latest/userguide/analysis-item-limits.html
- UI 側の説明（ビジュアル 30）: https://docs.aws.amazon.com/quick/latest/userguide/working-with-visual-types.html
- IAM フェデレーション（新規は Identity Center を推奨）: https://docs.aws.amazon.com/quick/latest/userguide/iam-federation.html
- 無料プロモーションの旧期限（AWS ブログ 2025-10）: https://aws.amazon.com/blogs/business-intelligence/reimagine-business-intelligence-amazon-quicksight-evolves-to-amazon-quick-suite/
- Q1 サインアップした人が Admin Pro になる事例（第三者情報）: https://dev.classmethod.jp/articles/quick-suite-subscribe/
- Q4 フィルターと SQL の関係（コミュニティ情報）: https://community.amazonquick.com/t/i-wanna-understand-filtering-quicksight-for-direct-query-reports/18985
- Q5 GetDashboard の欠落（コミュニティ情報）: https://community.amazonquick.com/t/why-does-cloudtrail-show-querydatabase-events-for-quicksight-users-without-corresponding-getdashboard-events/46722
- Q7 既定値が空のときの全件表示（コミュニティ情報）: https://community.amazonquick.com/t/parameter-field-when-kept-blank-and-is-minimized-displays-all/23017
- Q10 32 回の上限の単位（コミュニティ情報）: https://community.amazonquick.com/t/is-the-32-times-per-24-hours-createingestion-limit-applied-per-dataset-or-per-aws-account/51371
- Q13 IAM ユーザーのコンソールのセッション時間: https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_use_switch-role-console.html
- Q16 CLS の規則には見せる相手が最低 1 つ要る: https://docs.aws.amazon.com/quicksight/latest/APIReference/API_ColumnLevelPermissionRule.html
- 名前空間（IAM ユーザーは既定の名前空間だけ）: https://docs.aws.amazon.com/quick/latest/userguide/namespaces.html
- Q19 Control Tower のログ用アカウントと組織の証跡: https://docs.aws.amazon.com/controltower/latest/userguide/special-accounts.html
