---
title: "これだけは知っとこう WebAssembly (wasm)"
emoji: "🦀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [webassembly, wasm]
published: true
---

# はじめに

WebAssembly (wasm) の情報って分散していて、全体像を掴むの難しくないですか？
なんか、フロントエンドの処理が速くなるやつやなと思って調べてると「ランタイムが〜」、「コンテナで動かすには〜」「Rust の環境セットアップは〜」みたいな記事が出てきて、混乱しながら、「やっぱわからん」を繰り返してました。
真面目に、調べたり本を読んだりして、ある程度、全体像を理解してきたので、まとめてみることにしました。
「WebAssembly 興味あるけど、雰囲気しかわからん」「Rust で説明されてもわからん」って人に届け 🦀

# WebAssembly とは？

WebAssembly は、ブラウザで JavaScript 以外の言語を実行するための仕様・環境です。
と見せかけて、ブラウザ以外でも動きます。Node.js で動いたり、専用のランタイムで動いたり、Kubernetes 等のコンテナ環境で動いたりもします。
なんでもありで、フロントエンドでも、バックエンドでも、ほぼほぼどこでも動きます。
私の 2024/11 時点での理解だと、WebAssembly は、いろんな言語を動かせる高速でクロスプラットフォーム（ポータブル）でセキュアな仮想環境（サンドボックス）です。
以下に示す特徴のため、ブラウザだけでなく様々な場所で使えるようになってきてます。
エコシステムの成長速度が速いので、情報がまとまってないのかなと思います。
以下でざっくり、まとめていきます。

# WebAssembly の特徴

まずは、いくつか特徴を挙げてみます。

- 速い: 低レベルのバイナリ形式で、CPU が直接実行できるように設計されてます。
- 安全: サンドボックス化されており、ネットワークやファイルアクセスなど、必要な権限が明示的に与える必要があります。
- 多言語: いろんな言語で WebAssembly へのコンパイルが対応されてます。
- クロスプラットフォーム（ポータブル）: どこでも動くので、WebAssembly を使うことで、サーバーやクラウド上での実行環境を簡単に構築できます。

# WebAssembly vs

他との違いにも触れてみます。
理解できてない部分もありつつ、わかりやすさ重視で言い切ってます。詳細は、一次情報等参照です。

- JavaScript: WebAssembly はコンパイルするので、JavaScript よりも高速です。
- Java: Java は WebAssembly と同じく仮想環境で実行されますが、WebAssembly は多言語に対応しています。
- コンテナ: WebAssembly は OS 側を意識しなくても動作するのでよりセキュアです。ただ、コンテナのエコシステムにも WebAssembly が組み込まれていってます。
- アセンブリ: WebAssembly は OC 依存・CPU 依存がありません。

# WebAssembly の用語

調べるときに、よく出てくる用語をメモしておきます。

- Wasm: WebAssembly の略です
- WASI: WebAssembly System Interface (WebAssembly 用システムインターフェース) です。WebAssembly から OS の機能にアクセスするための API 群です。
- WAT: WebAssembly Text Format (WebAssembly テキストフォーマット) です。WebAssembly を記述するテキスト形式です。
- WIT: WebAssembly Interface Types です。WebAssembly モジュールとホスト環境（JavaScript や他の WebAssembly モジュールなど）の間でデータをやり取りするためのインターフェース定義形式です。
- バイトコード: WebAssembly のコード形式で、CPU が直接実行するマシンコードに近い中間コードです。.wasm 形式で、ブラウザがこれを解釈して実行します。
- ランタイム: WebAssembly を動かすための実装です。WebAssembly は OS に依存しないので、OS の機能を呼び出せるようになっています。
- V8: Google Chrome 用の JavaScript エンジンです。V8 は WebAssembly でも使われてます。

# WebAssembly 利用の流れ

ブラウザで動かす時と、専用ランタイムで動かす時とで、WebAssembly の利用方法が異なります。

## ブラウザで動かす場合

1. C や Rust 等で書いたコードを WebAssembly にコンパイル。
2. JavaScript からコンパイルされた WebAssembly を読み込み実行。

webpack やライブラリ・フレームワークを使うと、JavaScript と WebAssembly をいい感じに繋いでくれる。ただ、ブラックボックスになりがち。
WebAssembly から JavaScript 経由で DOM 操作も可能。

## 専用ランタイムで動かす場合

1. C や Rust 等で書いたコードを WebAssembly にコンパイル（同じ）。
2. Wasmtime 等ならそのまま実行（`wasmtime hello.wasm`）。

Node.js ならブラウザと同様 JavaScript から読み込み実行。

# WebAssembly のユースケース

以下のようなユースケースがあります。

## ブラウザ

- ゲーム
- グラフィックス処理

## ブラウザ以外

- エッジコンピューティング
- サーバーレスアーキテクチャ
- 組み込み
- プラグイン用ランタイム

# WebAssembly の事例

## ブラウザ

- Unity や Unreal Engine などのゲームエンジン
- Figma
- Google Earth
- PDFyamada ブーわー
- AutoCAD

## ブラウザ以外

事例というよりサポートされてるプラットフォームだけど

- Fastly
- Cloudflare Workers
- Kubernetes
- Envoy Proxy

# おわりに

かなり雑にまとめてみました。全体像伝わればと思います。
多くの記事が、ブラウザで動かす前提かブラウザ以外で動かす前提のどちらかで書かれてる気がしたので、この記事で全体像を知った上でいろんな情報にアクセスしてもらえると良いかなと思います。
フィードバックあればウェルカムです 🦀

# 参考

## WebAssembly のランタイム実装

はじめにと Wasm の概要は必読

https://zenn.dev/skanehira/books/writing-wasm-runtime-in-rust

## コンテナ・サーバレス

この記事と YouTube むちゃくちゃおすすめ。WebAssembly がブラウザだけじゃないってことを理解できた。

https://zenn.dev/koduki/articles/9f86d03cd703c4

## ブラウザ x Rust x WebAssembly

Rust を使って、JavaScript から WebAssembly 呼び出しや WebAssembly から JavaScript 呼び出しを試しながらゲームを作れる
Rust わからないと辛いけど、

https://www.amazon.co.jp/Rust%E3%81%A8WebAssembly%E3%81%AB%E3%82%88%E3%82%8B%E3%82%B2%E3%83%BC%E3%83%A0%E9%96%8B%E7%99%BA-%E2%80%95%E5%AE%89%E5%85%A8%E3%83%BB%E9%AB%98%E9%80%9F%E3%83%BB%E3%83%97%E3%83%A9%E3%83%83%E3%83%88%E3%83%95%E3%82%A9%E3%83%BC%E3%83%A0%E9%9D%9E%E4%BE%9D%E5%AD%98%E3%81%AEWeb%E3%82%A2%E3%83%97%E3%83%AA%E9%96%8B%E7%99%BA%E5%85%A5%E9%96%80-Eric-Smith/dp/481440039X/

## Rust x WebAssembly

新しくて、

https://www.amazon.co.jp/Rust%E3%81%A7%E5%AD%A6%E3%81%B6WebAssembly%E2%80%95%E2%80%95%E5%85%A5%E9%96%80%E3%81%8B%E3%82%89%E3%82%B3%E3%83%B3%E3%83%9D%E3%83%BC%E3%83%8D%E3%83%B3%E3%83%88%E3%83%A2%E3%83%87%E3%83%AB%E3%81%AB%E3%82%88%E3%82%8B%E9%96%8B%E7%99%BA%E3%81%BE%E3%81%A7-%E3%82%A8%E3%83%B3%E3%82%B8%E3%83%8B%E3%82%A2%E9%81%B8%E6%9B%B8-%E6%B8%85%E6%B0%B4-%E6%99%BA%E5%85%AC/dp/4297144131

## ハローワールド
