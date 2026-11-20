---
title: '脆弱性スキャンの数千件を KEV・EPSS で絞り、今日直す物だけ毎朝通知する（Grype・DuckDB）'
tags:
  - 脆弱性
  - DuckDB
  - Grype
  - EPSS
  - SQL
private: true
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

Grype の照合結果に、CISA の KEV・FIRST の EPSS・手書きの「外に出ているか」の表を DuckDB でつなぎます。4つの山に振り分けて「今日直す」だけを通知し、残りの山は消さずに末尾へ件数で残します。2回目からは、まだ届けていない組だけが届きます。

古い OS に載ったコンテナを照合すると、当たりは千件単位になります。AlmaLinux 8.4 に古い nginx・Tomcat・Rails・PHP を入れた4つのコンテナでは、コンテナ×CVE で数えて 4,393 件でした。全部を通知に流しても、どれから直すかは決まりません。

参考動画は記事の最後に紹介しています。

## 環境

- Linux（aarch64）・Docker Engine 29.6.1・Docker Compose v5.2.0
- git 2.53.0・GNU Make 4.4.1
- Syft 1.57.2・Grype 0.122.1（脆弱性 DB v6.1.10、2026-11-19 作成）・DuckDB v2.0.0。どれもコンテナの中で動きます
- KEV は catalogVersion 2026.11.18（1,786 件）、EPSS は score_date 2026-11-19 の CSV
- 確認日は 2026-11-20。数字は1回の実行の値です
- 検査したのはコンテナイメージだけです（`docker save` した tar を Syft に渡す経路）。動いているサーバーや VM へは試していません
- 通知はサンプル同梱のモック受信（Slack の Incoming Webhook の代わり）で受けました。実際の Slack へは送っていません
- macOS と amd64 では試していません

## 完成形

振り分けの本体は、`sql/02_piles.sql` の `CASE` 式です。`hits` は Grype の照合結果をコンテナ×CVE で1行にまとめた表で、`all_fixed` は「当たった部品すべてに修正版がある」を表します。

```sql:sql/02_piles.sql（抜粋）
SET VARIABLE epss_line = 0.1;

CREATE OR REPLACE TABLE piles AS
SELECT h.*,
       (k.cveID IS NOT NULL)                                AS in_kev,
       e.epss,
       (k.cveID IS NOT NULL OR coalesce(e.epss, 0) >= getvariable('epss_line')) AS exploited,
       coalesce(x.exposed, true)                           AS exposed,
       CASE WHEN NOT (k.cveID IS NOT NULL OR coalesce(e.epss, 0) >= getvariable('epss_line')) THEN '4 順番待ち（悪用されていない）'
            WHEN NOT h.all_fixed THEN '2 直す版待ち'
            WHEN NOT coalesce(x.exposed, true) THEN '4 順番待ち（外に出ていない）'
            WHEN k.cveID IS NOT NULL THEN '1 今日直す'
            ELSE '3 今週見る' END                            AS pile
FROM hits h
LEFT JOIN kev k   ON k.cveID = h.cve
LEFT JOIN epss e  ON e.cve = h.cve
LEFT JOIN exposure x USING (container);
```

KEV と EPSS は CVE 番号で、「外に出ているか」の表はコンテナ名でつなぎます。動かすのは次の3行です。

```bash
git clone https://github.com/zun-tools/vuln-watch-sample
cd vuln-watch-sample
make run
```

:::note alert
`WHERE in_kev AND all_fixed AND exposed` のように AND でつなぐと、「悪用されそうなのに修正版が無い」物が結果から消え、残りが安全に見えてしまいます。`CASE` で上から順に分け、外れた物は別の山へ送ります。
:::

## 1. サンプルを動かして、最初の通知を受け取る

ホストに要るのは Docker（Compose 付き）・git・make だけです。`make run` は次の順に進みます。

1. Grype の DB を更新し、KEV と EPSS を取得する
2. 4つのコンテナを build し、Syft で SBOM を作って Grype で照合する
3. DuckDB で4つの山に振り分け、通知を送る
4. 送った組を `state/watch.duckdb` に記録する

ベースのイメージと DB が手元にある状態で、`git clone` から通知まで 44〜56 秒でした。初回の pull と DB のダウンロードを含む時間は測っていません。

`.env` に `SLACK_WEBHOOK_URL` が無ければ、通知は `out/received/received_001.json` に届きます。`text` の中身（改行を展開）は次のとおりです。

```text:out/received/received_001.json の text
CVE-2020-11023（nginx・php・tomcat）悪用されている・修正版あり・外に出ている
CVE-2023-44487（nginx・php・tomcat）悪用されている・修正版あり・外に出ている
CVE-2023-4863（nginx・php）悪用されている・修正版あり・外に出ている
CVE-2023-4911（nginx・php・tomcat）悪用されている・修正版あり・外に出ている
CVE-2025-27363（nginx・php・tomcat）悪用されている・修正版あり・外に出ている
ほかの山（コンテナ×CVE）: 直す版待ち 14・今週見る 91・順番待ち 4274（悪用されていない 4236・外に出ていない 38）／今日直す（届け済み・未対応）5 CVE・14件
```

今日直すは CVE で5つ、コンテナ×CVE で数えると14件です。上の5行が CVE ごとの行で、括弧の中が当たったコンテナです。

この件数は、DB・KEV・EPSS を上の時点に固定した値です。既定の `make run` は毎回それぞれを最新にするので、後から回すと件数も並ぶ CVE も変わります。同じ入力で回し直すときは `make run FREEZE=1` を使います。

## 2. 何を、何のキーでつなぐか

DuckDB に入れる入力は4つで、取り込み用のスクリプトはありません。`read_json` と `read_csv` で直接読みます。

| 入力 | 出どころ | つなぐキー |
|---|---|---|
| 照合結果（部品・版・修正の状態） | Syft の SBOM を Grype で照合した JSON | — |
| 悪用が確認された一覧 | CISA の KEV（JSON） | CVE |
| 悪用されそうかの点数 | FIRST の EPSS（日次 CSV） | CVE |
| 外に出ているか | 手書きの `inventory/exposure.csv` | コンテナ名 |
| 届け済みの記録 | `state/watch.duckdb` の `notified` 表 | コンテナ×CVE |

Grype の JSON は `matches` 配列を `unnest` で1行ずつに展開します。

```sql:sql/01_load.sql（抜粋）
CREATE OR REPLACE TABLE grype_matches AS
SELECT t.container, g.m
FROM (
  SELECT regexp_extract(filename, 'grype_run/([^/]+)\.json$', 1) AS file, unnest(matches) AS m
  FROM read_json('out/grype_run/*.json', filename = true, maximum_object_size = 268435456)
) g
JOIN read_csv('out/grype_run/targets.csv', header = true, columns = {'file': 'VARCHAR', 'container': 'VARCHAR'}) t
  USING (file);

-- EPSS: 1 行目が「#model_version:…,score_date:…」のコメント行。skip=1 で読み飛ばす
CREATE OR REPLACE TABLE epss AS
SELECT cve, epss, percentile
FROM read_csv('out/data/epss.csv', skip = 1, header = true,
              columns = {'cve': 'VARCHAR', 'epss': 'DOUBLE', 'percentile': 'DOUBLE'});
```

:::note alert
`read_json` が1ファイル1オブジェクトとして読める大きさは、既定で 16MB までです。rails のコンテナの Grype の JSON は約 44MB（46,381,994 バイト）あり、`maximum_object_size` を指定しないと読めませんでした。
:::

数える単位はコンテナ×CVE です。同じ CVE が1つのコンテナの複数の部品に当たっても1件にし、3つのコンテナに当たれば3件と数えます。Grype の行のままだと 6,521 行でした。Grype の `vulnerability.id` が GHSA の行は、`relatedVulnerabilities` にある CVE を使います。CVE が無い GHSA の 22 行は KEV とも EPSS ともつながらず、順番待ちに入ります。

Grype 0.122 の JSON も、行ごとに KEV の印（`knownExploited`）と EPSS の点数（`epss`）を持ちます（[vulnerability_metadata.go](https://github.com/anchore/grype/blob/v0.122.1/grype/presenter/models/vulnerability_metadata.go)）。Grype の外にしか無いのは、外に出ているかの表と届け済みの記録です。DuckDB でつなぐ理由はこの2つにあります。

## 3. 3つの問いで4つの山に分ける

`CASE` の分岐は「悪用されている（されそう）か → 修正版があるか → 外に出ているか」の順です。

1. **悪用されている、またはされそうか**: KEV に載っているか、EPSS が線（`epss_line`）以上か。外れたら「順番待ち（悪用されていない）」
2. **修正版があるか**: 当たった部品すべてで Grype の `fix.state` が `fixed` か。外れたら「直す版待ち」
3. **外に出ているか**: 表で `はい` か（表に無いコンテナも `はい` 扱い）。外れたら「順番待ち（外に出ていない）」
4. 残りのうち KEV に載っている物が「今日直す」、EPSS だけで残った物が「今週見る」

KEV は、実際に悪用された信頼できる証拠がある脆弱性を CISA がまとめた一覧です（[CISA](https://www.cisa.gov/known-exploited-vulnerabilities-catalog/reducing-significant-risk-known-exploited-vulnerabilities)）。EPSS は、公開済みの CVE が今後30日以内に悪用される確率の推定で、0〜1 の値が毎日出ます（[FIRST](https://www.first.org/epss/)）。確認された物と見込みの物を分けるため、今日直すは KEV だけに絞り、EPSS で残った物は今週見るへ回しています。`fix.state` の値は `fixed`・`not-fixed`・`wont-fix`・`unknown` です（[Grype の JSON](https://oss.anchore.com/docs/guides/vulnerability/json/)）。

`make run` の後、山ごとの件数は `out/results4/piles4_summary.csv` に出ます。

| pile | コンテナ×CVE | CVE の数 | Grype の行 |
|---|--:|--:|--:|
| 1 今日直す | 14 | 5 | 26 |
| 2 直す版待ち | 14 | 4 | 20 |
| 3 今週見る | 91 | 35 | 137 |
| 4 順番待ち（外に出ていない） | 38 | 38 | 82 |
| 4 順番待ち（悪用されていない） | 4236 | 1471 | 6256 |

コンテナ×CVE の合計は 14＋14＋91＋38＋4,236＝4,393 で、元の件数と一致します。CVE の数は山をまたいで重なるので足せません。段ごとには 4,393 → 157 → 143 → 105 と減り、最後の 105 が今日直す 14 と今週見る 91 に分かれます。

直す版待ちの 14 件は、すべて KEV に無く EPSS が線以上の物でした。KEV に載る条件の1つに「ベンダーの更新など、直し方がはっきりしていること」があり、この実行では「悪用が確認されているのに修正版が無い」物は 0 件です。直す版待ちには `wont-fix`（curl の CVE-2024-7264、4件）も入っています。待っても修正版が来ない物が混ざる山です。

外に出ていない 38 件はすべて rails のコンテナで、KEV に載った CVE も3つ含みます。表で `いいえ` にしたので今日直すに入らなかっただけで、直さなくてよい物ではありません。今日直す以外の 4,379 件も同じで、順番が後ろになっただけです。SBOM に載らない物（パッケージ管理の外に置いた実行ファイルなど）と、脆弱性 DB がまだ知らない物は、どの山にも入りません。

### EPSS の線を動かすと、何が変わるか

線は `SET VARIABLE epss_line = 0.1;` の1か所です。0.1 は推奨値ではなく、この実行で引いた線です。`out/results4/funnel4.csv` に、線を4通りにしたときの件数が出ます。

```bash
cat out/results4/funnel4.csv
```

```text
theta,n0_hits,n1_exploited,pile1,pile2,pile3,pile4_not_exploited,pile4_not_exposed
0.05,4393,277,14,40,161,4116,62
0.10,4393,157,14,14,91,4236,38
0.20,4393,115,14,12,61,4278,28
0.50,4393,66,14,8,29,4327,15
```

`pile1`（今日直す）は線に依らず 14 のままです。今日直すは KEV で決まり、線が動かすのは直す版待ち・今週見る・順番待ちの3つです。

### 通知のコンテナ名は「どこ」で、「何」ではない

通知の行に部品の名前と上げる先の版はありません。`out/results4/pile1_today.csv` に出ます。

```text:out/results4/pile1_today.csv
cve,containers,n_container_cve,epss,fix_parts
CVE-2020-11023,"[nginx, php, tomcat]",3,0.84887,"[libgcc 8.5.0-23.el8_10.alma.1, libgomp 8.5.0-23.el8_10.alma.1, libstdc++ 8.5.0-23.el8_10.alma.1]"
CVE-2023-44487,"[nginx, php, tomcat]",3,0.99999,"[tomcat-coyote 8.5.94/9.0.81/10.1.14/11.0.0-M12, libnghttp2 1.33.0-5.el8_8, tomcat-coyote 9.0.81]"
CVE-2023-4863,"[nginx, php]",2,0.99979,[libwebp 1.0.0-8.el8_8.1]
CVE-2023-4911,"[nginx, php, tomcat]",3,0.63769,"[glibc-common 2.28-225.el8_8.6, glibc-minimal-langpack 2.28-225.el8_8.6, glibc 2.28-225.el8_8.6]"
CVE-2025-27363,"[nginx, php, tomcat]",3,0.27775,[freetype 2.9.1-10.el8_10]
```

`containers`・`fix_parts` の中の並びは、回すたびに変わることがあります。

当たっているのは nginx や PHP の本体ではありません。コンテナの中の OS の rpm（glibc・libwebp・freetype・libnghttp2・libgcc など）です。Tomcat のコンテナだけは、Tomcat 自身の jar（`tomcat-coyote`）にも CVE-2023-44487 が当たっています。

:::note alert
CVE-2020-11023 は jQuery の脆弱性ですが、ここでは gcc の rpm（libgcc など）に当たっています。AlmaLinux の勧告 ALSA-2025:1301 がソースパッケージ gcc 単位で出ているためです。Grype はソースパッケージ名で引いた結果を、バイナリの部品に割り当てます（`exact-indirect-match`）。当たった rpm に jQuery は入っていませんが、上げる先の版は勧告どおりです。
:::

## 4. 2回目は、まだ届けていない組だけ届ける

届けた組は、`sql/05_record.sql` が `notified` 表に記録します。次の回は `ANTI JOIN` で、記録に無い組だけを今日直すから拾います。

```sql:sql/04_notify.sql（抜粋）
CREATE OR REPLACE TABLE new_pile1 AS
SELECT p.container, p.cve FROM piles p
ANTI JOIN notified n ON n.container = p.container AND n.cve = p.cve
WHERE p.pile LIKE '1%';
```

サンプルの `make next-day` は、DB・KEV・EPSS を固定したまま、表に Apache（httpd）のコンテナを1行足して回します。

```bash
make next-day
```

```text:out/received/received_002.json の text
CVE-2020-11023（httpd）悪用されている・修正版あり・外に出ている
CVE-2023-44487（httpd）悪用されている・修正版あり・外に出ている
CVE-2023-4911（httpd）悪用されている・修正版あり・外に出ている
CVE-2026-111497（httpd）悪用されている・修正版あり・外に出ている
ほかの山（コンテナ×CVE）: 直す版待ち 18（前回 14 から +4）・今週見る 118（前回 91 から +27）・順番待ち 5141（前回 4274 から +867）（悪用されていない 5103・外に出ていない 38）／今日直す（届け済み・未対応）6 CVE・18件（前回 14件 から +4）
```

前回届けた 14 組の再送は 0 です。4行のうち3つは前回と同じ CVE で、足した httpd にも当たった分です。差分のキーがコンテナ×CVE なので、届くのは新しい CVE ではなく、新しく直す場所になります。初めて出た CVE は1つで、Brotli（圧縮データを展開する部品）の脆弱性でした。

`new_pile1` の条件は「今日直すにあって、届け済みの記録に無い組」です。コンテナを足したときのほか、KEV の更新で新しく載った組も同じ条件で届きます（この記事で実際に回したのは、コンテナを足す場合だけです）。

末尾の「今日直す（届け済み・未対応）」は、届け済みで、いまも今日直すに当たっている組の数です。サンプルは「直した」の判定を持たないので、部品を上げて照合に当たらなくなるまで、この数は減りません。

## 5. 毎朝回す・Slack に送る

README にある cron の設定例です。

```cron
0 7 * * * cd "$HOME/vuln-watch-sample" && make run >> logs/run.log 2>&1
```

この行は、cron と同じ最小の環境（`env -i`・`PATH=/usr/bin:/bin`）で1回通し、通知が出るところまで確かめました。crontab に登録して時刻で起動させてはいません。

Grype は既定で、脆弱性 DB が5日より古いと検査を止めます（[Grype の DB](https://oss.anchore.com/docs/guides/vulnerability/database/)）。`make run` は先頭で `grype db update` を走らせます。

Slack へ送るときは、`.env.example` を `.env` にコピーして `SLACK_WEBHOOK_URL` を書きます。`tools/post.py` が `{"text": "…"}` の JSON をその URL へ POST します。Webhook の URL は投稿先のチャンネルに書き込める秘密の値なので、`.env` はコミットしません。

## 6. 自分のイメージで試す

既存のイメージは、`IMAGES` に名前で渡します（空白区切りで複数可）。外に出ているかは、`inventory/exposure.csv` にイメージ名のまま1行足します。

```csv:inventory/exposure.csv に足す行
nginx:1.18.0,はい
```

サンプルを回した後なら、先に前回の記録を消します。残っていると、末尾の「前回から」の差がサンプルの件数との比較になります。

```bash
rm -f state/watch.duckdb
make run IMAGES="nginx:1.18.0"
```

:::note alert
表に書き忘れたイメージは「外に出ている」扱いで振り分けられ、通知の末尾に「台帳に無い箱 N（外に出ている扱い）」と名前が出ます。知らないコンテナを、黙って順番待ちへ落とさないための既定です。
:::

Docker Hub の `nginx:1.18.0`（Debian 10）を表に書かずに回すと、照合は 353 件でした。今日直す 1（CVE-2023-4863・libwebp）・直す版待ち 10・今週見る 23・順番待ち 319 に分かれ、末尾に台帳に無い箱の行が出ました。台帳に `nginx:1.18.0,いいえ` の1行を足して回し直すと、KEV に載って修正版もある CVE-2023-4863 は今日直すに入らず、順番待ちに入りました。

## 使った道具について

Syft と Grype は Anchore の OSS（Apache-2.0）です。Syft が部品一覧（SBOM）を作り、Grype が脆弱性 DB と照合します。DuckDB はサーバーを持たず、呼び出したプロセスの中で動く分析用のデータベースです。

## まとめ

| 症状 | 対処 |
|---|---|
| スキャン結果が数千件あり、通知が読まれない | KEV・修正版・外に出ているかで `CASE` を組み、「今日直す」だけ CVE ごとに通知。ほかの山は末尾に件数で残す |
| AND で絞ると、悪用されそうで修正版の無い物が消える | `CASE` の順で外れた物を別の山へ（直す版待ち・順番待ち） |
| 毎朝同じ CVE が届く | 届けた組（コンテナ×CVE）を `notified` 表に記録し、`ANTI JOIN` で除く |
| `read_json` が Grype の JSON を読めない | `maximum_object_size` を指定する（約 44MB の JSON で必要だった） |
| 通知に「nginx」とあるが、何を上げればよいか分からない | `out/results4/pile1_today.csv` の部品と修正版を見る |
| 表の書き忘れで今日直すから落ちる | 表に無いコンテナは「外に出ている」扱いにし、通知の末尾に名前を出す |

## 参考リンク

- [CISA: KEV に載る条件](https://www.cisa.gov/known-exploited-vulnerabilities-catalog/reducing-significant-risk-known-exploited-vulnerabilities)
- [FIRST: Exploit Prediction Scoring System (EPSS)](https://www.first.org/epss/)
- [Grype: JSON の出力（fix.state・match の種類）](https://oss.anchore.com/docs/guides/vulnerability/json/)
- [DuckDB: JSON の読み込み](https://duckdb.org/docs/2.0/data/json/overview)
- [サンプル: zun-tools/vuln-watch-sample（Makefile・sql/・inventory/）](https://github.com/zun-tools/vuln-watch-sample)

## 参考動画

<iframe width="560" height="315" src="https://www.youtube.com/embed/XxTwMfcxm_4" title="YouTube video player" frameborder="0" allowfullscreen></iframe>
