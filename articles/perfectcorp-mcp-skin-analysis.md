---
title: "「肌分析して」と頼んだら肌年齢50歳と言われた👼  by Claude x YouCam API"
emoji: "🪞"
type: "tech"
topics: ["claude", "mcp", "claudecode", "youcam", "python"]
published: true
published_at: 2026-06-07 20:00
---

# はじめに

Zenn のコンテストで YouCam API というのを知って、「肌分析できるのか、試してみたい」と思って作りました。

「カメラで写真撮って肌を分析して」と一言打つだけで動く MCP サーバーです。

疲れ具合わかればいいなと思ってたくらいでしたが、実年齢をまあまあ超える肌年齢を言われてショックを受けました。。。あんまり気にしてなかったけど、それなりにショック・・・。

---

# 実は公式 MCP サーバーがある

作り始めてから気づいたんですが、Perfect Corp（YouCam API の会社）には公式の MCP サーバーが既にありました 👼

```json
{
  "mcpServers": {
    "youcam-api-mcp": {
      "url": "https://mcp-api-01.makeupar.com/mcp",
      "headers": { "Authorization": "Bearer YOUR_API_KEY" }
    }
  }
}
```

これを Claude Code に登録するだけで、肌分析・メイク試着・ヘアスタイル変換など 50 種類以上のツールが使えます。ほぼ全部入りです。

ただ、これだと「画像ファイルのパスを渡す」ところまでは自分でやらないといけないし、Mac のカメラで撮影してそのまま分析、という体験にはならないので、そこだけ自分で作りました。カメラと一体化した MCP は自分の環境ではまあ使えるかなという感じです。

---

# 実際のやり取り

```
❯ カメラで写真撮って肌を分析して

⏺ カメラで撮影して肌分析を行います。
  Called perfectcorp-ai

⏺ 顔が小さすぎてうまく検出できませんでした。カメラにもう少し近づいて、
  顔が画面の大部分を占めるようにして、もう一度試してみてください。
  準備ができたら「もう一度」と言っていただければ再撮影します。

❯ もう一度

⏺ 再撮影します。カメラに近づいた状態でお待ちください。
  Called perfectcorp-ai

⏺ 分析が完了しました。結果をまとめます。

  肌分析レポート
  総合スコア: 83.7 / 100
  肌年齢: 50歳
  肌タイプ: ノーマル

  優れた項目 (90点以上)
  皮脂・オイリー度 99 / 毛穴（額）99 / 赤み 97 / ニキビ 97 / シワ（額）97 ...

  要注意な項目 (75点未満)
  ほうれい線 70 / 目袋 71 / ハリ・弾力 72 / 上まぶたのたるみ 61（最も気になる部位）

  まとめ・アドバイス
  - 肌質全体はノーマルで安定しており、赤みやニキビは非常に良好
  - 目元周り（上まぶたのたるみ・目袋）が最も気になるポイント
  - アイクリームや十分な睡眠が効果的
  - ほうれい線にはコラーゲン・レチノール配合スキンケアが有効
```

---

# 感想

肌年齢 50 歳と言われました。上まぶたのたるみが「最も気になる部位」とのことで、なかなかつらたん。

---

ちなみに一発目は「顔が小さすぎる」とエラーになりました。カメラから離れすぎていたようです。「もう一度」と打ったら Claude が自分で再撮影してくれました。

この流れは試してみて気持ちよかったです。MCP サーバーはエラーを返すだけで、どう伝えてどう促すかは Claude が考えてくれます。サーバー側でメッセージを作り込む必要がない。Claude に任せると、むしろ自分で書くより丁寧な言葉で案内してくれました。

---

# やったこと

## 構成

```
Claude Code（チャット）
    ↓ 「カメラで写真撮って分析して」
MCP Server（Python）
    ↓ imagesnap でカメラ撮影
    ↓ Perfect Corp API にアップロード
    ↓ 分析タスクを投げてポーリング
    ↓ 結果 JSON をそのまま返す
Claude Code（結果を解釈・要約）
```

MCP サーバーは API の橋渡しに徹して、解釈や要約は Claude に任せる設計にしました。

## カメラ撮影は imagesnap

Mac のカメラを CLI から操作できる `imagesnap` を使いました。

```bash
brew install imagesnap
imagesnap -w 3 photo.jpg  # 3秒後に撮影
```

これだけで FaceTime カメラから静止画が撮れます。

## MCP ツール

```python
capture_and_analyze_skin()   # 撮影 + 分析を一気にやる
```

imagesnap で撮影してそのまま API に投げます。Claude から見ると「撮影して分析する」が 1 つのアクションに見えます。

## サーバーのコード

```python
@mcp.tool()
def capture_and_analyze_skin() -> str:
    image_path = "/tmp/photo.jpg"
    subprocess.run(["imagesnap", "-w", "3", image_path])
    return analyze_skin_image(image_path)
```

:::message
エラー処理などは省略しています。詳細はリポジトリを参照してください。
:::

`analyze_skin_image` の中では、画像を base64 エンコードして Perfect Corp API にアップロード → 分析タスクを投げてポーリング → 結果 JSON をそのまま返しています。

# リポジトリ

https://github.com/optimisuke/perfectcorp-mcp

---

# おわりに

「カメラで撮って一言で完結する」体験は作れましたが、肌年齢 50 歳という現実も体験してしまいました 👼

現実を知る以外の体験としては快適でした。Claude に「カメラで写真撮って肌を分析して」とお願いするだけで動きます。MCP のおかげでサーバー側のコードはシンプルに保てて、エラーハンドリングや対話の制御を Claude に委ねられるのがよかったです。

肌年齢対策のアイクリームが何かよく分かっていませんが、探してみようと思います。

:::message
この記事は [Zenn × YouCam API コンテスト「ZennFes Spring 2026」](https://zenn.dev/contests/zennfes-spring-2026-perfect) への応募記事です。
:::

# 参考

- [Perfect Corp API Docs](https://docs.perfectcorp.com/reference/ai_skin_analysis)
- [MCP Python SDK](https://github.com/modelcontextprotocol/python-sdk)
- [perfectcorp-mcp（今回のリポジトリ）](https://github.com/optimisuke/perfectcorp-mcp)
