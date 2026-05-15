---
title: "積んでた Wi-Fi カメラで Claude にようやく視覚を持たせれた"
emoji: "👁️"
type: "tech"
topics: ["claude", "mcp", "tapo", "onvif", "claudecode"]
published: true
published_at: 2026-05-15 20:00
---

# はじめに

Twitter で kmizu さんがカメラで Claude に視覚を持たせてる動画を上げていて、見た瞬間に「これやりたい」と思ってそのままカメラを注文しました。TP-Link の Wi-Fi カメラです。

https://zenn.dev/nextbeat/articles/2026-02-embodied-claude

実際のコードも [embodied-claude](https://github.com/optimisuke/embodied-claude) というリポジトリにおいてくれていて、カメラを MCP サーバー経由で Claude につなぐ仕組みです。カメラが届いてからしばらく積んでたんですが、ようやく動かせました。

動かしてみて、「自然言語でカメラを操作できる」より「Claude が目を持った」という感じがして、AIのパートナーっぽさがまた一段上がった気がしています。

# カメラ選び

動かしたのは **TP-Link Tapo C220**（Crystal Clear 2K QHD）です。AC 電源の屋内カメラで、C210/C220 系は ONVIF に対応していて、README にも推奨カメラとして名前が書かれています。

![デスクに置いた TP-Link Tapo C220。赤いトラックボールの隣にいる](/images/wifi-cam-mcp-claude/camera.jpg)

試していて「他のカメラでも動くかな」と気になったので少し調べました。TP-Link には屋外向けのソーラー/バッテリー駆動モデル（Tapo C615F、C425、C460 など）もありますが、これらは ONVIF と RTSP が無効化されているようです。バッテリーの持ちを優先する設計で、常時接続が前提のプロトコルは切られているとのこと。TP-Link の公式 FAQ にも明記されていました。

屋外に設置したい場合は AC 電源を引けるかどうか確認が必要そうです。

:::message
**ONVIF** は IP カメラの操作 API の規格（スナップショット取得・PTZ 操作）、**RTSP** はカメラの映像ストリームを流すプロトコルです。wifi-cam-mcp はこの 2 つを使って動いています。
:::

# セットアップ

詳細は [README](https://github.com/optimisuke/embodied-claude/tree/main/wifi-cam-mcp) を見てもらうとして、自分がやったことだけ書きます。

1. Tapo アプリで **カメラアカウント** を作る（TP-Link クラウドアカウントとは別。アプリ → 歯車 → 高度な設定 → カメラのアカウント → オン）
2. `.env` にカメラの IP・ユーザー名・パスワードを書く
3. `uv sync` で依存関係を入れる
4. `claude mcp add` で MCP を登録して VSCode をリロード

これだけで動きました。

# 引っかかったところ

`.env` のユーザー名を間違えて `Authority failure` が出ましたが、直したらあっさり繋がりました。思ったよりすんなり動きました。

# 動かしてみた

Claude Code のセッションで MCP が有効になると、こんなツールが使えます。

| ツール                     | 動作                     |
| -------------------------- | ------------------------ |
| `see`                      | 今見えている画像を取得   |
| `look_left` / `look_right` | 左右に首を振る（1〜90°） |
| `look_up` / `look_down`    | 上下に首を振る（1〜90°） |
| `look_around`              | 4 方向を自動で見て回る   |
| `listen`                   | カメラのマイクで録音     |

「今カメラに何が映ってる？」と話しかけると `see` を呼んで画像を返してくれます。「もっと上を向いて、顔が見えるようにして」と言ったら `look_up` を数回呼んで、実際にカメラが動いて顔が映るようになりました。

![カメラから見たデスクの様子。トラックボールとキーボードが見える](/images/wifi-cam-mcp-claude/desk-from-camera.jpg)

# MCP サーバーの仕組み（補足）

せっかくなのでコードも少し読みました。

MCP サーバーの作りはシンプルです。Anthropic 公式の `mcp` ライブラリを使って、`list_tools()` と `call_tool()` の 2 つを登録しているだけでした。

```python
@server.list_tools()
async def list_tools() -> list[Tool]:
    return [Tool(name="see", description="...", inputSchema={...}), ...]

@server.call_tool()
async def call_tool(name: str, arguments: dict):
    match name:
        case "see":
            result = await camera.capture_image()
            return [ImageContent(type="image", data=result.image_base64, mimeType="image/jpeg")]
```

通信は JSON-RPC 2.0 over stdio で、Claude Code と JSON をやり取りします。`ImageContent` を返すと Claude Code が Claude API のマルチモーダル入力として渡してくれるので、Claude が実際に画像を「見る」ことができます。

# おわりに

Twitter で見かけて面白そうと思い、カメラを買って、しばらく積んで、ようやく動かせました。

仕組みも ONVIF という既存の規格を使っているだけで、思ったよりストレートな実装でした。

embodied-claude には hooks を使って面白い動きをいろいろ仕込んでいるみたいですが、今回はカメラが動くところまで確認しただけです。memory-mcp と組み合わせて「見たものを記憶する」あたりも、そのうち試してみたいと思っています。

間違いや補足があれば教えてください。

# 参考

- [3,980円のカメラでClaude Codeに「身体」を与えてみた（kmizu さん）](https://zenn.dev/nextbeat/articles/2026-02-embodied-claude)
- [embodied-claude](https://github.com/optimisuke/embodied-claude)
- [ONVIF 対応カメラ一覧（TP-Link 公式）](https://www.tp-link.com/us/support/faq/2680/)
- [MCP の公式ドキュメント](https://modelcontextprotocol.io/)
