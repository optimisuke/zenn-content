# zenn-content

Zenn に投稿するためのリポジトリ

# 管理

最初はブラウザから画像をアップロードしてたけど、途中から images フォルダに置くようになった。

images フォルダにある画像も最初は、フォルダ作って入れていたけど、途中から image paste のプラグインを入れて、時刻でファイルを作るようにした。

# 予約公開

Front Matter に `published: true` と `published_at: YYYY-MM-DD HH:mm`（JST）を指定して `main` に push する。
Zenn が指定時刻に自動公開する。
