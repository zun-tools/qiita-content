---
title: DuckDB で Shift_JIS の CSV や「N/A」混じりの列を読む（encodings・nullstr）
tags:
  - duckdb
  - CSV
  - SQL
  - 文字コード
  - Shift_JIS
private: true
updated_at: '2026-11-18T13:02:53+09:00'
id: e42c799a5d89996f1ce4
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

DuckDB で `This file is not utf-8 encoded` と出る Shift_JIS の CSV を読めるようになります。数字の列に「N/A」や「未定」が混じって `sum` が通らない CSV も、前処理のスクリプト無しに SQL だけで集計できます。

<iframe width="560" height="315" src="https://www.youtube.com/embed/C-XYBog9rM4" title="YouTube video player" frameborder="0" allowfullscreen></iframe>

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

-- 数字の列に「N/A」「未定」が混じった CSV を、数字の列として読む
FROM read_csv('sales_corrupted.csv', nullstr = ['N/A', '未定']);
```

ファイル名と、空とみなす文字（`'N/A'` と `'未定'`）を手元の CSV に合わせて書き換えれば使えます。

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

Shift_JIS の `sales_sjis.csv`、崩れた列を持つ `sales_corrupted.csv` などができます。最後の `duckdb` で対話の画面に入り、プロンプトが `D` になります。以降の SQL はこの中で打ちます。

## 2. Shift_JIS の CSV を読む

### そのまま読むと「not utf-8 encoded」

DuckDB は `FROM` の後ろにファイル名をそのまま書くと、その CSV を読みます。`SELECT *` を省いた書き方で、全列が出ます。区切りや列の型は、中身を見て自動で推論されます。Shift_JIS のファイルでは、ここで止まります。

```sql
FROM 'sales_sjis.csv';
```

```text
Invalid Input Error: CSV Error on Line: 1
Invalid unicode (byte sequence mismatch) detected. This file is not utf-8 encoded.
```

ファイルが UTF-8 ではない、というエラーです。

### `encoding` を付けても「does not support the encoding」

`read_csv` の `encoding` で文字コードを指定しても、拡張が無い状態では別のエラーになります。

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

CSV の読み込みが標準で対応する文字コードは、UTF-8・UTF-16・Latin-1 の3つだけです。DuckDB には、拡張を必要なときに自動で読み込む設定（自動ロード）があります。その設定が既定のままでも、拡張を一度も入れていない状態ではこのエラーになりました。

### `encodings` 拡張を入れて読む

エラー文が示すとおりに、拡張を入れてから同じ文を打ちます。

```sql
INSTALL encodings; LOAD encodings;
FROM read_csv('sales_sjis.csv', encoding = 'shift_jis');
```

| 注文番号<br>varchar | 商品名<br>varchar | 区分<br>varchar | 金額<br>int64 |
|---|---|---|--:|
| S001 | 日本語メカニカルキーボード | 通販 | 15000 |
| S002 | 静音マウスパッド | 店舗 | 2500 |
| S003 | 有線LANアダプタ | 法人 | 3800 |

列名も中身も日本語で読め、金額は整数の `int64` になっています。`encodings` は公式の拡張です。`INSTALL` でダウンロードし、`LOAD` でそのセッションに読み込みます。初回はネットにつながっている必要がありますが、登録は要りません。

## 3. 数字の列に「N/A」「未定」が混じった CSV を読む

### `SUMMARIZE` で、文字の列になった数字の列を見つける

`sales_corrupted.csv` は 100 行の売上で、単価と金額の列に「N/A」や「未定」が混じっています。DuckDB は CSV の中身を見て列の型を推論するので、文字が1つでも混じった列は文字の列になります。全列の型をまとめて見るには `SUMMARIZE` を当てます。

```sql
SUMMARIZE 'sales_corrupted.csv';
```

出力のうち、2つの列の部分です。

| column_name | column_type | min | max |
|---|---|---|---|
| unit_price | VARCHAR | 12000 | 未定 |
| amount | VARCHAR | 24000 | 未定 |

`VARCHAR` は文字の列を表します。文字として並べるので、最大が「未定」になっています。`SUMMARIZE` は全部の列について、型・最小・最大・おおよその種類数・平均・分位点・件数・`NULL` の割合を1回で返します。集計の前に当てておけば、数字のはずの列が `VARCHAR` になっていることに先に気づけます。対話の画面に入らず、シェルから `duckdb -c "SUMMARIZE 'ファイル名.csv';"` と打っても使えます。

### このまま `sum` を取るとエラーになる

```sql
SELECT sum(amount) FROM 'sales_corrupted.csv';
```

```text
Binder Error: No function matches the given name and argument types 'sum(VARCHAR)'. You might need to add explicit type casts.
```

文字の列に使える `sum` が無い、と言われます。

### `nullstr` に空とみなす文字を渡す

`read_csv` の `nullstr` に、`NULL` として扱う文字を並べます。混じっている文字が複数あるときは、このように `[ ]` の中にまとめて渡します。`DESCRIBE` は、列名と型だけを並べる文です。

```sql
DESCRIBE FROM read_csv('sales_corrupted.csv', nullstr = ['N/A', '未定']);
SELECT count(*) AS rows, sum(amount) AS total
FROM read_csv('sales_corrupted.csv', nullstr = ['N/A', '未定']);
```

`DESCRIBE` で見ると `unit_price` と `amount` が整数の列として読まれ、集計は 100 行・合計 2,352,000 になりました。崩れていたのは 100 行のうち2行で、金額が「N/A」の行と、単価も金額も「未定」の行です。この2行の金額は `NULL` になり、合計には入りません。

このデータは `nullstr` だけで直りました。それで直らない CSV は、`read_csv` の `columns` で列の型も指定します。

## DuckDB について

DuckDB はサーバーを持たず、`duckdb` と打ったそのプロセスの中で動く分析用のデータベースです。`FROM` の後ろに CSV・JSON・Parquet のファイル名を書けば、表の定義も取り込みの手順も無しに読めます。

## まとめ

- Shift_JIS の CSV は、`encoding = 'shift_jis'` だけでは読めない。`INSTALL encodings; LOAD encodings;` の後なら読める
- 数字の列に文字が混じると、その列は `VARCHAR` に推論され `sum` が通らない。`SUMMARIZE` で型を見て、`nullstr` に空とみなす文字を渡す
- `nullstr` で空とみなした値は `NULL` になり、合計に入らない。このサンプルでは 100 行のうち2行がそうなり、合計は 2,352,000 だった

## 参考リンク

- [CSV の読み込み（encoding・nullstr・columns）](https://duckdb.org/docs/2.0/data/csv/overview.html)
- [encodings 拡張](https://duckdb.org/docs/2.0/core_extensions/encodings.html)
- [サンプル: zun-tools/duckdb-sample（gen_sample.py・queries.sql）](https://github.com/zun-tools/duckdb-sample)

---

シリーズの他の記事: https://qiita.com/zundamon-tools/private/3dba4f8a2dfca9ddee79
