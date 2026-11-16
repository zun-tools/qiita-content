# qiita-content

Qiita（[zundamon-tools](https://qiita.com/zundamon-tools)）の記事リポジトリ。
`public/` の記事を main へ push すると、GitHub Actions（`.github/workflows/publish.yml`）が qiita-cli で投稿・更新する。

- 必要な secret: `QIITA_TOKEN`（Qiita の個人用アクセストークン・read_qiita / write_qiita）
- 投稿の管理は制作リポ側の台帳で行う。このリポへ手で記事を足さない
