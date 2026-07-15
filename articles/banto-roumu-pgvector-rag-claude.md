---
title: "労務の記憶をpgvectorに溜めてClaudeに文脈注入する — 個人開発の労務AI SaaSのRAG設計"
emoji: "🧠"
type: "tech"
topics: ["nextjs", "supabase", "pgvector", "claude", "rag"]
published: true
---

## 背景：キーワード検索だけでは「同じ会社」を覚えられない

番頭は、社労士試験に合格した知識をベースに、会社ごとの労務相談に答える個人開発のSaaSです。差別化の核は「その会社の過去の判断や規程を覚えていて、それを踏まえて答える」ことにあります。

最初のバージョンでは `company.ts` の中でキーワード＋2-gram部分一致による関連度スコアリングをしていました。ただこれには限界があって、「育休」と「育児休業」「産休明けの休み」のような言い換えを同じ話題として結びつけられません。労務相談は言い回しの揺れが大きい領域なので、意味で想起できる検索が必要でした。そこでpgvectorを使ったセマンティック検索を追加し、既存のキーワード検索は壊さずに併存させる設計にしています。

## 埋め込み層：SDKを足さずfetchで薄く作る

埋め込みの取得は `lib/embedding.ts` に閉じています。ここで意識したのは「壊れても黙って旧経路に戻す」ことです。

```ts
export function embeddingEnabled(): boolean {
  return Boolean(process.env.OPENAI_API_KEY)
}

export function toVectorLiteral(v: number[]): string {
  return `[${v.join(',')}]`
}
```

`OPENAI_API_KEY` が未設定なら `embeddingEnabled()` が `false` を返し、呼び出し側はネットワークすら叩かずキーワード検索に倒れます。API呼び出しが失敗してもthrowせずnullを返す方針も同じ思想で、意味検索はあくまで「効けば嬉しいオプション」として設計しています。個人開発でこの手のインフラ依存機能を作ると、外部APIの障害がそのままチャット機能全体の障害になりがちなので、フォールバックを最初から前提にしておくのは重要な判断でした。

もう一つの実装判断は依存を増やさないことです。OpenAIの公式SDKは足さず、`fetch` で直接 `/v1/embeddings` を叩いています。Vercelの関数サイズを太らせたくないのと、将来Voyageなど別プロバイダに差し替える可能性を考えると、このファイルの中だけで完結させたほうが移行コストが低いと判断しました。

次元数は `EMBEDDING_DIM = 1536` として `text-embedding-3-small` に固定し、SQLマイグレーション側の `vector(1536)` と厳密に一致させています。ここがずれると挿入・検索が問答無用で失敗するので、コメントで両ファイルを名指しして「変えるときは必ず揃える」と書いています。実際にモデル変更を検討したとき、次元数の不一致に気づかず一瞬ハマりかけた経験があり、コメントで縛るしかないと割り切りました。

もう一つ地味に効いているのが `toVectorLiteral` です。JS側の `number[]` をそのまま `supabase-js` のinsertやRPCに渡すとJSON配列としてシリアライズされてしまい、SQL側で `vector` にキャストできません。`[1,2,3]` のテキストリテラルに変換してから渡し、SQL側で `::vector` キャストする、という一手間が必要でした。

## チャンク設計：規程は原文まるごと、記憶は要約単位

チャンクの単位は2種類に分けています。

一つは `company_documents` テーブルに保存する規程の原文です。就業規則や賃金規程は条文単位で参照できないと「第◯条にはこうあります」という回答ができないため、要約せず原文まるごとを保持します。ただし暴走を防ぐためアプリ層で最大10万字、DBのCHECK制約で20万字を天井にしています。

```sql
content text NOT NULL CHECK (char_length(content) <= 200000),
char_count int NOT NULL DEFAULT 0,
...
UNIQUE (company_id, title)
```

タイトルを会社内でユニークにして、同名タイトルの再取込は上書き（upsert）にしています。規程は改定されるたびに全文が置き換わるものなので、バージョン管理より「常に最新の一本だけ持つ」方が運用がシンプルでした。

もう一つは `company_memories` の相談要約と `company_decisions` の過去判断です。こちらは1件あたり短い要約単位でチャンク化しており、埋め込みの入力上限（`MAX_INPUT_CHARS = 8000`）にも余裕を持って収まります。規程は原文そのまま、記憶は要約というチャンク粒度の使い分けが、番頭のRAG設計の軸になっています。

## 検索クエリと文脈注入：sonnetに渡す前のトークン予算

`/api/company/chat/route.ts` のフローは、まずユーザーを確認してcompany_idを確定させ、plan連動の利用上限をチェックしたうえで、`company_profiles` と直近の `company_memories` をsystemに注入し、法令キーワードが含まれていればDifyの一次情報も同梱してから、最終的にsonnetでストリーミング応答を返すという構成になっています。

```ts
const MAX_MEMORIES = 10
```

注入する記憶の件数を `MAX_MEMORIES = 10` に固定しているのは、トークン予算の都合です。会社の記憶は増え続けるので、全件をsystemに詰め込むとすぐにコンテキスト長を圧迫します。実運用では類似度上位10件＋直近のprofile数件で十分に「自社を覚えている」感触が出ることを確認し、この数値に落ち着きました。件数上限を関数のトップレベル定数として一箇所に出しているのは、後でチューニングする自分自身のためでもあります。

セマンティック検索が効く条件でも、キーワード検索の結果と完全に置き換えるのではなく、両方の結果を合成してから上位を取る設計にしています。これはembedding.tsのgraceful degrade思想と対になっていて、「意味検索が使えるときは足す、使えないときは黙って消える」という一貫性を保つためです。

## 想起の可視化：何を参照したかをPIIなしで見せる

番頭のもう一つの工夫が `lib/recall.ts` です。RAGで文脈注入した内容をユーザーに黙って使うのではなく、「この相談で自社のどの記憶を参照したか」を可視化するペイロードを組み立てています。

```ts
export interface RecalledMemory {
  profileCount: number
  memoryCount: number
  decisionCount: number
  ruleDocCount: number
  semantic: boolean
  items: RecalledItem[]
}
```

ここで最も気を使ったのは非PIIの原則です。相談の要約本文や氏名が入りうる `subject` は一切載せず、規程名・自社ルールのkey・判断のtopicラベルと日付だけをitemsに詰めています。相談記憶（memories）は本文を出さず件数のみにしているのも同じ理由です。

```ts
for (const p of ctx.profiles.slice(0, MAX_PROFILE_ITEMS)) {
  if (p.key) items.push({ kind: 'profile', label: p.key })
}
```

このペイロードは `X-Recalled-Memory` というレスポンスヘッダに載せて返しています。ボディのストリームを汚さずに済み、フロント側は既存のストリーミング処理を変えずに「今回はこの規程とこの過去の判断を踏まえています」という表示を追加できました。件数上限（`MAX_PROFILE_ITEMS` など）を設けているのはヘッダサイズを小さく保つためで、これもトークン予算とは別軸の「ヘッダ予算」の設計判断です。

## RLSとテナント分離

`company_documents.sql` では、読取りをメンバー全員、書込みをadminのみに絞るRLSポリシーを敷いています。既存の `is_company_member` / `is_company_admin` というSECURITY DEFINER関数を再利用し、新規関数を増やさないようにしたのは、認可ロジックが分散すると監査が難しくなるからです。RAGで参照する原文がテナントをまたいで漏れることは何としても避けたいポイントなので、埋め込み検索のRPCも同じ会社IDでフィルタしたクエリの中でしか呼ばれないようにしています。

## まとめ

番頭のRAG設計は、意味検索を「あれば嬉しい上乗せ」として位置づけ、キーワード検索や規程原文参照という既存の仕組みを壊さずに足していくことを徹底しました。埋め込みAPIの障害やキー未設定は黙ってフォールバックし、注入するチャンク数とヘッダのサイズには明示的な上限を置く。派手なRAGフレームワークを使わずとも、graceful degradeとトークン予算の管理さえ丁寧にやれば、個人開発でも実用に耐える労務AIの記憶機構は組めるという手応えを得ています。

---

毎回ゼロから説明する労務相談をやめるための労務AI「番頭」を、個人開発で作っています。無料で試せます。

https://banto-roumu.com/business?utm_source=zenn&utm_medium=cta&utm_campaign=banto_hint&utm_content=zenn_b
