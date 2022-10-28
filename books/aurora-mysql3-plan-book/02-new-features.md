---
title: "Aurora MySQL v3 の新機能"
---

## この章について

移行計画の前に、Aurora MySQL v3 の新機能について簡単に触れておきます。

## 本家（コミュニティ版）MySQL 8.0 由来の新機能

Aurora MySQL v3 はコミュニティ版 MySQL 8.0 をベースに開発されているため、コミュニティ版 MySQL 8.0 の新機能の多くが利用可能です。

https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.MySQL80.html#AuroraMySQL.8.0-features-community
https://dev.mysql.com/doc/refman/8.0/ja/mysql-nutshell.html

### 補足資料

https://speakerdeck.com/hmatsu47/mysql-8-dot-0hefalseyi-xing-wokao-eru
https://github.com/hmatsu47/mysql80_no_usui_hon

### 主な新機能

- **ウィンドウ関数**
  - **[MySQL 8.0.2 DMR でウィンドウ関数がサポートされたので、RANK 関数を試してみる](https://qiita.com/hmatsu47/items/6cc0e69f3895f3e4a486)**
- **共通テーブル式（CTE・`WITH`句）**
  - **[いまさら MySQL 8.0 で共通テーブル式（CTE : Common Table Expressions）](https://qiita.com/hmatsu47/items/01211556089b19913d05)**
- **`CHECK`制約**
  - **[MySQL 8.0.16 で実装された CHECK 制約を（いまさら）試してみる](https://qiita.com/hmatsu47/items/7526b5a4bfdc346b158c)**
- **ロールによる権限管理**
  - **[MySQL 8.0（8.0.2）DMR でロールを使う (1) 基本編](https://qiita.com/hmatsu47/items/e4a49d32685220d492a9)**
- **インスタント DDL**
  - 後述

### 強化された機能

- **インデックス関連**
  - 降順インデックス
    - **[いまさら MySQL 8.0 で降順 INDEX](https://qiita.com/hmatsu47/items/8c5e7abe204f7ecc5084)**
  - 関数＆式インデックス
  - 不可視インデックス
- **JSON 機能**
  - `JSON_TABLE()`関数
  - 複数値インデックス
    - **[MySQL 8.0.17 で Multi-Valued Indexes を試す](https://qiita.com/hmatsu47/items/3e49a473bc36aeefc706)**
  - 行更新の内部処理効率化
- **GIS 機能**
  - SRID 対応
    - **[「地球が丸いということを知った」MySQL 8.0 で距離を計算してみる](https://qiita.com/hmatsu47/items/97839fd9c3db1d2e9557)**
- **ロックの改良**
  - `NO WAIT`・`SKIP LOCKED`
    - **[MySQL 8.0 で NOWAIT / SKIP LOCKED（いまさら）](https://qiita.com/hmatsu47/items/7675b026e65762d2445f)**
- **結合（`JOIN`）処理の改良（高速化）**
  - ハッシュ結合（Hash join）
    - **[MySQL 8.0.20 で強化されたハッシュジョイン（Hash Join）を試してみる](https://qiita.com/hmatsu47/items/e9d3d4396fea42c8960e)**

### サポートされない機能など

MySQL 8.0 の一部（新）機能は Aurora MySQL v3 ではサポートされません。

- **`AUTO_INCREMENT`値の保持**
  - 再起動後は保持されるが、スナップショットからの復元時などには保持されない
    - **[AUTO_INCREMENT 値](https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/AuroraUserGuide/Aurora.AuroraMySQL.Compare-v2-v3.html#AuroraMySQL.mysql80-autoincrement)**
- **リソースグループ**
- **`UNDO`テーブル領域の新機能**
  - Aurora ではストレージレイヤのアーキテクチャが異なるため
- **TLS 1.3**
- **MySQL プラグインの設定**
- **X プラグイン（ドキュメントストア機能）**

### その他

一部のキーワードが MySQL 8.0.26 からバックポートされています。

- **[Aurora MySQL バージョン 3 に対する包括的な言語変更](https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/AuroraUserGuide/Aurora.AuroraMySQL.Compare-v2-v3.html#AuroraMySQL.8.0-inclusive-language)**

## Aurora MySQL v3 独自機能の変更点

全体的に Aurora 独自機能が減り本家 MySQL 8.0 のオリジナル実装に寄せた印象です。

なお、2022/04/21 に Aurora Serverless v2 が GA になり、Aurora MySQL v3 対応になりました。

### パラレルクエリ

独自機能として残り、適用範囲が拡大されました。

- **[新しいパラレルクエリの最適化](https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.MySQL80.html#AuroraMySQL.8.0-features-pq)**
  - 以下を含む SQL 文をサポート
    - `TEXT`・`BLOB`・`JSON`・`GEOMETRY`・`VARCHAR`・768 バイトより長い`CHAR`型を含んだテーブル
    - パーティショニングテーブル
    - `SELECT`のリスト（射影）内および`HAVING`句内の集計関数

### インスタント DDL

独自機能が廃止され、本家のインスタント DDL が採用されました。

- **[インスタント DDL (Aurora MySQL バージョン 3)](https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.Managing.FastDDL.html#AuroraMySQL.mysql80-instant-ddl)**

### クエリキャッシュ（廃止）

クエリキャッシュは I/O の低減に寄与する一方で並列スレッドのロック競合を引き起こすなどの問題があり、MySQL 5.6 の時点ですでに非推奨になっていましたが、MySQL 8.0 で廃止されました。

これに合わせて、Aurora MySQL v3 でも Aurora 独自仕様のクエリキャッシュが廃止されました。

Aurora MySQL v1 でクエリキャッシュを使用していた場合は、パフォーマンスの変化に気を付ける必要があります。

### Aurora Serverless v2（2020/04/22 追記）

Aurora MySQL v3 のバージョン 3.02.0 より、Aurora Serverless v2 がサポートされました。

Aurora Serverless v1 とは異なり、プロビジョンドインスタンスと同じクラスタにサーバレスインスタンスを配置できます。

- **[Amazon Aurora Serverless v2 is Generally Available: Instant Scaling for Demanding Workloads](https://aws.amazon.com/jp/blogs/aws/amazon-aurora-serverless-v2-is-generally-available-instant-scaling-for-demanding-workloads/)**
