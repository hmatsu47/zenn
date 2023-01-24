---
title: "Flutter で Supabase の PGroonga 全文検索を試してみた"
emoji: "🔍"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["flutter", "supabase", "全文検索"]
published: false
---

Supabase が PGroonga に対応したと聞いたので、Flutter で試してみました。

https://www.clear-code.com/blog/2023/1/17/supabase-support-pgroonga.html

題材のベースは、以前こちらの記事で言及した地図アプリです。

https://qiita.com/hmatsu47/items/c3f9cafb499aedaca1f1

:::message
記事に書いた後、若干改修しています。
:::

## PGroonga を有効にする

**「Database」-「Extensions」** の **「Available extensions」** から **「PGROONGA」** を探して有効化します。

![](/images/supabase_pgroonga_flutter/supabase_pgroonga_01.png)

有効化すると **「PGROONGA」** が **「Enabled extensions」** に移動します。

![](/images/supabase_pgroonga_flutter/supabase_pgroonga_02.png)

:::message
Extensions 一覧に「PGROONGA」が表示されない場合は、**「Setting」-「General」-「Infrastructure」** の **「Pause project」** でプロジェクトを一時停止し、その後再び起動します。
:::

## 対象テーブルに全文検索用インデックスを追加する

**「SQL Editor」** で **「New query」** から SQL 文を実行（RUN）します。

```sql:全文検索用インデックス追加
ALTER TABLE spot_opendata ADD COLUMN ft_text text GENERATED ALWAYS AS (title || ',' || describe || ',' || prefecture || municipality) STORED;
CREATE INDEX pgroonga_content_index
          ON spot_opendata
       USING pgroonga (ft_text)
        WITH (tokenizer='TokenMecab');
```

## ストアドファンクションを全文検索対応にする

同様に、**「SQL Editor」** で **「New query」** から SQL 文を実行（RUN）します。

```sql:ストアドファンクション全文検索対応化
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
    (CASE WHEN dist_limit = -1 THEN true
      ELSE (ST_POINT(point_longitude, point_latitude)::geography <-> spot_opendata.location::geography) <= dist_limit END)
  AND
    (CASE WHEN category_id_number = -1 THEN true
      ELSE category.id = category_id_number END)
  AND
    (CASE WHEN keywords = '' THEN true
      ELSE ft_text &@~ keywords END)
  ORDER BY distance;
END;
$$ LANGUAGE plpgsql;
```

:::message alert
以前のストアドプロシージャは、引数の数が違うので別物と見なされて`REPLACE`されずにそのまま残ってしまいます。
**「Database」-「Functions」** の **「schema public」** 一覧から **「Delete function」** するか、SQL Editor で`DROP FUNCTION`してください。
:::

## アプリケーションに全文検索を組み込む
