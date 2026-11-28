---
title: OSV-Scanner でコンテナを検査し、自分で足した層の脆弱性を直して消えるまで確かめる（scan image・AlmaLinux）
tags:
  - osv-scanner
  - Docker
  - 脆弱性
  - AlmaLinux
  - rpm
private: true
updated_at: '2026-11-28T18:10:30+09:00'
id: 95279e5271f3a1ba1888
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

OSV-Scanner の `scan image` に `--format vertical` を付けると、検出した脆弱性がベースイメージの層と自分で足した層に分かれて出ます。自分の層の1件から配布元の告知と CVE へ戻り、修正版に替えてその検出が消えるまでを確かめます。

古いベースイメージにパッケージを足したコンテナを検査すると、検出の大半はベースイメージから来ます。2023年7月に作られた `almalinux:8.8` に libwebp を1つ足したイメージでは、全体で 256 件でした。そのうち自分で足した層の分は1件です。

参考動画は記事の最後に紹介しています。

## 環境

- OSV-Scanner v2.7.1（osv-scalibr 0.5.3・linux_arm64）・Docker 29.6.1
- ベースは `almalinux:8.8`（2023-07-18 作成）。libwebp は vault.almalinux.org の 8.8 のリポジトリから入れた
- 確認日は 2026-11-22。数字は1回の実行の値です。件数は照合する日の OSV のデータで変わります
- 公式ドキュメントの対応一覧が挙げる OS のパッケージは、Alpine（APK）・Debian／Ubuntu（dpkg）・Ubuntu chiseled です。RPM と AlmaLinux は載っていませんが、v2.7.1 では AlmaLinux 8 の RPM を読めました
- イメージは `docker save` した tar を `--archive` で渡す形だけで試しました。イメージ名で渡す形、macOS、amd64 では試していません

## 完成形

```bash
docker build -t el8-old .
docker save -o el8-old.tar el8-old
osv-scanner scan image --archive el8-old.tar --format vertical
```

出力の冒頭に層ごとの件数が、パッケージごとのブロックに告知の ID と修正版が出ます（抜粋）。

```text:出力（抜粋）
Container Scanning Result (AlmaLinux 8.8 (Sapphire Caracal)) (Based on "almalinux" image):
Total 69 packages affected by 256 known vulnerabilities (0 Critical, 9 High, 7 Medium, 3 Low, 237 Unknown) from 2 ecosystems.
256 vulnerabilities can be fixed.

Container image information:
  OS version: AlmaLinux 8.8 (Sapphire Caracal)
  Base Image 1 (almalinux):
    Layer 0 /bin/sh -c #(nop) ADD file:UNKNOWN in / (255 vulns)
    Layer 1 /bin/sh -c #(nop) CMD ["/bin/bash"]
  Your Image:
    Layer 2 RUN /bin/sh -c dnf -y --disablerepo='*'... (1 vulns)
...
  libwebp@1.0.0-8.el8_7 has the following known vulnerabilities:
    introduced in # 2 Layer
    ALSA-2023:5309: Important: libwebp security update
      Severity: 'N/A'; Minimal Fix Version: '1.0.0-8.el8_8.1';
```

`Your Image` の下が、自分の Dockerfile で足した層です。この1件を、修正版に替えて0件にします。

:::note alert
既定の表形式（`--format` なし）にも、どの層で入ったか（`INTRODUCED LAYER`）は出ます。ただし告知の ID は出ません。1件から告知へ戻るときは `--format vertical` を使います。
:::

## 1. 修正前の libwebp を足したイメージを作る

vault.almalinux.org の 8.8 の AppStream には、libwebp が2つの版で置かれています。`1.0.0-8.el8_7`（修正前）と `1.0.0-8.el8_8.1`（ALSA-2023:5309 の修正版）です。修正前の版を名前と版で指定して入れます。

```dockerfile:Dockerfile
FROM almalinux:8.8
RUN dnf -y --disablerepo='*' \
      --repofrompath=v88app,http://vault.almalinux.org/8.8/AppStream/$(uname -m)/os/ \
      --repofrompath=v88base,http://vault.almalinux.org/8.8/BaseOS/$(uname -m)/os/ \
      --setopt=v88app.gpgcheck=1 --setopt=v88base.gpgcheck=1 \
      --setopt=v88app.gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-AlmaLinux \
      --setopt=v88base.gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-AlmaLinux \
      install libwebp-1.0.0-8.el8_7 \
 && dnf clean all
```

リポジトリは http ですが、`gpgcheck=1` のまま署名の照合を通りました。

```bash
docker build -t el8-old .
docker run --rm --network none el8-old rpm -q libwebp openssh-server
```

```text:出力
libwebp-1.0.0-8.el8_7.aarch64
package openssh-server is not installed
```

openssh-server が入っていないことも確かめています。入れると、自分の層に OpenSSH の検出が加わります。

## 2. scan image で照合し、層ごとの件数を見る

OSV-Scanner は Google が開発している CLI（Apache-2.0）です。macOS なら `brew install osv-scanner`、Windows なら `winget install Google.OSVScanner` で入ります。Linux は [GitHub Releases](https://github.com/google/osv-scanner/releases) のバイナリを使います。この記事では v2.7.1 の `osv-scanner_linux_arm64` を、同じリリースの `osv-scanner_SHA256SUMS` と照合してから使いました。

```bash
docker save -o el8-old.tar el8-old
osv-scanner scan image --archive el8-old.tar --format vertical
```

`--archive` で `docker save` した tar を渡すと、OSV-Scanner 自身は Docker に頼らずにファイルを読みます。照合は1回 5〜6 秒でした。

パッケージは2か所から検出されました。`from 2 ecosystems` の2つは `AlmaLinux:8`（RPM のデータベース `/var/lib/rpm/Packages`）と `PyPI`（イメージに入っていた pip・setuptools の wheel）です。`scan image` が既定で使うプラグインの組は `artifact` で、このイメージではこの2種類が対象になりました。検出したパッケージの名前と版は OSV（脆弱性の記録を OS や言語ごとに集めた公開のデータベース）と照合されます。README によると、照合のために OSV.dev へ送るのは、パッケージの名前・版・エコシステム・ファイルのハッシュです。

層ごとの件数は `Container image information` に出ます。`Layer 0` がベースイメージの中身で 255 件、`Layer 2` が Dockerfile の `RUN` で 1 件でした。既定の表形式なら、libwebp の行は次のとおりです（抜粋）。

```text:表形式の出力（抜粋）
| SOURCE PACKAGE             | INSTALLED VERSION     | FIX AVAILABLE | VULN COUNT | BINARY PACKAGES (COUNT)     | INTRODUCED LAYER | IN BASE IMAGE |
| libwebp                    | 1.0.0-8.el8_7         | Fix Available |          1 | libwebp                     | # 2 Layer        | --            |
```

`IN BASE IMAGE` が `--` の行が、自分で足したパッケージです。ベースイメージ由来の行には `almalinux` と入ります。

## 3. 自分の層の1件から、告知と CVE へ戻る

vertical の libwebp のブロックに、告知の ID・題名・修正版が並びます。

```text
  libwebp@1.0.0-8.el8_7 has the following known vulnerabilities:
    introduced in # 2 Layer
    ALSA-2023:5309: Important: libwebp security update
      Severity: 'N/A'; Minimal Fix Version: '1.0.0-8.el8_8.1';
```

`ALSA-2023:5309` は AlmaLinux の告知の ID です。このブロックに CVE 番号はありません。`Important` は告知の題名の文字列で、重さの欄 `Severity` は `N/A` でした。頼りになるのは `Minimal Fix Version`、つまり上げる先の版です。CVSS の点が要るなら、次に引く CVE 番号で NVD を見ます。

どの CVE かは、OSV の記録を開くと分かります。ブラウザなら `https://osv.dev/vulnerability/ALSA-2023:5309`、API なら次の1行です。

```bash
curl -s https://api.osv.dev/v1/vulns/ALSA-2023:5309
```

```text:出力（抜粋）
{"id":"ALSA-2023:5309","summary":"Important: libwebp security update", … "related":["CVE-2023-4863"],"upstream":["CVE-2023-4863"], … "affected":[{"package":{"name":"libwebp","ecosystem":"AlmaLinux:8","purl":"pkg:rpm/almalinux/libwebp"},"ranges":[{"type":"ECOSYSTEM","events":[{"introduced":"0"},{"fixed":"1.0.0-8.el8_8.1"}]}], …
```

`upstream` の CVE-2023-4863 は、libwebp のヒープバッファオーバーフローです。`affected` には、`ecosystem` が `AlmaLinux:8`、`fixed` が `1.0.0-8.el8_8.1` と書かれています。

## 4. 修正版に替えて、検出が消えるか確かめる

Dockerfile で替えるのは `install` の行の版だけです。

```diff:Dockerfile
-      install libwebp-1.0.0-8.el8_7 \
+      install libwebp-1.0.0-8.el8_8.1 \
```

```bash
docker build -t el8-fixed .
docker save -o el8-fixed.tar el8-fixed
osv-scanner scan image --archive el8-fixed.tar --format vertical
```

```text:出力（抜粋）
Total 68 packages affected by 255 known vulnerabilities (0 Critical, 9 High, 7 Medium, 3 Low, 236 Unknown) from 2 ecosystems.
...
  Your Image:
    Layer 2 RUN /bin/sh -c dnf -y --disablerepo='*'...
...
  libuuid@2.32.1-42.el8_8 has the following known vulnerabilities:
    introduced in # 0 Layer (almalinux)
    ALSA-2026:1852: Moderate: util-linux security update
      Severity: 'N/A'; Minimal Fix Version: '2.32.1-48.el8_10';
    ALSA-2026:96212: Important: util-linux security update
      Severity: 'N/A'; Minimal Fix Version: '2.32.1-50.el8_10';
  libxml2@2.9.7-16.el8 has the following known vulnerabilities:
```

`Layer 2` の末尾から件数が消えました。`libuuid` のブロックの次がすぐ `libxml2` で、libwebp のブロックはまるごとありません。ベースイメージの 255 件はそのままです。

上流の libwebp の修正版は 1.3.2 です。1.0.0 のままで消えたのは、ALSA の記録の `fixed` が AlmaLinux の版番号で書かれているからです。比べるのは、同じ OS の版どうしです。配布元が上流の修正だけを古い版へ持ち込むやり方を、バックポートと呼びます。番号が上流より古く見えても、この OS では修正済みとして扱われ、検出も消えました。

:::note alert
修正版の番号を、NVD などに載る上流の「1.3.2 未満に効く」と比べると、まだ直っていないように見えます。比べる相手は、同じ OS の告知の `fixed`（ここでは `1.0.0-8.el8_8.1`）です。
:::

## 5. 自分のイメージで試す

```bash
docker save -o my-image.tar イメージ名:タグ
osv-scanner scan image --archive my-image.tar --format vertical
```

見る場所は、`Your Image` の層の件数と、`introduced in # N Layer` が自分の層を指すブロックです。`scan image --help` には、`--archive` を付けずに `osv-scanner scan image イメージ名:タグ` と名前で渡す使い方も書かれており、手元に無いイメージは pull すると説明されています。

## まとめ

| 症状 | 対処 |
|---|---|
| 古いベースイメージで検出が数百件出て、自分の分が分からない | `--format vertical` の `Your Image` の層の件数と、`introduced in # N Layer` が自分の層のブロックを見る |
| 表形式に告知の ID が出ない | `--format vertical` で出す |
| 検出に CVE 番号が無い | `osv.dev/vulnerability/<告知ID>` か `api.osv.dev/v1/vulns/<告知ID>` の `upstream` を見る |
| `Severity` が `N/A` | 告知の題名の `Important` などは重さの欄ではない。重さは CVE 番号で NVD を引く |
| 修正版の番号が上流の修正版より小さい | バックポート。同じ OS の告知の `fixed` と比べる |

## 参考リンク

- [google/osv-scanner（GitHub・README）](https://github.com/google/osv-scanner)
- [OSV-Scanner: Scan image](https://google.github.io/osv-scanner/usage/scan-image)
- [OSV-Scanner: 出力の形式](https://google.github.io/osv-scanner/output/)
- [OSV: ALSA-2023:5309（AlmaLinux 8 の libwebp）](https://osv.dev/vulnerability/ALSA-2023:5309)

## 参考動画

<iframe width="560" height="315" src="https://www.youtube.com/embed/1u7T_DGgf18" title="YouTube video player" frameborder="0" allowfullscreen></iframe>
