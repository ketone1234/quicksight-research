# 構築手順チェックリスト

実装する順に並べた。各項目の詳細は括弧内のページを見る。

## 0. 事前に決めること

- [ ] 対象の AWS アカウントとリージョン（ap-northeast-1）
- [ ] 命名規約とタグ（環境名、システム名など）
- [ ] 管理の方法（手作業か IaC か。IaC なら CloudFormation か Terraform かを統一する）
- [ ] 対象の DB（本件は RDS for PostgreSQL の既存のリードレプリカ）と、その VPC・サブネット・SG
- [ ] 公開用ビューにするテーブルと、除く列（電話番号とそのハッシュなど）
- [ ] 問い合わせの種別と、分析で見たい指標
- [ ] 利用者の一覧と区分（ビジネス部門は Reader、開発部門は Reader または Author）
- [x] IAM ユーザーの作り方（個人ごとに作る）
- [ ] データの新しさの要件（想定は SPICE の取り込み 1 日 1 回。画面のデータは最大で約 1 日前になる）
- [x] CSV などのダウンロードを許すか（制限しない）
- [x] 証跡の保存期間（1 年間）
- [ ] DB の監査ログの保存期間
- [ ] マルチアカウントの統制（AWS Control Tower のガードレール、SCP）と矛盾しないか
- [ ] 本番の前に、検証環境（試用アカウント）で権限とアクセスを確かめるか
- [ ] 試用アカウントで、認証方式「Single-Sign On Only」のまま IAM ユーザーで使えるかを確かめる（4 章の認証方式を決める）

## 1. 証跡の基盤を先に作る（利用証跡）

- [ ] 組織の CloudTrail 証跡が既にあるか確認する（AWS Control Tower の CloudTrail 連携が有効なら、組織の証跡がある）
- [ ] 組織の証跡の保存期間が 1 年になっているか確認する（Control Tower の既定は 1 年）
- [ ] なければ、全リージョンの証跡を作り、専用の S3 バケットに保存する（暗号化、整合性検証、長期保管、改ざん防止）
- [ ] Athena の CloudTrail テーブル

## 2. IAM の準備（権限）

- [ ] アカウントのサインイン URL（アカウントエイリアス）を整える
- [ ] パスワードポリシー（長さ、有効期限、再利用の制限）を設定する
- [ ] IAM グループを作る: ビジネス部門用（Quick Sight 専用）と開発部門用
- [ ] MFA 必須のポリシーを両方のグループに付ける（仮想 MFA を既定、FIDO セキュリティキーも可。初回サインイン時に登録させる）
- [ ] ビジネス部門用グループに、Quick Sight 以外の AWS 操作を拒否するポリシーを付ける
- [ ] Quick Sight の自動登録用のポリシーを付ける（ビジネス部門は `CreateReader`、Author にする開発者は `CreateUser`）
- [ ] ビジネス部門の IAM ユーザーを新しく作り、グループに入れる。アクセスキーは発行しない
- [ ] 開発部門は既存の IAM ユーザーを使い、グループに入れる
- [ ] 権限境界や SCP と矛盾しないか確かめる

## 3. DB 側の準備（接続方式、ビュー）

- [ ] 公開用ビュー（個人情報の列を除く。先頭列 `account_id`、文字列型、取り込み時刻の列）
- [ ] 必要なら、縦持ちに変換するビューと、サービス横断のビュー
- [ ] `account_id` のインデックス
- [ ] DB ユーザー `qs_spice_ro`、`qs_direct_ro`（プライマリで作る。公開用ビューへの SELECT のみ、TLS 必須、文タイムアウト）
- [ ] DB の監査ログ（pgaudit。対象を上の 2 ユーザーに絞り、CloudWatch Logs へ出力、保存期間を設定）
- [ ] Secrets Manager に 2 つのシークレット（`username` と `password` の JSON）。自動ローテーションの接続先はプライマリ

## 4. Quick Sight アカウントの作成（権限）

- [ ] サインアップ専用の IAM ロールを用意する（AdministratorAccess を使わない）
- [ ] AWS コンソールで Quick を開き、サインアップする
  - リージョン: アジアパシフィック（東京）
  - 認証方式: Single-Sign On Only（試用で使えなかった場合は (Recommended) Password-based or Single-Sign On）
- [ ] 自分のロールが Admin Pro になっていないか確認し、なっていれば Admin に変える（$250/月 を避ける）
- [ ] SPICE の自動購入をオフにする（足りなくなってきたら手動で追加購入する）
- [ ] Account termination protection がオンか確認する
- [ ] 生成 AI の機能を、カスタム権限のガバナンス設定で既定拒否にする

## 5. ネットワーク（接続方式）

- [ ] VPC 接続の実行ロール（信頼ポリシーと ec2 の 5 権限）
- [ ] Quick Sight ENI 専用の SG と、リードレプリカの SG の inbound ルール（ENI 用 SG からの 5432 だけを許可）
- [ ] VPC 接続を作り（異なる AZ のプライベートサブネット 2 つ以上）、状態が AVAILABLE になることを確認する

## 6. データソース（接続方式）

- [ ] Security & permissions で、Secrets Manager の 2 つのシークレットを許可する
- [ ] データソース `prod-db-spice` と `prod-db-direct` を CLI で作る（リードレプリカのエンドポイント、VPC 接続、シークレット、TLS）
- [ ] 両方とも接続できることを確認する

## 7. ユーザー、グループ、共有（権限）

- [ ] 各利用者に Quick Sight を一度開いてもらい、自動登録されたロールが想定どおりか確認する
- [ ] Quick Sight のグループ `business-readers`、`dev-members` を作り、メンバーを入れる
- [ ] カスタム権限: Author ロールの共有とデータソース作成を禁止（ダウンロードは制限しない）
- [ ] データソースの共有: `prod-db-spice` と `prod-db-direct` を `dev-members` に（ビジネス部門には共有しない）
- [ ] 必要なら IP 制限（社内の IP だけ許可）

## 8. データセット（データセット、SPICE）

- [ ] 公開用ビューごとに、SPICE のデータセットと直接クエリのデータセットを作る
- [ ] 個人情報の列を含むデータセットがあれば、CLS でその列を禁止する
- [ ] 参照範囲を絞る場合は RLS のルールを用意してから有効にする
- [ ] SPICE の更新スケジュール（1 日 1 回の全件更新。DB の負荷が低い時間帯）と、失敗時のメール通知
- [ ] 容量の使用量を確認する（見積もりとの比較）

## 9. 画面（ダッシュボード）

- [ ] 問い合わせ用のシート: パラメータ `AccountId`、テキスト入力のコントロール、データセットごとのフィルター、種別ごとのシート
- [ ] 分析用のシート: オーダーの推移、予約から実行までの日数、契約状態別の件数など
- [ ] 取り込み時刻の表示
- [ ] ダッシュボードとして公開（ダウンロードは許可する）
- [ ] ビジネス部門用ダッシュボードを `business-readers` に共有
- [ ] 開発部門用（直接クエリ）の分析とダッシュボードを作り、`dev-members` にだけ共有
- [ ] フォルダを「ビジネス部門公開用」と「開発部門用」に分ける
- [ ] 表示の速さとデータの新しさを確認する

## 10. 監視とアラート（SPICE、利用証跡）

- [ ] `IngestionErrorCount` のアラーム（SPICE 更新の失敗を通知）
- [ ] `SPICECapacityConsumedInMB` のアラーム（上限に近づいたら手動で追加購入する）
- [ ] EventBridge: データソース・データセットの作成、ダッシュボードの共有範囲の変更
- [ ] `ConsoleLogin` 失敗の検知

## 11. 受け入れテスト

| #   | 確認すること                                                         | 期待する結果                                   |
| --- | -------------------------------------------------------------------- | ---------------------------------------------- |
| T1  | ビジネス部門がコンソールにサインインし、ビジネス部門用ダッシュボードを開く | アカウント識別子で各シートが絞り込まれる       |
| T2  | ビジネス部門がデータセットやデータソースの一覧を開く                 | 作成できない。直接クエリの資産が見えない       |
| T3  | ビジネス部門が開発部門用ダッシュボードの URL を直接開く              | 権限エラーになる                               |
| T4  | ビジネス部門が Quick Sight 以外の AWS サービス（EC2 など）を開く     | 拒否される                                     |
| T5  | MFA を登録していない IAM ユーザーでサインインする                    | MFA の登録など以外の操作が拒否される           |
| T6  | 全利用者の画面とデータセットで、電話番号とそのハッシュの列を探す     | どこにも表示されない                           |
| T7  | 開発部門が直接クエリのダッシュボードで検索する                       | 最新の値が出る。DB の監査ログに SQL が残る     |
| T8  | 開発部門がダッシュボードをビジネス部門に共有しようとする             | カスタム権限で拒否される                       |
| T9  | ビジネス部門が CSV と Excel で出力する                               | 出力できる                                     |
| T10 | CloudTrail を Athena で検索する                                      | `ConsoleLogin` と `GetDashboard` が個人名で残る |
| T11 | SPICE の取り込みをわざと失敗させる（DB ユーザーを一時停止など）      | メール通知と CloudWatch アラームが届く         |
| T12 | シークレットをローテーションした後、取り込みと直接クエリを実行する   | どちらも成功する                               |
| T13 | 請求の画面で Quick Sight の料金を確認する                            | $250 のアカウント料金がかかっていない          |

## 12. 運用

- 利用者の追加・削除の手順を決める（IAM ユーザー、Quick Sight のユーザー、グループ）
- 退職や異動のときは IAM ユーザーと Quick Sight ユーザーの両方を消す。月の途中で消しても、その月は満額
- ユーザーを消すと、その人だけが所有する資産も消える。消す前に所有者を移す
- 共有範囲と証跡を定期的に確認する（月 1 回など）
- 認証方式を Password-based or Single-Sign On にした場合は、Quick Sight 専用の ID が混ざっていないかも定期的に確認する

## 13. 出典

確認日は 2026-10-20 または 2026-10-21。「第三者情報」「コミュニティ情報」と書いたもの以外は AWS 公式。

- 各手順の根拠は、括弧内に示したページの「出典」節にある。ここには、このページで初めて出てくる事実の出典だけを載せる
- 1 章 証跡の基盤
  - Control Tower のログの保存期間: https://docs.aws.amazon.com/controltower/latest/userguide/configure-log-retention.html
- 2 章 IAM の準備
  - パスワードポリシー: https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_passwords_account-policy.html
  - アカウントエイリアス: https://docs.aws.amazon.com/IAM/latest/UserGuide/console_account-alias.html
  - MFA 必須化のポリシー例: https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_examples_aws_my-sec-creds-self-manage.html
- 4 章 Quick Sight アカウントの作成
  - サインアップ手順: https://docs.aws.amazon.com/quick/latest/userguide/signing-up.html
  - サインアップに要る IAM 権限: https://docs.aws.amazon.com/quicksight/latest/user/iam-policy-examples.html
  - サインアップした人が Admin Pro になる事例（第三者情報）: https://dev.classmethod.jp/articles/quick-suite-subscribe/
  - SPICE の自動購入: https://docs.aws.amazon.com/quicksight/latest/user/managing-spice-capacity.html
  - 削除保護: https://docs.aws.amazon.com/quicksight/latest/user/closing-account.html
  - 生成 AI の一括拒否: https://docs.aws.amazon.com/quick/latest/userguide/custom-permissions-governance.html
- 12 章 運用
  - 月途中のユーザー削除と課金: https://docs.aws.amazon.com/quick/latest/userguide/understanding-quick-user-billing.html
  - ユーザー削除時の資産の扱い: https://docs.aws.amazon.com/quick/latest/userguide/managing-users.html
- 出典なし: 0 章の事前決定事項、11 章の受け入れテスト、12 章の定期確認は、チームの方針とこの調査から出した提案
