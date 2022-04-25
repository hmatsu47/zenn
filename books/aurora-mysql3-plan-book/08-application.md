---
title: "アプリケーションコードの修正"
---
## この章について

v1 → v3 変更点の調査結果をもとに、アプリケーションコードの修正対象になりやすい点を示します。

## アプリケーションコードの修正対象になりやすい変更点

すべてをカバーするものではありませんが、いくつかピックアップしていきます。

:::message
Chapter 05 で挙げた`tx_isolation`→`transaction-isolation`の変更もあります。
:::

### ライブラリ

DB 接続用のライブラリ（MySQL Connector の各言語対応版）を MySQL 8.0 対応バージョンに変更します。

その際、TLS 接続（TLS バージョンが合わない、非 SSL では接続できない等）の問題が生じる可能性がある点を留意しておきます。

#### TLS 接続の場合

TLS バージョンの問題が生じる可能性があります（Aurora MySQL 移行より前に新しいバージョンのライブラリに更新する場合）。

https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.Security.html#AuroraMySQL.Security.SSL.TLS_Version

#### 非 TLS 接続の場合

ライブラリのバージョンによっては、接続パラメータで SSL/TLS の無効化が必要になる場合があります。

例）

- MySQL Connector/J で、接続パラメータに`sslMode=DISABLED`を（追加）指定する

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
- Aurora MySQL v3・MySQL Connector/J 8.0 の組み合わせ:「`NOTNULL`列に`null`で`UPDATE`することはできない」旨のエラーが発生

のように変化するケースがあります。

もともと Aurora MySQL v1（MySQL 5.6）の時点でこのような SQL 文を実行するのは非推奨でしたが、同様の事象が発生した場合は`null`ではなく`NOW()`などで`UPDATE`する形に SQL 文（プリペアドステートメント）を書き換えます。

### 予約語のバッティング

すでに使用しているテーブル名・カラム名などに新しく予約語となったキーワード（`RANK`など）が含まれる場合は、前後を「`」で囲みます。

例）

- 既存コードについてはバッティング箇所のみ直す（「\`」で囲む）
- 新規コードについてはバッティングの有無に関わらずデータベース（スキーマ）名・テーブル名・カラム名・インデックス名やエイリアスを「\`」で囲む
  - `.`で連結する形式で書かれている箇所は必ずしも「\`」で囲む必要がないが、ミス防止のため全て「\`」で囲む

### `FOUND_ROWS()`（および`SQL_CALC_FOUND_ROWS`）

ページャ機能の実装で使っていたケースが多いと思います。

先に改修してリリースすると v1 運用中に性能低下の恐れがあることなどから、v3 移行後に改修するのが妥当でしょう。

### 文字セット・照合順序の問題

#### `utf8`・`utf8mb3`

`utf8`（`utf8mb3`のエイリアス）が残っている場合は、移行前または移行後に`utf8mb4`に変更します。

その際、照合順序の選択に注意が必要です。`utf8_general_ci`を使っていた場合は`utf8mb4_general_ci`が一番近い挙動になりますが、4 バイトの文字が区別できない欠点もあります。

https://mita2db.hateblo.jp/entry/2020/12/07/000000

#### `utf8mb4`のデフォルト照合順序

v2（MySQL 5.7）までとは異なり、`utf8mb4_0900_as_ci`がデフォルトになります。

無意識にデフォルトの指定を使っていると不具合が生じる可能性があるので、可能な限り明示的に指定します。

### データ型

`YEAR(2)`・`YEAR(4)`、`FLOAT(M,D)`・`DOUBLE(M,D)`、数値データ型の`ZEROFILL`属性などがありそうです。

中でも`YEAR(2)`はすでに廃止されているので移行前の修正が必要です。

### `GROUP BY 【列名】 ASC/DESC`

`GROUP BY 【列名】 ASC/DESC`の形式で書かれているもの以上に、 **暗黙の昇順ソート** （`ASC`が省略されているもの）が多そうです。

これは見逃しやすいので注意が必要です。

**`ORDER BY`を伴わない`GROUP BY`を含む SQL 文が、暗黙ソートを期待して書かれたものかどうかを必ず確認しましょう。**

### `UNION`の構文解析と動作の変更

`FOR UPDATE`との組み合わせがありそうです。

### Aurora MySQL 独自の問題

#### 参照専用（Reader）インスタンスでの`CREATE TEMPORARY TABLE (AS SELECT)`挙動変化

`innodb_read_only`が`1`のときに InnoDB ストレージエンジンで一時（テンポラリ）テーブルが作成できなくなったのに伴い、Reader インスタンスで`CREATE TEMPORARY TABLE … ENGINE=InnoDB`を実行する場合は SQL モード`NO_ENGINE_SUBSTITUTION`を無効にする必要があります。

ただし`AS SELECT`付きで実行する場合は SQL モードにかかわらずエラーになります。

そのため、Reader インスタンスで`CREATE TEMPORARY TABLE`を実行する場合は`ENGINE=InnoDB`を削除しておくのが良いでしょう。

- **[リーダー DB インスタンスのテンポラリテーブル](https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.MySQL80.html#AuroraMySQL.mysql80-temp-tables-readers)**