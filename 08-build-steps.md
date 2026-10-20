# 構築手順チェックリスト

実装する順に並べた。各項目の詳細は括弧内のページを見る。

## 0. 事前に決めること

- [ ] 対象の DB（エンジン、バージョン、VPC、サブネット、リードレプリカの有無）
- [ ] 問い合わせの種別と、種別ごとに参照する表と列
- [ ] 利用者の一覧（PO、開発、管理者）
- [ ] データの新しさの要件（想定は SPICE の取り込み 1 日 1 回。画面のデータは最大で約 1 日前になる）
- [ ] CSV などのダウンロードを許すか
- [ ] 証跡と監査ログの保存期間
- [ ] 本番の前に試用アカウント（30 日無料）で検証するか

## 1. 証跡の基盤を先に作る（利用証跡）

- [ ] ログ保存用の S3 バケット（専用、暗号化、ライフサイクル）
- [ ] CloudTrail のマルチリージョン証跡（整合性検証あり、読み取り・書き込みの両方）
- [ ] Athena の CloudTrail テーブル

## 2. DB 側の準備（接続方式、ビュー）

- [ ] 問い合わせ種別ごとの DB ビュー（先頭列 `account_id`、文字列型、取り込み時刻の列）
- [ ] `account_id` のインデックス
- [ ] DB ユーザー `qs_spice_ro`、`qs_direct_ro`（ビューへの SELECT のみ、TLS 必須、文タイムアウト）
- [ ] DB の監査ログ（対象を上の 2 ユーザーに絞り、CloudWatch Logs へ出力、保存期間を設定）
- [ ] Secrets Manager に 2 つのシークレット（`username` と `password` の JSON）

## 3. Quick Sight アカウントの作成（権限）

- [ ] サインアップ専用の IAM ロールを用意する（AdministratorAccess を使わない）
- [ ] AWS コンソールで Quick を開き、サインアップする
  - リージョン: アジアパシフィック（東京）
  - 認証方式: (Recommended) Password-based or Single-Sign On
- [ ] 自分のロールが Admin Pro になっていないか確認し、なっていれば Admin に変える（$250/月 を避ける）
- [ ] SPICE の自動購入をオフにする
- [ ] Account termination protection がオンか確認する
- [ ] 生成 AI の機能を、カスタム権限のガバナンス設定で既定拒否にする

## 4. ネットワーク（接続方式）

- [ ] VPC 接続の実行ロール（信頼ポリシーと ec2 の 5 権限）
- [ ] Quick Sight ENI 用の SG と、DB の SG の inbound ルール
- [ ] VPC 接続を作り、状態が AVAILABLE になることを確認する

## 5. データソース（接続方式）

- [ ] Security & permissions で、Secrets Manager の 2 つのシークレットを許可する
- [ ] データソース `prod-db-spice` と `prod-db-direct` を CLI で作る（VPC 接続、シークレット、TLS）
- [ ] 両方とも Validate connection が通ることを確認する

## 6. ユーザーとグループ（権限）

- [ ] PO と開発者の個人 IAM ユーザー（MFA 必須）
- [ ] Quick Sight への登録（PO は READER、開発は AUTHOR）
- [ ] グループ `po-readers`、`dev-authors` を作り、メンバーを入れる
- [ ] カスタム権限: Author ロールの共有とデータソース作成を禁止。必要なら Reader ロールのダウンロードを禁止
- [ ] データソースの共有: `prod-db-spice` と `prod-db-direct` を `dev-authors` に（PO には共有しない）
- [ ] 必要なら IP 制限（社内の IP だけ許可）

## 7. データセット（データセット、SPICE）

- [ ] 種別ごとに SPICE のデータセットを作る（入力は DB ビュー）
- [ ] 種別ごとに直接クエリのデータセットを作る
- [ ] SPICE の更新スケジュール（1 日 1 回の全件更新。DB の負荷が低い時間帯）と、失敗時のメール通知
- [ ] 容量の使用量を確認する（見積もりとの比較）

## 8. 画面（ビュー）

- [ ] PO 用の分析: パラメータ `AccountId`、テキスト入力のコントロール、データセットごとのフィルター、種別ごとのシート
- [ ] 取り込み時刻の表示
- [ ] ダッシュボードとして公開（ダウンロードの可否を設定）
- [ ] PO 用ダッシュボードを `po-readers` に共有
- [ ] 開発用（直接クエリ）の分析とダッシュボードを作り、`dev-authors` にだけ共有
- [ ] フォルダを「PO 公開用」と「開発用」に分ける

## 9. 監視とアラート（SPICE、利用証跡）

- [ ] `IngestionErrorCount` のアラーム
- [ ] `SPICECapacityConsumedInMB` のアラーム
- [ ] EventBridge: データソース・データセットの作成、ダッシュボードの共有範囲の変更
- [ ] `ConsoleLogin` 失敗の検知

## 10. 受け入れテスト

| #  | 確認すること                                                     | 期待する結果                         |
| -- | ---------------------------------------------------------------- | ------------------------------------ |
| T1 | PO がコンソールにサインインし、PO 用ダッシュボードを開く          | アカウント識別子で各シートが絞り込まれる |
| T2 | PO がデータセットやデータソースの一覧を開く                      | 作成できない。直接クエリの資産が見えない |
| T3 | PO が開発用ダッシュボードの URL を直接開く                       | 権限エラーになる                     |
| T4 | 開発者が直接クエリのダッシュボードで検索する                     | 最新の値が出る。DB の監査ログに SQL が残る |
| T5 | 開発者がダッシュボードを PO に共有しようとする                   | カスタム権限で拒否される             |
| T6 | ダウンロードを禁止した場合、PO が CSV 出力を試す                 | メニューが出ない                     |
| T7 | CloudTrail を Athena で検索する                                  | `ConsoleLogin` と `GetDashboard` が個人名で残る |
| T8 | SPICE の取り込みをわざと失敗させる（DB ユーザーを一時停止など）  | メール通知と CloudWatch アラームが届く |
| T9 | 請求の画面で Quick Sight の料金を確認する                        | $250 のアカウント料金がかかっていない |

## 11. 運用

- 利用者の追加・削除の手順を決める（IAM ユーザー、Quick Sight の登録、グループ）
- 退職や異動のときは IAM ユーザーと Quick Sight ユーザーの両方を消す。月の途中で消しても、その月は満額
- ユーザーを消すと、その人だけが所有する資産も消える。消す前に所有者を移す
- 共有範囲と証跡を定期的に確認する（月 1 回など）

## 12. 出典

確認日はいずれも 2026-10-20。「第三者情報」「コミュニティ情報」と書いたもの以外は AWS 公式。

- 各手順の根拠は、括弧内に示したページの「出典」節にある。ここには、このページで初めて出てくる事実の出典だけを載せる
- 3 章 Quick Sight アカウントの作成
  - サインアップ手順: https://docs.aws.amazon.com/quick/latest/userguide/signing-up.html
  - サインアップに要る IAM 権限: https://docs.aws.amazon.com/quicksight/latest/user/iam-policy-examples.html
  - サインアップした人が Admin Pro になる事例（第三者情報）: https://dev.classmethod.jp/articles/quick-suite-subscribe/
  - SPICE の自動購入: https://docs.aws.amazon.com/quicksight/latest/user/managing-spice-capacity.html
  - 削除保護: https://docs.aws.amazon.com/quicksight/latest/user/closing-account.html
  - 生成 AI の一括拒否: https://docs.aws.amazon.com/quick/latest/userguide/custom-permissions-governance.html
- 11 章 運用
  - 月途中のユーザー削除と課金: https://docs.aws.amazon.com/quick/latest/userguide/understanding-quick-user-billing.html
  - ユーザー削除時の資産の扱い: https://docs.aws.amazon.com/quick/latest/userguide/managing-users.html
- 出典なし: 0 章の事前決定事項、10 章の受け入れテスト、11 章の定期確認は、この調査から出した提案
