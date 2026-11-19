---
title: 'DuckDB で Shift_JIS の CSV や「N/A」混じりの列を読む（encodings・nullstr）'
tags:
  - DuckDB
  - CSV
  - SQL
  - 文字コード
  - Shift_JIS
private: false
updated_at: '2026-11-19T07:26:42+09:00'
id: e42c799a5d89996f1ce4
organization_url_name: null
slide: false
ignorePublish: false
---

DuckDB で `This file is not utf-8 encoded` と出る Shift_JIS の CSV を読めるようになります。数値列に「N/A」や「未定」が混じって `sum` が通らない CSV も、前処理のスクリプト無しに SQL だけで集計できます。

日本語版 Windows の Excel で「CSV（コンマ区切り）」として保存した CSV は、文字コードが Shift_JIS です。DuckDB は既定で CSV を UTF-8 として読むので、そのままでは開けません。

参考動画は記事の最後に紹介しています。

## 環境

- DuckDB v2.0.0 (Cyanoptera) d54cbbe3df（CLI。Homebrew で導入）
- macOS（Apple シリコン）
- 確認日は 2026-11-14。Linux と Windows では試していません
- サンプルデータの生成に Python 3（標準ライブラリだけ）

## 完成形

```sql
-- Shift_JIS の CSV を読む
INSTALL encodings; LOAD encodings;
FROM read_csv('sales_sjis.csv', encoding = 'shift_jis');

-- 数値列に「N/A」「未定」が混じった CSV を、数値列として読む
FROM read_csv('sales_corrupted.csv', nullstr = ['N/A', '未定']);
```

ファイル名と、`NULL` とみなす文字列（`'N/A'` と `'未定'`）を手元の CSV に合わせて書き換えてください。

:::note alert
`encodings` 拡張は、自動ロードが既定のままでも一度 `INSTALL` するまで使えません。`encoding = 'shift_jis'` を付けただけでは通りません。
数値のはずの列も、100 行のうち2行に文字が混じっただけで列ごと `VARCHAR` に推論されます。
:::

## 1. DuckDB を入れて、サンプルのデータを作る

Mac なら Homebrew の1行で入ります。

```bash
brew install duckdb
```

Linux と macOS は、公式のインストールスクリプト `curl https://install.duckdb.org | bash` でも入ります。Windows は公式のインストールのページを見てください。アカウントや設定ファイルは要りません。

データはサンプルのスクリプトで作ります。乱数の種を固定しているので、件数と合計はこの記事と同じになります。

```bash
git clone https://github.com/zun-tools/duckdb-sample.git
mkdir duckdb-trial && cd duckdb-trial
python3 ../duckdb-sample/gen_sample.py
duckdb
```

Shift_JIS の `sales_sjis.csv` や、数値列に文字が混じった `sales_corrupted.csv` ができます。最後の `duckdb` で対話シェルが起動し、プロンプトが `D` に変わります。以降の SQL はこの中で打ちます。

## 2. Shift_JIS の CSV を読む

### そのまま読むと「not utf-8 encoded」

`FROM` の後ろにファイル名を書くだけで、DuckDB はその CSV を読みます（`SELECT *` を省いた書き方）。区切り文字と列の型は型推論に任せています。

```sql
FROM 'sales_sjis.csv';
```

```text
Invalid Input Error: CSV Error on Line: 1
Invalid unicode (byte sequence mismatch) detected. This file is not utf-8 encoded.
```

### `encoding` を付けても「does not support the encoding」

`read_csv` の `encoding` で文字コードを指定しても、拡張が無い状態では別のエラーに変わるだけです。

```sql
FROM read_csv('sales_sjis.csv', encoding = 'shift_jis');
```

```text
Invalid Input Error: The CSV Reader does not support the encoding: "shift_jis"
It is possible that the encoding exists in the encodings extension. You can try "INSTALL encodings; LOAD encodings"
The currently supported encodings are: 
*  utf-16
*  latin-1
*  utf-8
```

CSV リーダー本体が扱える文字コードは、UTF-8・UTF-16・Latin-1 の3つに限られます。Shift_JIS は `encodings` 拡張の担当です。

### `encodings` 拡張を入れて読む

```sql
INSTALL encodings; LOAD encodings;
FROM read_csv('sales_sjis.csv', encoding = 'shift_jis');
```

| 注文番号<br>varchar | 商品名<br>varchar | 区分<br>varchar | 金額<br>int64 |
|---|---|---|--:|
| S001 | 日本語メカニカルキーボード | 通販 | 15000 |
| S002 | 静音マウスパッド | 店舗 | 2500 |
| S003 | 有線LANアダプタ | 法人 | 3800 |

`encodings` は公式の拡張です。`INSTALL` でダウンロードし、`LOAD` でそのセッションに読み込みます。初回だけネットワーク接続が要りますが、登録は要りません。

## 3. 数値列に「N/A」「未定」が混じった CSV を読む

### `SUMMARIZE` で、VARCHAR になった数値列を見つける

`sales_corrupted.csv` は 100 行の売上で、単価と金額の列に「N/A」や「未定」が混じっています。全列の推論結果をまとめて見るなら `SUMMARIZE` です。

```sql
SUMMARIZE 'sales_corrupted.csv';
```

出力のうち、2つの列の部分です。

| column_name | column_type | min | max |
|---|---|---|---|
| unit_price | VARCHAR | 12000 | 未定 |
| amount | VARCHAR | 24000 | 未定 |

max が「未定」なのは、文字列として比べているためです。`SUMMARIZE` は全列の型・最小・最大・おおよその種類数・平均・分位点・件数・`NULL` の割合を1回で返します。集計の前に当てておくと、数値のはずの列が `VARCHAR` になっていても先に気づけます。シェルから直接 `duckdb -c "SUMMARIZE 'ファイル名.csv';"` と打つこともできます。

### このまま `sum` を取るとエラーになる

```sql
SELECT sum(amount) FROM 'sales_corrupted.csv';
```

```text
Binder Error: No function matches the given name and argument types 'sum(VARCHAR)'. You might need to add explicit type casts.
```

### `nullstr` に NULL とみなす文字列を渡す

`read_csv` の `nullstr` に、`NULL` として扱う文字列を渡します。複数あれば `[ ]` のリストにまとめます。`DESCRIBE` は列名と型だけを返す文です。

```sql
DESCRIBE FROM read_csv('sales_corrupted.csv', nullstr = ['N/A', '未定']);
SELECT count(*) AS rows, sum(amount) AS total
FROM read_csv('sales_corrupted.csv', nullstr = ['N/A', '未定']);
```

`unit_price` と `amount` は `BIGINT`（上の表の `int64` と同じ型）として読まれ、集計は 100 行・合計 2,352,000 になりました。崩れていたのは 100 行のうち2行で、金額が「N/A」の行と、単価も金額も「未定」の行です。この2行の金額は `NULL` なので、合計には入りません。

このデータは `nullstr` だけで直りました。それでも型が合わない CSV は、`read_csv` の `columns` で列の型を指定します。

## DuckDB について

DuckDB はサーバーを持たず、`duckdb` と打ったそのプロセスの中で動く分析用のデータベースです。`FROM` の後ろに CSV・JSON・Parquet のファイル名を書けば、表の定義も取り込みの手順も無しに読めます。

## まとめ

| 症状 | 対処 |
|---|---|
| `This file is not utf-8 encoded` | `INSTALL encodings; LOAD encodings;` の後に `read_csv(..., encoding = 'shift_jis')` |
| `does not support the encoding: "shift_jis"` | 同上（`INSTALL` を一度も打っていない） |
| `sum(VARCHAR)` の Binder Error。`SUMMARIZE` で数値のはずの列が `VARCHAR` | `read_csv(..., nullstr = ['N/A', '未定'])` で該当の文字列を `NULL` に |
| `nullstr` でも型が合わない | `read_csv` の `columns` で列の型を指定 |

## 参考リンク

- [CSV の読み込み（encoding・nullstr・columns）](https://duckdb.org/docs/2.0/data/csv/overview.html)
- [encodings 拡張](https://duckdb.org/docs/2.0/core_extensions/encodings.html)
- [サンプル: zun-tools/duckdb-sample（gen_sample.py・queries.sql）](https://github.com/zun-tools/duckdb-sample)

## 参考動画

<iframe width="560" height="315" src="https://www.youtube.com/embed/iSwRUF_RQkU" title="YouTube video player" frameborder="0" allowfullscreen></iframe>

---

シリーズの他の記事: https://qiita.com/zundamon-tools/items/3dba4f8a2dfca9ddee79
