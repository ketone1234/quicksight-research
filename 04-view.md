# ビュー（画面と DB ビュー）

「ビュー」は 2 つの意味で調べた。

- Quick Sight でデータを見せる仕組み（分析、ダッシュボード、ビジュアル、パラメータ、フィルター）
- DB 側のビュー（問い合わせ種別ごとに参照する列をまとめたもの）

## 1. 結論

- ダッシュボードを 1 つ作り、問い合わせ種別ごとにシートを分ける
- 画面上部にアカウント識別子の入力欄（テキスト入力）を置き、全シートの表をその値で絞り込む
- 明細は表ビジュアルで出す
- PO 用（SPICE）と開発用（直接クエリ）でダッシュボードを分ける。画面の作りは同じでよい

画面のイメージ:

```text
+-- ダッシュボード「アカウント調査」 --------------------------+
| アカウント識別子: [ 1234567890      ]  (テキスト入力)        |
+-- シート: [契約] [利用履歴] [決済] [問い合わせ履歴] ---------+
| 表ビジュアル: account_id = 入力値 の行だけを表示             |
| 取り込み時刻: 2026-10-20 03:00 (SPICE 版のみ、1 日 1 回)     |
+--------------------------------------------------------------+
```

## 2. 分析からダッシュボード公開までの流れ

1. Analyses → New analysis → データセットを選んで USE IN ANALYSIS
2. シートの種類（Interactive sheet）とレイアウトを選ぶ
3. Insert → Add Visual でビジュアルを置く（表、ピボット、KPI など 30 種以上）
4. パラメータ、コントロール、フィルターを設定する（3 章）
5. Publish → New dashboard（2 回目以降は Replace an existing dashboard）→ 公開するシートを選ぶ
6. More settings で表示オプションを選び、Publish dashboard
7. Share でユーザーかグループに共有する

- 分析は Author 以上が使う編集画面。ダッシュボードは Reader でも見られる閲覧画面
- 1 つの分析に置けるシートは 20 枚まで

## 3. アカウント識別子で検索する画面の作り方

### 3.1 手順

1. データセットの準備: 問い合わせ種別ごとのデータセットに、同じ名前・同じ型の `account_id`（String）列を持たせる
2. パラメータを作る: Parameters → Add
   - Name: `AccountId`
   - Data type: String
   - Values: Single value
   - Static default value: 存在しない値（例 `__NONE__`）。理由は 3.2
3. コントロールを作る: パラメータのメニュー → Add control → Style: Text field
   - 複数の ID をまとめて入れたい場合は Text field - multiline と複数値パラメータにする
4. フィルターを作る: シートごと、データセットごとに次を設定する
   - Filter → Add → `account_id` → Filter type: Custom filter → Equals → Use parameters → `AccountId`
5. シートを分ける: 種別ごとにシートを作る。パラメータは分析全体で共通なので、1 回の入力で全シートに効く
6. 公開して、PO のグループに共有する

### 3.2 注意点

- 既定値を空にすると全件が表示されるという報告がある（コミュニティ情報）。存在しない値を既定値にして避ける（要検証）
- 文字列の比較は大文字と小文字を区別する
- フィルターは「1 つのフィールド、1 つのデータセット」に対して作る。データセットが 4 つならフィルターも 4 つ要る
- 同じ名前・同じ型の列なら、1 つのフィルターを複数データセットに効かせる設定もある。パラメータとの併用は要検証
- 表ビジュアルでは、列を Value にだけ入れる。Group by に入れると、全件を取ってから集約するので重くなる

### 3.3 URL で値を渡す

- 形式: `https://ap-northeast-1.quicksight.aws.amazon.com/sn/dashboards/<ダッシュボード ID>#p.AccountId=1234567890`
- `p.` の後ろはパラメータ名（コントロール名ではない）。複数は `&` でつなぐ
- `#` より後ろはサーバーに送られない。つまり URL で渡した値も、AWS 側のログには残らない
- 特定のシートを直接開く URL の形式は公式にない（要検証）

### 3.4 データ鮮度の表示（SPICE 版）

- DB ビューに取り込み時刻の列を足し、KPI ビジュアルでその最大値を表示する（6 章の SQL 例）
- SPICE は取り込みに失敗しても古いデータを表示し続ける。時刻を出しておかないと、古さに気づけない

## 4. 直接クエリでのフィルターの効き方

| データセットの入力         | 画面のフィルター値の行き先                       | DB 負荷 |
| -------------------------- | ------------------------------------------------ | ------- |
| テーブル / DB ビュー直指定 | SQL の WHERE 句に入る                            | 小さい  |
| カスタム SQL               | 外側のクエリに付く。内側の SQL は全件を返し得る  | 大きい  |
| カスタム SQL + データセットパラメータ | 内側の SQL に値が埋め込まれる          | 小さい  |

- 直接クエリでは DB ビュー直指定を基本にする
- 「フィルターが必ず WHERE になる」とまでは公式に書かれていない。DB の監査ログで実際の SQL を確かめる（要検証）

## 5. 表示件数とダウンロード

### 5.1 上限

| 対象           | 上限                                                       |
| -------------- | ---------------------------------------------------------- |
| 表ビジュアル   | 200 列。表示行数の公式な上限は見つからなかった            |
| ピボット       | 50 万レコード。行と列のフィールドはそれぞれ 20 まで        |
| CSV 出力       | 500MB または 100 万行                                      |
| Excel 出力     | 表とピボットのみ。表は 80 万セル / 10 万行                 |

### 5.2 ダウンロードを止める方法

| 方法                     | 設定                                                                     |
| ------------------------ | ------------------------------------------------------------------------ |
| ダッシュボードの公開設定 | Publish → More settings → Enable download options のチェックを外す      |
| API                      | `DashboardPublishOptions.ExportToCSVOption.AvailabilityStatus=DISABLED`  |
| カスタム権限（ロール単位）| `ExportToCsv`、`ExportToExcel`、`ExportToPdf`、`PrintReports` を DENY   |

- 公開設定で Excel も止まるかは、公式の文面が CSV にしか触れていない（要検証）
- 確実に止めるなら、Reader ロールのカスタム権限で止める（「権限」ページ）
- ページ分割レポート（Pixel-Perfect Reports）は、帳票の定期配信用の有料アドオン（月 $500 から）。本件では不要

## 6. DB ビューの設計

### 6.1 方針

- 問い合わせ種別ごとに 1 ビュー。名前の例: `v_inq_contract`、`v_inq_payment`
- 先頭列を `account_id` にし、元テーブルの `account_id` 列にはインデックスを張る
- 問い合わせの回答に要る列だけを選ぶ。個人情報など不要な列は入れない
- 集計、DISTINCT、LIMIT を含むビューは、条件が内側まで届かず遅くなることがある。明細のビューにする
- 1 アカウントに大量の行がある種別は、期間で絞った列（例: 直近 90 日）にするかを決めておく

### 6.2 SQL の例（PostgreSQL）

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

- `extracted_at` は SPICE に取り込んだ時刻になる（直接クエリでは表示した時刻になる）

## 7. 分析とダッシュボードの主な制限値

| 項目                         | 上限                                         |
| ---------------------------- | -------------------------------------------- |
| シート                       | 1 分析あたり 20                              |
| ビジュアル                   | 1 シートあたり 50（UI の説明では 30。食い違いあり）|
| データセット                 | 1 ダッシュボードあたり 50                    |
| パラメータ                   | 1 分析あたり 400                             |
| コントロール                 | 1 シートあたり 200                           |
| 計算フィールド               | 1 分析あたり 500                             |
| ドロップダウンの選択肢表示   | 1,000 件                                     |
| 1 ユーザーの同時セッション   | 3（調整不可）                                |

## 8. 出典

確認日はいずれも 2026-10-20。「第三者情報」「コミュニティ情報」と書いたもの以外は AWS 公式。

- 2 章 分析からダッシュボード公開までの流れ
  - 分析の作成: https://docs.aws.amazon.com/quick/latest/userguide/creating-an-analysis.html
  - ビジュアルの種類: https://docs.aws.amazon.com/quick/latest/userguide/working-with-visual-types.html
  - ダッシュボードの公開: https://docs.aws.amazon.com/quick/latest/userguide/creating-a-dashboard.html
  - ダッシュボードの共有: https://docs.aws.amazon.com/quick/latest/userguide/sharing-a-dashboard.html
  - 複数のシート: https://docs.aws.amazon.com/quick/latest/userguide/working-with-multiple-sheets.html
- 3 章 アカウント識別子で検索する画面の作り方
  - パラメータの作成: https://docs.aws.amazon.com/quick/latest/userguide/parameters-set-up.html
  - コントロール: https://docs.aws.amazon.com/quick/latest/userguide/parameters-controls.html
  - パラメータを使うフィルター: https://docs.aws.amazon.com/quick/latest/userguide/parameters-filtering-by.html
  - 文字列フィルター（大文字と小文字の区別）: https://docs.aws.amazon.com/quick/latest/userguide/add-a-text-filter-data-prep.html
  - フィルターの追加: https://docs.aws.amazon.com/quick/latest/userguide/adding-a-filter.html
  - 複数データセット・複数シートへのフィルター: https://docs.aws.amazon.com/quick/latest/userguide/cross-sheet-filters.html
  - URL でのパラメータ指定: https://docs.aws.amazon.com/quick/latest/userguide/parameters-in-a-url.html
  - 表ビジュアルの性能（AWS ブログ）: https://aws.amazon.com/blogs/big-data/tips-and-tricks-for-high-performant-dashboards-in-amazon-quicksight
  - 既定値が空のときの全件表示（コミュニティ情報）: https://community.amazonquick.com/t/parameter-field-when-kept-blank-and-is-minimized-displays-all/23017
  - シートを直接開く URL（コミュニティ情報）: https://community.amazonquick.com/t/filtering-and-navigating-to-a-particular-report-in-dashboard/48089
  - 取り込みに失敗しても古いデータを表示（AWS ブログ）: https://aws.amazon.com/blogs/business-intelligence/automate-spice-dataset-refreshes-using-amazon-quick-sight-apis/
- 4 章 直接クエリでのフィルターの効き方
  - カスタム SQL とフィルター（AWS ブログ）: https://aws.amazon.com/blogs/business-intelligence/optimize-queries-using-dataset-parameters-in-amazon-quicksight/
  - フィルターと SQL の関係（コミュニティ情報）: https://community.amazonquick.com/t/i-wanna-understand-filtering-quicksight-for-direct-query-reports/18985
- 5 章 表示件数とダウンロード
  - 表ビジュアル: https://docs.aws.amazon.com/quick/latest/userguide/tabular.html
  - ピボット: https://docs.aws.amazon.com/quick/latest/userguide/pivot-table.html
  - データの書き出し: https://docs.aws.amazon.com/quick/latest/userguide/exporting-data.html
  - ダウンロードの無効化（API）: https://docs.aws.amazon.com/quicksight/latest/APIReference/API_CreateDashboard.html
  - カスタム権限: https://docs.aws.amazon.com/quick/latest/userguide/custom-permissions.html
  - ページ分割レポート: https://docs.aws.amazon.com/quick/latest/userguide/qs-reports-getting-started.html
  - ページ分割レポートの料金: https://aws.amazon.com/quicksight/pricing/
- 6 章 DB ビューの設計
  - 出典なし（一般的な DB 設計の指針）: ビューの分け方、インデックス、集計を含むビューの性能、`CURRENT_TIMESTAMP` の列
- 7 章 分析とダッシュボードの主な制限値
  - 分析の制限値: https://docs.aws.amazon.com/quick/latest/userguide/analysis-item-limits.html
  - UI 側の説明（ビジュアル 30、データセット 50）: https://docs.aws.amazon.com/quick/latest/userguide/working-with-visual-types.html
  - ドロップダウンの表示件数: https://docs.aws.amazon.com/quick/latest/userguide/parameters-controls.html
  - 同時セッション: https://docs.aws.amazon.com/general/latest/gr/quicksight.html
