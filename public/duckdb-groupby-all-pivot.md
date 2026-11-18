---
title: DuckDB で CSV の集計を短く書く（SUMMARIZE・GROUP BY ALL・PIVOT）
tags:
  - duckdb
  - SQL
  - CSV
private: true
updated_at: '2026-11-18T13:03:22+09:00'
id: 595748af201e33f3973f
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

DuckDB の独自の構文を使うと、初めて見る CSV の中身の確認から、月×カテゴリの集計表づくりまでを短い SQL で書けます。`SUMMARIZE` で全列の型と欠けを1回で見て、`GROUP BY ALL` と `PIVOT` でまとめます。

参考動画は記事の最後に紹介しています。

## 環境

- DuckDB v2.0.0 (Cyanoptera) d54cbbe3df（CLI。Homebrew で導入）
- macOS（Apple シリコン）
- 確認日は 2026-11-14。Linux と Windows では試していません
- サンプルデータの生成に Python 3（標準ライブラリだけ）

## 完成形

```sql
PIVOT (
  SELECT strftime(s.order_date, '%Y-%m') AS month, p.category, s.amount
  FROM 'sales_2026_*.csv' s
  JOIN 'products.json' p ON s.product_id = p.product_id
)
ON category USING sum(amount)
GROUP BY month
ORDER BY month;
```

月ごとの売上の CSV を商品一覧の JSON とつなぎ、月を行・カテゴリを列にして金額を合計する文です。ファイル名と列名を手元のものに合わせれば使えます。

## 1. DuckDB を入れて、サンプルのデータを作る

Mac なら Homebrew の1行で入ります。

```bash
brew install duckdb
```

Linux と macOS は、公式のインストールスクリプト `curl https://install.duckdb.org | bash` でも入ります。Windows は公式のインストールのページを見てください。アカウントや設定ファイルは要りません。

データは架空のガジェット店の売上で、サンプルのスクリプトで作ります。乱数の種を固定しているので、件数と合計はこの記事と同じになります。

```bash
git clone https://github.com/zun-tools/duckdb-sample.git
mkdir duckdb-trial && cd duckdb-trial
python3 ../duckdb-sample/gen_sample.py
duckdb
```

月ごとの売上 `sales_2026_01.csv`〜`sales_2026_03.csv`（計 6,000 行）と、商品一覧の `products.json` などができます。売上の CSV は9列で、注文番号 `order_id`・注文日 `order_date`・商品 ID `product_id`・顧客区分 `customer_type`・数量 `quantity`・金額 `amount`・備考 `note` などが並びます。商品一覧の JSON は、商品 ID `product_id`・商品名 `name`・カテゴリ `category` などを持ちます。最後の `duckdb` で対話の画面に入り、プロンプトが `D` になります。以降の SQL はこの中で打ちます。

## 2. `FROM` から書く・`DESCRIBE` で型を見る

DuckDB には、手で探るときに書く量を減らす独自の構文があり、公式は「Friendly SQL」と呼んでいます。まずは `FROM` から書く形と `DESCRIBE` です。

```sql
FROM 'sales_2026_01.csv' LIMIT 3;
DESCRIBE FROM 'sales_2026_01.csv';
```

1行目は `SELECT *` を省いた形で、全列が出ます。`DESCRIBE` は列名と型だけを並べます。どちらも、`FROM` の後ろに CSV のファイル名をそのまま書いています。DuckDB は CSV の区切りや列の型を中身から推論するので、表の定義も取り込みも要りません。

## 3. `SUMMARIZE` で全列を一度に見る

初めて見る CSV に最初に当てるなら `SUMMARIZE` です。

```sql
SUMMARIZE 'sales_2026_01.csv';
```

全部の列について、型・最小・最大・おおよその種類数・平均・分位点・件数・`NULL` の割合を1回で返します。出力は12列と横に長いので、一部を抜き出しました。

| column_name | column_type | min | max | null_percentage |
|---|---|---|---|--:|
| order_date | DATE | 2026-01-01 | 2026-01-31 | 0.00 |
| quantity | BIGINT | 1 | 5 | 0.00 |
| amount | BIGINT | 9458 | 390990 | 0.00 |
| note | VARCHAR | ギフト包装 | 通常発送 | 16.35 |

見る所は `column_type` です。`BIGINT` は整数、`VARCHAR` は文字の列を表します。数字の列が `VARCHAR` になっていれば、集計の前にここで気づけます。備考の `note` は 16.35% が空で、欠けている列も先に分かります。

## 4. `GROUP BY ALL` でまとめる列を書かない

```sql
SELECT customer_type, product_id, count(*) AS count, sum(amount) AS total_amount
FROM 'sales_2026_01.csv'
GROUP BY ALL
ORDER BY total_amount DESC
LIMIT 5;
```

| customer_type<br>varchar | product_id<br>varchar | count<br>int64 | total_amount<br>int128 |
|---|---|--:|--:|
| プレミアム | P003 | 103 | 22769168 |
| ゲスト | P003 | 107 | 22528947 |
| 法人 | P003 | 102 | 21935642 |
| 一般会員 | P003 | 91 | 19959635 |
| ゲスト | P004 | 118 | 10597135 |

`GROUP BY ALL` は、`SELECT` に並んだ集計でない列（ここでは `customer_type` と `product_id`）を、まとめる列として推論します。`SELECT` の列を足し引きしても、`GROUP BY` を書き直さずに済みます。

## 5. `PIVOT` で月×カテゴリの表にする

月とカテゴリの組み合わせを縦に並べると12行になり、見比べにくくなります。`PIVOT` で、カテゴリを列に回します。カテゴリは商品一覧の `products.json` にあるので、売上の CSV 3本と商品 ID でつないでから回しています。

```sql
PIVOT (
  SELECT strftime(s.order_date, '%Y-%m') AS month, p.category, s.amount
  FROM 'sales_2026_*.csv' s
  JOIN 'products.json' p ON s.product_id = p.product_id
)
ON category USING sum(amount)
GROUP BY month
ORDER BY month;
```

![ターミナルの画面。上に PIVOT の SQL、下に month と PC周辺機器・アクセサリ・オーディオ・ディスプレイの5列の表。2026-01 から 2026-03 の3行で、ディスプレイの列は 87193392・87381929・74864879](https://raw.githubusercontent.com/zun-tools/qiita-content/main/images/duckdb-groupby-all-pivot/duckdb-pivot-month-category.png)
*右端のディスプレイの列だけ、3月が2月より約1,250万円少ない。ほかの3列は大きく動いていない*

`ON` に横へ並べる列、`USING` に升目に入れる集計を書き、`GROUP BY` に書いた `month` が1行ずつの行になります。`FROM 'sales_2026_*.csv'` の `*` は、一致するファイルを全部まとめて1つの表として読む書き方です。

## DuckDB について

DuckDB はサーバーを持たず、`duckdb` と打ったそのプロセスの中で動く分析用のデータベースです。`FROM` の後ろに CSV・JSON・Parquet のファイル名を書けば、表の定義も取り込みの手順も無しに読めます。

## まとめ

- 初めて見る CSV には `SUMMARIZE` を当て、`column_type` で数字の列が数字として読まれているか、欠けがどれだけあるかを先に見る
- `GROUP BY ALL` なら、まとめる列を `SELECT` から推論してくれる
- `PIVOT ... ON 列 USING 集計` で、縦に長い集計を見比べやすい表に回せる

## 参考リンク

- [Friendly SQL](https://duckdb.org/docs/2.0/sql/dialect/friendly_sql.html)
- [SUMMARIZE](https://duckdb.org/docs/2.0/guides/meta/summarize.html)
- [PIVOT](https://duckdb.org/docs/2.0/sql/statements/pivot.html)
- [サンプル: zun-tools/duckdb-sample（gen_sample.py・queries.sql）](https://github.com/zun-tools/duckdb-sample)

## 参考動画

<iframe width="560" height="315" src="https://www.youtube.com/embed/C-XYBog9rM4" title="YouTube video player" frameborder="0" allowfullscreen></iframe>

---

シリーズの他の記事: https://qiita.com/zundamon-tools/private/3dba4f8a2dfca9ddee79
