# 権限

## 1. 結論

- 認証方式は「(Recommended) Password-based or Single-Sign On」を選ぶ（作成後は変更できない）
- IAM ユーザーは個人ごとに作る（2026-10-22 に決定）。共有の IAM ユーザーや共有ロールは使わない（証跡で個人を特定できなくなる）
  - ビジネス部門は IAM ユーザーを新しく作る。開発部門は既存の IAM ユーザーを使う
  - アクセスキーは発行せず、マネジメントコンソールだけで使う
- MFA を全員に必須にする（仮想 MFA を既定、FIDO セキュリティキーも可）。初回サインイン時に登録させる
- ビジネス部門は、Quick Sight 以外の AWS 操作を IAM で拒否する（4.3）
- Quick Sight のユーザーは、初回アクセス時に自動で登録する（自己プロビジョニング。4.4）
- ロール: ビジネス部門は Reader。開発部門は Reader または Author。管理者は Admin
- グループ: `business-readers`（ビジネス部門）、`dev-members`（開発部門）
- 直接クエリを開発部門に限る専用の設定はない。次の 3 つで実現する
  - 直接クエリ用のデータソースとダッシュボードを `dev-members` にだけ共有する
  - ビジネス部門を Reader にする（Reader はデータソースもデータセットも作れない）
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

## 4. IAM ユーザーと Quick Sight ユーザー

### 4.1 IAM ユーザーの準備

| 項目                 | 方針                                                                 |
| -------------------- | -------------------------------------------------------------------- |
| ビジネス部門         | IAM ユーザーを新しく作る                                             |
| 開発部門             | 既存の IAM ユーザーを使う                                            |
| アクセスキー         | 発行しない。マネジメントコンソールだけで使う                         |
| サインイン URL       | アカウントエイリアスを付け、覚えやすい URL にする                    |
| パスワードポリシー   | 長さ、有効期限、再利用の制限を決める                                 |
| IAM グループ         | ビジネス部門用（Quick Sight 専用）と開発部門用に分け、ポリシーはグループに付ける |

### 4.2 MFA の必須化

- 方式: 仮想 MFA（認証アプリ）を既定にし、FIDO セキュリティキーも認める
- 「MFA で認証していないときは、MFA の登録とパスワード変更に要る操作以外を拒否する」ポリシーを、両方のグループに付ける
  - こうすると、初回サインイン時に MFA を登録しない限り、何もできない
  - AWS 公式のポリシー例（自分の MFA とパスワードを管理させる例）をもとに作る
- 拒否の部分は次の形になる（`NotAction` に並べる操作は公式の例に合わせる）

```json
{
  "Sid": "DenyAllExceptListedIfNoMFA",
  "Effect": "Deny",
  "NotAction": [
    "iam:CreateVirtualMFADevice",
    "iam:EnableMFADevice",
    "iam:GetUser",
    "iam:GetMFADevice",
    "iam:ListMFADevices",
    "iam:ListVirtualMFADevices",
    "iam:ResyncMFADevice",
    "sts:GetSessionToken"
  ],
  "Resource": "*",
  "Condition": {"BoolIfExists": {"aws:MultiFactorAuthPresent": "false"}}
}
```

- 公式の例には、この拒否に加えて「自分の MFA とパスワードを管理する」許可の部分がある。両方をそのまま使う
- 公式の例では、MFA を登録する前のパスワード変更（`iam:ChangePassword`）も拒否される
  - 初回サインイン時のパスワード変更を強制すると、MFA を登録する前に変更できず先へ進めない
  - 対処は 2 つ。初回のパスワード変更を強制しないか、`iam:ChangePassword` と `iam:GetAccountPasswordPolicy` を `NotAction` に足す（公式は後者を推奨していない）

### 4.3 ビジネス部門の AWS 操作を制限する

- ビジネス部門用グループには、次の 2 つだけを許可し、それ以外は拒否する
  - Quick Sight の自動登録（`quicksight:CreateReader`。自分のユーザーに限る。4.4）
  - 自分のパスワード変更と MFA の登録（4.2 の公式例の Allow 部分）
- IAM で `quicksight:*` を許可しない
  - 利用者向けの画面の動きは Quick Sight のロールと名前空間で決まるが、管理画面（Manage Quick）の操作は IAM の権限だけで決まる
  - IAM の Quick Sight 権限は、Quick Sight 内のロール（Reader など）とは別に、API の操作を許可する
  - たとえば `quicksight:RegisterUser` や `quicksight:UpdateUser` を持つと、画面では Reader でも、API から自分のロールを変えられるおそれがある（設計上の注意。公式の記載はない）
  - アクセスキーを発行しなくても、コンソールの CloudShell から API を呼べる。拒否の対象に CloudShell も含める
- 登録済みのユーザーが Quick Sight の画面を使うのに、IAM の権限がどこまで要るかは公式に記載がない（要検証）

### 4.4 Quick Sight ユーザーの自動登録（本件の方式）

- IAM ポリシーで次のアクションを許可しておくと、本人が初めて Quick Sight を開いた時点でユーザーが作られる
  - `quicksight:CreateReader`: Reader として作る（ビジネス部門、Reader の開発部門）
  - `quicksight:CreateUser`: Author として作る（Author にする開発部門）
  - `quicksight:CreateAdmin`: Admin として作る（管理者だけ）
- 複数持っている場合は、いちばん上のロールで作られる。開発部門用グループに `CreateUser` を付けると、全員が Author（$24）になる
  - Author にする人だけを別のグループにするか、ユーザーに直接付ける
- 作られた時点で課金が始まる
- 本人が初回にメールアドレスを入力する。管理者は値を制御できない

自動登録用のポリシーの例（ビジネス部門用）:

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

### 4.5 管理者が登録する方法（参考）

コンソール:

1. Manage Quick → Manage users → Invite users
2. User name に IAM ユーザー名、Email、Role を入れ、IAM User を Yes にする
3. 招待メールが届く。本人は 7 日以内に承諾し、IAM のパスワードで認証する

- メールアドレスは大文字と小文字を区別する。配布リストのアドレスは使えない

CLI（招待メールは送られない。登録した時点で課金が始まる）:

```bash
aws quicksight register-user --region ap-northeast-1 --aws-account-id 111122223333 --namespace default \
  --identity-type IAM --iam-arn arn:aws:iam::111122223333:user/biz-yamada \
  --email biz-yamada@example.com --user-role READER
```

### 4.6 IAM ロールで使う場合（参考）

- Quick Sight のユーザー名は「ロール名/セッション名」になる（例: `QuickSightReader/biz-yamada`）
- 同じロールでも、セッション名が違えば別のユーザーとして扱われる
- 登録は `register-user` の `--iam-arn` にロールの ARN、`--session-name` にセッション名を指定する
- コンソールでロールを切り替えたときに、セッション名が IAM ユーザー名になるかは明記がない（要検証）

## 5. ユーザーロール

| ロール（API の値） | できること                                                     | 月額 |
| ------------------ | -------------------------------------------------------------- | ---- |
| Reader（`READER`） | 共有されたダッシュボードの閲覧と操作（フィルター、入力）       | $3   |
| Author（`AUTHOR`） | データソース、データセット、分析、ダッシュボードの作成         | $24  |
| Admin（`ADMIN`）   | Author の操作に加え、ユーザー管理、SPICE の購入、アカウント設定 | $24  |
| Pro 系             | 上記に生成 AI の機能を加えたもの                               | 本件では使わない |

- 本件の割り当て
  - ビジネス部門: Reader（SPICE のダッシュボードだけを見る）
  - 開発部門: Reader（直接クエリのダッシュボードも見る）、または Author（データセットや画面を作る。試算では 1 名）
- ロールの変更: Manage users の画面、または `aws quicksight update-user --role AUTHOR`

## 6. グループ

```bash
aws quicksight create-group --region ap-northeast-1 --aws-account-id 111122223333 --namespace default \
  --group-name dev-members
aws quicksight create-group-membership --region ap-northeast-1 --aws-account-id 111122223333 --namespace default \
  --group-name dev-members --member-name dev-suzuki
```

- コンソール: Manage Quick → Manage groups → NEW GROUP（グループの削除は CLI でしかできない）
- 資産の共有はグループ単位にする。1 つの資産に共有できる相手は 100 までで、グループは 1 と数える
- 部門の分け方は、グループとフォルダで行う。名前空間（namespace）では分けられない
  - IAM ユーザーは既定の名前空間（`default`）にしか入れられない。既定以外の名前空間は、IAM フェデレーションの SSO ユーザー向け
  - 名前空間を分けると、利用者の画面では別の名前空間のユーザーが見えず、共有の操作もできない

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

ダッシュボードをビジネス部門グループに共有する CLI:

```bash
aws quicksight update-dashboard-permissions --region ap-northeast-1 --aws-account-id 111122223333 \
  --dashboard-id <DASHBOARD_ID> --grant-permissions \
  'Principal=arn:aws:quicksight:ap-northeast-1:111122223333:group/default/business-readers,Actions=quicksight:DescribeDashboard,quicksight:QueryDashboard,quicksight:ListDashboardVersions'
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
| Reader ロール | なし（ダウンロードは制限しないと決定）                             | -                                    |
| Author ロール | `CreateAndUpdateDataSources`、`ShareDataSources`、`ShareDashboards` | 直接クエリの資産がビジネス部門へ渡るのを防ぐ |
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

| 資産                                   | `business-readers`       | `dev-members`          | 管理者 |
| -------------------------------------- | ------------------ | ---------------------- | ------ |
| データソース（SPICE 用）               | 共有しない         | 利用                   | 所有者 |
| データソース（直接クエリ用）           | 共有しない         | 利用                   | 所有者 |
| SPICE のデータセット                   | 共有しない         | User または Owner      | Owner  |
| 直接クエリのデータセット               | 共有しない         | User または Owner      | Owner  |
| ビジネス部門用ダッシュボード（SPICE）           | Viewer             | Viewer                 | 所有者 |
| 開発部門用ダッシュボード（直接クエリ） | 共有しない         | Viewer または Co-owner | 所有者 |

- 注意: 直接クエリのダッシュボードは、閲覧者の操作でもデータソースの認証情報を使って DB に問い合わせる
  - ビジネス部門に誤って共有すると、実質的にビジネス部門に直接クエリを開放したことになる
  - 直接クエリのダッシュボードは「全員に共有」やリンク共有を使わない
- 開発部門の Reader も、`dev-members` に入れておけば直接クエリのダッシュボードを見られる（Reader は作成はできない）
- フォルダを「ビジネス部門公開用」と「開発部門用」に分け、フォルダ単位で共有すると誤りが減る
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
- この管理者用ポリシーは、管理者にだけ付ける。ビジネス部門や開発部門に `quicksight:*` を付けない（4.3）
- 利用者側のポリシーは 4 章を参照（MFA の必須化は 4.2、ビジネス部門の制限は 4.3、自動登録は 4.4）

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
| MFA                          | IAM 側で必須にする（4.2）                                                |
| 削除保護                     | Account termination protection をオンにしておく                         |
| SCP                          | 別アカウントでの勝手な契約や、エディション・認証方式を縛れる             |
| Control Tower のガードレール | 既存の SCP が Quick Sight の操作やリージョンを拒否していないか確かめる   |

- IP 制限の場所: Manage Quick → Security & permissions → IP and VPC endpoint restrictions
  - 反映まで最大 10 分。全リージョンに効く。自分の IP を含むルールがないと保存できない
  - 対象は画面（Web）、埋め込み、モバイル。API には効かないので、API は IAM の `aws:SourceIp` で絞る
- コンソールのセッションの長さ
  - IAM ユーザーのセッションは既定で 12 時間。IAM ユーザー向けにこれを変える設定は、公式に記載がない
  - 短くしたい場合は、IAM ロールに切り替えて使う運用にする。ロールの最大セッション時間（1〜12 時間）で上限を決められる
  - Quick Sight 独自のセッション時間の設定があるかは、公式に記載がない（要検証）

## 13. 行レベル・列レベルのセキュリティ

- 個人情報の列（電話番号とそのハッシュなど）は、全利用者・全サービスに一律で見せない
  - DB ユーザーの権限、公開用ビューからの除外、列レベルセキュリティ（CLS）の 3 段階で守る
- 行レベルセキュリティ（RLS）は、部門などで参照範囲を絞る場合に使う。ルールのない利用者には何も見えなくなるので、全員分のルールを先に用意する
- 詳細は「データセット」ページの 1 章と 7 章

## 14. 出典

確認日は 2026-10-20 または 2026-10-21。「第三者情報」「コミュニティ情報」と書いたもの以外は AWS 公式。

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
- 4 章 IAM ユーザーと Quick Sight ユーザー
  - MFA 必須化のポリシー例: https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_examples_aws_my-sec-creds-self-manage.html
  - 自分の MFA を管理させるポリシー例: https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_examples_iam_mfa-selfmanage.html
  - パスワードポリシー: https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_passwords_account-policy.html
  - アカウントエイリアス: https://docs.aws.amazon.com/IAM/latest/UserGuide/console_account-alias.html
  - 利用者の画面と管理画面の権限の違い: https://docs.aws.amazon.com/quick/latest/userguide/managing-user-access-qs-iam.html
  - 出典なし（設計上の注意）: IAM の `quicksight:*` で API からロールを変えられるおそれ、CloudShell の拒否
  - 出典なし（チームの方針）: ビジネス部門は新規の IAM ユーザー、開発部門は既存の IAM ユーザー、アクセスキーを発行しない
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
  - 名前空間（IAM ユーザーは既定の名前空間だけ）: https://docs.aws.amazon.com/quick/latest/userguide/namespaces.html
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
  - ロールの最大セッション時間: https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_update-role-settings.html
  - 出典なし（チームの方針）: Control Tower のガードレールとの整合を確かめること
  - IP・VPC エンドポイント制限: https://docs.aws.amazon.com/quick/latest/userguide/manage-ip-vpc.html
  - VPC エンドポイント: https://docs.aws.amazon.com/quicksight/latest/developerguide/vpc-interface-endpoints.html
  - SCP と条件キー: https://docs.aws.amazon.com/quicksight/latest/user/security-scp.html
  - 削除保護: https://docs.aws.amazon.com/quicksight/latest/user/closing-account.html
  - IAM のセッション時間: https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_use_switch-role-console.html
  - 出典なし（一般的な IAM の設定）: MFA を必須にする条件キー `aws:MultiFactorAuthPresent`
- 13 章 行レベル・列レベルのセキュリティ
  - CLS の規則には見せる相手が最低 1 つ要る: https://docs.aws.amazon.com/quicksight/latest/APIReference/API_ColumnLevelPermissionRule.html
  - RLS: https://docs.aws.amazon.com/quicksight/latest/user/restrict-access-to-a-data-set-using-row-level-security.html
  - CLS: https://docs.aws.amazon.com/quicksuite/latest/userguide/restrict-access-to-a-data-set-using-column-level-security.html
