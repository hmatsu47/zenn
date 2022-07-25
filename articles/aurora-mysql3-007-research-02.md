---
title: "Amazon Aurora MySQL v1（5.6 互換）→ v3（8.0 互換）移行を計画する（7）パラメータ・コード確認"
emoji: "📑"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["aws", "aurora", "mysql", "移行", "バージョンアップ"]
published: true
---

これは

- **[Amazon Aurora MySQL v1（5.6 互換）→ v3（8.0 互換）移行を計画する（6）詳細調査について](/hmatsu47/articles/aurora-mysql3-006-research-01)**

の続きです。

詳細調査結果を実際のパラメータグループやアプリケーション・コード（SQL 文）などと突き合わせて確認してみます。

:::message
ここでは、私が関わっているサービスを例に確認しています。
ただし、一部実際とは異なる部分もあります。
:::

## パラメータグループの検討

https://github.com/hmatsu47/aurora_mysql1to3diff/blob/main/aurora-mysql1_3_param.md

こちらと実際に使用しているパラメータグループ（クラスタと DB）を比較して、対応が必要な点を見付けます。

私が関わっているサービスでは、以下の点が出てきました。

### `innodb_strict_mode`のデフォルトが **厳密モード** に

https://dev.mysql.com/doc/refman/8.0/ja/innodb-parameters.html#sysvar_innodb_strict_mode

これは正確には Aurora MySQL v2（MySQL 5.7）の時点でデフォルトが変わっています。

一部の SQL 文の実行で、従来は警告（Warning）で済ませていたのをエラーで止める動作に変わりました。ほぼ問題にならない見込みですが、念のため従来の設定に合わせて厳密モードを OFF にします。

:::message
関連する項目に **[`sql-mode`](https://dev.mysql.com/doc/refman/8.0/ja/sql-mode.html)** があります。
こちらは本家 MySQL のデフォルトとは違い、Aurora MySQL のデフォルトパラメータグループでは v1 から v3 まで`0`（本家 MySQL では **無指定** に相当）です。
:::

### `innodb_autoinc_lock_mode`のデフォルトが`1`から`2`に

**`AUTO_INCREMENT`値** 生成時のロックモードです。

https://dev.mysql.com/doc/refman/8.0/ja/innodb-parameters.html#sysvar_innodb_autoinc_lock_mode

`2`のほうが性能上有利なのですが、私が関わっているサービスでは、MySQL Connector/J の **[`rewriteBatchedStatements`オプション](https://dev.mysql.com/doc/connector-j/8.0/en/connector-j-connp-props-performance-extensions.html#cj-conn-prop_rewriteBatchedStatements)** を使ったバッチ`INSERT`と **[LAST_INSERT_ID()](https://dev.mysql.com/doc/refman/8.0/ja/information-functions.html#function_last-insert-id)** を組み合わせた処理を行っている部分があったので、動作の問題が生じる可能性を考えて従来と同じ`1`に戻します。

一方、以下の点については特に問題にならないことがわかりました。

### `tx_isolation`が`transaction-isolation`に変わった

これはパラメータグループというよりも、アプリケーションから DB に接続する部分の実装やライブラリのオプション設定に関わる問題ですが、以前触れた **[サイボウズのブログ記事](https://blog.cybozu.io/entry/2021/05/24/175000#%E3%83%88%E3%83%A9%E3%83%B3%E3%82%B6%E3%82%AF%E3%82%B7%E3%83%A7%E3%83%B3%E5%88%86%E9%9B%A2%E3%83%AC%E3%83%99%E3%83%AB%E3%82%92%E8%A8%AD%E5%AE%9A%E3%81%99%E3%82%8B%E5%A4%89%E6%95%B0%E5%90%8D%E3%81%AE%E5%A4%89%E6%9B%B4)** ではこの変更の影響を受けていました。

私が関わっているサービスではこのパラメータをアプリケーションから変更していないので、対応は不要です。

## アプリケーション・コードとテーブル定義の検討

### ライブラリ

DB 接続用のライブラリ（MySQL Connector/J）を MySQL 8.0 対応バージョンに変更します。

その際、TLS 接続（TLS バージョンが合わない、非 SSL では接続できない等）の問題が生じる可能性がある点を留意しておきます。

#### TLS 接続の場合

TLS バージョンの問題が生じる可能性があります（Aurora MySQL 移行より前に新しいバージョンのライブラリに更新する場合）。

https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.Security.html#AuroraMySQL.Security.SSL.TLS_Version

#### 非 TLS 接続の場合

ライブラリのバージョンによっては、接続パラメータに`sslMode=DISABLED`が必要になる場合があります。

https://dev.mysql.com/doc/connector-j/8.0/en/connector-j-connp-props-security.html

:::message
移行作業中に互換性の問題が生じる恐れがある場合は、代わりに非推奨の`useSSL=false`を指定し、問題が生じる恐れがなくなった時点で`sslMode=DISABLED`に書き換えます。
:::

#### その他

パラメータグループでデフォルトどおり`explicit_defaults_for_timestamp`に`1`（有効）を指定した状態で、

- MySQL Connector/J のプリペアドステートメントを使って
- `NOT NULL`かつ`DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP`の`TIMESTAMP`・`DATETIME`列に対して
- `null`で`UPDATE`したとき

の挙動が、

- Aurora MySQL v1・MySQL Connector/J 5.1 の組み合わせ:`UPDATE`時のタイムスタンプで更新
- Aurora MySQL v3・MySQL Connector/J 8.0 の組み合わせ:「`NOT NULL`列に`null`で`UPDATE`することはできない」旨のエラーが発生

のように変化するケースがあります。

同様の事象が発生した場合は`null`ではなく`NOW()`などで`UPDATE`する形に SQL 文（プリペアドステートメント）を書き換えます。

また、Connector/J 8.0.29 で`rewriteBatchedStatements=true`のときに`ROW_COUNT()`の結果が不正な事象も見つかっています（8.0.28 に戻して回避）。

### SQL 文とテーブル定義

こちらもいくつか検討事項が出てきましたが、結局、事前改修の対象は予約語とのバッティング箇所だけになりそうです。

#### 予約語

https://github.com/hmatsu47/aurora_mysql1to3diff/blob/main/mysql57_80_reserved.md

既存コードでは`OF`と`RANK`のバッティングが確認されました。この結果をふまえて、今回は一旦以下の対応とします。

- 既存コードについてはバッティング箇所のみ直す（「\`」で括る）
- 新規コードについてはバッティングの有無に関わらずデータベース（スキーマ）名・テーブル名・カラム名・インデックス名やエイリアスを「\`」で括る
  - `.`で連結する形式で書かれている箇所は必ずしも「\`」で括る必要がないが、ミス防止のため全て「\`」で括る

#### ビルトイン関数

https://github.com/hmatsu47/aurora_mysql1to3diff/blob/main/mysql57_80_func_oper.md#%E3%83%93%E3%83%AB%E3%83%88%E3%82%A4%E3%83%B3%E9%96%A2%E6%95%B0

**[サイボウズのブログ記事](https://blog.cybozu.io/entry/2021/05/24/175000#SQL_CALC_FOUND_ROWS-%E3%81%A8-FOUND_ROWS-%E3%81%8C-deprecated-%E3%81%AB)** にもあった、`FOUND_ROWS()`（および`SQL_CALC_FOUND_ROWS`）が見つかりました。

こちらはページャ機能の実装で使っていますが、

- 先に改修してから Aurora MySQL v3 に移行すると、Aurora MySQL v1 運用中の性能低下の恐れがあること
- 今後ページャ機能からスクロールなど別の UI に切り替える検討を進めていること

などから、移行前の改修は見送ることにします。

#### 文字セット・照合順序

https://github.com/hmatsu47/aurora_mysql1to3diff/blob/main/mysql57_80_manual_all.md#%E6%96%87%E5%AD%97%E3%82%BB%E3%83%83%E3%83%88%E7%85%A7%E5%90%88%E9%A0%86%E5%BA%8F

一部`utf8`（`utf8mb3`のエイリアス）が残っていますが、現時点ではまだ「非推奨」なので、都合上今回の移行とは別のタイミングで対応します。

#### データ型

https://github.com/hmatsu47/aurora_mysql1to3diff/blob/main/mysql57_80_manual_all.md#%E3%83%87%E3%83%BC%E3%82%BF%E5%9E%8B

`DOUBLE(M,D)`がありました。こちらも現時点では「非推奨」であり、一旦別タイミングでの対応とします。

#### その他 SQL ステートメントなど

https://github.com/hmatsu47/aurora_mysql1to3diff/blob/main/mysql57_80_manual_all.md#sql-%E3%82%B9%E3%83%86%E3%83%BC%E3%83%88%E3%83%A1%E3%83%B3%E3%83%88%E3%81%AA%E3%81%A9

**[サイボウズのブログ記事](https://blog.cybozu.io/entry/2021/05/24/175000#GROUP-BY-a-ASC-%E3%81%AE%E5%BB%83%E6%AD%A2)** にもあった`GROUP BY 【列名】 ASC/DESC`ですが、 **暗黙の昇順ソート** （`ASC`が省略されているもの）が比較的多数見つかりました。

これは見逃しやすいので注意が必要です。

## 実際に動かしてみないとわからないものもある

ここまでは調査資料から判別できましたが、一部、それが難しい項目もあります。

### `RIGHT JOIN`の修正（MySQL 8.0.22）

MySQL 8.0.22 のリリースノートの **[Optimizer Notes](https://dev.mysql.com/doc/relnotes/mysql/8.0/en/news-8-0-22.html#mysqld-8-0-22-optimizer)** に、

> When using a RIGHT JOIN, some internal objects, were not converted to those suitable for use with a LEFT JOIN as intended. These included some lists of tables built at parse time, but which did not have their order reversed. This required maintaining code to handle instances in which a LEFT JOIN was originally a RIGHT JOIN as special cases, and was the source of several bugs. Now the server performs any necessary reversals at parse time, so that after parsing, a RIGHT JOIN is in fact, in all respects, a LEFT JOIN. (Bug #30887665, Bug #30964002)
>
> References: See also: Bug #12567331, Bug #21350125.

という項目があります。`RIGHT JOIN`実行時、内部的に`LEFT JOIN`に書き換える処理に不適切な部分があったのを改修したという趣旨ですが、これらのバグトラッキング番号の情報がクローズされてしまい内容が読めないので、こちらは実際に動作を確認してみる必要がありそうです。

### `UNION`の構文解析と動作の変更（MySQL 8.0.19 など）

マニュアルの UNION 句のページ中、

https://dev.mysql.com/doc/refman/8.0/ja/union.html#union-distinct-all

ここから下の各項目については内容を読む限りでは問題はなさそうですが、カッコの記述などが動作に影響する可能性があるため、念のため実際の動作確認の際に変化が生じないか気を付ける必要がありそうです。

:::message
**[サイボウズの記事](https://blog.cybozu.io/entry/2021/05/24/175000#UNION-%E5%8F%A5%E3%81%A8-FOR-UPDATE-%E3%82%92%E4%BD%BF%E3%81%86%E5%A0%B4%E5%90%88%E3%81%AB%E6%8B%AC%E5%BC%A7%E3%81%8C%E8%BF%BD%E5%8A%A0%E3%81%A7%E5%BF%85%E8%A6%81)** では`FOR UPDATE`との組み合わせについて触れています。
:::

---

- **[Amazon Aurora MySQL v1（5.6 互換）→ v3（8.0 互換）移行を計画する（8）動作確認](/hmatsu47/articles/aurora-mysql3-008-ope-check)**

に続きます。
