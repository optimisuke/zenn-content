---
title: "Ollama () をMacで動かしてみる"
emoji: "🦙"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [llama, ollama, llm, mac, openai]
published: true
---

# はじめに

[Ollama](https://ollama.ai/) （Llama2 とかいろんなモデルをローカル環境で動かすツール）をすーごく簡単に使えたので、書くほどじゃないけどメモ。
使い方は github の README を見た。
[jmorganca/ollama: Get up and running with Llama 2 and other large language models locally](https://github.com/jmorganca/ollama/tree/main)

使えるモデルは今こんな感じ。
![](/images/2023-11-17-17-30-54.png)

環境はこんな感じ。

```
MacBook Pro 16-inch, 2021
Apple M1 Pro
Memory 16GB
Sonoma 14.0
```

# インストール

インストールはこんな感じ。

1. [Ollama](https://ollama.ai/)にアクセスして、ダウンロード。
2. アプリがダウンロードフォルダに入るので、アプリケーションフォルダに移動。
3. アプリを開く。
4. Mac の右上のバーにラマのアイコンが現れる。

ラマかわいい。
![](/images/2023-11-17-13-57-35.png)

ちなみに、Linux 用のコマンドは使えなかった 😇
`curl https://ollama.ai/install.sh | sh`
ちなみに、Windows 版も Coming soon!らしい。
Docker image もあるみたい。

# 使い方

## コマンドライン

こんな感じでコマンドラインで使える。

```bash
% ollama run llama2
>>> hello
Hello! It's nice to meet you. Is there something I can help you with or would you like to chat?

```

日本語は怪しい。プロンプトを工夫すればいけるのかも。

```bash
>>> こんにちは
Konnichiwa! Ogenki desu ka? (お元気ですか？) How are you today?
```

## ブラウザ

ブラウザで[localhost:11434](http://localhost:11434/)にアクセスすると

```
Ollama is running
```

ってかえってくる。

## API

API サーバーも動いてる。

```bash
% curl -X POST http://localhost:11434/api/generate -d '{
  "model": "llama2",
  "prompt":"Why is the sky blue?"
}'
{"model":"llama2","created_at":"2023-11-17T05:09:26.035946Z","response":"\n","done":false}
{"model":"llama2","created_at":"2023-11-17T05:09:26.065294Z","response":"The","done":false}
{"model":"llama2","created_at":"2023-11-17T05:09:26.092508Z","response":" sky","done":false}
{"model":"llama2","created_at":"2023-11-17T05:09:26.120137Z","response":" appears","done":false}
{"model":"llama2","created_at":"2023-11-17T05:09:26.14749Z","response":" blue","done":false}
{"model":"llama2","created_at":"2023-11-17T05:09:26.175Z","response":" because","done":false}
{"model":"llama2","created_at":"2023-11-17T05:09:26.202586Z","response":" of","done":false}
{"model":"llama2","created_at":"2023-11-17T05:09:26.229704Z","response":" a","done":false}
{"model":"llama2","created_at":"2023-11-17T05:09:26.256979Z","response":" phenomen","done":false}
{"model":"llama2","created_at":"2023-11-17T05:09:26.284129Z","response":"on","done":false}
```

API ドキュメントはここ。
[ollama/docs/api.md at main · jmorganca/ollama](https://github.com/jmorganca/ollama/blob/main/docs/api.md)

## 止め方

右上のアイコンから止める。
![](/images/2023-11-17-14-55-19.png)

# おわりに

LLM をローカルで動かすには、GPU とか必要なんかなと思ってたけど、サクサク動いてびっくり。
Llama 作った Meta の方々と ollama の Contributors の方々に感謝。
