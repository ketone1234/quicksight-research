# ビュー（DB ビュー）

Quick Sight が読む DB 側のビューをまとめる。画面（分析・ダッシュボード）は「ダッシュボード」ページにまとめた。

## 1. 結論

- Quick Sight からは DB のテーブルを直接読ませず、ビューだけを読ませる
- ビューは 3 種類を用意する
  - 公開用ビュー: テーブルごとに、個人情報の列（電話番号とそのハッシュなど）を除いたもの
  - 縦持ちのビュー: プランごとに列が分かれた表を、プランを行に展開したもの
  - サービス横断のビュー: 同じ構成のテーブルを、複数サービス分まとめたもの
- Quick Sight 用の DB ユーザーには、ビューにだけ SELECT を付ける
- ビューの作成と GRANT はプライマリで行う（リードレプリカに反映される）
- ビューの定義を変えたら、Quick Sight のデータセットを開いて保存し直す

## 2. ビューを使う理由

| 目的                         | ビューでできること                                                                 |
| ---------------------------- | ---------------------------------------------------------------------------------- |
| 個人情報の保護               | 列を除いたビューにだけ権限を与えれば、Quick Sight からは元の列を読めない           |
| 列名と型の統一               | アカウント識別子を同じ列名・文字列型にそろえ、全データセットで同じフィルターを使える |
| 集計しやすい形               | 縦持ちへの変換やサービス横断の統合を、SPICE でも直接クエリでも同じように使える     |
| Quick Sight 側の加工を減らす | 結合や変換を DB 側にまとめると、データセットの設定が単純になる                     |

- Quick Sight は、ビューをテーブルと同じように扱う（データセットの作成画面でテーブルの一覧に出る）
- Quick Sight の Unpivot や Append のステップでも同じ変換はできるが、SPICE 専用。ビューなら直接クエリでも使える

## 3. 設計の方針

- スキーマを Quick Sight 用に分ける（例: `inquiry`）。名前の例は、問い合わせ用が `v_inq_<種別>`、分析用が `v_<内容>`
- 先頭列を `account_id`（文字列型）にし、元テーブルの `account_id` 列にはインデックスを張る
- 回答や分析に要る列だけを選ぶ。個人情報の列は入れない
- 取り込み時刻の列（`extracted_at`）を入れる。SPICE のデータの古さを画面に出すのに使う
- 集計、DISTINCT、LIMIT を含むビューは、条件が内側まで届かず遅くなることがある。問い合わせ用は明細のビューにする
- 1 アカウントに大量の行がある種別は、期間で絞るか（例: 直近 90 日）を決めておく
- 桁数の多い ID は文字列型にする（SPICE は 19 桁を超える数値の行を取り込まない）

## 4. ビューの種類と SQL の例（PostgreSQL）

テーブル名と列名は例。

### 4.1 公開用ビュー（問い合わせ用）

```sql
CREATE VIEW inquiry.v_inq_contract AS
SELECT
  CAST(c.account_id AS VARCHAR(64)) AS account_id,
  c.plan_name,
  c.status,
  c.started_at,
  c.updated_at,
  CURRENT_TIMESTAMP AS extracted_at
FROM app.contracts c;
```

- 電話番号とそのハッシュなどの列は、SELECT に書かない
- `extracted_at` は SPICE に取り込んだ時刻になる（直接クエリでは表示した時刻になる）

### 4.2 縦持ちに変換するビュー

プラン（容量など）ごとにステータスの列が分かれている表（横持ち）を、プランを行に展開した形（縦持ち）にする。
プラン別の件数や推移のグラフが、1 つの列で作れるようになる。

```sql
CREATE VIEW inquiry.v_order_by_plan AS
SELECT o.account_id, p.plan_name, p.status, o.recorded_at
FROM app.orders o
CROSS JOIN LATERAL (VALUES
  ('plan_a', o.plan_a_status),
  ('plan_b', o.plan_b_status),
  ('plan_c', o.plan_c_status)
) AS p(plan_name, status);
```

### 4.3 サービス横断のビュー

複数のサービスが同じ構成のテーブルを持っている場合は、1 つにまとめてサービス名の列を足す。

```sql
CREATE VIEW inquiry.v_contract_all AS
SELECT 'service_a' AS service, account_id, status, updated_at FROM service_a.contracts
UNION ALL
SELECT 'service_b' AS service, account_id, status, updated_at FROM service_b.contracts;
```

- サービスによって結合キー（アカウント識別子、提携先側の利用者識別子など）が違う場合は、ビューの中で共通の列名にそろえる
- 画面では、サービス名の列でサービスを選べるようにする

## 5. DB ユーザーの権限

```sql
GRANT USAGE ON SCHEMA inquiry TO qs_spice_ro, qs_direct_ro;
GRANT SELECT ON ALL TABLES IN SCHEMA inquiry TO qs_spice_ro, qs_direct_ro;
```

- `ALL TABLES IN SCHEMA` にはビューも含まれる。Quick Sight 用のスキーマにはビューだけを置く
- 後から作るビューにも自動で権限を付けるには、`ALTER DEFAULT PRIVILEGES` を使う
- ビューを通して読むとき、元のテーブルにはビューの所有者の権限でアクセスする
  - Quick Sight 用の DB ユーザーに、元のテーブルの権限は要らない（与えない）
- ビューの作成と GRANT はプライマリで行う。リードレプリカは読み取り専用で、プライマリの変更がそのまま反映される
- DB ユーザーの作成、TLS、文のタイムアウトは「接続方式」ページの 5 章

## 6. 直接クエリでのフィルターの効き方

| データセットの入力         | 画面のフィルター値の行き先                       | DB 負荷 |
| -------------------------- | ------------------------------------------------ | ------- |
| テーブル / DB ビュー直指定 | SQL の WHERE 句に入る                            | 小さい  |
| カスタム SQL               | 外側のクエリに付く。内側の SQL は全件を返し得る  | 大きい  |
| カスタム SQL + データセットパラメータ | 内側の SQL に値が埋め込まれる          | 小さい  |

- 直接クエリでは、データセットの入力に DB ビューを直接指定するのを基本にする
- 「フィルターが必ず WHERE になる」とまでは公式に書かれていない。DB の監査ログで実際の SQL を確かめる（要検証）

## 7. ビューを変更するとき

- 列の追加・削除・型の変更をしたら、Quick Sight のデータセットを開いて保存し直す
  - そうしないと SPICE の取り込みが失敗する（`SQL_SCHEMA_MISMATCH_ERROR`）
- 列を消すと、その列を使っているビジュアルや計算フィールドが動かなくなる。先に Quick Sight 側の使用箇所を確かめる
- 公開用ビューに個人情報の列を誤って足さないよう、ビューの変更はレビューを通す

## 8. 出典

確認日は 2026-10-20 または 2026-10-21。「第三者情報」「コミュニティ情報」と書いたもの以外は AWS 公式。

- 2 章 ビューを使う理由
  - ビューをテーブルと同じように扱う（結合の説明）: https://docs.aws.amazon.com/quick/latest/userguide/joining-data.html
  - Unpivot と Append は SPICE 専用: https://docs.aws.amazon.com/quick/latest/userguide/spice-only-features.html
- 3 章 設計の方針
  - 19 桁を超える数値の取り込みエラー: https://docs.aws.amazon.com/quicksight/latest/user/errors-spice-ingestion.html
  - 出典なし（一般的な DB 設計の指針）: スキーマの分け方、インデックス、集計を含むビューの性能、`extracted_at` の列
- 4 章 ビューの種類と SQL の例
  - 出典なし（一般的な PostgreSQL の SQL）
- 5 章 DB ユーザーの権限
  - Quick Sight 用の DB ユーザーに要る権限: https://docs.aws.amazon.com/quicksight/latest/user/required-permissions.html
  - 出典なし（一般的な PostgreSQL・RDS の仕様）: ビューの権限の働き、`ALTER DEFAULT PRIVILEGES`、リードレプリカへの反映
- 6 章 直接クエリでのフィルターの効き方
  - カスタム SQL とフィルター（AWS ブログ）: https://aws.amazon.com/blogs/business-intelligence/optimize-queries-using-dataset-parameters-in-amazon-quicksight/
  - フィルターと SQL の関係（コミュニティ情報）: https://community.amazonquick.com/t/i-wanna-understand-filtering-quicksight-for-direct-query-reports/18985
- 7 章 ビューを変更するとき
  - スキーマ変更時の取り込み失敗: https://docs.aws.amazon.com/quick/latest/userguide/refreshing-imported-data.html
  - 取り込みのエラーの種類: https://docs.aws.amazon.com/quicksight/latest/user/errors-spice-ingestion.html
