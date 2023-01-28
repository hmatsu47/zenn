---
title: "Supabase の PGroonga 全文検索で同義語検索してみる"
emoji: "📖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["supabase", "全文検索", "pgroonga"]
published: true
---

こちらの記事の続きです。

https://zenn.dev/hmatsu47/articles/supabase_pgroonga_flutter

PGroonga による全文検索で同義語検索できるようにしてみます。

:::message
Flutter アプリケーション側に変更はありません。
:::

PGroonga のドキュメントを参考に実装していきます。

https://pgroonga.github.io/ja/how-to/synonym-expansion.html

:::message
~~これを書いている 2023/1/28 現在、当該ページに Typo があるようです（Pull Request を送っておきました）~~ 修正されました。
https://github.com/pgroonga/pgroonga.github.io/pull/97
:::

## 同義語テーブルとインデックスを作成する

まずは同義語テーブルを作成します。

```sql:同義語テーブル作成
CREATE TABLE synonyms (
  term text PRIMARY KEY,
  synonyms text[]
);
```

そして、そのテーブルにインデックスを作成します。

```sql:同義語テーブル用インデックス作成
CREATE INDEX synonyms_search ON synonyms USING pgroonga (term pgroonga_text_term_search_ops_v2);
```

テーブルとインデックスの作成ができたら、同義語をテーブルに追加します。

```sql:同義語をテーブルに追加
INSERT INTO synonyms (term, synonyms) VALUES ('美術館', ARRAY['美術館', 'ミュージアム']);
INSERT INTO synonyms (term, synonyms) VALUES ('博物館', ARRAY['博物館', 'ミュージアム']);
INSERT INTO synonyms (term, synonyms) VALUES ('ミュージアム', ARRAY['ミュージアム', '美術館', '博物館']);
```

## ストアドファンクションを同義語検索対応にする

[前回の記事](https://zenn.dev/hmatsu47/articles/supabase_pgroonga_flutter#%E3%82%B9%E3%83%88%E3%82%A2%E3%83%89%E3%83%95%E3%82%A1%E3%83%B3%E3%82%AF%E3%82%B7%E3%83%A7%E3%83%B3%E3%82%92%E5%85%A8%E6%96%87%E6%A4%9C%E7%B4%A2%E5%AF%BE%E5%BF%9C%E3%81%AB%E3%81%99%E3%82%8B)で、`WHERE`の条件に

- **`ft_text &@~ keywords`**

と指定したところを、

- **`ft_text &@~ pgroonga_query_expand('synonyms', 'term', 'synonyms', keywords)`**

に書き換えます。

https://pgroonga.github.io/ja/reference/functions/pgroonga-query-expand.html

:::message
同義語が`OR`条件で列挙（展開）されるイメージです。
:::

```sql:CREATE_FUNCTION
CREATE OR REPLACE
 FUNCTION get_spots(point_latitude double precision, point_longitude double precision, dist_limit int, category_id_number int, keywords text)
RETURNS TABLE (
  distance double precision,
  category_name text,
  title text,
  describe text,
  latitude double precision,
  longitude double precision,
  prefecture text,
  municipality text
) AS $$
BEGIN
  RETURN QUERY
  SELECT ((ST_POINT(point_longitude, point_latitude)::geography <-> spot_opendata.location::geography) / 1000) AS distance,
    category.category_name,
    spot_opendata.title,
    spot_opendata.describe,
    ST_Y(spot_opendata.location),
    ST_X(spot_opendata.location),
    spot_opendata.prefecture,
    spot_opendata.municipality
  FROM spot_opendata
  INNER JOIN category ON spot_opendata.category_id = category.id
  WHERE
    (CASE WHEN dist_limit = -1 AND keywords = '' THEN false ELSE true END)
  AND
    (CASE WHEN dist_limit = -1 THEN true
      ELSE (ST_POINT(point_longitude, point_latitude)::geography <-> spot_opendata.location::geography) <= dist_limit END)
  AND
    (CASE WHEN category_id_number = -1 THEN true
      ELSE category.id = category_id_number END)
  AND
    (CASE WHEN keywords = '' THEN true
      ELSE ft_text &@~ pgroonga_query_expand('synonyms', 'term', 'synonyms', keywords) END)
  ORDER BY distance;
END;
$$ LANGUAGE plpgsql;
```

以上で全文検索が同義語対応になりました。

## SQL Editor で確認

SQL Editor で「ミュージアム」を検索してみると、

![](/images/supabase_pgroonga_synonyms/supabase_pgroonga_synonyms_01.png)

「美術館」「博物館」もヒットしていることがわかります。

:::message

- **[サンプルデータ](https://github.com/hmatsu47/maptool/tree/main/sampleData/supabase)**
  - このサンプルデータは、以下の著作物を改変して利用しています。
    - 愛知県文化財マップ（ナビ愛知）、愛知県、クリエイティブ・コモンズ・ライセンス 表示２.１日本
    - https://www.pref.aichi.jp/soshiki/joho/0000069385.html

:::

:::message
前述のとおり、Flutter アプリケーション側は変更不要です。
:::
