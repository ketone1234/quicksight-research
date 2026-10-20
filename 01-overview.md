# 概要と推奨構成

## 1. 背景と要件

- 現状: 運用者から DB の情報を問い合わせられてから回答するまでに数日かかる
- 目標: 緊急時などに、PO が自分で DB の情報を参照できるようにする
- 問い合わせは種類ごとに参照するテーブルが違う。受け取るキーは、ほぼ共通の「アカウント識別子」

| #  | 要件                                                | 実現方法（概要）                                                     | 詳細          |
| -- | --------------------------------------------------- | -------------------------------------------------------------------- | ------------- |
| R1 | アカウント管理は IAM。AWS にログインしてから使う    | 認証方式を IAM 対応のものにし、IAM ユーザーを Quick Sight に登録する | 権限          |
| R2 | 利用証跡を残す                                      | CloudTrail の証跡と DB の監査ログを組み合わせる                      | 利用証跡      |
| R3 | DB を SPICE に取り込んで分析する                    | SPICE データセットを 1 日 1 回、スケジュールで更新する               | SPICE         |
| R4 | DB 直接クエリも使えるようにし、開発部門だけに限る   | 直接クエリ用の資産を開発グループだけに共有する                       | 権限          |

## 2. 最初に知っておくべき制約

- 認証方式とアカウント名は、作成後に変えられない。変えるにはアカウントを削除して作り直す（全資産が消える）
- Quick Sight は、ダッシュボードで入力した検索値（アカウント識別子）を記録しない。補い方は「利用証跡」ページ
- SPICE のデータは最後に取り込んだ時点のもの。取り込みは 1 日 1 回の想定なので、最大で約 1 日前のデータになる
  - 最新の値が必要なときは、開発部門が直接クエリで確かめる
- 直接クエリはビジュアル 1 つあたり 120 秒でタイムアウトする
- 2025-10 以降の新しいデータ準備画面は、データセットパラメータに対応していない。使うには旧画面への切り替えが要る
- VPC 接続は Quick Sight と同じリージョンの VPC にしか作れない

## 3. 製品名の整理

- 2025-10: Amazon QuickSight は「Amazon Quick Suite」に変わり、BI 機能は「Amazon Quick Sight」になった
- 2026 年: 製品群の名前が「Amazon Quick」に短縮された（公式の改称告知は見つからず、ドキュメントの表記で確認）
- 変わらないもの: API と CLI（`aws quicksight`）、IAM アクション（`quicksight:`）、CloudTrail のイベントソース
- コンソールの管理メニューは「Manage Quick」。古い資料では「Manage QuickSight」
- 公式ドキュメントは `/quick/`、`/quicksuite/`、`/quicksight/` の 3 系統が混在している。検索では 3 つの名前を試す
- この資料では「Quick Sight」と書く

## 4. 用語

| 用語          | 意味                                                                         |
| ------------- | ---------------------------------------------------------------------------- |
| データソース  | DB への接続定義。エンドポイント、認証情報、VPC 接続を持つ                    |
| データセット  | データソースから取り出す表と加工の定義。クエリモード（SPICE / 直接クエリ）を持つ |
| SPICE         | Quick Sight 内のインメモリ領域。取り込んだ時点のデータ（スナップショット）を持つ |
| 直接クエリ    | 画面を表示するたびに DB へ SQL を発行するモード（Direct Query）              |
| 分析          | ビジュアルを組み立てる編集画面。Author 以上が使う                            |
| ダッシュボード| 分析を公開した閲覧用の画面。Reader 以上が使う                                |
| ビジュアル    | 表、ピボット、グラフなどの部品                                               |
| パラメータ    | 画面の入力値を入れておく変数。フィルターから参照する                         |
| コントロール  | パラメータに値を入れるための入力欄（テキスト入力、ドロップダウンなど）       |
| VPC 接続      | Quick Sight が自社 VPC 内に ENI を作り、非公開の DB へ接続する仕組み         |
| ENI           | ネットワークインターフェース（Elastic Network Interface）。VPC 内の通信の出入口 |
| SG            | セキュリティグループ。ENI や DB に付ける通信許可のルール                     |

## 5. 全体構成（推奨）

```text
[PO / 開発者]
   | (1) AWS マネジメントコンソールに個人の IAM ユーザーでサインイン（MFA 必須）
   v
[Quick Sight  東京リージョン / Enterprise]
   |  PO   : Reader。SPICE のダッシュボードを閲覧するだけ
   |  開発 : Author。データセットの作成と直接クエリ
   | (2) VPC 接続（Quick Sight の ENI がプライベートサブネットに置かれる）
   v
[VPC プライベートサブネット]
   RDS / Aurora のリードレプリカ
   DB ユーザー: qs_spice_ro（SPICE 取り込み用） / qs_direct_ro（直接クエリ用）

[利用証跡]
   CloudTrail 証跡 -> S3（整合性検証あり） -> Athena で検索
   DB 監査ログ     -> CloudWatch Logs
```

## 6. データの流れ

データは、利用者と処理のタイミングによって 3 つの経路を通る。SPICE の取り込みは 1 日 1 回を想定する。

```text
[閲覧時: PO]  DB には問い合わせない
利用者 → ダッシュボード → データセット（SPICE モード） → SPICE に保存済みのデータ

[取り込み時]  1 日 1 回、スケジュールで実行（例: 毎日 3:00）
DB のテーブル → DB のビュー → データソース（接続定義・VPC 接続） → データセットの定義 → SPICE に保存

[直接クエリ: 開発部門だけ]  表示のたびに DB へ問い合わせる
利用者 → ダッシュボード → データセット（直接クエリ） → データソース → DB のビュー → DB のテーブル
```

| 層             | 役割                                                                 | 詳細ページ   |
| -------------- | -------------------------------------------------------------------- | ------------ |
| ダッシュボード | 利用者が見る画面。アカウント識別子を入れて絞り込む                   | ビュー       |
| データセット   | どのビューのどの列を使うかの定義。SPICE か直接クエリかのモードを持つ | データセット |
| SPICE          | SPICE モードのデータセットのデータを入れておく場所                   | SPICE        |
| データソース   | DB への接続定義（エンドポイント、認証情報、VPC 接続）                | 接続方式     |
| DB のビュー    | 問い合わせ種別ごとに、回答に要る列をまとめたもの                     | ビュー       |
| DB のテーブル  | 元のデータ                                                           | -            |

- 「利用者 → ダッシュボード → SPICE → データセット → ビュー → DB」と 1 本の流れで書くこともある。その場合は次の 3 点に注意する
  - SPICE はデータセットの次の層ではなく、データセットのデータの保存先。ダッシュボードはデータセットを参照し、そのデータが SPICE から読まれる
  - 1 本の流れには、閲覧時と取り込み時の 2 つのタイミングが混ざっている。閲覧時は SPICE で止まり、DB には届かない
  - 開発部門の直接クエリは SPICE を通らない。直接クエリを開発部門に限る制御は、データソースの共有範囲で行う
- 1 日 1 回の取り込みで生じること
  - 画面のデータは最大で約 1 日前のもの。当日の値が要るときは、開発部門が直接クエリで確かめる
  - PO の閲覧は DB に問い合わせないので、DB の監査ログには残らない（DB に残るのは 1 日 1 回の取り込みの SQL だけ）
  - 取り込みに失敗すると、前日のデータが表示され続ける。失敗の通知を必ず設定する（「SPICE」ページの 5.5 と 7 章）

## 7. 推奨構成の要点

| 観点               | 推奨                                                         | 理由                                                                 |
| ------------------ | ------------------------------------------------------------ | -------------------------------------------------------------------- |
| エディション       | Enterprise                                                   | VPC 接続、Reader ロール、カスタム権限が Enterprise 限定              |
| リージョン         | 東京（ap-northeast-1）                                       | DB と同じリージョンにする。同梱の SPICE 容量は最初のリージョンに付く |
| 認証方式           | (Recommended) Password-based or Single-Sign On               | IAM ユーザーでコンソールから入れる。詳細は「権限」ページ             |
| ユーザーロール     | PO は Reader、開発は Author、管理者は Admin。Pro 系は使わない | Pro 系が 1 人でもいると、アカウント料金 $250/月 がかかる             |
| 接続               | VPC 接続、リードレプリカ、読み取り専用 DB ユーザー、TLS      | DB をインターネットに公開しない                                      |
| データの持ち方     | PO 向けは SPICE（1 日 1 回取り込み）、開発向けは直接クエリ   | 直接クエリは画面操作のたびに DB に負荷がかかる                       |
| 画面               | 問い合わせ種別ごとにシートを分け、アカウント識別子は共通入力 | 1 回の入力で全種別を引ける                                           |
| 直接クエリの限定   | 共有範囲で制御する（専用の設定項目はない）                   | 直接クエリのデータソースと資産を PO に共有しない                     |
| 証跡               | CloudTrail（読み取りと書き込みの両方）と DB 監査ログ         | Quick Sight は画面で入力した検索値を記録しない                       |

## 8. 料金の概算（PO 5 名、開発 3 名の場合）

| ロール         | 月額単価 | 人数 | 小計 |
| -------------- | -------- | ---- | ---- |
| Reader         | $3       | 5    | $15  |
| Author / Admin | $24      | 3    | $72  |
| 合計           |          |      | $87  |

- SPICE: Author 1 人につき 10GB が付く（3 人で 30GB）。追加は $0.38/GB/月
- 次のどれかに当たると、アカウント料金 $250/月 が加わる
  - Pro 系のユーザー（Reader Pro、Author Pro、Admin Pro）が 1 人でもいる
  - Topics 経由の Q&A を有効にしている
  - ダッシュボードの Q&A を有効にしている
- 料金ページの脚注 1 か所だけは条件を「Pro ユーザーがいて、かつ Q&A が有効」と書いており、本文と食い違う。どちらも避ければ確実
- 注意: サインアップした人が自動で Admin Pro になったという報告がある（第三者情報）。作成直後にロールを確認する
- 課金はユーザーを作った時点で始まる。未ログインでも、月の途中で消しても、その月は満額になる
- 初回は 30 日間の無料試用がある（25 ユーザーまで）
- 別に料金がかかるもの: CloudTrail（2 つ目以降のコピー）、S3、Athena、CloudWatch Logs、KMS、Secrets Manager

## 9. 出典

確認日はいずれも 2026-10-20。「第三者情報」「コミュニティ情報」と書いたもの以外は AWS 公式。

- 2 章 最初に知っておくべき制約
  - 認証方式は作成後に変えられない: https://docs.aws.amazon.com/quicksuite/latest/userguide/setting-up-sso.html
  - アカウント削除で全資産が消える: https://docs.aws.amazon.com/quicksight/latest/user/closing-account.html
  - 検索値が記録されない（コミュニティ情報）: https://community.amazonquick.com/t/definition-of-a-eventname-in-cloudtrail-logs/22103
  - 直接クエリのタイムアウト: https://docs.aws.amazon.com/quick/latest/userguide/data-source-limits.html
  - 新しいデータ準備画面の未対応機能: https://docs.aws.amazon.com/quick/latest/userguide/unsupported-features.html
  - VPC 接続は同じリージョンだけ: https://docs.aws.amazon.com/quicksight/latest/user/vpc-finding-setup-information.html
- 3 章 製品名の整理
  - Quick Suite への移行（AWS ブログ 2025-10）: https://aws.amazon.com/blogs/business-intelligence/reimagine-business-intelligence-amazon-quicksight-evolves-to-amazon-quick-suite/
  - 現在の製品の説明: https://docs.aws.amazon.com/quick/latest/userguide/what-is.html
  - Amazon Quick への改称時期（第三者情報）: https://blog.serverworks.co.jp/amazon-quick-suite-renamed-to-amazon-quick
- 5〜7 章 全体構成、データの流れ、推奨構成
  - VPC 接続は Enterprise 限定: https://docs.aws.amazon.com/quicksight/latest/user/working-with-aws-vpc.html
  - カスタム権限は Enterprise 限定: https://docs.aws.amazon.com/quick/latest/userguide/custom-permissions.html
  - 閲覧時は SPICE から読み、DB には取り込み時だけ問い合わせる: https://docs.aws.amazon.com/quick/latest/userguide/spice.html
  - スケジュールでの取り込み: https://docs.aws.amazon.com/quick/latest/userguide/refreshing-imported-data.html
  - 取り込みに失敗しても古いデータを表示（AWS ブログ）: https://aws.amazon.com/blogs/business-intelligence/automate-spice-dataset-refreshes-using-amazon-quick-sight-apis/
  - SPICE 容量とリージョン: https://docs.aws.amazon.com/quicksight/latest/user/managing-spice-capacity.html
  - Reader ロール、データと同じリージョンの推奨（FAQ）: https://aws.amazon.com/quicksight/faqs/
  - サインアップの認証方式: https://docs.aws.amazon.com/quick/latest/userguide/signing-up.html
- 8 章 料金の概算
  - 料金: https://aws.amazon.com/quicksight/pricing/
  - 課金の開始と月途中の削除: https://docs.aws.amazon.com/quick/latest/userguide/understanding-quick-user-billing.html
  - 30 日の無料試用: https://docs.aws.amazon.com/quick/latest/userguide/signing-up.html
  - サインアップした人が Admin Pro になる事例（第三者情報）: https://dev.classmethod.jp/articles/quick-suite-subscribe/
  - CloudTrail の料金: https://aws.amazon.com/cloudtrail/pricing/
