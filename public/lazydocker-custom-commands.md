---
title: lazydocker の customCommands で長い docker compose exec をメニューから呼ぶ
tags:
  - lazydocker
  - Docker
  - docker-compose
  - Rails
  - PostgreSQL
private: true
updated_at: '2026-11-19T07:27:26+09:00'
id: e92d6b5d31c884fdd368
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

`docker compose exec web bin/rails db:migrate` のような長いコマンドを、lazydocker の設定ファイルの `customCommands` に登録すると、サービスを選んで `c` を押したメニューから走らせられます。`serviceNames` で、コマンドを出すサービスも絞れます。

参考動画は記事の最後に紹介しています。

## 環境

- lazydocker v0.25.2（Linux arm64 の公式バイナリ）
- Docker Engine 29.6.1・Docker Compose v5.2.0（`docker-compose` コマンドは入っていない環境）
- Linux（aarch64）。確認日は 2026-11-12
- macOS・Docker Desktop・OrbStack・Colima では試していません
- サンプルのアプリは Rails 8.1 と Postgres 17 で、どちらもコンテナの中で動きます

## 完成形

lazydocker の設定ファイル `config.yml` に、次の内容を書きます。

```yaml:config.yml
customCommands:
  services:
    - name: rails db:migrate
      attach: true
      command: '{{ .DockerCompose }} exec {{ .Service.Name }} bin/rails db:migrate'
      serviceNames: [web]
    - name: rails console
      attach: true
      command: '{{ .DockerCompose }} exec {{ .Service.Name }} bin/rails console'
      serviceNames: [web]
    - name: psql
      attach: true
      command: '{{ .DockerCompose }} exec {{ .Service.Name }} psql -U postgres'
      serviceNames: [db]
```

設定ファイルの場所は OS ごとに決まっています。

| OS | 場所 |
|---|---|
| macOS | `~/Library/Application Support/jesseduffield/lazydocker/config.yml` |
| Linux | `~/.config/lazydocker/config.yml` |
| Windows | `%AppData%\lazydocker\config.yml` |

サンプルの `lazydocker/config.yml` はこれと同じ中身なので、上の場所の設定ファイルへ写せば書き写す手間を省けます。手元のプロジェクトに合わせて書き換えるのは、`serviceNames` のサービス名と、`exec` の後ろのコマンドの2か所です。保存したら、lazydocker を一度閉じて開き直します。

## 1. サンプルを立てる

lazydocker は、Mac なら公式の README の手順どおり `brew install lazydocker` で入ります。ほかの OS の入れ方は README の Installation の節にあります。lazydocker の README は、要件を Docker 29.0.0 以上としています。

練習には、Rails の web と Postgres の db を `docker compose` で立てるサンプルを使います。サンプルの `compose.yml` は接続先をわざと間違えた版なので、直した版の `compose.fixed.yml` で置き換えてから立てます。

```bash
git clone https://github.com/zun-tools/lazydocker-sample.git memo-app
cd memo-app
cp compose.fixed.yml compose.yml
docker compose up -d
```

フォルダ名を `memo-app` にしておくと、画面に出る名前がこの記事と同じになります。この時点で `http://localhost:3000/` を開くと、Rails のエラー画面に `Migrations are pending.`（マイグレーションがまだ）と出ます。この状態から、マイグレーションをメニューで流します。

## 2. lazydocker から設定ファイルを開く

同じフォルダで `lazydocker` と打ち、`1` で Project のパネルへ移って `e` を押します。設定ファイルがエディタで開きます。開くエディタは環境変数 `$VISUAL`、無ければ `$EDITOR` で決まり、どちらも無ければ vi です。

設定ファイルを初めて開いたときは、中身は空でした。ここに完成形の `customCommands` を書いて保存し、エディタを閉じます。閉じると次の行が出るので、`enter` で lazydocker の画面へ戻ります。

```text
Press enter to return to lazydocker (this prompt can be disabled in your config by setting `gui.returnImmediately: true`)
```

文にあるとおり、設定で `gui.returnImmediately: true` にすると、この確認を省けます。

## 3. 各項目の意味

| 項目 | 意味 |
|---|---|
| `name` | メニューに出る名前 |
| `attach` | `true` なら lazydocker の画面からコマンドの画面へ切り替えて走らせる（対話できる）。`false` なら裏で走らせる |
| `command` | 走らせるコマンド。Go のテンプレートで書ける |
| `serviceNames` | このコマンドを出すサービス |

`{{ .DockerCompose }}` の既定は `docker compose` です。この環境では、プロジェクト名を付けた `docker compose -p memo-app` に展開されました。`{{ .Service.Name }}` には、選んでいるサービスの名前が入ります。

`customCommands` の下の `services` は、Services のパネルで使うコマンドという意味です。コマンドごとにキーを割り当てる欄は無く、呼ぶのは `c` のメニューからです。

## 4. 開き直して、`c` で rails db:migrate を流す

保存した直後に web を選んで `c` を押しても、メニューには `cancel` しか出ませんでした。`q` で lazydocker を一度閉じ、もう一度 `lazydocker` と打って開き直します。

`2` で Services へ移り、web を選んで `c` を押すと、web 用に登録した2つだけが出ます。

![web で c を押したときの Custom Command のメニュー。rails console と rails db:migrate が docker compose -p memo-app exec web bin/ から始まるコマンドと並び、最後に cancel](https://raw.githubusercontent.com/zun-tools/qiita-content/main/images/lazydocker-custom-commands/c02_web_c_menu.png)

このときのメニューは config.yml に書いた順とは逆で、`rails console` が上に並びました。下のキーで `rails db:migrate` を選んで `enter` を押すと、画面がコマンドの出力に切り替わります。

```text
+ docker compose -p memo-app exec web bin/rails db:migrate

== 20261112114337 CreateMemos: migrating ======================================
-- create_table(:memos)
   -> 0.0030s
== 20261112114337 CreateMemos: migrated (0.0030s) =============================

Press enter to return to lazydocker (this prompt can be disabled in your config by setting `gui.returnImmediately: true`)
```

先頭の `+` の行が、実際に走ったコマンドです。`enter` で lazydocker に戻り、ブラウザを開き直すと、メモの一覧の画面が出ました。公式の設定の説明文には `docker-compose` と書いた行もありますが、既定値は `docker compose` なので、`docker-compose` コマンドが無いこの環境でもそのまま通りました。

## 5. 対話するコマンドも同じメニューから開く

`attach: true` にしたコマンドは、対話の画面もそのまま使えます。web で `c` を押して `rails console` を選ぶと、Rails のコンソールが開きます。

```text
+ docker compose -p memo-app exec web bin/rails console

Loading development environment (Rails 8.1.4.1)
memo-app(dev):001> Memo.count
  Memo Count (17.7ms)  SELECT COUNT(*) FROM "memos" /*application='MemoApp'*/
=> 0
memo-app(dev):002> exit
```

`exit` で抜けると、また `enter` を求める行が出ます。押せば lazydocker の画面に戻ります。

db を選んで `c` を押すと、db 用に登録した `psql` だけが出ます。

![db で c を押したときの Custom Command のメニュー。psql と docker compose -p memo-app exec db psql -U postgre までのコマンドの行、その下に cancel](https://raw.githubusercontent.com/zun-tools/qiita-content/main/images/lazydocker-custom-commands/c08_db_c_menu.png)

`psql -U postgres` で入った先は `postgres` のデータベースです。アプリのテーブルを見るには、`\c` でつなぎ替えてから `\dt` を打ちます。

```text
postgres=# \c memo_development
You are now connected to database "memo_development" as user "postgres".
memo_development=# \dt
                List of relations
 Schema |         Name         | Type  |  Owner
--------+----------------------+-------+----------
 public | ar_internal_metadata | table | postgres
 public | memos                | table | postgres
 public | schema_migrations    | table | postgres
(3 rows)
```

`\q` で抜け、`enter` で lazydocker に戻ります。compose.yml にあるパスワードの `postgres` は、このサンプル専用の値です。

## 6. メニューに出ないときに確かめる2点

- `serviceNames` に書いた名前が、compose.yml のサービス名と同じか
- 設定を保存した後に、lazydocker を開き直したか

## lazydocker について

lazydocker は、Docker と docker compose をターミナルの1画面で扱う TUI です。作者は lazygit と同じ Jesse Duffield で、Go で書かれた MIT ライセンスの OSS です。公式の README は、必要な情報を1つの窓に集め、よく使うコマンドをキー1つで呼べて、自分のコマンドも足せる道具だと説明しています。

## まとめ

- 長いコマンドは `customCommands` の `services` に書くと、サービスを選んで `c` のメニューから走らせられる
- `serviceNames` で、コマンドを出すサービスを絞れる。このサンプルでは web に migrate と console、db に psql を出した
- `attach: true` なら、rails console や psql の対話の画面もそのまま使える。終わると `enter` で lazydocker に戻る
- 保存しただけではメニューに出ない。`q` で閉じて開き直す

## 参考リンク

- [lazydocker の設定（customCommands・設定ファイルの場所）](https://github.com/jesseduffield/lazydocker/blob/v0.25.2/docs/Config.md)
- [lazydocker のキー一覧（v0.25.2）](https://github.com/jesseduffield/lazydocker/blob/v0.25.2/docs/keybindings/Keybindings_en.md)
- [docker compose exec](https://docs.docker.com/reference/cli/docker/compose/exec/)
- [サンプル: zun-tools/lazydocker-sample（lazydocker/config.yml）](https://github.com/zun-tools/lazydocker-sample)

## 参考動画

<iframe width="560" height="315" src="https://www.youtube.com/embed/fJPAYU2DhpE" title="YouTube video player" frameborder="0" allowfullscreen></iframe>

---

