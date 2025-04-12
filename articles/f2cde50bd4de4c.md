---
title: "JavaでAPIサーバーを作る3つの選択肢 - 他言語エンジニアに贈る、モダンJava開発の入り口 -"
emoji: "☕️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [java, liberty, websphere, spring, quarkus]
published: true
---

## 1. はじめに

筆者はもともと Go や Node.js で API サーバーを構築していた経験がありました。
シンプルな構成で高速に動作し、オープンなライブラリを自由に組み合わせられる世界はとても快適で、Java は正直「古い」「重い」「大企業向けで閉じた世界」という先入観がありました。

特に、エンタープライズ企業で使われている **Java の商用環境（WebSphere、WebLogic など）** はどうなっているのか理解しづらく、言語というより“専用プラットフォーム”のような印象さえありました。

しかし調べてみると、**Java にもオープンソース化とクラウド対応の流れが急速に進んでいる**ことが分かりました。
昔ながらの「アプリケーションサーバー」も今や軽量化され、**他言語と同じように自己完結型・クラウドネイティブな開発が可能**になっています。

本記事では、そんな Java の最新事情を、「Open Liberty」「Spring Boot」「Quarkus」という 3 つのアプローチから俯瞰します。

---

## 2. Java で API を作るということ

他言語では「アプリ＝サーバー」スタイルが一般的です（例：Node.js の Express や Python の FastAPI）。
Java にもそのスタイルはありますが、歴史的には **「アプリケーションサーバーにアプリを載せる」** という考え方からスタートしました。

この構造を理解しておくと、後述する 3 つの選択肢（Open Liberty, Spring Boot, Quarkus）の違いがよく見えてきます。

---

## 3. Open Liberty：Java の伝統と標準仕様に基づいたアーキテクチャ

### 3.1 アプリケーションサーバーとは？

Java の「アプリケーションサーバー（Application Server）」とは、**アプリケーションのホスティングと運用を担う実行環境**のことです。

Node.js でいうところの `express()` に相当するような Web サーバー部分を、Java では**別プロセスとして管理する**文化が長く続いていました。

### 3.2 Open Liberty とは？

Open Liberty は、IBM が提供するオープンソースの軽量なアプリケーションサーバーです。**Jakarta EE（旧 Java EE）や MicroProfile に準拠**しており、以下のような特徴があります：

| 機能               | 内容                                                                                    |
| ------------------ | --------------------------------------------------------------------------------------- |
| Jakarta EE         | Java の標準仕様（Servlet, JAX-RS, CDI, JPA など）に準拠                                 |
| MicroProfile       | クラウドネイティブな API（Health Check, Config, OpenAPI, Metrics など）を追加でサポート |
| Dev Mode           | 開発中のホットリロードや即時反映が可能                                                  |
| 拡張機能ベース構成 | 使いたい機能だけを選んで構成できる（軽量化が可能）                                      |
| WAR デプロイ対応   | アプリを「WAR ファイル」としてデプロイする従来型にも対応                                |

### 3.3 構成例（REST API）

```java
@Path("/hello")
public class HelloResource {
  @GET
  @Produces(MediaType.TEXT_PLAIN)
  public String hello() {
    return "Hello from Liberty!";
  }
}
```

### 3.4 用語解説

| 用語         | 意味と補足                                                              |
| ------------ | ----------------------------------------------------------------------- |
| WAR ファイル | Web Application Archive。Java Web アプリの配布単位。                    |
| Jakarta EE   | Java のサーバーサイド向け仕様セット（JPA, CDI, JAX-RS など）            |
| CDI          | Java での DI（依存性注入）の標準機能。Spring の `@Autowired` に似ている |
| MicroProfile | Jakarta EE を補完して、クラウドネイティブ対応を強化する API 群          |

### 3.5 どんなときに使う？

- Java EE / WebSphere アプリの資産を活かしたい
- 複数 WAR 構成や複数アプリを同一環境で管理したい
- 保守性・標準性を重視した企業向けシステム

---

## 4. Spring Boot：Java エンジニアのデファクトスタンダード

### 4.1 Spring Boot とは？

Spring Boot は、Spring Framework をベースにした「設定不要ですぐ動く Java アプリケーション構築のためのフレームワーク」です。2014 年の登場以来、Java の Web/API アプリ開発におけるデファクトになりました。

### 4.2 登場の背景

Java EE の設定の多さや冗長さを解消するために生まれた。設定ファイルを極力省略し、アノテーションベースでアプリを構成可能。

### 4.3 構成と特徴

| 機能                  | 内容                                                             |
| --------------------- | ---------------------------------------------------------------- |
| 組み込み Web サーバー | Tomcat / Jetty / Undertow を選択可能。自己完結型の HTTP サーバー |
| スターター依存        | `spring-boot-starter-web`, `data-jpa` などの便利な依存群         |
| アノテーション構成    | `@RestController`, `@GetMapping` など直感的な API 記述           |
| Fat JAR 形式          | `java -jar` で起動できる自己完結アプリ                           |
| 豊富な拡張機能        | Security, Actuator, OpenAPI など多彩な機能                       |

### 4.4 構成例

```java
@RestController
public class HelloController {

  @GetMapping("/hello")
  public String hello() {
    return "Hello from Spring Boot!";
  }
}
```

### 4.5 用語解説

| 用語               | 意味と補足                                       |
| ------------------ | ------------------------------------------------ |
| Spring Framework   | DI や AOP を提供する Java の代表的フレームワーク |
| スターター         | よく使うライブラリ群をまとめた依存セット         |
| Fat JAR            | 必要なライブラリ込みの自己完結型 JAR             |
| Auto Configuration | アプリの構成を自動で検出・設定する機能           |

### 4.6 どんなときに使う？

- 最短で API を立ち上げたい
- Node.js 的な開発体験を Java でも得たい
- ドキュメントや人材の多さを重視したい

---

## 5. Quarkus：クラウドネイティブ時代に最適化された Java

### 5.1 Quarkus とは？

Red Hat 製のモダンな Java フレームワークで、クラウドや Kubernetes を前提に設計。ネイティブバイナリ化により高速起動・省メモリを実現。

### 5.2 登場の背景

「Java は重い・遅い」という課題を解決するために、ビルド時最適化・GraalVM 対応・K8s 連携などを重視して設計。

### 5.3 構成と特徴

| 機能                   | 内容                                       |
| ---------------------- | ------------------------------------------ |
| JVM + ネイティブ両対応 | JVM で開発、GraalVM でバイナリ化可能       |
| Dev モード             | `quarkus dev` によるライブリロード対応     |
| JAX-RS ベース          | Jakarta EE 準拠の API 構文を使用           |
| GraalVM 対応           | 数 ms で起動、数十 MB の実行サイズ         |
| K8s 連携               | YAML 生成や S2I 連携など豊富なクラウド支援 |

### 5.4 構成例

```java
@Path("/hello")
public class HelloResource {

  @GET
  @Produces(MediaType.TEXT_PLAIN)
  public String hello() {
    return "Hello from Quarkus!";
  }
}
```

### 5.5 用語解説

| 用語               | 意味と補足                                                     |
| ------------------ | -------------------------------------------------------------- |
| ネイティブバイナリ | JVM を使わない OS 上の実行ファイル                             |
| GraalVM            | Java などを高速に動かす仮想マシン＋ AOT コンパイラ             |
| AOT                | Ahead-of-Time コンパイル。起動前にコードを全て変換しておく方式 |

### 5.6 どんなときに使う？

- サーバーレスや K8s などクラウド中心の構成にしたい
- 起動速度やメモリ効率を最重要視したい
- Spring より軽いフレームワークを探している

---

## 6. 比較まとめ表

| 項目       | Open Liberty              | Spring Boot                | Quarkus                        |
| ---------- | ------------------------- | -------------------------- | ------------------------------ |
| 型         | アプリケーションサーバー  | フレームワーク             | フレームワーク（クラウド特化） |
| 起動形式   | WAR / Dev Mode            | `java -jar`                | `java -jar` / ネイティブ       |
| 主な仕様   | Jakarta EE + MicroProfile | Spring 独自 + 一部 Jakarta | MicroProfile + GraalVM         |
| 起動速度   | 遅め〜普通                | 普通〜速い                 | 非常に速い（ネイティブ）       |
| メモリ効率 | 安定重視                  | 中程度                     | 軽量（特にネイティブ）         |
| サポート   | IBM                       | VMware                     | Red Hat                        |
| 向き不向き | 企業内資産活用            | モダン Web 全般            | K8s やサーバーレス             |

![](/images/2025-04-12-22-58-41.png)

---

## 7. おわりに

筆者自身、Node.js や Go といった言語で API サーバーを開発してきた中で、Java には“商用環境で使われる複雑な仕組み”という印象を持っていました。

しかし、Open Liberty、Spring Boot、Quarkus といった選択肢を知ることで、Java でもモダンな開発スタイルが可能であることがわかりました。

特に注目すべきは、Java がこれまで商用環境で築いてきた信頼性や標準仕様の強さを保ちつつ、**オープン化**によって他言語と同様の開発体験が得られるようになった点です。

**今や Java は「重い言語」ではなく、「選択肢の多い言語」になった**と感じています。

他言語エンジニアの視点からも、Java の進化と多様性をぜひ体験してみてください。

---

## 8. 参考リンク

- [Open Liberty 公式](https://openliberty.io/)
- [Spring Boot 公式](https://spring.io/projects/spring-boot)
- [Quarkus 公式](https://quarkus.io/)
- [GraalVM Native Image](https://www.graalvm.org/native-image/)
- [Jakarta EE 公式](https://jakarta.ee/)
- [MicroProfile 公式](https://microprofile.io/)
