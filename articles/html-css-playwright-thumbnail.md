---
title: "Canvaを開かずに、HTML/CSSとPlaywrightでYouTubeサムネを作った"
emoji: "🎬"
type: "tech"
topics: ["playwright", "html", "css", "youtube"]
published: false
---

# はじめに

YouTube のサムネを作る必要がありました。

普通は Canva、Figma、Keynote あたりを開くところだと思います。自分も普段ならそうすると思うんですが、今回はハッカソン提出用のデモ動画で、アプリ画面のキャプチャを使ったサムネをさっと作りたい状況でした。

サムネを1枚だけ作りたいのに、Canvaでテンプレートを探して、文字サイズを調整して、画像を書き出すのが地味にめんどくさいです。

そこで、Claude に HTML/CSS を書いてもらって、ブラウザで表示し、Playwright でスクリーンショットを撮る方法を試しました。

完成したサムネはこれです。

![](/images/html-css-playwright-thumbnail/thumbnail.png)

この記事では、やったことを短く整理します。

# やったこと

流れはこんな感じです。

1. アプリ画面のスクリーンショットを用意する
2. `1280x720` 固定の HTML/CSS を書く
3. ローカル HTTP サーバーで表示する
4. Playwright でスクリーンショットを撮る

元にした画面は、アプリで作品を AI 評価している画面です。

![](/images/html-css-playwright-thumbnail/source-screen.png)

スクリーンショットをそのまま使うだけだと弱いので、左側にアプリ画面、右側に大きいコピー、左上に吹き出しを置く構成にしました。

Claude には、だいたい次のように頼みました。

- YouTube サムネ用に `1280x720` の HTML を作る
- 左側にアプリ画面のスクリーンショットを置く
- 右側に大きい日本語コピーを置く
- 黄色い吹き出しを足す
- 日本語が太く見えるフォントを使う

最初から完璧なものが出るわけではないですが、「吹き出しがキャラアイコンと被るので上に移動して」「右側の文字をもっと大きくして」みたいに、見た目を見ながら何度か直していくと使える形になりました。

# HTML/CSS でサムネを作る

サムネ用なので、レスポンシブは考えずに `1280x720` 固定にしました。

```html
<style>
  * { box-sizing: border-box; margin: 0; padding: 0; }
  html, body { width: 1280px; height: 720px; overflow: hidden; }
  body {
    font-family: "M PLUS Rounded 1c", "Noto Sans JP", sans-serif;
    position: relative;
    background: #fff;
  }
</style>
```

左側は実アプリのキャプチャ、右側はコピー用のエリアです。

```html
<div class="bg"></div>

<div class="bubble">この絵、なんて声かける？</div>

<div class="right">
  <span class="tag">DEMO &amp; 紹介</span>
  <div class="hook-main">
    <span class="accent">「いいねぇ」</span><br>
    以上の言葉を、<br>
    AIがくれる。
  </div>
  <div class="hook-sub">
    <span class="em">3人のAIキャラ</span>が、<br>
    作品を語る。
  </div>
</div>
```

日本語の太字をきれいに出したかったので、Google Fonts で `M PLUS Rounded 1c` と `Noto Sans JP` を読み込んでいます。

```html
<link href="https://fonts.googleapis.com/css2?family=M+PLUS+Rounded+1c:wght@800;900&family=Noto+Sans+JP:wght@700;900&display=swap" rel="stylesheet">
```

実際のファイルはこのように残しました。

```text
docs/submission/video/thumbnail.html
docs/submission/video/thumbnail.png
```

# Playwright で画像にする

HTML ができたら、ローカルで配信します。

```bash
cd docs/submission/video
python3 -m http.server 8000
```

別ターミナルで Playwright CLI を使って、表示、リサイズ、スクリーンショットを実行します。

```bash
playwright-cli -s=thumb open http://localhost:8000/thumbnail.html
playwright-cli -s=thumb resize 1280 720
playwright-cli -s=thumb screenshot thumbnail.png
```

これで YouTube にアップロードできる `1280x720` の PNG ができます。

# 良かったところ

## コードとして残る

サムネのレイアウトが HTML/CSS として残るので、あとから見返しやすいです。

文言を変えた、色を変えた、位置を変えた、という変更も Git の差分で追えます。

## LLM に修正を頼みやすい

「吹き出しを少し上に」「右側の見出しをもっと大きく」「黄色いマーカーを足す」みたいな修正は、画像を直接いじるより HTML/CSS のほうが頼みやすかったです。

特に日本語文字入りのサムネを画像生成 AI に丸ごと作らせると、文字が崩れることがあります。文字入れは HTML/CSS 側に寄せたほうが安心でした。

## 実アプリ画面と相性がいい

技術記事やデモ動画のサムネでは、実際の画面を見せたいことが多いです。

HTML/CSS なら、スクリーンショットを背景にして、その上に文字や吹き出しを置くだけで済みます。

# 注意点

これは Canva や Figma の完全な代替ではないです。

写真の切り抜き、複雑な合成、細かい手作業の調整は、普通にデザインツールのほうが楽です。

今回の方法が向いているのは、こういうケースだと思います。

- スクリーンショットと文字が中心
- 同じフォーマットで何枚か作りたい
- コードで変更履歴を残したい
- LLM に CSS 修正を頼みたい

逆に、写真の細かい切り抜きや、一枚絵としての作り込みが大事なサムネなら、普通に Canva や Figma を使ったほうが早いと思います。

あと、Google Fonts を使う場合は、スクリーンショットを撮る前にフォント読み込みが終わっているかは見ておいたほうがよさそうです。

# おわりに

YouTube サムネは Canva や Figma で作るものだと思っていましたが、HTML/CSS と Playwright でも意外といけました。

特に、エンジニアがアプリ画面を使ってサムネを作る用途だと相性が良いです。

全部をこの方法にする必要はないですが、「スクリーンショット + 大きい文字」のサムネをコードで作る選択肢としては、けっこう便利だと思いました。

YouTube サムネだけでなく、技術記事の OGP 画像や、登壇資料の表紙画像を作るときにも使えそうです。
