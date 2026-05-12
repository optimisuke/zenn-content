# zenn-content

Zenn に投稿するためのリポジトリ

# 管理

最初はブラウザから画像をアップロードしてたけど、途中から images フォルダに置くようになった。

images フォルダにある画像も最初は、フォルダ作って入れていたけど、途中から image paste のプラグインを入れて、時刻でファイルを作るようにした。

# 予約公開

`scheduled-publish` ブランチに投稿したい記事を push しておくと、GitHub Actions が毎日 20:00 JST に `main` へマージする。
Zenn は `main` の更新を検知して投稿する。

手動で実行したい場合は、GitHub Actions の `Scheduled publish` から `Run workflow` を押す。
別のブランチを使いたい場合は、手動実行時の `source_branch` にブランチ名を指定する。
