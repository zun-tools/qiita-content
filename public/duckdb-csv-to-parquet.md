---
title: DuckDB で複数の CSV を Parquet 1本にする（COPY・zstd）
tags:
  - duckdb
  - CSV
  - Parquet
  - SQL
private: false
updated_at: '2026-11-19T09:35:28+09:00'
id: c1518d2d65dae8f7fe94
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

DuckDB の `COPY` 文を使うと、複数の CSV ファイルを zstd で圧縮した Parquet 1本にまとめて書き出せます。同じ集計を何度も回すときに、読み込むファイルの数と容量を減らせます。

月ごとに追記していく CSV は、ファイルの本数も容量も増える一方です。CSV はテキストなので、集計のたびに文字列から数値や日付への変換をやり直すことにもなります。

参考動画は記事の最後に紹介しています。

## 環境

- DuckDB v2.0.0 (Cyanoptera) d54cbbe3df（CLI。Homebrew で導入）
- macOS（Apple シリコン）
- 確認日は 2026-11-14。Linux と Windows では試していません
- サンプルデータの生成に Python 3（標準ライブラリだけ）
- Parquet の縮み方はデータによって変わります

## 完成形

```sql
COPY (FROM 'sales_2026_*.csv')
  TO 'sales_all.parquet' (FORMAT parquet, COMPRESSION zstd);
```

`sales_2026_*.csv` に一致する CSV を、列はそのままで Parquet ファイル1本に書き出します。ファイル名を手元のものに合わせてください。

:::note alert
`COPY` は成功しても何も表示しません。
`COMPRESSION` を省くと、圧縮は zstd ではなく既定の Snappy になります。
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

月ごとの売上 `sales_2026_01.csv`〜`sales_2026_03.csv`（計 6,000 行）などができます。最後の `duckdb` で対話シェルが起動し、プロンプトが `D` に変わります。以降の SQL はこの中で打ちます。

## 2. `COPY` で Parquet 1本に書き出す

`FROM 'sales_2026_*.csv'` の `*` は glob で、一致するファイルを全部まとめて1つのテーブルとして読みます。そのクエリ結果を `COPY ... TO` で Parquet に書き出します。

```sql
COPY (FROM 'sales_2026_*.csv')
  TO 'sales_all.parquet' (FORMAT parquet, COMPRESSION zstd);
```

CSV の9列には手を付けず、そのまま書き出しています。

## 3. ファイルの大きさを確かめる

対話シェルからは、`.shell` でシェルのコマンドを実行できます。

```sql
.shell ls -lh sales_2026_*.csv sales_all.parquet
```

![ターミナルの画面。.shell ls -lh の出力4行。sales_2026_01.csv から sales_2026_03.csv がそれぞれ 174K、sales_all.parquet が 66K](https://raw.githubusercontent.com/zun-tools/qiita-content/main/images/duckdb-csv-to-parquet/duckdb-parquet-size.png)
*CSV は 174K が3本、Parquet は 66K の1本*

CSV 3本の合計は約522KB、Parquet は 66KB の1本です。このデータでは約8分の1になりました。

## 4. 件数と合計が CSV と一致するか確かめる

中身が変わっていないかは、両方に同じ集計を当てて比べます。

```sql
SELECT count(*), sum(amount) FROM 'sales_all.parquet';
SELECT count(*), sum(amount) FROM 'sales_2026_*.csv';
```

どちらも 6,000 行・513,894,610 で一致しました。以降は `FROM 'sales_all.parquet'` と書くだけで、CSV と同じ書き方で集計できます。

## DuckDB について

DuckDB はサーバーを持たず、`duckdb` と打ったそのプロセスの中で動く分析用のデータベースです。`FROM` の後ろに CSV・JSON・Parquet のファイル名を書けば、表の定義も取り込みの手順も無しに読めます。

## まとめ

| 症状 | 対処 |
|---|---|
| 月ごとの CSV が増え、本数と容量がかさむ | `COPY (FROM 'sales_2026_*.csv') TO 'sales_all.parquet' (FORMAT parquet, COMPRESSION zstd);` |
| 書き出した Parquet の中身が CSV と同じか不安 | 両方に `SELECT count(*), sum(amount)` を当てて比べる |

## 参考リンク

- [Parquet（COPY・圧縮の既定）](https://duckdb.org/docs/2.0/data/parquet/overview.html)
- [サンプル: zun-tools/duckdb-sample（gen_sample.py・queries.sql）](https://github.com/zun-tools/duckdb-sample)

## 参考動画

<iframe width="560" height="315" src="https://www.youtube.com/embed/iSwRUF_RQkU" title="YouTube video player" frameborder="0" allowfullscreen></iframe>

---

シリーズの他の記事: https://qiita.com/zundamon-tools/items/3dba4f8a2dfca9ddee79
