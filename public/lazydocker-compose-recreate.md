---
title: 'lazydocker で docker compose の落ちたコンテナを直す（r で直らなければ R → recreate）'
tags:
  - lazydocker
  - Docker
  - docker-compose
  - Rails
  - PostgreSQL
private: true
updated_at: '2026-11-19T07:28:22+09:00'
id: 3834b64259d4b101cd22
organization_url_name: null
slide: false
ignorePublish: false
---

`docker compose up -d` で「Started」と出たのにブラウザから接続できないとき、lazydocker を開けば、どのサービスが落ちてログに何が出たかを1画面で追えます。compose.yml を直した後は、`r`（restart）ではなく `R` → recreate でコンテナを作り直すと、直した設定が入ります。

`up -d` は、起動直後に exit するサービスがあっても `Started` を出して戻るので、出力だけでは落ちたことに気づけません。原因を直した後も、`docker compose restart` は compose.yml の変更を反映しないため、同じエラーが続きます。

参考動画は記事の最後に紹介しています。

## 環境

- lazydocker v0.25.2（Linux arm64 の公式バイナリ）
- Docker Engine 29.6.1・Docker Compose v5.2.0
- Linux（aarch64）。確認日は 2026-11-12
- macOS・Docker Desktop・OrbStack・Colima では試していません
- サンプルのアプリは Rails 8.1 と Postgres 17 で、どちらもコンテナの中で動きます

## 完成形

compose.yml のあるディレクトリで起動します。

```bash
lazydocker
```

1. `2` で Services へ移り、↑↓ で Exited のサービスを選ぶ
2. 右の Logs で原因を探し、compose.yml を直す
3. 同じサービスで `R` → `recreate` → `enter`
4. `]` を2回押して Container Env タブへ替え、直した値が入ったかを見る

:::note alert
`r` が走らせる `docker compose restart` は、compose.yml の変更を読み込みません。compose.yml を直したら、`R` → recreate で作り直します。
:::

## 1. lazydocker を入れて、壊れたサンプルを立てる

Mac なら、公式の README にある Homebrew の1行で入ります。

```bash
brew install lazydocker
```

Linux と Windows 向けには、README の Installation の節にバイナリ・Scoop・Chocolatey・`go install` などが並んでいます。要件は、lazydocker の README では Docker 29.0.0 以上です。手元の版は `docker version` で確認できます。

サンプルは Rails（web）と Postgres（db）の compose 構成で、DB の接続先を1か所わざと間違えてあります。

```bash
git clone https://github.com/zun-tools/lazydocker-sample.git memo-app
cd memo-app
docker compose up -d
```

ディレクトリ名を `memo-app` にすると、compose のプロジェクト名がこの記事の出力と揃います。

```text
[+] up 4/4
 ✔ Network memo-app_default Created
 ✔ Volume memo-app_pgdata   Created
 ✔ Container memo-app-db-1  Healthy
 ✔ Container memo-app-web-1 Started
```

この状態で `http://localhost:3000/` を開いても接続できません。

## 2. lazydocker で、落ちたサービスとログを見る

同じディレクトリで `lazydocker` を起動します。compose のサービスは2番目の Services パネルに並びます。3番目の Standalone Containers は compose の外で立てたコンテナ用なので、web と db は出てきません。

![lazydocker の Services パネル。db の行は running (healthy) で 5432/tcp、web の行は exited (1) と出ている](https://raw.githubusercontent.com/zun-tools/qiita-content/main/images/lazydocker-compose-recreate/p01_services_web_exited_services_rows.png)

web を選ぶと、右の Logs に最後のログが出ます。

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

サンプルの Dockerfile は `CMD` で `bin/rails db:create && bin/rails server -b 0.0.0.0` を走らせます。最初の `db:create` が DB に接続できず失敗するので、コンテナはサーバーを起動する前に exit 1 で終わります。`docker logs` と `docker compose ps` を打ち分ける必要はなく、状態とログは同じ画面にあります。

## 3. 接続先をサービス名 `db` に直す

コンテナの中の `127.0.0.1` は、そのコンテナ自身のループバックです。Postgres は db という別のコンテナで動いているため、web から `127.0.0.1:5432` に接続しても相手がいません。compose のサービスは同じネットワークにつながり、サービス名で名前解決できるので、接続先を `db` にします。

サンプルには修正済みの `compose.fixed.yml` があります。別のターミナルで差分を確かめてから置き換えます。

```bash
diff -u compose.yml compose.fixed.yml
cp compose.fixed.yml compose.yml
```

```diff
-      DATABASE_URL: postgres://postgres:postgres@127.0.0.1:5432/memo_development
+      DATABASE_URL: postgres://postgres:postgres@db:5432/memo_development
```

## 4. `r` で再起動しても直らない

lazydocker で web を選んだまま `r` を押しても、web は exited (1) のままでした。ログには同じ接続エラーが積み増されるだけです。`]` を2回押して Container Env タブを開くと、値が古いままなのが分かります。

![lazydocker の Container Env タブ。DATABASE_URL が postgres://postgres:postgres@127.0.0.1:5432/memo_development のまま](https://raw.githubusercontent.com/zun-tools/qiita-content/main/images/lazydocker-compose-recreate/p07_env_after_r.png)

`r` の中身は `docker compose restart` で、Docker の公式ドキュメントにはこうあります。

> If you make changes to your `compose.yml` configuration, these changes are not reflected after running this command.
> （`compose.yml` の設定を変えても、このコマンドを実行した後には反映されません）

## 5. `R` → recreate でコンテナを作り直す

web を選んだまま `R` を押すと、restart の選択肢のメニューが開きます。各行には、実際に走るコマンドも出ます。

![R で出たメニュー。restart は docker compose -p memo-app restart web、recreate は docker compose -p memo-app up -d --force-recreat まで見え、rebuild は docker compose -p memo-app up -d --build web、最後に cancel](https://raw.githubusercontent.com/zun-tools/qiita-content/main/images/lazydocker-compose-recreate/p08_R_menu.png)

| 選ぶもの | 走るコマンド | 何が起きるか | compose.yml の変更 |
|---|---|---|---|
| restart | `docker compose restart` | 同じ設定のまま立ち上げ直す | 入らない |
| recreate | `docker compose up -d --force-recreate` | コンテナを作り直す | 入る |
| rebuild | `docker compose up -d --build` | イメージを作り直してから立てる | 入る（Dockerfile やソースの変更も） |

`recreate` を選んで `enter` を押すと、web も running になりました。

![recreate の後の Services パネル。db は running (healthy)、web は running で 3000->3000 と出ている](https://raw.githubusercontent.com/zun-tools/qiita-content/main/images/lazydocker-compose-recreate/p09_after_recreate_running_services_rows.png)

Container Env タブの値:

```text
DATABASE_URL:
postgres://postgres:postgres@db:5432/memo_development
```

ソースを変えたときは rebuild の出番です。このサンプルの Dockerfile は `COPY . .` でソースをイメージに焼き込む作りです。`app/views/memos/index.html.erb` の `<h1>Memos</h1>` を書き換えたときは、`R` → rebuild で画面に反映されました。

lazydocker を開いてから外で触ったのは、compose.yml の1行だけです。サービスの特定、ログと環境変数の確認、作り直しは、すべてキー操作で済みました。

## 6. ブラウザで確かめる

web を選んで `w` を押すと、最初のポートの URL（ここでは `http://localhost:3000/`）をブラウザで開きます。今度は Rails から応答が返り、エラー画面にこう出ます。

```text
ActiveRecord::PendingMigrationError
Migrations are pending. To resolve this issue, run:
bin/rails db:migrate
```

DB 接続は通ったので、残りはマイグレーションです。別のターミナルで流すと、メモの一覧が表示されます。

```bash
docker compose exec web bin/rails db:migrate
```

```text
== 20261112114337 CreateMemos: migrating ======================================
-- create_table(:memos)
   -> 0.0023s
== 20261112114337 CreateMemos: migrated (0.0023s) =============================
```

lazydocker の終了は `q` です。`esc` では終わりません。

## lazydocker について

Docker と docker compose をターミナルの1画面で扱う TUI です。作者は lazygit と同じ Jesse Duffield で、Go 製・MIT ライセンスの OSS です。裏で走るのは通常の `docker` と `docker compose` なので、シェルからの操作と混ぜて使えます。

## まとめ

| 症状 | 原因 | 対処 |
|---|---|---|
| `up -d` で `Started` と出たのに接続できない | サービスが起動直後に exit している | Services で Exited のサービスを選び、Logs を読む |
| ログに `127.0.0.1` の 5432 番への `Connection refused` | コンテナ内の `127.0.0.1` はそのコンテナ自身 | 接続先をサービス名（ここでは `db`）にする |
| compose.yml を直して `r` しても Exited のまま。Container Env も古い値 | `docker compose restart` は compose.yml の変更を反映しない | `R` → recreate |
| ビューなどのソースを変えた | ソースが `COPY . .` でイメージに焼き込まれている | `R` → rebuild |
| `ActiveRecord::PendingMigrationError` | マイグレーションが未実行 | `docker compose exec web bin/rails db:migrate` |

## 参考リンク

- [lazydocker（公式リポジトリ・入れ方の一覧）](https://github.com/jesseduffield/lazydocker)
- [lazydocker のキー一覧（v0.25.2）](https://github.com/jesseduffield/lazydocker/blob/v0.25.2/docs/keybindings/Keybindings_en.md)
- [docker compose restart（compose.yml の変更は反映されない）](https://docs.docker.com/reference/cli/docker/compose/restart/)
- [Compose のネットワーク（サービス名で呼ぶ）](https://docs.docker.com/compose/how-tos/networking/)
- [サンプル: zun-tools/lazydocker-sample（壊れた compose.yml・compose.fixed.yml）](https://github.com/zun-tools/lazydocker-sample)

## 参考動画

<iframe width="560" height="315" src="https://www.youtube.com/embed/fJPAYU2DhpE" title="YouTube video player" frameborder="0" allowfullscreen></iframe>

---

シリーズの他の記事: https://qiita.com/zundamon-tools/private/975c61779740d6113fd7
