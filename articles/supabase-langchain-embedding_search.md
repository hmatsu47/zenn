---
title: "langchain-embedding_search を使って Supabase を Vector Store として使う"
emoji: "↗"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["supabase", "langchain", "embeddings", "typescript", "openaiapi"]
published: true
---

先日（2023/4/24）の **[第 41 回 PostgreSQL アンカンファレンス @ オンライン](https://pgunconf.connpass.com/event/278329/)** で話したネタです。

https://pgunconf.connpass.com/event/278329/

**[PostgreSQL パッケージマネージャー dbdev](https://database.dev/)** を使って **[Supabase](https://supabase.com/)** に **[langchain-embedding_search](https://database.dev/langchain/embedding_search)** を導入し、Supabase を Vector Store として使ってみます。

:::message alert
**2024/4/30 追記：**
2024 年現在、LangChain のパッケージ構成およびコードの書き方が当記事の執筆時点とは大幅に変わっています。
:::

## dbdev とは？

PostgreSQL の拡張機能や SQL 集を管理するパッケージマネージャーとして Supabase が提供するものです。

https://database.dev/

[Supabase のブログ記事](https://supabase.com/blog/dbdev) には以下のような記述があります。

- （2023/4/14 現在）パブリックプレビュー中
- PostgreSQL において、JavaScript にとっての`npm`や Python にとっての`pip`、Rust にとっての`cargo`の役割を目指している
- バージョニング・名前空間の管理機能を持つ

## langchain-embedding_search とは？

Supabase の DB（PostgreSQL）を Vector Store として使うためのパッケージです。

https://database.dev/langchain/embedding_search

Vector Store は、

- ベクトル化した文書を保存し
- LLM（大規模言語モデル）のプロンプトに文脈を与える際に関連文書を検索・抽出する

目的で使うデータストアです。

::: message
説明としては、[このあたりのスライド資料](https://speakerdeck.com/os1ma/puronputoenziniaringukarashi-merulangchainru-men?slide=49)がわかりやすいと思います。
:::

**[langchain-embedding_search](https://database.dev/langchain/embedding_search)** では、拡張機能 **[pgvector](https://github.com/pgvector/pgvector)** を用いて、

- ベクトル型の列を持つ`documents`テーブル
- コサイン類似度を使って関連文書を検索する`match_documents`ストアドファンクション

を Supabase プロジェクト上に生成します。

## 使ってみる

### Supabase プロジェクト作成

新規のプロジェクトを作成するか、既存プロジェクトの環境を最新にして、最新の拡張機能が使える状態にします。

:::message
既存プロジェクトの環境を最新にするには、[こちらの記事の注意書き](https://zenn.dev/hmatsu47/articles/supabase_pgroonga_flutter#pgroonga-%E3%82%92%E6%9C%89%E5%8A%B9%E3%81%AB%E3%81%99%E3%82%8B)を参考にしてください。
:::

### dbdev 有効化

**「SQL Editor」** で[こちらのページ](https://database.dev/installer)に書かれている SQL 文をそのまま実行します。

:::message
SQL 文中の`apiKey`の内容も変更不要です。
:::

### pgvector 有効化

同じく **「SQL Editor」** で次の SQL 文を実行します。

```sql:pgvector有効化
create extension if not exists vector;
```

### langchain-embedding_search インストール・有効化

さらに **「SQL Editor」** で次の SQL 文を実行します。

```sql:langchain-embedding_searchインストール・有効化
select dbdev.install('langchain-embedding_search');
create extension "langchain-embedding_search"
    version '1.0.0';
```

- `documents`テーブル
- `match_documents`ストアドファンクション

が生成されます。

:::message alert
このとき生成される`documents`テーブルは RLS 無効状態です。プロダクト利用する際は RLS を適切に設定しましょう。
:::

:::message alert
**2023/6/8 追記：**
2023/6/8 現在、最新バージョンは 1.1.0 のはずですが、dbdev ではまだ対応していないようです（1.1.0 を指定するとエラーが発生）。

> _Failed to run sql query: extension "langchain-embedding_search" has no installation script nor update path for version "1.1.0"_

また、その関係で、`langchainjs`バージョン 0.0.88 以降を使うと、`match_documents`ストアドファンクションと引数が合わず、langchainjs からの呼び出しがエラーになります。

そのため、`langchainjs`バージョン 0.0.88 以降を使うときは、`langchain-embedding_search`バージョン 1.1.0 と同じ`fliter`引数つきの`match_documents`ストアドファンクションを手動で追加します。

```sql:match_documentsストアドファンクション手動追加
create function match_documents (
  query_embedding vector(1536),
  match_count int DEFAULT null,
  filter jsonb DEFAULT '{}'
) returns table (
  id bigint,
  content text,
  metadata jsonb,
  similarity float
)
language plpgsql
as $$
#variable_conflict use_column
begin
  return query
  select
    id,
    content,
    metadata,
    1 - (documents.embedding <=> query_embedding) as similarity
  from documents
  where metadata @> filter
  order by documents.embedding <=> query_embedding
  limit match_count;
end;
$$;
```

なお、このワークアラウンドで対応した後に`langchain-embedding_search`バージョン 1.1.0 が有効化できるようになったときは、この`match_documents`ストアドファンクションを手動で削除してから有効化します。
:::

### Supabase プロジェクトの URL と匿名（anon）キーを確認

**「API Settings」** で **「Project URL」** と **「Project API keys」** の `anon`・`public`キーを確認しておきます。

### OpenAI API キーを発行・確認

**[OpenAI API keys](https://platform.openai.com/account/api-keys)** からキーを発行し確認しておきます。

:::message
OpenAI への登録直後であれば、無料枠で十分試せる範囲です。
:::

### サンプルコードを取得

[こちらの GitHub リポジトリ](https://github.com/hmatsu47/dbdev-langchain-test) にサンプルコードを用意しておきました。

https://github.com/hmatsu47/dbdev-langchain-test

手元に clone 後`npm install`します。

その後、clone したディレクトリ直下に`.env`ファイルを作成し、Supabase のプロジェクト URL / 匿名キーと OpenAI の API キーを記述します。

```text:.env
VITE_SUPABASE_URL=【SupabaseのプロジェクトURL】
VITE_SUPABASE_ANON_KEY=【Supabaseのプロジェクトanon key】
VITE_OPENAI_KEY=【OpenAIのAPI key】
```

:::message
Vector Store への文書追加と検索のコードはこちらです。

```typescript:文書追加
// ドキュメント追加
const postDocuments = async (body: { contents: string[]; metadata: Embeddings; }) => {
  await SupabaseVectorStore.fromTexts(
    body.contents,
    body.metadata,
    new OpenAIEmbeddings({
      openAIApiKey: import.meta.env.VITE_OPENAI_KEY,
    }),
    {
      client: supabaseClient,
      tableName: "documents",
      queryName: "match_documents",
    }
  );
  const results = {
    message: "OK"
  }
  return results;
}
```

```typescript:文書検索
// 検索
const search = async (keyword: string, count: number) => {
  const vectorStore = await SupabaseVectorStore.fromExistingIndex(
    new OpenAIEmbeddings({
      openAIApiKey: import.meta.env.VITE_OPENAI_KEY,
    }),
    {
      client: supabaseClient,
      tableName: "documents",
      queryName: "match_documents",
    }
  );
  const results = await vectorStore.similaritySearch(keyword, count);
  return results;
};
```

:::

### 起動

`npm run dev`で起動します。

### テストデータ投入

_`curl -X POST -H 'Content-Type: application/json; charset=UTF-8' http://localhost:【起動ポート番号】 -d '{"contents":【ドキュメント配列】, "metadata":【メタデータ配列】}'`_ で投入できます。 _`-d @【投入するファイルのパス】`_ も可能です。

[こちらのテストデータ](https://github.com/hmatsu47/dbdev-langchain-test/blob/master/test-data.json) を`curl`で投入します。

https://github.com/hmatsu47/dbdev-langchain-test/blob/master/test-data.json

```json:テストデータ投入
% curl -X POST -H 'Content-Type: application/json; charset=UTF-8' http://localhost:3699 -d @./test-data.json
{"message":"OK"}%
```

![](/images/supabase-langchain-embedding_search/documents_table.png)

:::message
日本 PostgreSQL ユーザ会に馴染みの深い方の著書 4 冊の紹介ページから引用しました。

- 失敗から学ぶ RDB の正しい歩き方
  - https://gihyo.jp/book/2019/978-4-297-10408-5
- ［改訂 3 版］内部構造から学ぶ PostgreSQL
  - https://gihyo.jp/book/2022/978-4-297-13206-4
- データベース初心者のための PostgreSQL 教室
  - https://nextpublishing.jp/book/11673.html
- これからはじめる PostgreSQL 入門
  - https://gihyo.jp/book/2018/978-4-7741-9814-9

:::

### 検索

_`curl http://localhost:【起動ポート番号】/【検索キーワード】/【結果の最大数】`_ で検索が可能です。

実際に検索してみます。

```json:検索1
% curl http://localhost:3699/初心者/2 | jq
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   966  100   966    0     0    841      0  0:00:01  0:00:01 --:--:--   845
[
  {
    "pageContent": "本書はデータベース初心者およびPostgreSQL初心者向けの入門書です。データベースとは何か？からPostgreSQLのインストール、SQLの実行、トランザクションについて、レプリケーション、バックアップまでを解説しています。",
    "metadata": {
      "title": "データベース初心者のためのPostgreSQL教室",
      "author": [
        "目黒 聖"
      ]
    }
  },
  {
    "pageContent": "本書は，データベース初学者を対象にPostgreSQLを使って，データベース操作の基本から運用までを学ぶための本です。付属DVD収録のファイルを利用することで，自宅のWindowsパソコンやMacで実際にデータの検索や更新などを行いながら，PostgreSQLによるリレーショナルデータベースの操作をマスターすることができます。",
    "metadata": {
      "title": "これからはじめる PostgreSQL入門",
      "author": [
        "高塚 遙",
        "桑村 潤"
      ]
    }
  }
]
```

```json:検索2
% curl http://localhost:3699/辛い/1 | jq
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   914  100   914    0     0    444      0  0:00:02  0:00:02 --:--:--   445
[
  {
    "pageContent": "「データベースがよく落ちる」「前任者が残したテーブル，SQLが読み解けない」「RDBMSを入れ替えたら予期せぬバグが」――MySQLやPostgreSQLといったRDBMS（リレーショナルデータベース管理システム）を使った業務システム，Webサービスを設計・運用していると，こういった問題によく直面するのではないでしょうか。本書はRDB（リレーショナルデータベース）の間違った使い方（＝アンチパターン）を紹介しながら，アンチパターンを生まないためのノウハウを解説します。それぞれの章では，問題解決に必要なRDBやSQLの基礎知識も押さえるので，最近RDBMSを触り始めた新人の方にもお勧めです。",
    "metadata": {
      "title": "失敗から学ぶ RDBの正しい歩き方",
      "author": [
        "曽根壮大"
      ]
    }
  }
]
```

まあまあそれっぽい検索結果が出ていますね。

## 注意

- Node.js v16 では動きません。`fetch`を（試験的に）サポートする Node.js v18 以降で実行してください。
- 全体的に入力値チェックなどは実装していません。

## 参考

### LangChain JS/TS Docs Supabase

https://js.langchain.com/docs/modules/indexes/vector_stores/integrations/supabase

### LangChain Python Docs SupabaseVectorStore

https://python.langchain.com/en/latest/modules/indexes/vectorstores/examples/supabase.html

:::message
今回は Python のライブラリは使っていません。
Supabase に関しては JS/TS のライブラリのほうが実装が先行しているようです。
:::

### サンプルコード（`app.ts`）全体

https://github.com/hmatsu47/dbdev-langchain-test/blob/master/app.ts

### その他

文書ベクトルと全文検索を併用する **[langchain-hybrid_search](https://database.dev/langchain/hybrid_search)** もあります。

https://database.dev/langchain/hybrid_search
https://js.langchain.com/docs/modules/indexes/retrievers/supabase-hybrid
