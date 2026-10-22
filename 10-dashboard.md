# ダッシュボード（画面）

Quick Sight でデータを見せる仕組み（分析、ダッシュボード、ビジュアル、パラメータ、フィルター）をまとめる。
DB 側のビューは「ビュー」ページにまとめた。

## 1. 結論

- ダッシュボードを 1 つ作り、問い合わせ種別ごとにシートを分ける
- 画面上部にアカウント識別子の入力欄（テキスト入力）を置き、全シートの表をその値で絞り込む
- 明細は表ビジュアルで出す
- 問い合わせ用のシートとは別に、サービス横断で推移や件数を見る分析用のシートを置く（4 章）
- ビジネス部門用（SPICE）と開発部門用（直接クエリ）でダッシュボードを分ける。画面の作りは同じでよい

画面のイメージ:

```text
+-- ダッシュボード「アカウント調査」 --------------------------+
| アカウント識別子: [ 1234567890      ]  (テキスト入力)        |
+-- シート: [契約] [利用履歴] [決済] [問い合わせ履歴] [分析] --+
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
6. 公開して、ビジネス部門のグループに共有する

### 3.2 注意点

- 既定値を空にすると全件が表示されるという報告がある（コミュニティ情報）。存在しない値を既定値にして避ける（要検証）
- 文字列の比較は大文字と小文字を区別する
- フィルターは「1 つのフィールド、1 つのデータセット」に対して作る。データセットが 4 つならフィルターも 4 つ要る
- 同じ名前・同じ型の列なら、1 つのフィルターを複数データセットに効かせる設定もある。パラメータとの併用は要検証
- 表ビジュアルでは、列を Value にだけ入れる。Group by に入れると、全件を取ってから集約するので重くなる
- 直接クエリのダッシュボードで、フィルターの値が DB にどう渡るかは「ビュー」ページの 6 章

### 3.3 URL で値を渡す

- 形式: `https://ap-northeast-1.quicksight.aws.amazon.com/sn/dashboards/<ダッシュボード ID>#p.AccountId=1234567890`
- `p.` の後ろはパラメータ名（コントロール名ではない）。複数は `&` でつなぐ
- `#` より後ろはサーバーに送られない。つまり URL で渡した値も、AWS 側のログには残らない
- 特定のシートを直接開く URL の形式は公式にない（要検証）

### 3.4 データ鮮度の表示（SPICE 版）

- DB ビューに取り込み時刻の列（`extracted_at`）を足し、KPI ビジュアルでその最大値を表示する（「ビュー」ページの 4.1）
- SPICE は取り込みに失敗しても古いデータを表示し続ける。時刻を出しておかないと、古さに気づけない

## 4. 分析用のシート（サービス横断の集計）

問い合わせ用のシートとは別に、サービス横断で推移や件数を見るシートを作る。

| 見たいこと                             | ビジュアル               | 設定の例                                                    |
| -------------------------------------- | ------------------------ | ----------------------------------------------------------- |
| プラン別のオーダー件数の推移           | 折れ線グラフ             | X 軸: 日付（月単位）、値: 件数、色: プラン                  |
| 予約から実行までの日数                 | 棒グラフ、KPI            | 値: 日数の計算フィールド（下記）の平均                      |
| 処理ステータス別の件数                 | 棒グラフ、ピボット       | 行: ステータス、値: 件数                                    |
| 契約状態・製品別の契約者数と解約の推移 | 積み上げ棒グラフ、折れ線 | X 軸: 月、値: 契約者数（重複を除いた件数）、色: 契約状態    |

- サービスを選ぶドロップダウンを置き、サービス名の列（「ビュー」ページの 4.3）で絞り込む
- 件数は Count、契約者数のように重複を除きたいものは Count distinct で集計する
- 縦持ちに変換したビュー（「ビュー」ページの 4.2）を使うと、プランを「色」に入れるだけでプラン別のグラフになる
- 分析用のシートは SPICE で作る。集計は全件を読むので、直接クエリにすると DB の負荷が大きい
- 予約から実行までの日数は、計算フィールドで求める
  - `dateDiff(date1, date2, "DD")` は date2 から date1 を引いた日数を整数で返す。単位を省略すると日数になる
  - 時間で求める場合は `"HH"` にする

```text
dateDiff({reserved_at}, {executed_at}, "DD")
```

## 5. 表示件数とダウンロード

### 5.1 上限

| 対象           | 上限                                                       |
| -------------- | ---------------------------------------------------------- |
| 表ビジュアル   | 200 列。表示行数の公式な上限は見つからなかった            |
| ピボット       | 50 万レコード。行と列のフィールドはそれぞれ 20 まで        |
| CSV 出力       | 500MB または 100 万行                                      |
| Excel 出力     | 表とピボットのみ。表は 80 万セル / 10 万行                 |

### 5.2 ダウンロードを止める方法

本件では、ダウンロードは制限しないと決めた（2026-10-22）。以下は、方針を変える場合の参考。

| 方法                     | 設定                                                                     |
| ------------------------ | ------------------------------------------------------------------------ |
| ダッシュボードの公開設定 | Publish → More settings → Enable download options のチェックを外す      |
| API                      | `DashboardPublishOptions.ExportToCSVOption.AvailabilityStatus=DISABLED`  |
| カスタム権限（ロール単位）| `ExportToCsv`、`ExportToExcel`、`ExportToPdf`、`PrintReports` を DENY   |

- 公開設定で Excel も止まるかは、公式の文面が CSV にしか触れていない（要検証）
- 確実に止めるなら、Reader ロールのカスタム権限で止める（「権限」ページ）
- ページ分割レポート（Pixel-Perfect Reports）は、帳票の定期配信用の有料アドオン（月 $500 から）。本件では不要

## 6. 分析とダッシュボードの主な制限値

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

## 7. 出典

確認日は 2026-10-20 または 2026-10-21。「第三者情報」「コミュニティ情報」と書いたもの以外は AWS 公式。

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
- 4 章 分析用のシート
  - ビジュアルの種類: https://docs.aws.amazon.com/quick/latest/userguide/working-with-visual-types.html
  - dateDiff 関数: https://docs.aws.amazon.com/quick/latest/userguide/dateDiff-function.html
- 5 章 表示件数とダウンロード
  - 表ビジュアル: https://docs.aws.amazon.com/quick/latest/userguide/tabular.html
  - ピボット: https://docs.aws.amazon.com/quick/latest/userguide/pivot-table.html
  - データの書き出し: https://docs.aws.amazon.com/quick/latest/userguide/exporting-data.html
  - ダウンロードの無効化（API）: https://docs.aws.amazon.com/quicksight/latest/APIReference/API_CreateDashboard.html
  - カスタム権限: https://docs.aws.amazon.com/quick/latest/userguide/custom-permissions.html
  - ページ分割レポート: https://docs.aws.amazon.com/quick/latest/userguide/qs-reports-getting-started.html
  - ページ分割レポートの料金: https://aws.amazon.com/quicksight/pricing/
- 6 章 分析とダッシュボードの主な制限値
  - 分析の制限値: https://docs.aws.amazon.com/quick/latest/userguide/analysis-item-limits.html
  - UI 側の説明（ビジュアル 30、データセット 50）: https://docs.aws.amazon.com/quick/latest/userguide/working-with-visual-types.html
  - ドロップダウンの表示件数: https://docs.aws.amazon.com/quick/latest/userguide/parameters-controls.html
  - 同時セッション: https://docs.aws.amazon.com/general/latest/gr/quicksight.html
