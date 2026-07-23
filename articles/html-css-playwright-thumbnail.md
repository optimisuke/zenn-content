---
title: "YouTubeサムネ作りがわからなさすぎて、AIにHTML/CSSで作ってもらった"
emoji: "🎬"
type: "tech"
topics: ["playwright", "html", "css", "youtube"]
published: true
published_at: 2026-07-23 20:00
---

# はじめに

先日、[Microsoft Agent Hackathon](https://zenn.dev/hackathons/microsoft-agent-hackathon-2026)にチャレンジしてその提出で、YouTube 動画を用意する必要がありました。

残念ながら賞取れずでしたが、ぜひ読んでみてください。自分でももう読みたくないくらいの超大作です。↓↓

https://zenn.dev/optimisuke/articles/ai-mediator-eeyan

さて、動画ができると、サムネもそこそこちゃんと作りたくなりますが、画像編集ツールを開いてゼロから配置を考えるのは、かなりめんどくさそうです。そこで、AI に HTML/CSS を書いてもらい、Playwright でスクリーンショットを撮ってみました。

完成したサムネがこちらです。

![](/images/html-css-playwright-thumbnail/thumbnail.png)

プロが作るようなサムネとは天と地かと思いますが、素人の自分がゼロから作るよりは良いものに、かなり早くたどり着けたのでやったこと共有です。

# やったこと

流れは4ステップです。

1. 動画の推しポイントの画像を用意する（今回は紹介するアプリのキャプチャ）
2. AI に `1280x720` 固定の HTML/CSS を作ってもらう
3. ブラウザで表示して、気になるところを直す
4. Playwright で PNG にする

元にしたのは、アプリのキャプチャ画面です。

![](/images/html-css-playwright-thumbnail/source-screen.png)

そのままだとサムネとしてはわかりにくいので、左にアプリ画面、右に大きなコピー、左上に吹き出しを置きました。

AI には、だいたい次のように頼みました。

- YouTube サムネ用に `1280x720` で作る
- 左にアプリ画面、右に大きい日本語コピーを置く
- もっといいかんじにして

一度で完成はしませんでしたが、「吹き出しがキャラと被るので上へ」「右の文字をもっと大きく」のように、表示を見ながら何度か直しています。ゴールとか与えてループエンジニアリング的な工夫をしたら、もうちょい勝手にやってくれるかもですが、そこそこのやりとりで満足できました。以下、成果物や過程で気になったところです。

# HTML/CSS でレイアウトする

サムネ専用なので、レスポンシブ対応はせず `1280x720` に固定されてました。

```html
<style>
  * {
    box-sizing: border-box;
    margin: 0;
    padding: 0;
  }
  html,
  body {
    width: 1280px;
    height: 720px;
    overflow: hidden;
  }
  body {
    font-family: "M PLUS Rounded 1c", "Noto Sans JP", sans-serif;
    position: relative;
    background: #fff;
  }
</style>
```

中身は、背景画像と吹き出し、コピー用の領域という単純な構成でした。

```html
<div class="bg"></div>
<div class="bubble">この絵、なんて声かける？</div>
<div class="right">
  <span class="tag">DEMO &amp; 紹介</span>
  <div class="hook-main">
    <span class="accent">「いいねぇ」</span><br />
    以上の言葉を、<br />AIがくれる。
  </div>
</div>
```

# Playwright で PNG にする

HTML ができたら、ローカルで配信してくれます。open でよいかもですが。

```bash
cd docs/submission/video
python3 -m http.server 8000
```

別ターミナルで表示サイズを固定し、スクリーンショットを撮ります。というか、勝手にやってくれました。

```bash
playwright-cli -s=thumb open http://localhost:8000/thumbnail.html
playwright-cli -s=thumb resize 1280 720
playwright-cli -s=thumb screenshot thumbnail.png
```

これで YouTube にアップロードできる PNG ができました。

# 良かったところ

HTML/CSS なので、「文字を大きく」「吹き出しを上へ」といった修正を AI に日本語で頼めます。追加する文字も画像生成 AI のように崩れず、Git で変更も追えます。

特に、実際のアプリ画面に文字や吹き出しを重ねるだけなら、かなり相性が良いと感じました。

# ただし、ここから先は時間が溶けそう

最初の1枚を形にするまでは速かったですが、サムネ作りはもっと奥が深いはずです。文字の読みやすさ、視線誘導、余白、色、スマホで見たときの印象等、真面目に考え始めると、無限に調整できそうです。第一歩は速いですが、良いサムネを作るまでには道のりが長いかもです。「良いサムネとは」がわからないので、それっぽいのができて満足しておわっちゃいました。どなたか、こだわりのskillとかにして共有してくれるともっと良いもの作れるようになる気がします。探したらすでにあるんやろか？

# おわりに

今回の方法は、デザインツールを置き換えるものではないと思いますが、素人の自分がゼロから作るより気軽にできました。完成度を上げようとすると時間はかかりそうですが、最初のたたき台をAIで素早く作る方法としては、かなりありな気がします。やってるうちに欲が出てきて時間が溶けそうな気もしますが。
