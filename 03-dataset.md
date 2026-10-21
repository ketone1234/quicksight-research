# データセット

## 1. 結論

- DB の各テーブルに「公開用ビュー」を作り、それを入力にしてデータセットを作る
  - 公開用ビューには、個人情報の列（電話番号とそのハッシュなど）を含めない
- データセットは用途で 2 種類に分ける
  - 問い合わせ用: アカウント識別子で絞り込んで明細を見る
  - 分析用: サービス横断で推移や件数を集計する
- 同じビューから 2 つのデータセットを作る
  - SPICE 版（ビジネス部門向け）: SPICE 用データソース経由
  - 直接クエリ版（開発部門向け）: 直接クエリ用データソース経由
- アカウント識別子の列は、全データセットで同じ列名・文字列型にそろえる（例: `account_id`）
- 個人情報の列は 3 段階で守る
  - DB: Quick Sight 用の DB ユーザーには、公開用ビューにだけ SELECT を付ける（「ビュー」ページの 5 章）
  - ビュー: 公開用ビューから列を除く
  - Quick Sight: 列レベルセキュリティ（CLS）でも禁止する（7 章）
- 列の絞り込み、縦持ちへの変換、サービス横断の統合は DB ビュー側で行う。Quick Sight 側の加工は最小限にする

## 2. データセットが持つ設定

| 設定           | 内容                                                         |
| -------------- | ------------------------------------------------------------ |
| 入力           | テーブル、DB ビュー、カスタム SQL のどれか                   |
| データ準備     | 結合、計算フィールド、列名の変更、列の除外、型の変更、フィルター |
| クエリモード   | SPICE または直接クエリ（1 データセットに 1 つ）              |
| 権限           | 誰が使えるか（User / Owner）                                 |
| 行・列の制限   | 行レベルセキュリティ（RLS）、列レベルセキュリティ（CLS）     |
| 更新設定       | SPICE の取り込みスケジュール（SPICE のときだけ）             |

## 3. 作成手順（コンソール）

1. Data → Create → New dataset
2. 作成済みのデータソースを選ぶ（または新しく作る。手順は「接続方式」ページ）
3. Schema を選び、テーブル（DB ビュー）を選ぶ。SQL を書く場合は Use custom SQL
4. クエリモードを選ぶ
   - Import to SPICE for quicker analytics（SPICE）
   - Directly query your data（直接クエリ）
5. Edit/Preview data でデータ準備の画面に入り、必要なら加工して保存する

クエリモードを選べる場所は 3 つある。

- 作成直後の画面（上の手順 4）
- データ準備画面の Query mode
- API の `ImportMode`（`SPICE` / `DIRECT_QUERY`）

## 4. 入力方式の使い分け

| 方式                     | 長所                                              | 短所                                               | 本件     |
| ------------------------ | ------------------------------------------------- | -------------------------------------------------- | -------- |
| テーブル / DB ビュー直指定 | 直接クエリで、画面のフィルター値が WHERE に入る   | 列の追加や変更は DB 側の作業になる                 | 採用     |
| Quick Sight 側の結合     | DB を変えずに表を組み合わせられる                 | 直接クエリで結合できるのは同じデータソース内だけ   | 小規模のみ |
| カスタム SQL             | 自由に書ける                                      | 画面のフィルターは外側にかかり、内側は全件取得になり得る | 必要時のみ |

- DB ビューは、データセットの作成画面でテーブルと同じように扱える（ビュー専用の手順は公式にない）
- DB ビューの定義を変えたら、データセットを開いて保存し直す（そうしないと SPICE の取り込みが失敗する）

## 5. データ準備

### 5.1 新しいデータ準備画面（2025-10 以降の既定）

- 処理を「ステップ」で積み上げる方式。ステップは次のとおり
  - Input、Add Calculated Columns、Change Data Type、Rename Columns、Select Columns、Filter、Join
  - Append、Aggregate、Pivot、Unpivot
- 直接クエリで使えるのは Input、計算列、型変更、列名変更、列選択、Filter、Join まで
- Append、Aggregate、Pivot、Unpivot は SPICE 専用
- 新画面はデータセットパラメータに対応していない。使うには保存前に Switch to legacy experience で旧画面へ切り替える
- 新画面で保存したデータセットは、旧画面に戻せない

### 5.2 結合

- 種類: Inner、Left、Right、Full outer。1 データセットで最大 32 テーブル
- 直接クエリで結合できるのは「同じデータソース」の表どうしだけ（同じ DB を指す別のデータソースは対象外）
- 異なるデータソースの表を結合するには SPICE が要る（結合する側の合計は 20GB まで）
- 直接クエリでは Unique key を設定すると、表示に必要な表だけを結合するので速くなる

### 5.3 計算フィールドと列の操作

- データセットで作った計算フィールドは、そのデータセットを使う全ての分析で使える
- 集計関数を含む計算は、分析側にしか作れない
- 列名の変更、列の除外、型の変更は、新画面ではそれぞれのステップで行う

### 5.4 データセットのフィルター

- 文字列は Equals、Starts with、Contains、Is in などで絞れる。複数条件は AND
- プレビューに反映されるのは先頭 1,000 行だけ

### 5.5 縦持ちへの変換とサービス横断の統合

- どちらも DB ビューで行う（SQL の例は「ビュー」ページの 4.2 と 4.3）
- Quick Sight の Unpivot（縦持ちへの変換）や Append（複数の表の統合）のステップでもできるが、SPICE 専用

## 6. データセットパラメータ（直接クエリでカスタム SQL を使う場合）

画面で入力した値を、カスタム SQL の中に直接埋め込む機能。内側のクエリで絞り込めるので、DB の負荷を減らせる。

| 項目           | 内容                                                                 |
| -------------- | -------------------------------------------------------------------- |
| 使える条件     | 直接クエリのデータセットだけ。SPICE のカスタム SQL には入れられない  |
| 作成場所       | データセットの編集画面 → Parameters → +                              |
| 型             | String / Integer / Number / Datetime（作成後は変えられない）         |
| SQL への埋め込み | `WHERE account_id = <<$AccountId>>`。複数値は `IN (<<$AccountIds>>)` |
| 分析との接続   | 分析の Parameters で、分析パラメータとデータセットパラメータを 1 対 1 で対応付ける（Map Parameter） |
| 制約           | 新しいデータ準備画面では使えない（5.1 参照）                         |

```sql
SELECT c.account_id, c.plan_name, c.status, c.updated_at
FROM inquiry.v_inq_contract c
WHERE c.account_id = <<$AccountId>>
```

## 7. 行レベル・列レベルのセキュリティ

| 機能 | 内容                                                                    | 本件                                       |
| ---- | ----------------------------------------------------------------------- | ------------------------------------------ |
| CLS  | 指定した列を、許可したユーザー / グループにだけ見せる                   | 個人情報の列を全員に禁止する（二重の防御） |
| RLS  | ルール用のデータセットで「どのユーザー / グループがどの行を見られるか」を決める | 参照範囲を部門などで絞る場合に使う |

- どちらも Enterprise 限定

CLS の注意:

- 公開用ビューから除いた列は、データセットにそもそも現れない。CLS をかける対象がないので、そのデータセットでは設定不要
- CLS が効くのは、その列を含むデータセット（元のテーブルを入力にしたものなど）。CLS はその漏れを防ぐ二重の防御になる
- DB ユーザーに元のテーブルの権限を与えなければ、そうしたデータセットは作れない。DB の権限を一番の防御にする
- CLS では「誰にも見せない」設定を直接は作れない。1 つの規則に、見せる相手（ユーザーかグループ）が最低 1 つ要る
  - 全員に禁止したい場合は、メンバーのいないグループ（または管理者だけのグループ）を見せる相手にする（要検証）
  - データセットの所有者に CLS が効くかは公式に記載がない（要検証）
- 制限した列を含むビジュアルは表示されない。表とピボットでは値が「Not Authorized」と表示される

RLS の注意:

- ルールのないユーザーには行が 1 件も見えない。RLS を入れる場合は、全利用者のルールを用意してから有効にする
- データセットの所有者は RLS の影響を受けない
- IAM ユーザーをルールに書くときの形式（ユーザー名か ARN か）は実機で確かめる（要検証）
- タグベースの RLS は匿名の埋め込み専用で、IAM のユーザーには使えない

## 8. データセットの権限

| 権限  | できること                                         |
| ----- | -------------------------------------------------- |
| User  | そのデータセットで分析や派生データセットを作る     |
| Owner | User に加えて、編集、更新、削除、他者への共有      |

- 付与先はユーザーかグループ。グループで付与すると管理が楽になる
- ダッシュボードを見るだけのビジネス部門には、データセットの権限は要らないと読める（明記はない。要検証）

## 9. 主な制限値

| 区分             | 制限                                                                  |
| ---------------- | --------------------------------------------------------------------- |
| SPICE（Enterprise）| 1 データセットあたり 20 億行または 2TB                              |
| SPICE（Standard） | 1 データセットあたり 2,500 万行または 25GB                           |
| 列               | 2,000 列。列名 127 文字                                               |
| 文字列           | 1 フィールド 2,047 文字（新しいデータ準備画面では 65,534 文字）      |
| 結合             | 32 テーブル                                                           |
| 新しいデータ準備 | 256 ステップ、データセットの入れ子 10 階層                            |
| カスタム SQL     | API では 168,000 文字まで                                             |

- 料金ページには「10 億行 / 1TB」という古い値が残っている。ユーザーガイド（20 億行 / 2TB）を採用した
- ID のように桁数の多い数値は、19 桁を超えると取り込みで行が落ちる。文字列型にしておく

## 10. API とコードでの管理

### 10.1 データセットを API で作る最小例

```json
{
  "DataSetId": "inq-contract-spice",
  "Name": "inq-contract (SPICE)",
  "ImportMode": "SPICE",
  "PhysicalTableMap": {"t1": {"RelationalTable": {
    "DataSourceArn": "arn:aws:quicksight:ap-northeast-1:111122223333:datasource/prod-db-spice",
    "Schema": "inquiry", "Name": "v_inq_contract",
    "InputColumns": [
      {"Name": "account_id", "Type": "STRING"},
      {"Name": "status", "Type": "STRING"}
    ]}}}
}
```

```bash
aws quicksight create-data-set --region ap-northeast-1 --aws-account-id 111122223333 --cli-input-json file://dataset.json
```

- この形式（`PhysicalTableMap` だけを使う従来形式）で作ると、旧画面形式のデータセットになる

### 10.2 環境間の移行とコード管理

- アセットバンドル: ダッシュボード、分析、データセット、データソースなどをまとめて書き出し、別環境に読み込める
  - 書き出し: `start-asset-bundle-export-job`（形式は `QUICKSIGHT_JSON` か `CLOUDFORMATION_JSON`）
  - 読み込み: `start-asset-bundle-import-job`。データソースの認証情報は上書き指定が必須。権限は引き継がれない
  - SPICE の中身は含まれない
- CloudFormation: `AWS::QuickSight::DataSource`、`DataSet`、`Dashboard`、`Analysis`、`RefreshSchedule`、`VPCConnection` などがある
- 検証環境で作った画面を本番に移す運用にするなら、アセットバンドルを使う

## 11. 出典

確認日は 2026-10-20 または 2026-10-21。「第三者情報」「コミュニティ情報」と書いたもの以外は AWS 公式。

- 3 章 作成手順
  - データベースからのデータセット作成: https://docs.aws.amazon.com/quick/latest/userguide/create-a-database-data-set.html
  - カスタム SQL: https://docs.aws.amazon.com/quick/latest/userguide/adding-a-SQL-query.html
  - CreateDataSet の API（ImportMode）: https://docs.aws.amazon.com/quicksight/latest/APIReference/API_CreateDataSet.html
- 4 章 入力方式の使い分け
  - 結合（テーブル、ビュー、SQL の扱い）: https://docs.aws.amazon.com/quick/latest/userguide/joining-data.html
  - カスタム SQL とフィルターの関係（AWS ブログ）: https://aws.amazon.com/blogs/business-intelligence/optimize-queries-using-dataset-parameters-in-amazon-quicksight/
  - スキーマ変更時の取り込み失敗: https://docs.aws.amazon.com/quick/latest/userguide/refreshing-imported-data.html
- 5 章 データ準備
  - データ準備のステップ: https://docs.aws.amazon.com/quick/latest/userguide/data-prep-steps.html
  - SPICE 専用の機能: https://docs.aws.amazon.com/quick/latest/userguide/spice-only-features.html
  - 新しい画面の未対応機能: https://docs.aws.amazon.com/quick/latest/userguide/unsupported-features.html
  - 新旧画面の切り替え: https://docs.aws.amazon.com/quick/latest/userguide/switching-between-data-prep-experiences.html
  - 結合: https://docs.aws.amazon.com/quick/latest/userguide/joining-data.html
  - Unique key と動的クエリ: https://aws.amazon.com/about-aws/whats-new/2020/12/amazon-quicksight-enhances-dashboard-performance-with-dynamic-querying-and-other-dataset-optimizations
  - 計算フィールド: https://docs.aws.amazon.com/quick/latest/userguide/adding-a-calculated-field-analysis.html
  - データセットのフィルター: https://docs.aws.amazon.com/quick/latest/userguide/add-a-filter-data-prep.html
- 6 章 データセットパラメータ
  - 概要: https://docs.aws.amazon.com/quick/latest/userguide/dataset-parameters.html
  - 作成: https://docs.aws.amazon.com/quick/latest/userguide/dataset-parameters-insert-parameter.html
  - カスタム SQL での利用: https://docs.aws.amazon.com/quick/latest/userguide/dataset-parameters-SQL.html
  - 分析パラメータとの対応付け: https://docs.aws.amazon.com/quick/latest/userguide/dataset-parameters-analysis.html
- 7 章 行レベル・列レベルのセキュリティ
  - CLS の規則には見せる相手が最低 1 つ要る: https://docs.aws.amazon.com/quicksight/latest/APIReference/API_ColumnLevelPermissionRule.html
  - 出典なし（チームの方針）: 個人情報の列を 3 段階で守ること
  - RLS: https://docs.aws.amazon.com/quick/latest/userguide/restrict-access-to-a-data-set-using-row-level-security.html
  - タグベースの RLS: https://docs.aws.amazon.com/quick/latest/userguide/quicksight-dev-rls-tags.html
  - CLS: https://docs.aws.amazon.com/quick/latest/userguide/restrict-access-to-a-data-set-using-column-level-security.html
- 8 章 データセットの権限
  - データセットの共有: https://docs.aws.amazon.com/quick/latest/userguide/sharing-data-sets.html
  - ダッシュボード共有時のデータの見え方: https://docs.aws.amazon.com/quick/latest/userguide/sharing-a-dashboard.html
- 9 章 主な制限値
  - データソースの制限値: https://docs.aws.amazon.com/quick/latest/userguide/data-source-limits.html
  - データ準備の制限値: https://docs.aws.amazon.com/quick/latest/userguide/data-preparation-limits.html
  - SPICE の上限拡大（2026-01）: https://aws.amazon.com/about-aws/whats-new/2026/01/amazon-quick-suite-launches-expanded-spice
  - 料金ページに残る古い上限値: https://aws.amazon.com/quicksight/pricing/
  - 19 桁を超える数値の取り込みエラー: https://docs.aws.amazon.com/quicksight/latest/user/errors-spice-ingestion.html
  - カスタム SQL の文字数: https://docs.aws.amazon.com/quicksight/latest/APIReference/API_CustomSql.html
- 10 章 API とコードでの管理
  - CreateDataSet: https://docs.aws.amazon.com/quicksight/latest/APIReference/API_CreateDataSet.html
  - 従来形式で作ると旧画面形式になる: https://docs.aws.amazon.com/quick/latest/userguide/new-data-prep-faqs.html
  - アセットバンドルの書き出し: https://docs.aws.amazon.com/quicksight/latest/developerguide/assetbundle-export.html
  - アセットバンドルの読み込み: https://docs.aws.amazon.com/quicksight/latest/developerguide/assetbundle-import.html
  - CloudFormation のリソース: https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/AWS_QuickSight.html
