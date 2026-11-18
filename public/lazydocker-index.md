---
title: 'lazydocker の使い方：docker compose の落ちたコンテナを直す・長いコマンドを登録する・困ったときのキー'
tags:
  - lazydocker
  - Docker
  - docker-compose
  - TUI
private: true
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

lazydocker で docker compose の開発環境を扱う方法を、課題ごとに2本の記事で紹介します。この記事には、入れ方と画面の見方、困ったときに押すキーの早見をまとめています。

参考動画は記事の最後に紹介しています。

## 記事の一覧

- 「落ちたコンテナを直す」https://qiita.com/zundamon-tools/private/3834b64259d4b101cd22
  落ちたサービスとログを1画面で見つけ、compose.yml を直したら `r` ではなく `R` → recreate で作り直す
- 「長いコマンドを登録する」https://qiita.com/zundamon-tools/private/e92d6b5d31c884fdd368
  `docker compose exec` で打つ rails db:migrate・rails console・psql を `customCommands` に登録し、`c` のメニューから呼ぶ

どちらも Rails と Postgres のサンプル（[zun-tools/lazydocker-sample](https://github.com/zun-tools/lazydocker-sample)）を使います。手順と出力は、lazydocker v0.25.2・Docker Engine 29.6.1（Linux）で 2026-11-12 に確かめたものです。

## 入れて開く

lazydocker の README は、要件を Docker 29.0.0 以上としています。入れ方は README の Installation の節にまとまっていて、Mac 向けには Homebrew の1行があります。

```bash
brew install lazydocker
```

Linux と Windows 向けには、バイナリ・Scoop・Chocolatey・`go install` などが並んでいます。この記事の手順を確かめたのは Linux で、macOS・Docker Desktop・OrbStack・Colima では試していません。

compose のファイルがあるフォルダで `lazydocker` と打つと開き、`q` で終わります（`esc` では終わりません）。

## 画面の見方

![lazydocker の画面。左に Project・Services・Standalone Containers・Images・Volumes・Networks の6つの一覧、右に web のログ、一番下の行に PgUp/PgDn: scroll, b: view bulk commands, q: quit, x: menu の案内](https://raw.githubusercontent.com/zun-tools/qiita-content/main/images/lazydocker-index/u01_overview.png)

左に一覧が6つ、右に選んだ物のログやタブ、一番下に押せるキーの案内が出ます。

| キー | 一覧 | 中身 |
|---|---|---|
| `1` | Project | compose のプロジェクト |
| `2` | Services | compose で立てたサービス（web・db など） |
| `3` | Standalone Containers | compose の外で立てたコンテナ |
| `4` | Images | イメージ |
| `5` | Volumes | ボリューム |
| `6` | Networks | ネットワーク |

compose で立てた物は、3番目ではなく2番目の Services に並びます。右のタブは `[` と `]` で替えます。Services のタブは Logs・Stats・Container Env・Container Config・Top の5つです。

## 困ったときのキー

全部を覚えなくても、`x`（`?` でも同じ）でいま押せるキーの一覧が出ます。あとは困りごとから引きます。

| 困りごと | キー | すること・注意 |
|---|---|---|
| 見たい | `enter` → `+` | 右のログへ移ってから `+`。1回目で左の一覧が消えてログが全幅、2回目で Services とログが同じ幅、3回目で開いたときの幅（左の一覧が細い）に戻る |
| 見たい | `]` | Stats のタブで CPU とメモリのグラフ、Container Env で環境変数 |
| 見たい | `1` → `]` | Project の Docker-Compose Config のタブ。`docker compose config` の出力で、build の場所が絶対パスになるなど、compose が既定値を補った後の設定が出る |
| 見たい | `/` | 名前の一部を入れて一覧を絞る |
| 直したい | `r` | 再起動。compose.yml の変更は入らない |
| 直したい | `R` | recreate（設定を直したとき）・rebuild（イメージの中身を変えたとき） |
| 直したい | `c` | `customCommands` に登録したコマンドのメニュー |
| 直したい | `E` | コンテナの中のシェル。bash が無いイメージでは `/bin/sh` |
| 直したい | `w` | 最初のポートの URL をブラウザで開く |
| 直したい | `D` | プロジェクトの一式を止める（`docker compose down`）。down project with volumes はボリュームごと消す |
| 直したい | `U` | プロジェクトの一式を立てる（`docker compose up -d`）。確認の後に走る |
| 片付けたい | `b` | 一括のメニュー。3番目の Standalone Containers では prune exited containers、Images では prune unused images、Volumes では prune unused volumes が並ぶ |

`U` は `D` で止めた後に一式を立て直すときに使い、compose.yml を直した後のサービスには `R` → recreate を押します。`R` は選んだサービスを作り直すメニューで、`r` の再起動では compose.yml の変更は入りません。

`D` のメニューでは、ふつうの down project を選びます。Docker の公式ドキュメントでは、ふつうの down はボリュームを消さず、消すのは `--volumes` を付けたときです。prune は使っていない物をまとめて消す操作で、消したら戻りません。押す前に確認の文を読みます。

## lazydocker とは

lazydocker は、Docker と docker compose をターミナルの1画面で扱う TUI です。コンテナの状態・ログ・CPU とメモリをまとめて見られ、再起動や作り直しをキー1つで走らせます。中で走るのはいつもの `docker` と `docker compose` のコマンドなので、ターミナルで打つ操作と混ぜて使えます。

作者は lazygit と同じ Jesse Duffield で、Go で書かれた MIT ライセンスの OSS です。最新のリリースは v0.25.2（2026-04-19）です。

## 参考リンク

- [lazydocker（公式リポジトリ・入れ方の一覧）](https://github.com/jesseduffield/lazydocker)
- [lazydocker のキー一覧（v0.25.2）](https://github.com/jesseduffield/lazydocker/blob/v0.25.2/docs/keybindings/Keybindings_en.md)
- [lazydocker の設定（v0.25.2）](https://github.com/jesseduffield/lazydocker/blob/v0.25.2/docs/Config.md)
- [サンプル: zun-tools/lazydocker-sample](https://github.com/zun-tools/lazydocker-sample)

## 参考動画

<iframe width="560" height="315" src="https://www.youtube.com/embed/fJPAYU2DhpE" title="YouTube video player" frameborder="0" allowfullscreen></iframe>
