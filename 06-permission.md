# 権限

## 1. 結論

- 認証方式は「(Recommended) Password-based or Single-Sign On」を選ぶ（作成後は変更できない）
- 利用者ごとに個人の IAM ユーザーを用意し、MFA を必須にする。共有の IAM ユーザーや共有ロールは使わない（証跡で個人を特定できなくなる）
- 管理者が IAM ユーザーを Quick Sight に登録する。利用者が自分で登録する方式（自己プロビジョニング）は使わない
- ロール: PO は Reader、開発は Author、管理者は Admin
- グループ: `po-readers`（PO）、`dev-authors`（開発）
- 直接クエリを開発部門に限る専用の設定はない。次の 3 つで実現する
  - 直接クエリ用のデータソースとダッシュボードを `dev-authors` にだけ共有する
  - PO を Reader にする（Reader はデータソースもデータセットも作れない）
  - カスタム権限で、Author の共有操作とデータソース作成を管理者だけに寄せる

## 2. 権限のしくみ（5 つの層）

| 層                  | 制御する内容                                                   | 設定する場所                      |
| ------------------- | -------------------------------------------------------------- | --------------------------------- |
| IAM                 | AWS へのサインイン、Quick Sight の管理操作                     | IAM のユーザーとポリシー          |
| Quick Sight ロール  | Reader / Author / Admin ごとにできる操作                       | Manage Quick → Manage users       |
| カスタム権限        | ロールでできる操作の一部を禁止する（Enterprise）               | Manage Quick、または CLI          |
| 資産ごとの権限      | どのデータソース、データセット、ダッシュボードを使えるか       | 各資産の共有設定                  |
| DB                  | DB ユーザーが読める表                                          | DB の GRANT                       |

- Quick Sight の Admin ロールだけでは足りない管理操作がある
  - VPC 接続、KMS、カスタム権限、IP 制限、AWS リソースへのアクセス設定など
  - これらには IAM 側の権限（`quicksight:*` など）も要る

## 3. 認証方式

### 3.1 選択肢

| サインアップ画面の表示                          | API の値（推定）      | 内容                                                  |
| ----------------------------------------------- | --------------------- | ----------------------------------------------------- |
| (Recommended) Password-based or Single-Sign On  | `IAM_AND_QUICKSIGHT`  | IAM のユーザー・ロールと、Quick Sight 独自のユーザー  |
| Single-Sign On Only                             | `IAM_ONLY`            | IAM のフェデレーションだけ                            |
| IAM Identity Center                             | `IAM_IDENTITY_CENTER` | Identity Center のグループを Quick Sight のロールに割り当てる |
| Active Directory                                | `ACTIVE_DIRECTORY`    | AD のグループで管理する                               |

- 表示名と API の値の対応は、公式に一覧がなく推定（第三者情報で補った）
- 東京リージョンでは 4 つとも選べる

### 3.2 この要件で Password-based or Single-Sign On を選ぶ理由

- IAM ユーザーで AWS コンソールにサインインし、そのまま Quick Sight を開ける
- IAM Identity Center は新規アカウントに推奨されているが、利用者は Identity Center のアクセスポータルから入る形になる
  - 「IAM で管理し、AWS にログインしてから使う」という要件と合わない
  - 他の方式と併用できない
- 将来 Identity Center へ移るには、アカウントを作り直す必要がある（全資産が消える）

### 3.3 作成後に変えられないもの

- 認証方式
- アカウント名（AWS 全体で一意）
- 最初に選んだリージョン（同梱の SPICE 容量が付く場所）

## 4. ユーザーの登録

### 4.1 コンソールで登録する

1. Manage Quick → Manage users → Invite users
2. User name に IAM ユーザー名、Email、Role を入れ、IAM User を Yes にする
3. 招待メールが届く。本人は 7 日以内に承諾し、IAM のパスワードで認証する

- メールアドレスは大文字と小文字を区別する。配布リストのアドレスは使えない

### 4.2 CLI で登録する

```bash
aws quicksight register-user --region ap-northeast-1 --aws-account-id 111122223333 --namespace default \
  --identity-type IAM --iam-arn arn:aws:iam::111122223333:user/po-yamada \
  --email po-yamada@example.com --user-role READER
```

- CLI で登録した場合、招待メールは送られない
- 登録した時点で課金が始まる
- 登録済みのユーザーに、Quick Sight 用の IAM 権限が別に要るかは明記がない（要検証）

### 4.3 IAM ロールで使う場合（参考）

- Quick Sight のユーザー名は「ロール名/セッション名」になる（例: `QuickSightReader/po-yamada`）
- 同じロールでも、セッション名が違えば別のユーザーとして扱われる
- 登録は `register-user` の `--iam-arn` にロールの ARN、`--session-name` にセッション名を指定する
- コンソールでロールを切り替えたときに、セッション名が IAM ユーザー名になるかは明記がない（要検証）

### 4.4 自己プロビジョニング（使わない方式の参考）

- IAM ポリシーで `quicksight:CreateReader`（Reader）、`CreateUser`（Author）、`CreateAdmin`（Admin）を許可する
- 本人が初めて Quick Sight を開いた時点でユーザーが作られ、課金が始まる
- 本人が初回にメールアドレスを入力するので、管理者は値を制御できない

## 5. ユーザーロール

| ロール（API の値） | できること                                                     | 月額 |
| ------------------ | -------------------------------------------------------------- | ---- |
| Reader（`READER`） | 共有されたダッシュボードの閲覧と操作（フィルター、入力）       | $3   |
| Author（`AUTHOR`） | データソース、データセット、分析、ダッシュボードの作成         | $24  |
| Admin（`ADMIN`）   | Author の操作に加え、ユーザー管理、SPICE の購入、アカウント設定 | $24  |
| Pro 系             | 上記に生成 AI の機能を加えたもの                               | 本件では使わない |

- ロールの変更: Manage users の画面、または `aws quicksight update-user --role AUTHOR`

## 6. グループ

```bash
aws quicksight create-group --region ap-northeast-1 --aws-account-id 111122223333 --namespace default \
  --group-name dev-authors
aws quicksight create-group-membership --region ap-northeast-1 --aws-account-id 111122223333 --namespace default \
  --group-name dev-authors --member-name dev-suzuki
```

- コンソール: Manage Quick → Manage groups → NEW GROUP（グループの削除は CLI でしかできない）
- 資産の共有はグループ単位にする。1 つの資産に共有できる相手は 100 までで、グループは 1 と数える

## 7. 資産ごとの権限

| 資産           | 権限の種類                                           | 付与先               |
| -------------- | ---------------------------------------------------- | -------------------- |
| データソース   | 利用（データセットを作れる）/ 所有者                 | ユーザー、グループ   |
| データセット   | User（分析を作れる）/ Owner（編集、削除、共有）      | ユーザー、グループ   |
| 分析           | 閲覧 / 共同所有                                      | ユーザー、グループ   |
| ダッシュボード | Viewer / Co-owner（グループに付けられるのは Viewer） | ユーザー、グループ、全員 |
| フォルダ       | Viewer / Contributor / Owner                         | ユーザー、グループ   |

- フォルダに入れた資産は、フォルダの権限を引き継ぐ
- ダッシュボードの Viewer に要るアクション: `DescribeDashboard`、`QueryDashboard`、`ListDashboardVersions`

ダッシュボードを PO グループに共有する CLI:

```bash
aws quicksight update-dashboard-permissions --region ap-northeast-1 --aws-account-id 111122223333 \
  --dashboard-id <DASHBOARD_ID> --grant-permissions \
  'Principal=arn:aws:quicksight:ap-northeast-1:111122223333:group/default/po-readers,Actions=quicksight:DescribeDashboard,quicksight:QueryDashboard,quicksight:ListDashboardVersions'
```

## 8. カスタム権限（Enterprise）

- ロールで許された操作の一部を禁止する機能。権限を広げることはできない
- 適用先はアカウント、ロール（Admin / Author / Reader）、ユーザーの 3 種類。優先順位はユーザー > ロール > アカウント
- グループへの適用は公式に記載がない
- コンソール（Manage Quick）でも CLI でも、ユーザーとロールに割り当てられる

主な項目（値は `ALLOW` / `DENY`）:

| 目的                         | 項目名                                                       |
| ---------------------------- | ------------------------------------------------------------ |
| データソースの作成・共有     | `CreateAndUpdateDataSources`、`ShareDataSources`             |
| データセットの作成・共有     | `CreateAndUpdateDatasets`、`CreateSPICEDataset`、`ShareDatasets` |
| ダッシュボード・分析の共有   | `ShareDashboards`、`ShareAnalyses`                           |
| ダウンロード・印刷           | `ExportToCsv`、`ExportToExcel`、`ExportToPdf`、`PrintReports` |

本件での設定案:

| 適用先        | 禁止する項目                                                       | 目的                           |
| ------------- | ------------------------------------------------------------------ | ------------------------------ |
| Reader ロール | `ExportToCsv`、`ExportToExcel`、`ExportToPdf`、`PrintReports`      | データの持ち出し防止（社内方針次第） |
| Author ロール | `CreateAndUpdateDataSources`、`ShareDataSources`、`ShareDashboards` | 直接クエリの資産が PO へ渡るのを防ぐ |
| アカウント    | 生成 AI の機能（`--governance` で既定を拒否）                      | 不要な機能と課金の防止         |

```bash
aws quicksight create-custom-permissions --region ap-northeast-1 --aws-account-id 111122223333 \
  --custom-permissions-name author-restricted \
  --capabilities '{"CreateAndUpdateDataSources":"DENY","ShareDataSources":"DENY","ShareDashboards":"DENY"}'
aws quicksight update-role-custom-permission --region ap-northeast-1 --aws-account-id 111122223333 \
  --namespace default --role AUTHOR --custom-permissions-name author-restricted
```

- 生成 AI の一括拒否は `--governance '{"DefaultCategoryEffects":{"AI":"DENY_BY_DEFAULT"}}'` で指定する
- 更新すると設定全体が置き換わる。1 項目だけ変える場合も、全項目を指定して送る
- 親の項目を DENY にすると、子の項目も拒否される
- Author の共有を止めると、ダッシュボードの共有は Admin の作業になる

## 9. 要件の実現: 直接クエリを開発部門だけにする

| 資産                                   | `po-readers`       | `dev-authors`          | 管理者 |
| -------------------------------------- | ------------------ | ---------------------- | ------ |
| データソース（SPICE 用）               | 共有しない         | 利用                   | 所有者 |
| データソース（直接クエリ用）           | 共有しない         | 利用                   | 所有者 |
| SPICE のデータセット                   | 共有しない         | User または Owner      | Owner  |
| 直接クエリのデータセット               | 共有しない         | User または Owner      | Owner  |
| PO 用ダッシュボード（SPICE）           | Viewer             | Viewer                 | 所有者 |
| 開発用ダッシュボード（直接クエリ）     | 共有しない         | Viewer または Co-owner | 所有者 |

- 注意: 直接クエリのダッシュボードは、閲覧者の操作でもデータソースの認証情報を使って DB に問い合わせる
  - PO に誤って共有すると、実質的に PO に直接クエリを開放したことになる
  - 直接クエリのダッシュボードは「全員に共有」やリンク共有を使わない
- フォルダを「PO 公開用」と「開発用」に分け、フォルダ単位で共有すると誤りが減る
- 共有の状態は定期的に確認する（Manage Quick → Manage assets、または CloudTrail の `UpdateDashboardAccess`）

## 10. IAM ポリシーの例

Quick Sight の管理者（契約の誤削除を防ぐ Deny 付き）:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {"Effect": "Allow", "Action": "quicksight:*", "Resource": "*"},
    {"Effect": "Deny",
     "Action": ["quicksight:Unsubscribe", "quicksight:DeleteAccountSubscription"],
     "Resource": "*"}
  ]
}
```

- VPC 接続や AWS リソース設定には、これに加えて `ec2:Describe*`、`iam:ListRoles`、`iam:PassRole` などが要る（「接続方式」ページ）

自己プロビジョニングを使う場合の例（PO 用。開発用は `CreateReader` を `CreateUser` にする）:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": "quicksight:CreateReader",
    "Resource": "arn:aws:quicksight::111122223333:user/${aws:userid}"
  }]
}
```

## 11. AWS リソースへのアクセス設定

- 場所: Manage Quick → Security & permissions → AWS サービスへのアクセス（Manage）
- ここで選んだサービスに対し、Quick Sight が IAM ロールとポリシーを作る。既存の IAM ロールを指定することもできる
- 本件で使う項目
  - Secrets Manager: データソースで使うシークレットを選ぶ
  - RDS: 自動検出を使う場合だけ
- プライベートサブネットの DB へは、この設定ではなく VPC 接続で到達する

## 12. アカウント全体のセキュリティ設定

| 設定                         | 内容                                                                     |
| ---------------------------- | ------------------------------------------------------------------------ |
| IP・VPC エンドポイント制限   | 社内の IP からだけ画面を使えるようにする（Enterprise、最大 100 ルール） |
| MFA                          | IAM 側で必須にする（`aws:MultiFactorAuthPresent` 条件など）              |
| 削除保護                     | Account termination protection をオンにしておく                         |
| SCP                          | 別アカウントでの勝手な契約や、エディション・認証方式を縛れる             |

- IP 制限の場所: Manage Quick → Security & permissions → IP and VPC endpoint restrictions
  - 反映まで最大 10 分。全リージョンに効く。自分の IP を含むルールがないと保存できない
  - 対象は画面（Web）、埋め込み、モバイル。API には効かないので、API は IAM の `aws:SourceIp` で絞る
- セッションの長さは IAM 側の設定に従う。Quick Sight 独自の設定があるかは明記がない（要検証）

## 13. 行レベル・列レベルのセキュリティ

- 本件は「アカウント識別子で全件を引く」用途なので、行レベルセキュリティ（RLS）は不要
- 列を隠したい場合は、Quick Sight の列レベルセキュリティより DB ビューで列を除くほうが管理しやすい
- 詳細は「データセット」ページの 7 章

## 14. 出典

確認日はいずれも 2026-10-20。「第三者情報」「コミュニティ情報」と書いたもの以外は AWS 公式。

- 1・3 章 認証方式
  - サインアップ画面の選択肢: https://docs.aws.amazon.com/quick/latest/userguide/signing-up.html
  - API の認証方式の値: https://docs.aws.amazon.com/quicksight/latest/APIReference/API_CreateAccountSubscription.html
  - 表示名と API の値の対応（第三者情報）: https://dev.classmethod.jp/articles/quick-suite-subscribe/
  - Identity Center 連携と変えられない設定: https://docs.aws.amazon.com/quicksuite/latest/userguide/setting-up-sso.html
  - IAM フェデレーション（新規は Identity Center を推奨）: https://docs.aws.amazon.com/quick/latest/userguide/iam-federation.html
  - 認証方式の選び方（AWS 規範ガイダンス）: https://docs.aws.amazon.com/prescriptive-guidance/latest/quick-suite-access-approach/approach-overview.html
  - Identity Center 連携の利用方法: https://docs.aws.amazon.com/prescriptive-guidance/latest/quick-suite-access-approach/iam-identity-center-integration.html
- 2 章 権限のしくみ
  - IAM 権限が要る管理操作: https://docs.aws.amazon.com/quick/latest/userguide/identity.html
- 4 章 ユーザーの登録
  - IAM ユーザーの招待: https://docs.aws.amazon.com/quicksight/latest/user/managing-user-access-qs-iam.html
  - IAM ユーザーでの利用（AWS 規範ガイダンス）: https://docs.aws.amazon.com/prescriptive-guidance/latest/quick-suite-access-approach/iam-users.html
  - RegisterUser: https://docs.aws.amazon.com/quicksight/latest/APIReference/API_RegisterUser.html
  - register-user の CLI: https://docs.aws.amazon.com/cli/latest/reference/quicksight/register-user.html
  - ロール利用時のユーザー名: https://docs.aws.amazon.com/prescriptive-guidance/latest/quick-suite-access-approach/iam-identity-center-federation.html
  - 自己プロビジョニング: https://docs.aws.amazon.com/quick/latest/userguide/provisioning-users.html
  - SAML 連携でのユーザー作成: https://docs.aws.amazon.com/quick/latest/userguide/external-identity-providers-setting-up-saml.html
- 5 章 ユーザーロール
  - ユーザーの種類: https://docs.aws.amazon.com/quicksuite/latest/userguide/user-types.html
  - 料金: https://aws.amazon.com/quicksight/pricing/
  - ロールの変更（UpdateUser）: https://docs.aws.amazon.com/quicksight/latest/APIReference/API_UpdateUser.html
- 6 章 グループ
  - グループの作成: https://docs.aws.amazon.com/quicksight/latest/user/creating-quicksight-groups.html
  - 共有先の上限: https://docs.aws.amazon.com/quicksight/latest/APIReference/qs-api-permissions.html
- 7 章 資産ごとの権限
  - ダッシュボードの共有: https://docs.aws.amazon.com/quick/latest/userguide/share-a-dashboard-grant-access-users.html
  - データセットの共有: https://docs.aws.amazon.com/quicksight/latest/user/sharing-data-sets.html
  - データソースの権限: https://docs.aws.amazon.com/quicksight/latest/developerguide/update-data-source-permissions.html
  - UpdateDashboardPermissions: https://docs.aws.amazon.com/quicksight/latest/APIReference/API_UpdateDashboardPermissions.html
  - フォルダの権限: https://docs.aws.amazon.com/quicksight/latest/user/folders-security.html
- 8 章 カスタム権限
  - カスタム権限: https://docs.aws.amazon.com/quick/latest/userguide/custom-permissions.html
  - 項目の一覧（Capabilities）: https://docs.aws.amazon.com/quicksight/latest/APIReference/API_Capabilities.html
  - 生成 AI の一括拒否: https://docs.aws.amazon.com/quick/latest/userguide/custom-permissions-governance.html
- 9 章 直接クエリを開発部門だけにする
  - 出典なし（2〜8 章の仕様を組み合わせた設計案）
  - 閲覧時にもデータソースの認証情報が使われる: https://docs.aws.amazon.com/quicksight/latest/user/secrets-manager-integration.html
- 10 章 IAM ポリシーの例
  - IAM ポリシーの例: https://docs.aws.amazon.com/quick/latest/userguide/iam-policy-examples.html
- 11 章 AWS リソースへのアクセス設定
  - AWS リソースへのアクセス: https://docs.aws.amazon.com/quick/latest/userguide/accessing-data-sources.html
  - 既存の IAM ロールの利用: https://docs.aws.amazon.com/quick/latest/userguide/security-create-iam-role.html
- 12 章 アカウント全体のセキュリティ設定
  - IP・VPC エンドポイント制限: https://docs.aws.amazon.com/quick/latest/userguide/manage-ip-vpc.html
  - VPC エンドポイント: https://docs.aws.amazon.com/quicksight/latest/developerguide/vpc-interface-endpoints.html
  - SCP と条件キー: https://docs.aws.amazon.com/quicksight/latest/user/security-scp.html
  - 削除保護: https://docs.aws.amazon.com/quicksight/latest/user/closing-account.html
  - IAM のセッション時間: https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_use_switch-role-console.html
  - 出典なし（一般的な IAM の設定）: MFA を必須にする条件キー `aws:MultiFactorAuthPresent`
- 13 章 行レベル・列レベルのセキュリティ
  - RLS: https://docs.aws.amazon.com/quicksight/latest/user/restrict-access-to-a-data-set-using-row-level-security.html
  - CLS: https://docs.aws.amazon.com/quicksuite/latest/userguide/restrict-access-to-a-data-set-using-column-level-security.html
