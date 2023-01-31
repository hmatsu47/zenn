---
title: "Supabase の PGroonga 全文検索でストップワード対応のワークアラウンドを試してみる"
emoji: "⏹"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["supabase", "全文検索", "pgroonga"]
published: true
---

こちらの記事の続きです。

https://zenn.dev/hmatsu47/articles/supabase_pgroonga_flutter
https://zenn.dev/hmatsu47/articles/supabase_pgroonga_synonyms

同義語検索の次は、ストップワード（検索対象から除外するキーワードや文字）の設定を…と思ったのですが、

https://ja.osdn.net/projects/groonga/lists/archive/dev/2018-October/004705.html
https://ja.osdn.net/projects/groonga/lists/archive/dev/2018-November/004706.html

こちらのスレッドに書かれているとおり、**PGroonga は Groonga の`TokenFilterStopWord`には対応していない** ようです。

:::message
`CREATE INDEX ... WITH`で`token_filters`に`TokenFilterStopWord`を指定することはできるものの、ストップワード用の辞書を指定することができないようです。
:::

そのため、**インデックスおよび検索時の`WHERE`句の指定（正規表現による文字列の除外）で、特定のキーワードや文字種を除外するワークアラウンド** を試してみます。

:::message
今回も Flutter アプリケーション側は変更不要です。
:::

## ワークアラウンドの内容

先ほどのメーリングリストのアーカイブのスレッドで示されているワークアラウンドは、

https://ja.osdn.net/projects/groonga/lists/archive/dev/2018-November/004708.html

PostgreSQL の式インデックスを使う方法でした。

今回は PGroonga の全文検索インデックスの適用対象が生成列なので、式インデックスは使わず、列生成時の式を変更して対応します。

:::message
ストップワードが少ないケースで利用可能なワークアラウンドです。ストップワードが多いケースにはこのワークアラウンドは向きません。
:::

## 検索対象テーブルの生成列とインデックスを変更してストップワードに対応する

今回ストップワード対応するアプリケーションのサンプルデータはこちらです。

- **[サンプルデータ](https://github.com/hmatsu47/maptool/tree/main/sampleData/supabase)**
  - このサンプルデータは、以下の著作物を改変して利用しています。
    - 愛知県文化財マップ（ナビ愛知）、愛知県、クリエイティブ・コモンズ・ライセンス 表示２.１日本
    - https://www.pref.aichi.jp/soshiki/joho/0000069385.html

検索対象文字列の中では、「の」および「・」の 2 種類の文字が入力キーワードの揺らぎでうまく検索できない可能性がありそうです。

そのため、これらを`REGEXP_REPLACE`関数で正規表現を使って除外します。

```sql:テーブル・インデックスをストップワード対応に
DROP INDEX pgroonga_content_index;
ALTER TABLE spot_opendata
  DROP COLUMN ft_text,
  ADD COLUMN ft_text text GENERATED ALWAYS AS
    (REGEXP_REPLACE((title || ',' || describe || ',' || prefecture || municipality), '[の・]', '', 'g')) STORED;
CREATE INDEX pgroonga_content_index
          ON spot_opendata
       USING pgroonga (ft_text)
        WITH (tokenizer='TokenMecab');
```

:::message
この例は **文字の除外** なので正規表現として`[]`の中に除外対象文字種を指定していますが、キーワードを除外する場合は正規表現として複数のキーワードを`|`（パイプ）で区切りで列挙する形で指定します。
:::

## ストアドファンクションを変更してストップワードに対応する

`WHERE`句の全文検索部分の条件を変更します。

同義語対応のときと同様、1 行だけの変更です。

:::message
`pgroonga_query_expand`関数の 4 つ目のパラメータ（`keywords`：検索キーワード）を、全文検索用の生成列と同じように`REGEXP_REPLACE`関数を使う形に書き換えます。
:::

```sql:ストアドファンクション・ストップワード対応
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
      ELSE
        ft_text &@~ pgroonga_query_expand('synonyms', 'term', 'synonyms', REGEXP_REPLACE(keywords, '[の・]', '', 'g'))
      END)
  ORDER BY distance;
END;
$$ LANGUAGE plpgsql;
```

## SQL Editor で確認

SQL Editor で「マインドスケープミュージアム」を検索してみると、

![](/images/supabase_pgroonga_stopword_wa/supabase_pgroonga_stopword_wa_01.png)

途中に「・」を含む「マインドスケープ・ミュージアム」がヒットしていることがわかります。

:::message
前述のとおり、Flutter アプリケーション側は変更不要です。
:::
