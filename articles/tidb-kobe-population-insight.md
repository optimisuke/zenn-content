---
title: "RAGのRDB・ベクトルDB・全文検索めんどすぎて滅! からの TiDB Cloud"
emoji: "🏙️"
type: "tech"
topics: ["tidb", "nextjs", "openai", "rag", "typescript"]
published: true
published_at: 2026-06-05 20:00
---

# はじめに

RAG（Retrieval-Augmented Generation）を作るとき、「構造化データ（RDB）」と「非構造化データ（ベクトルDB）」を別々のサービスで管理するのが地味に面倒だと思ってました。

たとえば「統計データはSQLで検索しつつ、PDF文書はベクトル検索と全文検索で引いてきて、まとめてAIに渡す」みたいな処理をしようとすると、RDB・ベクトルDB・全文検索をそれぞれ別につないで、結果をマージして...ってやる必要があります。シンプルなユースケースのわりに、管理するサービスが増えてしんどいんですよね。

TiDB Cloudを調べたら、SQL・ベクトル検索・全文検索を単一のDBで扱えると知りました。技術誌の[ソフトウェアデザイン](https://gihyo.jp/magazine/SD)で特集されていたこともあってずっと気になっていたので、ちょうどいい機会でした。

作ったのは、**神戸市の人口統計データ（CSV）と政策文書（PDF）を組み合わせて、自然文で質問できるAI分析アプリ**です（まじめ）。題材は神戸市ですが、アーキテクチャ自体は「構造化CSVと非構造化PDF」があれば何にでも応用できます。

# やりたいこと

こんな質問に答えられるアプリを目指しました。

- 神戸市はどの年代が流出している？（→ 人口統計CSVをSQL検索）
- 神戸市は人口減少をどう捉えている？（→ 政策PDF文書をベクトル検索・全文検索）
- 若者流出の実態と政策対応は？（→ 両方を組み合わせてAIが回答）

**1つのDBで構造化データ（統計）と非構造化データ（文書チャンク + embedding）を管理する**というのが今回の実験のポイントです。

![アプリの画面](/images/tidb-kobe-population-insight/01-top.png)

Vercel にデプロイして公開しています。LLM APIを呼んでいるので野放しにするのも怖く、パスワード認証をつけて一旦とめています。

https://kobe-population-insight.vercel.app

# アーキテクチャ

```mermaid
flowchart TD
    A[ブラウザ / Next.js] -->|POST message + history| B[orchestrator.ts]
    B -->|質問分析| C[gpt-5.4-mini\n人口統計が必要?\n政策文書が必要?]
    C -->|needsSql| D[SQL検索\nパラメータ抽出 → 固定テンプレートSQL]
    C -->|needsPolicy| E[ベクトル検索\nVEC_COSINE_DISTANCE]
    C -->|needsPolicy| F[全文検索\nLIKE検索]
    D --> G[(TiDB Cloud\npopulation テーブル)]
    E --> H[(TiDB Cloud\npolicy_chunks テーブル)]
    F --> H
    D -->|結果| I[gpt-5.5\n回答生成]
    E -->|チャンク| I
    F -->|チャンク| I
    I --> A
```

TiDB Cloud Serverless の**1つのDB**に、人口統計テーブルと政策文書チャンクテーブルの両方を置いています。

# DBスキーマ

```sql
-- 人口統計（構造化データ）
CREATE TABLE population (
  id         BIGINT PRIMARY KEY AUTO_INCREMENT,
  year       INT NOT NULL,
  ward       VARCHAR(100),          -- 区名
  age_group  VARCHAR(100),          -- 年齢グループ（例: "20-29"）
  metric     VARCHAR(100) NOT NULL, -- population / transfer_in / transfer_out / projection
  value      BIGINT NOT NULL
);

-- 政策文書チャンク（非構造化データ）
CREATE TABLE policy_chunks (
  id           BIGINT PRIMARY KEY AUTO_INCREMENT,
  source_title VARCHAR(255) NOT NULL,
  source_url   TEXT,
  chunk_text   TEXT NOT NULL,
  embedding    VECTOR(1536) COMMENT 'hnsw(distance=cosine)'
);
```

`population` はCSVから投入した普通のテーブル。`policy_chunks` はPDFをチャンク分割してembeddingを生成したものです。同じDBに両方あるだけで、接続先を増やさずに済みます。

# データ投入

## 人口統計CSV（構造化）

神戸市オープンデータから年齢別転入・転出・将来推計のCSVをダウンロードして投入しました。

注意点として、このCSVはShift-JIS エンコードです。`iconv-lite` で変換してから処理しています。

```typescript
import iconv from "iconv-lite";

function decodeCsv(filePath: string): string[][] {
  const buf = readFileSync(filePath);
  const text = iconv.decode(buf, "Shift_JIS");
  return text
    .split(/\r?\n/)
    .filter((l) => l.trim())
    .map((l) => l.split(",").map((c) => c.trim()));
}
```

## 政策文書PDF（非構造化）

神戸市総合基本計画や神戸2030ビジョンの資料などをダウンロードして処理しました。`pdf-parse` でテキスト抽出 → 800文字・overlap100文字でチャンク分割 → `text-embedding-3-small` でembedding生成 → TiDBに挿入、という流れです。今回は236チャンク投入しました。

# 3種類の検索

## SQL検索（人口統計）

AIに直接SQLを生成させるとインジェクションのリスクがあるので、AIはパラメータだけ抽出して固定テンプレートに当てはめる方式にしています。

```typescript
// AIが抽出するのはパラメータのみ（年・区・年齢グループ・指標）
const params = await extractParams(question);
// 固定テンプレートSQLにパラメータを渡す
const result = await db.execute(
  `SELECT year, ward, age_group, metric, SUM(value) AS value
   FROM population
   WHERE metric = ? AND year = ?
   GROUP BY year, ward, age_group, metric
   ORDER BY year DESC`,
  [params.metric, params.year],
);
```

## ベクトル検索（政策文書）

質問文をembeddingして `VEC_COSINE_DISTANCE` で近いチャンクを引きます。

```typescript
const embedding = await openai.embeddings.create({
  model: "text-embedding-3-small",
  input: question,
});
const embStr = `[${embedding.data[0].embedding.join(",")}]`;

const result = await db.execute(
  `SELECT id, source_title, chunk_text,
          VEC_COSINE_DISTANCE(embedding, ?) AS distance
   FROM policy_chunks
   ORDER BY distance ASC
   LIMIT 5`,
  [embStr],
  { fullResult: true },
);
```

## 全文検索（政策文書）

AIでキーワードを抽出してLIKE検索しています。マッチ数をスコア代わりにして並び替えています。

```typescript
const keywords = await extractKeywords(question); // ["若年層", "人口減少", ...]

const likeConditions = keywords.map(() => `chunk_text LIKE ?`).join(" OR ");
const scoreExpr = keywords
  .map(() => `(CASE WHEN chunk_text LIKE ? THEN 1 ELSE 0 END)`)
  .join(" + ");

const result = await db.execute(
  `SELECT id, source_title, chunk_text, (${scoreExpr}) AS score
   FROM policy_chunks
   WHERE ${likeConditions}
   ORDER BY score DESC LIMIT 5`,
  [...likeValues, ...likeValues, limit],
  { fullResult: true },
);
```

# 動作デモ

質問ごとにどの検索が使われたかをアコーディオンで確認できるようにしました。

SQL検索では、AIが抽出したパラメータとSQL結果の先頭数行を表示しています。

![SQL検索の結果とアコーディオン](/images/tidb-kobe-population-insight/03-accordion-sql.png)

政策文書の質問では、ヒットしたチャンクに「全文検索」「ベクトル」のどちらでヒットしたかタグが付きます。

![ベクトル・全文検索のチャンクタグ](/images/tidb-kobe-population-insight/05-accordion-chunks.png)

# Vercel デプロイ

Vercel にデプロイするとき、設定する環境変数はこれだけです。

```
TIDB_HOST=
TIDB_PORT=
TIDB_USER=
TIDB_PASSWORD=
TIDB_DATABASE=
OPENAI_API_KEY=
AUTH_PASSWORD=   # 簡易認証用
```

少ないのは良いことなんですが、「本当にこれだけで動くのか？」と少し不安になりました。実際に動いたのでよかったです。`@tidbcloud/serverless` がHTTP経由で接続するので、VPCピアリングや特別なネットワーク設定が不要なのが楽なポイントだと思います。

# ハマりポイント

今回はClaude Codeに実装してもらいながら進めました。いくつかハマってたようです。自分は事後報告を聞いただけですが。

## 1. 接続はURL形式が必要

individual params形式で繋いだら「Missing user name prefix」。URL形式にしたら動きました。

```typescript
// NG
connect({ host: "...", user: "xxxxxxxx.root", password: "..." });
// OK
connect({ url: "mysql://xxxxxxxx.root:password@host:4000/dbname?ssl=true" });
```

## 2. MATCH AGAINST が使えない

`MATCH AGAINST` で全文検索を実装したら `Error 1105 (HY000): UnknownType: *ast.MatchAgainst`。Serverless のHTTP APIは非対応のようです。LIKEベースで実装し直しました。

## 3. USE文がHTTP APIで効かない

`USE kobe_population;` が毎回リセットされます。接続URLにDB名を入れる形式で解決。

```typescript
connect({ url: `mysql://user:pass@host:4000/kobe_population?ssl=true` });
```

## 4. pdf-parse v2 → v1

v2はAPIが変わっていて `require("pdf-parse")` が関数を返さなくなっていました。

```bash
npm install pdf-parse@1
```

# おわりに

TiDB Cloud Serverlessを使ってみて、SQL・ベクトル検索・（LIKEベースの）全文検索を1つのDBで扱えるのは確かに便利でした。サービスが分かれていないのでクライアントの設定がシンプルで済みますし、Vercelにデプロイするときの環境変数も少なく済みました。

今回はデータ量が少ないのでパフォーマンスはまだ気にしていませんが、ベクトル検索のHNSWインデックスがどう効いてくるかは、データを増やして試してみたいです。

今の実装は「質問を分類 → 検索 → 回答」という固定パイプラインです。OpenAIのFunction Callingを使ったAgentic RAGにすると、GPTが自分で検索ツールを選んで呼び出し、結果を見てさらに検索するかを判断する、みたいな形にできます。そっちの方が複雑な質問への対応が柔軟になりそうで、次にやってみたいことの一つです。

ずっと気になっていたTiDBを今回ようやく試せました。まだ触れていないモダンな機能もたくさんありそうで、今後のデータ基盤の選択肢の一つになりそうだと感じています。

間違いや改善点があればコメントで教えてもらえると助かります。

# 参考

- [TiDB Cloud Serverless](https://tidbcloud.com/)
- [TiDB Vector Search](https://docs.pingcap.com/tidbcloud/vector-search-overview)
- [神戸市オープンデータ](https://www.city.kobe.lg.jp/a47946/shise/toke/toukei/jinkoudata/jinkoudata.html)
  https://github.com/optimisuke/kobe-population-insight
