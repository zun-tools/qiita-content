---
title: 'DuckDB で CSV と JSON を扱う：取り込まずに SQL で読む・集計する・読めない CSV を直す'
tags:
  - DuckDB
  - SQL
  - CSV
  - JSON
  - Parquet
private: false
updated_at: '2026-11-19T07:24:46+09:00'
id: 3dba4f8a2dfca9ddee79
organization_url_name: null
slide: false
ignorePublish: false
---

DuckDB で手元の CSV と JSON を、表を作らず取り込みもせずに SQL で扱う方法を、課題ごとに4本の記事で紹介します。どれも同じサンプルのスクリプトで作ったデータで、コマンドと出力をそのまま載せています。

参考動画は記事の最後に紹介しています。

## 記事の一覧

- 「CSV と JSON の JOIN」https://qiita.com/zundamon-tools/items/dd2c93034ba11461eb7e
  1本の CSV、`*` でまとめた複数の CSV、JSON の配列と入れ子を読み、CSV と JSON を `JOIN` する
- 「集計を短く書く」https://qiita.com/zundamon-tools/items/595748af201e33f3973f
  `SUMMARIZE` で全列の型と欠けを見て、`GROUP BY ALL` と `PIVOT` で短く集計する
- 「CSV を Parquet 1本にする」https://qiita.com/zundamon-tools/items/c1518d2d65dae8f7fe94
  複数の CSV を `COPY ... (FORMAT parquet, COMPRESSION zstd)` で1本の Parquet にまとめる
- 「Shift_JIS・N/A を読む」https://qiita.com/zundamon-tools/items/e42c799a5d89996f1ce4
  Shift_JIS の CSV を `encodings` 拡張で読み、「N/A」「未定」が混じった数字の列を `nullstr` で直す

## 4本でやったこと

架空のガジェット店の売上データ（月ごとの CSV 3本・計 6,000 行と、商品一覧の JSON）で試した結果です。最後の2行は、同じスクリプトが作る別のファイル（Shift_JIS の CSV と、数字の列が崩れた 100 行の CSV）の結果です。

| やったこと | 要の書き方 | 結果 | 記事 |
|---|---|---|---|
| CSV 3本をまとめて読む | `FROM 'sales_2026_*.csv'` | 6,000 行・売上合計 513,894,610 | CSV と JSON の JOIN |
| JSON の商品一覧とつなぐ | `JOIN 'products.json'` | カテゴリ・商品別の5行 | CSV と JSON の JOIN |
| 全列の中身を一度に見る | `SUMMARIZE` | 9列の型・最小・最大・欠けの割合 | 集計を短く書く |
| 月×カテゴリの表にする | `PIVOT ... ON category` | 3か月×4カテゴリ | 集計を短く書く |
| Parquet 1本に変える | `COPY ... (FORMAT parquet)` | 522KB → 66KB・件数と合計が一致 | CSV を Parquet 1本にする |
| Shift_JIS の CSV を読む | `INSTALL encodings` | 日本語の3行 | Shift_JIS・N/A を読む |
| 「N/A」「未定」が混じった CSV | `nullstr` | 100 行・合計 2,352,000 | Shift_JIS・N/A を読む |

どの記事の手順でも、`CREATE TABLE` は一度も打っていません。

## DuckDB とは

DuckDB は分析用のデータベースです。ふつうのデータベースは先に立てたサーバーへつないで使いますが、DuckDB はサーバーを持たず、`duckdb` と打ったそのプロセスの中で動きます。`FROM` の後ろにファイル名をそのまま書けば、CSV・JSON・Parquet を表の定義も取り込みの手順も無しに読めます。CSV の区切りや列の型は、中身から自動で推論されます。

オランダの研究所 CWI で作られた MIT ライセンスの OSS で、v2.0.0 は 2026年10月21日に公開された長期サポート版です。

## 参考リンク

- [Why DuckDB](https://duckdb.org/why_duckdb)
- [DuckDB 2.0.0 の発表（LTS）](https://duckdb.org/2026/10/21/announcing-duckdb-200.html)
- [サンプル: zun-tools/duckdb-sample（gen_sample.py・queries.sql）](https://github.com/zun-tools/duckdb-sample)

## 参考動画

<iframe width="560" height="315" src="https://www.youtube.com/embed/iSwRUF_RQkU" title="YouTube video player" frameborder="0" allowfullscreen></iframe>
