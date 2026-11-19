---
title: 'DuckDB で CSV と JSON を取り込まずに SQL で読み、JOIN する'
tags:
  - DuckDB
  - SQL
  - CSV
  - JSON
private: false
updated_at: '2026-11-19T07:26:43+09:00'
id: dd2c93034ba11461eb7e
organization_url_name: null
slide: false
ignorePublish: false
---

DuckDB を使うと、手元の CSV と JSON を `CREATE TABLE` も取り込みもせずに、`FROM` にファイル名を書くだけで SQL で読めます。月ごとに分かれた CSV 3本（計 6,000 行）と商品一覧の JSON を、1本の SQL でつないで集計するところまで進めます。

多くのデータベースでは、CSV に SQL を当てる前にサーバーを用意し、`CREATE TABLE` で表を定義して取り込む手順が要ります。ファイルが月ごとに増えれば、その取り込みも増えた分だけ繰り返すことになります。

参考動画は記事の最後に紹介しています。

## 環境

- DuckDB v2.0.0 (Cyanoptera) d54cbbe3df（CLI。Homebrew で導入）
- macOS（Apple シリコン）
- 確認日は 2026-11-14。Linux と Windows では試していません
- サンプルデータの生成に Python 3（標準ライブラリだけ）

## 完成形

```sql
SELECT p.category, p.name,
       count(s.order_id) AS order_count,
       sum(s.quantity)   AS total_qty,
       sum(s.amount)     AS total_revenue
FROM 'sales_2026_*.csv' s
JOIN 'products.json' p ON s.product_id = p.product_id
GROUP BY p.category, p.name
ORDER BY total_revenue DESC;
```

売上の CSV（`sales_2026_*.csv` に一致する全部）と商品一覧の JSON を商品 ID で JOIN し、カテゴリ・商品ごとに集計します。ファイル名と列名を手元のものに合わせてください。

:::note alert
`sales_2026_*.csv` の `*` は、後から同じフォルダに置いたファイルにも一致します。4月分を置けば、同じ SQL の結果に4月分が入ります。
JSON の配列の添字は 1 始まりです。先頭の要素は `tags[1]` です。
:::

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

月ごとの売上 `sales_2026_01.csv`〜`sales_2026_03.csv` と、商品一覧の `products.json` などができます。最後の `duckdb` で対話シェルが起動し、プロンプトが `D` に変わります。以降の SQL はこの中で打ちます。

## 2. 1本の CSV を読む

```sql
SELECT order_id, order_date, product_id, quantity, amount FROM 'sales_2026_01.csv' LIMIT 5;
```

```text
┌──────────────────┬────────────┬────────────┬──────────┬────────┐
│     order_id     │ order_date │ product_id │ quantity │ amount │
│     varchar      │    date    │  varchar   │  int64   │ int64  │
├──────────────────┼────────────┼────────────┼──────────┼────────┤
│ ORD-2026_01-0001 │ 2026-01-30 │ P001       │        2 │  29720 │
│ ORD-2026_01-0002 │ 2026-01-08 │ P001       │        4 │  62196 │
│ ORD-2026_01-0003 │ 2026-01-18 │ P004       │        5 │ 151605 │
│ ORD-2026_01-0004 │ 2026-01-05 │ P003       │        2 │ 134526 │
│ ORD-2026_01-0005 │ 2026-01-10 │ P002       │        4 │  45660 │
└──────────────────┴────────────┴────────────┴──────────┴────────┘
```

`FROM` の後ろにファイル名を引用符で書けば、そのままテーブルとして扱えます。列名の下の行は型推論の結果で、区切り文字も同じく中身から推論されます。推論が手元の CSV に合わないなら、`read_csv('ファイル名.csv', ...)` の形にして `columns` で列の型を指定します。

## 3. `*` で複数の CSV をまとめて読む

ファイル名の月の部分を `*`（glob）にすると、一致するファイルを全部まとめて1つのテーブルとして読みます。

```sql
SELECT count(*) AS total_rows, sum(amount) AS total_sales FROM 'sales_2026_*.csv';
```

```text
┌────────────┬─────────────┐
│ total_rows │ total_sales │
│   int64    │   int128    │
├────────────┼─────────────┤
│       6000 │   513894610 │
└────────────┴─────────────┘
```

いまフォルダにあるのは1〜3月の3本なので、3か月分です。月を絞るなら、ファイル名を具体的に書くか、`WHERE` で `order_date` を絞ります。

## 4. JSON の配列と入れ子を取り出す

`products.json` の1件は、次の形です。タグは配列で、電池の持ちは `spec` の中に入れ子になっています。

```json:products.json（先頭の1件）
  {
    "product_id": "P001",
    "name": "メカニカルキーボード 60%",
    "category": "PC周辺機器",
    "base_cost": 12000,
    "tags": [
      "ハードウェア",
      "人気",
      "静音赤軸"
    ],
    "spec": {
      "switch": "silent-red",
      "wireless": true,
      "battery_hours": 40
    }
  },
```

JSON も `FROM` に書けば読めます。配列は `tags[1]` のように添字で、入れ子は `spec.battery_hours` のようにドットでつないで取り出します。

```sql
SELECT product_id, name, category, tags[1] AS first_tag, spec.battery_hours AS battery FROM 'products.json';
```

| product_id<br>varchar | name<br>varchar | category<br>varchar | first_tag<br>varchar | battery<br>int64 |
|---|---|---|---|--:|
| P001 | メカニカルキーボード 60% | PC周辺機器 | ハードウェア | 40 |
| P002 | トラックボールマウス | PC周辺機器 | ハードウェア | 120 |
| P003 | ウルトラワイドモニター 34型 | ディスプレイ | 大型 | NULL |
| P004 | ノイズキャンセリングヘッドセット | オーディオ | マイク | NULL |
| P005 | USB-C デスク用ドック 12-in-1 | アクセサリ | ハブ | NULL |

モニターなど `spec` に `battery_hours` のキーが無い商品は `NULL` です。

## 5. CSV と JSON を JOIN する

売上の行には商品 ID しかありません。商品名とカテゴリは JSON 側にあるので、商品 ID をキーに JOIN します（完成形と同じ文）。

```sql
SELECT p.category, p.name,
       count(s.order_id) AS order_count,
       sum(s.quantity)   AS total_qty,
       sum(s.amount)     AS total_revenue
FROM 'sales_2026_*.csv' s
JOIN 'products.json' p ON s.product_id = p.product_id
GROUP BY p.category, p.name
ORDER BY total_revenue DESC;
```

| category<br>varchar | name<br>varchar | order_count<br>int64 | total_qty<br>int128 | total_revenue<br>int128 |
|---|---|--:|--:|--:|
| ディスプレイ | ウルトラワイドモニター 34型 | 1177 | 3513 | 249440200 |
| オーディオ | ノイズキャンセリングヘッドセット | 1214 | 3699 | 108775577 |
| アクセサリ | USB-C デスク用ドック 12-in-1 | 1161 | 3510 | 64477414 |
| PC周辺機器 | メカニカルキーボード 60% | 1263 | 3735 | 54958049 |
| PC周辺機器 | トラックボールマウス | 1185 | 3489 | 36243370 |

ファイル名に `s` や `p` の別名を付ければ、テーブルと同じ書き方で `JOIN ... ON` が書けます。`order_count` の合計は 6,000、`total_revenue` の合計は 513,894,610 で、`*` で読んだときの件数・合計と一致しました。ここまで `CREATE TABLE` は一度も打っていません。

## DuckDB について

DuckDB はサーバーを持たず、`duckdb` と打ったそのプロセスの中で動く分析用のデータベースです。`FROM` の後ろに CSV・JSON・Parquet のファイル名を書けば、表の定義も取り込みの手順も無しに読めます。

## まとめ

| 症状 | 対処 |
|---|---|
| CSV に SQL を当てたいが、表を作って取り込むのが手間 | `FROM 'ファイル名.csv'` |
| 型推論が手元の CSV に合わない | `read_csv('ファイル名.csv', ...)` の `columns` で型を指定 |
| 月ごとに分かれた CSV をまとめて集計したい | `FROM 'sales_2026_*.csv'`。絞るならファイル名か `WHERE` |
| JSON の配列・入れ子の値が要る | `tags[1]`・`spec.battery_hours` |
| CSV に ID しかなく、名前やカテゴリは JSON 側にある | 別名を付けて `JOIN 'products.json' p ON ...` |

## 参考リンク

- [CSV の読み込み](https://duckdb.org/docs/2.0/data/csv/overview.html)
- [複数ファイルを読む（glob）](https://duckdb.org/docs/2.0/data/multiple_files/overview.html)
- [JSON を読む](https://duckdb.org/docs/2.0/data/json/overview.html)
- [サンプル: zun-tools/duckdb-sample（gen_sample.py・queries.sql）](https://github.com/zun-tools/duckdb-sample)

## 参考動画

<iframe width="560" height="315" src="https://www.youtube.com/embed/iSwRUF_RQkU" title="YouTube video player" frameborder="0" allowfullscreen></iframe>

---

シリーズの他の記事: https://qiita.com/zundamon-tools/items/3dba4f8a2dfca9ddee79
