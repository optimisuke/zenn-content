---
title: "TiDB CloudでSQL・ベクトル検索・全文検索を1つのDBにまとめてAIアプリを作った"
emoji: "🏙️"
type: "tech"
topics: ["tidb", "nextjs", "openai", "rag", "typescript"]
published: false
---

# はじめに

RAGを作るとき、「構造化データ（DB）」と「非構造化データ（ベクトルDB）」を別々のサービスで管理するのが地味に面倒だと思ってました。

たとえば「統計データを SQLで検索しつつ、PDF文書はベクトル検索で引いてきて、まとめてAIに渡す」みたいな処理をしようとすると、RDBとベクトルDBとFTSを別々につないで、結果をマージして...ってやる必要があります。シンプルなユースケースのわりに、管理するサービスが増えてしんどいんですよね。

TiDB Cloudを調べたら、SQL・ベクトル検索・全文検索を単一のDBで扱えると知りました。ちょうどいい題材もあったので、実際に作って試してみました。

作ったのは、**神戸市の人口統計データ（CSV）と政策文書（PDF）を組み合わせて、自然文で質問できるAI分析アプリ**です。

https://github.com/optimisuke/kobe-population-insight

https://kobe-population-insight.vercel.app

# やりたいこと

こんな質問に答えられるアプリを目指しました。

- 神戸市はどの年代が流出している？（→ 人口統計CSVをSQL検索）
- 神戸市は人口減少をどう捉えている？（→ 政策PDF文書をベクトル検索・全文検索）
- 若者流出の実態と政策対応は？（→ 両方を組み合わせてAIが回答）

**1つのDBで構造化データ（統計）と非構造化データ（文書チャンク + embedding）を管理する**というのが今回の実験のポイントです。

# アーキテクチャ

```text
Browser (Next.js)
  └─ POST /api/chat { message, history }
       └─ orchestrator.ts
            ├─ [1] 質問分析 (gpt-5.4-mini)
            │       → 人口統計が必要? 政策文書が必要?
            │
            ├─ [2] 並列検索
            │   ├─ SQL検索: パラメータ抽出 → population テーブル
            │   └─ ベクトル検索 + 全文検索: policy_chunks テーブル
            │
            └─ [3] 回答生成 (gpt-5.5)
                    → 直近8件の会話履歴を積んで回答
```

TiDB Cloud Serverless の1つのDBに、人口統計テーブルと政策文書チャンクテーブルの両方を置いています。

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

`population` はCSVから投入した普通のテーブル。`policy_chunks` はPDFをチャンク分割してembeddingを生成したものです。同じDBに両方あるだけで、管理がだいぶ楽になります。

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

神戸市総合基本計画や神戸2030ビジョンの資料などをダウンロードして処理しました。

```typescript
// PDF → テキスト抽出 → チャンク分割 → embedding生成 → 挿入
const pdfBuffer = readFileSync(filePath);
const parsed = await pdfParse(pdfBuffer);

// 800文字・overlap100文字でチャンク分割
const chunks = chunkText(parsed.text, 800, 100);

// text-embedding-3-small でembedding生成してバルクインサート
const embeddings = await openai.embeddings.create({
  model: "text-embedding-3-small",
  input: chunks,
});
```

今回は236チャンク投入しました。

# 3種類の検索

## SQL検索（人口統計）

AIに直接SQLを生成させると インジェクションのリスクがあるので、AIはパラメータだけ抽出して固定テンプレートに当てはめる方式にしています。

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
  [params.metric, params.year]
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
  { fullResult: true }
);
```

## 全文検索（政策文書）

AIでキーワードを抽出してLIKE検索しています。マッチ数をスコア代わりにして並び替えています。

```typescript
// キーワード抽出
const keywords = await extractKeywords(question); // ["若年層", "人口減少", ...]

// キーワードごとにLIKE条件を生成
const likeConditions = keywords.map(() => `chunk_text LIKE ?`).join(" OR ");
const scoreExpr = keywords.map(() => `(CASE WHEN chunk_text LIKE ? THEN 1 ELSE 0 END)`).join(" + ");

const result = await db.execute(
  `SELECT id, source_title, chunk_text, (${scoreExpr}) AS score
   FROM policy_chunks
   WHERE ${likeConditions}
   ORDER BY score DESC LIMIT 5`,
  [...likeValues, ...likeValues, limit],
  { fullResult: true }
);
```

# ハマりポイント

実装中にいくつかハマりました。同じ構成を試す人の参考になれば。

## 1. 接続形式はURL形式が必要だった

`@tidbcloud/serverless` でindividual params形式（host, user, password を個別に渡す）で接続したら、「Missing user name prefix」というエラーが出ました。

TiDB Cloud Serverlessのユーザー名は `xxxxxxxx.root` という形式でプレフィックスが必要なのですが、individual params形式だとうまく渡せないようでした。URL形式にしたら動きました。

```typescript
// NG
connect({ host: "...", user: "xxxxxxxx.root", password: "..." })

// OK
connect({ url: "mysql://xxxxxxxx.root:password@host:4000/dbname?ssl=true" })
```

## 2. MATCH AGAINST が使えなかった

最初は全文検索を `MATCH AGAINST` で実装していましたが、実行したら `Error 1105 (HY000): UnknownType: *ast.MatchAgainst` というエラーが出ました。

TiDB Cloud Serverless のHTTP APIは MATCH AGAINST 構文に対応していないようです。

代わりにLIKEベースで実装し直しました。精度はそれなりですが、ベクトル検索と組み合わせればカバーできる感じがしてます。

## 3. USEステートメントがHTTP APIで効かなかった

スキーマ適用スクリプトで `USE kobe_population;` を実行したら効きませんでした。

TiDB Cloud Serverless のHTTP APIはステートレスなので、`USE` でのDB切り替えが毎回リセットされるようです。接続URLにDB名を含める形式にしたら解決しました。

```typescript
// DB名はURLに含める
connect({ url: `mysql://user:pass@host:4000/kobe_population?ssl=true` })
```

## 4. pdf-parse v2 → v1 にダウングレード

`pdf-parse` の最新版（v2）はAPIが変わっていて `require("pdf-parse")` が関数を返さなくなっていました。v1に戻したら動きました。

```bash
npm install pdf-parse@1
```

# 動作デモ

質問ごとにどの検索が使われたか、アコーディオンで確認できるようにしています。

- SQL検索のとき: 抽出されたパラメータ（年、指標など）とSQL結果の先頭数行を表示
- ベクトル検索・全文検索のとき: ヒットしたチャンクにどちらの検索でヒットしたかタグを表示

デバッグや精度確認のときに地味に便利です。

# おわりに

TiDB Cloud Serverlessを使ってみて、SQL・ベクトル検索・（LIKEベースの）全文検索を1つのDBで扱えるのは確かに便利でした。サービスが分かれていないのでクライアントの設定もシンプルで済みます。

ただ、MATCH AGAINST が使えないのは想定外でした。LIKEベースで代替できるとはいえ、スケールするとつらくなりそうなので、そのあたりの対応は今後気になるところです。

今回はデータ量が少ないのでパフォーマンスはまだ気にしていませんが、ベクトル検索のHNSWインデックスがどう効いてくるかは、データを増やして試してみたいです。

あと、今の実装は「質問を分類 → 検索 → 回答」という固定パイプラインなので、OpenAIのFunction Callingを使ってAgentic RAG的にしたいとも思っています。GPTが自分で検索ツールを選んで呼び出し、結果を見てさらに検索するかを判断する、みたいな形にできると面白そうです。

間違いや改善点があればコメントで教えてもらえると助かります。

# 参考

- [TiDB Cloud Serverless](https://tidbcloud.com/)
- [TiDB Vector Search](https://docs.pingcap.com/tidbcloud/vector-search-overview)
- [神戸市オープンデータ](https://www.city.kobe.lg.jp/a47946/shise/toke/toukei/jinkoudata/jinkoudata.html)
- [GitHub: kobe-population-insight](https://github.com/optimisuke/kobe-population-insight)
