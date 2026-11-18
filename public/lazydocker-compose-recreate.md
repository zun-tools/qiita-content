---
title: 'lazydocker で docker compose の落ちたコンテナを直す（r で直らなければ R → recreate）'
tags:
  - lazydocker
  - Docker
  - docker-compose
  - Rails
  - PostgreSQL
private: true
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

`docker compose up -d` で「Started」と出たのにブラウザでつながらないとき、lazydocker を開くと、どのサービスが落ちてログに何が出たかを1画面で読めます。compose.yml を直した後は、`r`（restart）ではなく `R` → recreate でコンテナを作り直すと、直した設定が入ります。

参考動画は記事の最後に紹介しています。

## 環境

- lazydocker v0.25.2（Linux arm64 の公式バイナリ）
- Docker Engine 29.6.1・Docker Compose v5.2.0
- Linux（aarch64）。確認日は 2026-11-12
- macOS・Docker Desktop・OrbStack・Colima では試していません
- サンプルのアプリは Rails 8.1 と Postgres 17 で、どちらもコンテナの中で動きます

## 完成形

compose.yml のあるフォルダで開きます。

```bash
lazydocker
```

1. `2` で Services へ移り、上下のキーで Exited のサービスを選ぶ
2. 右の Logs を読んで原因を探し、compose.yml を直す
3. 同じサービスで `R` を押し、`recreate` を選んで `enter`
4. `]` を2回押してタブを Container Env へ替え、直した値が入ったかを見る

`r` の再起動では、compose.yml の変更は入りません。

## 1. lazydocker を入れて、壊れたサンプルを立てる

Mac では、公式の README の手順なら Homebrew の1行で入ります。

```bash
brew install lazydocker
```

Linux と Windows は、README の Installation の節にバイナリ・Scoop・Chocolatey・`go install` などの入れ方が並んでいます。lazydocker の README は、要件を Docker 29.0.0 以上としています。手元の版は `docker version` で見られます。

練習には、接続先を1か所わざと間違えたサンプルを使います。Rails の web と Postgres の db を、`docker compose` で立てる構成です。

```bash
git clone https://github.com/zun-tools/lazydocker-sample.git memo-app
cd memo-app
docker compose up -d
```

フォルダ名を `memo-app` にしておくと、画面に出る名前がこの記事と同じになります。

```text
[+] up 4/4
 ✔ Network memo-app_default Created
 ✔ Volume memo-app_pgdata   Created
 ✔ Container memo-app-db-1  Healthy
 ✔ Container memo-app-web-1 Started
```

web も `Started` と出ますが、`http://localhost:3000/` を開くとつながりません。

## 2. lazydocker で、落ちたサービスとログを見る

同じフォルダで `lazydocker` と打ちます。左に一覧が6つ、右に選んだ物のログが出ます。compose で立てた web と db は、2番目の Services に並びます。3番目の Standalone Containers は compose の外で立てたコンテナの一覧で、web と db はここには出ません。

`2` で Services へ移ると、db は動いていて、web だけが止まっています。

![lazydocker の Services パネル。db の行は running (healthy) で 5432/tcp、web の行は exited (1) と出ている](https://raw.githubusercontent.com/zun-tools/qiita-content/main/images/lazydocker-compose-recreate/p01_services_web_exited_services_rows.png)

上下のキーで web を選ぶと、右の Logs に web が最後に出したログが出ます。

```text
connection to server at "127.0.0.1", port 5432 failed: Connection refused
    Is the server running on that host and accepting TCP/IP connections?
Couldn't create 'memo_development' database. Please check your configuration.
bin/rails aborted!
ActiveRecord::ConnectionNotEstablished: connection to server at "127.0.0.1",
port 5432 failed: Connection refused (ActiveRecord::ConnectionNotEstablished)
    Is the server running on that host and accepting TCP/IP connections?

Caused by:
PG::ConnectionBad: connection to server at "127.0.0.1", port 5432 failed:
Connection refused (PG::ConnectionBad)
    Is the server running on that host and accepting TCP/IP connections?

Tasks: TOP => db:create
```

web は起動の最初に `bin/rails db:create` を走らせ、`127.0.0.1` の 5432 番につなげずに終了しています。`docker logs` や `docker compose ps` を打たなくても、止まったサービスとその理由が同じ画面に並びます。

## 3. 接続先をサービス名 `db` に直す

コンテナの中の `127.0.0.1` は、そのコンテナ自身を指します。Postgres は別のコンテナ（db）で動いているので、web の中の `127.0.0.1` には何もいません。compose のサービスは同じネットワークにつながり、サービス名で互いを呼べます。そこで接続先を `db` に直します。

サンプルには直した版の `compose.fixed.yml` があります。別のターミナルで、差分を見てから置き換えます。

```bash
diff -u compose.yml compose.fixed.yml
cp compose.fixed.yml compose.yml
```

```diff
-      DATABASE_URL: postgres://postgres:postgres@127.0.0.1:5432/memo_development
+      DATABASE_URL: postgres://postgres:postgres@db:5432/memo_development
```

## 4. `r` で再起動しても直らない

lazydocker に戻り、web を選んだまま `r` を押します。web は exited (1) のままで、ログには同じ接続エラーの行が増えるだけでした。

`]` を2回押すと、右のタブが Logs → Stats → Container Env と進みます。Container Env は、コンテナに入っている環境変数の一覧です。

![lazydocker の Container Env タブ。DATABASE_URL が postgres://postgres:postgres@127.0.0.1:5432/memo_development のまま](https://raw.githubusercontent.com/zun-tools/qiita-content/main/images/lazydocker-compose-recreate/p07_env_after_r.png)

compose.yml は直したのに、`DATABASE_URL` は `127.0.0.1` のままです。lazydocker の `r` が走らせるのは `docker compose restart` で、Docker の公式ドキュメントにはこうあります。

> If you make changes to your `compose.yml` configuration, these changes are not reflected after running this command.
> （`compose.yml` の設定を変えても、このコマンドを実行した後には反映されません）

`r` はコンテナを同じ設定のまま立ち上げ直すので、環境変数の変更は入りません。

## 5. `R` → recreate でコンテナを作り直す

web を選んだまま、大文字の `R` を押します。restart の選択肢のメニューが出て、各行に実際に走るコマンドも並びます。

![R で出たメニュー。restart は docker compose -p memo-app restart web、recreate は docker compose -p memo-app up -d --force-recreat まで見え、rebuild は docker compose -p memo-app up -d --build web、最後に cancel](https://raw.githubusercontent.com/zun-tools/qiita-content/main/images/lazydocker-compose-recreate/p08_R_menu.png)

| 選ぶもの | 走るコマンド | 何が起きるか | compose.yml の変更 |
|---|---|---|---|
| restart | `docker compose restart` | 同じ設定のまま立ち上げ直す | 入らない |
| recreate | `docker compose up -d --force-recreate` | コンテナを作り直す | 入る |
| rebuild | `docker compose up -d --build` | イメージを作り直してから立てる | 入る（Dockerfile やソースの変更も） |

下のキーで `recreate` へ移り、`enter` を押します。作り直しが終わると、web も running に変わりました。

![recreate の後の Services パネル。db は running (healthy)、web は running で 3000->3000 と出ている](https://raw.githubusercontent.com/zun-tools/qiita-content/main/images/lazydocker-compose-recreate/p09_after_recreate_running_services_rows.png)

Container Env のタブでは、接続先が `db` に替わっています。

```text
DATABASE_URL:
postgres://postgres:postgres@db:5432/memo_development
```

compose.yml などの設定を直したときは recreate、ソースなどイメージの中身を変えたときは rebuild を選びます。このサンプルは Dockerfile の `COPY . .` でソースをイメージに焼き込む作りです。`app/views/memos/index.html.erb` の見出し `<h1>Memos</h1>` を書き換えたときは、`R` → rebuild で画面に変更が出ました。

compose.yml を1行直したほかは、落ちたサービスを探すのも、ログと環境変数を読むのも、作り直すのも lazydocker のキーで済みました。

## 6. ブラウザで確かめる

web を選んで `w` を押すと、最初のポートの URL（ここでは `http://localhost:3000/`）をブラウザで開きます。今度はアプリが返事をし、Rails のエラー画面に次の文が出ます。

```text
ActiveRecord::PendingMigrationError
Migrations are pending. To resolve this issue, run:
bin/rails db:migrate
```

データベースにつながり、テーブルを作る手順（マイグレーション）がまだ、という表示です。別のターミナルでマイグレーションを流すと、メモの一覧の画面が出ます。

```bash
docker compose exec web bin/rails db:migrate
```

```text
== 20261112114337 CreateMemos: migrating ======================================
-- create_table(:memos)
   -> 0.0023s
== 20261112114337 CreateMemos: migrated (0.0023s) =============================
```

lazydocker は `q` で終わります（`esc` では終わりません）。

## lazydocker について

lazydocker は、Docker と docker compose をターミナルの1画面で扱う TUI です。作者は lazygit と同じ Jesse Duffield で、Go で書かれた MIT ライセンスの OSS です。中で走るのはいつもの `docker` と `docker compose` のコマンドなので、ターミナルで打つ操作と混ぜて使えます。

## まとめ

- lazydocker の Services で落ちたサービスを選ぶと、状態とログが同じ画面に出る
- コンテナの中の `127.0.0.1` はそのコンテナ自身で、別のサービスはサービス名（ここでは `db`）で呼ぶ
- `r` は `docker compose restart` を走らせるだけで、compose.yml の変更は入らない。Container Env のタブで値を見ると分かる
- 設定を直したら `R` → recreate、イメージの中身を変えたら `R` → rebuild を選ぶ

## 参考リンク

- [lazydocker（公式リポジトリ・入れ方の一覧）](https://github.com/jesseduffield/lazydocker)
- [lazydocker のキー一覧（v0.25.2）](https://github.com/jesseduffield/lazydocker/blob/v0.25.2/docs/keybindings/Keybindings_en.md)
- [docker compose restart（compose.yml の変更は反映されない）](https://docs.docker.com/reference/cli/docker/compose/restart/)
- [Compose のネットワーク（サービス名で呼ぶ）](https://docs.docker.com/compose/how-tos/networking/)
- [サンプル: zun-tools/lazydocker-sample（壊れた compose.yml・compose.fixed.yml）](https://github.com/zun-tools/lazydocker-sample)

## 参考動画

<iframe width="560" height="315" src="https://www.youtube.com/embed/fJPAYU2DhpE" title="YouTube video player" frameborder="0" allowfullscreen></iframe>

---

