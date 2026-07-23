# zenn-content

Zenn に投稿するためのリポジトリ

# 管理

最初はブラウザから画像をアップロードしてたけど、途中から images フォルダに置くようになった。

images フォルダにある画像も最初は、フォルダ作って入れていたけど、途中から image paste のプラグインを入れて、時刻でファイルを作るようにした。

# 予約公開

Front Matter に `published: true` と `published_at: YYYY-MM-DD HH:mm`（JST）を指定して `main` に push する。
Zenn が指定時刻に自動公開する。

# 記事ネタ

未公開記事として置いていたもの。書きたくなったら、最新情報を調べ直して新しく書く。

- FinOps 完全に理解した
  - FinOps とは何かを自分なりに整理する
  - 参考: https://www.finops.org/introduction/what-is-finops/
- RFID 入門
- NestJS 入門
- CQRS 完全に理解した
- TypeScript 完全に理解した
- LangChain.js × Next.js テンプレートの Retrieval 解説
  - 公式テンプレートの RAG 処理を、コードを追いながら説明する
  - LangChain の入門後、実際のアプリ構成を理解したい人向け
  - 以前のテンプレートなので、書くなら現行 LangChain.js との違いを確認する
  - 参考: https://github.com/langchain-ai/langchain-nextjs-template
