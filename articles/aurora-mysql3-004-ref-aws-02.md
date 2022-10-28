---
title: "Amazon Aurora MySQL v1（5.6 互換）→ v3（8.0 互換）移行を計画する（4）AWS 公式ドキュメントを読む（2）"
emoji: "📙"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["aws", "aurora", "mysql", "移行", "バージョンアップ"]
published: true
---

これは

- **[Amazon Aurora MySQL v1（5.6 互換）→ v3（8.0 互換）移行を計画する（3）AWS 公式ドキュメントを読む（1）](/hmatsu47/articles/aurora-mysql3-003-ref-aws-01)**

の続きです。

今回は AWS 公式の Aurora 関連ドキュメントのうち、

- **[ベースとなる MySQL のバージョンアップ（5.6 → 8.0）による差異](/hmatsu47/articles/aurora-mysql3-004-ref-aws-02#%E3%83%99%E3%83%BC%E3%82%B9%E3%81%A8%E3%81%AA%E3%82%8B-mysql-%E3%81%AE%E3%83%90%E3%83%BC%E3%82%B8%E3%83%A7%E3%83%B3%E3%82%A2%E3%83%83%E3%83%97%EF%BC%885.6-%E2%86%92-8.0%EF%BC%89%E3%81%AB%E3%82%88%E3%82%8B%E5%B7%AE%E7%95%B0)**
- **[本家 MySQL 8.0 と Aurora MySQL v3 の相違点](/hmatsu47/articles/aurora-mysql3-004-ref-aws-02#%E6%9C%AC%E5%AE%B6-mysql-8.0-%E3%81%A8-aurora-mysql-v3-%E3%81%AE%E7%9B%B8%E9%81%95%E7%82%B9)**
- **[Aurora 独自機能の変更点](/hmatsu47/articles/aurora-mysql3-004-ref-aws-02#aurora-%E7%8B%AC%E8%87%AA%E6%A9%9F%E8%83%BD%E3%81%AE%E5%A4%89%E6%9B%B4%E7%82%B9)**

を中心に取り上げてみます。

:::message
[前回の記事](/hmatsu47/articles/aurora-mysql3-003-ref-aws-01)と重複する点は省略します。また、v1 → v3 の移行に直接関係しない点も省略します。
:::

## 「Aurora MySQL バージョン 3 は MYSQL 8.0 との互換性があります。」を読む

https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.MySQL80.html

より拾っていきます。

:::message
2022/2/26 時点で公開されている情報をもとに書いています。
:::

### ベースとなる MySQL のバージョンアップ（5.6 → 8.0）による差異

- **[インスタント DDL をサポート](/hmatsu47/articles/aurora-mysql3-004-ref-aws-02#%E3%82%A4%E3%83%B3%E3%82%B9%E3%82%BF%E3%83%B3%E3%83%88-ddl-%E3%82%92%E3%82%B5%E3%83%9D%E3%83%BC%E3%83%88)**
- **[`Administrator`権限の分割](/hmatsu47/articles/aurora-mysql3-004-ref-aws-02#administrator%E6%A8%A9%E9%99%90%E3%81%AE%E5%88%86%E5%89%B2)**
- **[クエリキャッシュの削除](/hmatsu47/articles/aurora-mysql3-004-ref-aws-02#%E3%82%AF%E3%82%A8%E3%83%AA%E3%82%AD%E3%83%A3%E3%83%83%E3%82%B7%E3%83%A5%E3%81%AE%E5%89%8A%E9%99%A4)**
- **[ハッシュ結合（Hash join）を実装](/hmatsu47/articles/aurora-mysql3-004-ref-aws-02#%E3%83%8F%E3%83%83%E3%82%B7%E3%83%A5%E7%B5%90%E5%90%88%EF%BC%88hash-join%EF%BC%89%E3%82%92%E5%AE%9F%E8%A3%85)**
- **[デフォルト文字セットが`latin1`から`utf8mb4`に変更](/hmatsu47/articles/aurora-mysql3-004-ref-aws-02#%E3%83%87%E3%83%95%E3%82%A9%E3%83%AB%E3%83%88%E6%96%87%E5%AD%97%E3%82%BB%E3%83%83%E3%83%88%E3%81%8Clatin1%E3%81%8B%E3%82%89utf8mb4%E3%81%AB%E5%A4%89%E6%9B%B4)**
- **[参照専用（Reader）インスタンスでの`CREATE TEMPORARY TABLE (AS SELECT)`挙動変化](</hmatsu47/articles/aurora-mysql3-004-ref-aws-02#%E5%8F%82%E7%85%A7%E5%B0%82%E7%94%A8%EF%BC%88reader%EF%BC%89%E3%82%A4%E3%83%B3%E3%82%B9%E3%82%BF%E3%83%B3%E3%82%B9%E3%81%A7%E3%81%AEcreate-temporary-table-(as-select)%E6%8C%99%E5%8B%95%E5%A4%89%E5%8C%96>)**
- **[内部テンポラリテーブル変更](/hmatsu47/articles/aurora-mysql3-004-ref-aws-02#%E5%86%85%E9%83%A8%E3%83%86%E3%83%B3%E3%83%9D%E3%83%A9%E3%83%AA%E3%83%86%E3%83%BC%E3%83%96%E3%83%AB%E5%A4%89%E6%9B%B4)**

#### インスタント DDL をサポート

MySQL 8.0 では新たにインスタント DDL がサポートされました。

https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.Managing.FastDDL.html#AuroraMySQL.mysql80-instant-ddl

これに伴い、Aurora MySQL v1 で提供されていた（ラボモードの）高速 DDL が廃止されました。

カラムやインデックスの追加・変更・削除など、DDL の実行時の所要時間とロックの有無などが変わる可能性があります。

#### `Administrator`権限の分割

MySQL 8.0 では以前のバージョンと比べて管理者権限が細分化されています。

- **[ロールベースの特権モデル](https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/AuroraUserGuide/Aurora.AuroraMySQL.Compare-80-v3.html#AuroraMySQL.privilege-model)**

Aurora MySQL v1 で実行していた SQL 文などが権限エラーになる場合は、実行に必要な権限を確認してください。

また、MySQL 8.0 ではロールベースの権限管理が導入されています。

https://dev.mysql.com/doc/refman/8.0/ja/roles.html

これにあわせて`rds_superuser_role`という管理者権限（特権）を持つロールが追加されています。

#### クエリキャッシュの削除

クエリキャッシュは I/O の低減に寄与する一方で並列スレッドのロック競合を引き起こすなどの問題があり、MySQL 5.6 の時点ですでに非推奨になっていましたが、MySQL 8.0 で廃止されました。

これに合わせて、Aurora MySQL v3 でも Aurora 独自仕様のクエリキャッシュが廃止されました。

Aurora MySQL v1 でクエリキャッシュを使用していた場合は、パフォーマンスの変化に気を付ける必要があります。

#### ハッシュ結合（Hash join）を実装

MySQL 8.0 に ハッシュ結合（Hash join）が実装され、Aurora 独自仕様のハッシュ結合が廃止されました。

https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/AuroraUserGuide/aurora-mysql-parallel-query.html#aurora-mysql-parallel-query-enabling-hash-join

あわせて、ハッシュ結合の有効化・無効化の指定方法が変わりました。

#### デフォルト文字セットが`latin1`から`utf8mb4`に変更

https://dev.mysql.com/doc/refman/8.0/ja/charset-unicode-utf8mb4.html

あわせて、デフォルトの照合順序（Collation）も変更されています。

https://dev.mysql.com/doc/refman/8.0/ja/charset-charsets.html

地味ですが、文字セットと照合順序は

- 文字化け
- 文字の照合問題（どの文字を同一と見做すか？）
- パフォーマンス

に関係します。

以前紹介した **[サイボウズのブログ記事](https://blog.cybozu.io/entry/2021/05/24/175000#utf8mb4-%E3%81%AE%E7%85%A7%E5%90%88%E9%A0%86%E5%BA%8F%E3%81%AE%E3%83%87%E3%83%95%E3%82%A9%E3%83%AB%E3%83%88%E5%80%A4%E3%81%AE%E5%A4%89%E6%9B%B4)** でも`utf8mb4`のデフォルト照合順序が変わった問題について触れていました。

:::message
Aurora MySQL v1 同様、v3 初期リリースでは`utf8`は`utf8mb3`（3 バイトまでの UTF-8）のエイリアスですが、今後のマイナーバージョンで`utf8mb4`のエイリアスに変更される可能性があります。
:::

パフォーマンス面では、

https://yoku0825.blogspot.com/2018/12/utf8mb40900aici.html

のような違いがあります。

:::message
さらに MySQL 8.0.17 で`utf8mb4_0900_bin`が追加されています。
https://qiita.com/hmatsu47/items/d66830c8a00c21f5edad
:::

#### 参照専用（Reader）インスタンスでの`CREATE TEMPORARY TABLE (AS SELECT)`挙動変化

`innodb_read_only`が`1`のときに InnoDB ストレージエンジンでテンポラリテーブルが作成できなくなったのに伴い、Reader インスタンスで`CREATE TEMPORARY TABLE … ENGINE=InnoDB`を実行する場合は SQL モード`NO_ENGINE_SUBSTITUTION`を無効にする必要があります。

ただし`AS SELECT`付きで実行する場合は SQL モードにかかわらずエラーになります。

そのため、Reader インスタンスで`CREATE TEMPORARY TABLE`を実行する場合は`ENGINE=InnoDB`を削除しておくのが良さそうです。

- **[リーダー DB インスタンスでユーザーが作成した (明示的な) 一時テーブル](https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/AuroraUserGuide/ams3-temptable-behavior.html#ams3-temptable-behavior.user)**

#### 内部テンポラリテーブル変更

:::message
`CREATE TEMPORARY TABLE`ではなく、SQL 文の実行時に内部的に使用されるテンポラリテーブルの話です。
:::

MyISAM ストレージエンジンが廃止され、アーキテクチャが変わりました。

- **[内部 (黙示的) 一時テーブルのストレージエンジン](https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/AuroraUserGuide/ams3-temptable-behavior.html#ams3-temptable-behavior-engine)**

動作調整のために`internal_tmp_mem_storage_engine`パラメータが追加されました。

:::message
**2022/4/5 追記：**
Reader インスタンスでは共有ストレージへの書き込みができないため、`internal_tmp_mem_storage_engine`パラメータの設定値が重要になります。
https://aws.amazon.com/jp/blogs/database/use-the-temptable-storage-engine-on-amazon-rds-for-mysql-and-amazon-aurora-mysql/
:::

### 本家 MySQL 8.0 と Aurora MySQL v3 の相違点

- **[初期リリースは MySQL 8.0.26 ベースだが、一部 8.0.26 からバックポートしているキーワードが存在](/hmatsu47/articles/aurora-mysql3-004-ref-aws-02#%E5%88%9D%E6%9C%9F%E3%83%AA%E3%83%AA%E3%83%BC%E3%82%B9%E3%81%AF-mysql-8.0.26-%E3%83%99%E3%83%BC%E3%82%B9%E3%81%A0%E3%81%8C%E3%80%81%E4%B8%80%E9%83%A8-8.0.26-%E3%81%8B%E3%82%89%E3%83%90%E3%83%83%E3%82%AF%E3%83%9D%E3%83%BC%E3%83%88%E3%81%97%E3%81%A6%E3%81%84%E3%82%8B%E3%82%AD%E3%83%BC%E3%83%AF%E3%83%BC%E3%83%89%E3%81%8C%E5%AD%98%E5%9C%A8)**
- **[Percona XtraBackup ツールからの物理バックアップ復元は未対応](/hmatsu47/articles/aurora-mysql3-004-ref-aws-02#percona-xtrabackup-%E3%83%84%E3%83%BC%E3%83%AB%E3%81%8B%E3%82%89%E3%81%AE%E7%89%A9%E7%90%86%E3%83%90%E3%83%83%E3%82%AF%E3%82%A2%E3%83%83%E3%83%97%E5%BE%A9%E5%85%83%E3%81%AF%E6%9C%AA%E5%AF%BE%E5%BF%9C)**
- **[パラメータ`innodb_flush_log_at_trx_commit`変更不可に](/hmatsu47/articles/aurora-mysql3-004-ref-aws-02#%E3%83%91%E3%83%A9%E3%83%A1%E3%83%BC%E3%82%BFinnodb_flush_log_at_trx_commit%E5%A4%89%E6%9B%B4%E4%B8%8D%E5%8F%AF%E3%81%AB)**
- **[一部ステータス変数が非適応](/hmatsu47/articles/aurora-mysql3-004-ref-aws-02#%E4%B8%80%E9%83%A8%E3%82%B9%E3%83%86%E3%83%BC%E3%82%BF%E3%82%B9%E5%A4%89%E6%95%B0%E3%81%8C%E9%9D%9E%E9%81%A9%E5%BF%9C)**
- デフォルトで binlog が OFF
- リソースグループは非対応
- **[UNDO テーブルスペースの処理方法相違](/hmatsu47/articles/aurora-mysql3-004-ref-aws-02#undo-%E3%83%86%E3%83%BC%E3%83%96%E3%83%AB%E3%82%B9%E3%83%9A%E3%83%BC%E3%82%B9%E3%81%AE%E5%87%A6%E7%90%86%E6%96%B9%E6%B3%95%E7%9B%B8%E9%81%95)**
- TLS 1.3 非対応
- **[プラグインの設定変更不可](/hmatsu47/articles/aurora-mysql3-004-ref-aws-02#%E3%83%97%E3%83%A9%E3%82%B0%E3%82%A4%E3%83%B3%E3%81%AE%E8%A8%AD%E5%AE%9A%E5%A4%89%E6%9B%B4%E4%B8%8D%E5%8F%AF)**

#### 初期リリースは MySQL 8.0.26 ベースだが、一部 8.0.26 からバックポートしているキーワードが存在

「Master」「Slave」などの用語の読み替えがバックポートされています。

- **[Aurora MySQL バージョン 3 に対する包括的な言語変更](https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/AuroraUserGuide/Aurora.AuroraMySQL.Compare-v2-v3.html#AuroraMySQL.8.0-inclusive-language)**

#### Percona XtraBackup ツールからの物理バックアップ復元は未対応

今後のマイナーバージョンでサポート予定です。

#### パラメータ`innodb_flush_log_at_trx_commit`変更不可に

`1`で固定になりました。

#### 一部ステータス変数が非適応

- **[Aurora MySQL に適応されない MySQL ステータス変数](https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.Reference.html#AuroraMySQL.Reference.StatusVars.Inapplicable)**

:::message
Aurora 独自のステータス変数を含みます。
:::

#### UNDO テーブルスペースの処理方法相違

そもそも本家 MySQL と Aurora ではストレージ層のアーキテクチャが異なりますので、本家 MySQL 8.0 の UNDO テーブルスペース関連機能は使えません。

> - Aurora MySQL は名前付きテーブルスペースをサポートしていません。
> - `innodb_undo_log_truncate`構成設定はオフになっており、オンにできません。Aurora には、ストレージスペースを再利用するための独自のメカニズムがあります。
> - Aurora MySQL には`CREATE UNDO TABLESPACE`、`ALTER UNDO TABLESPACE ... SET INACTIVE`、および`DROP UNDO TABLESPACE`ステートメントがありません。
> - Aurora は、UNDO テーブルスペースの数を自動的に設定し、それらのテーブルスペースを自動的に管理します。

#### プラグインの設定変更不可

ドキュメントストア向けの X プラグイン、クエリリライトプラグインなどが使えません。

### Aurora 独自機能の変更点

重複する点は省略します。

- **[Aurora 並列クエリの最適化対象が拡大](/hmatsu47/articles/aurora-mysql3-004-ref-aws-02#aurora-%E4%B8%A6%E5%88%97%E3%82%AF%E3%82%A8%E3%83%AA%E3%81%AE%E6%9C%80%E9%81%A9%E5%8C%96%E5%AF%BE%E8%B1%A1%E3%81%8C%E6%8B%A1%E5%A4%A7)**
- **[バックトラック未サポート](/hmatsu47/articles/aurora-mysql3-004-ref-aws-02#%E3%83%90%E3%83%83%E3%82%AF%E3%83%88%E3%83%A9%E3%83%83%E3%82%AF%E6%9C%AA%E3%82%B5%E3%83%9D%E3%83%BC%E3%83%88)**
- **[Aurora Serverless v1 クラスタ非サポート](/hmatsu47/articles/aurora-mysql3-004-ref-aws-02#aurora-serverless-v1-%E3%82%AF%E3%83%A9%E3%82%B9%E3%82%BF%E9%9D%9E%E3%82%B5%E3%83%9D%E3%83%BC%E3%83%88)**
- **[`mysql.lambda_async`ストアドプロシージャ削除](/hmatsu47/articles/aurora-mysql3-004-ref-aws-02#mysql.lambda_async%E3%82%B9%E3%83%88%E3%82%A2%E3%83%89%E3%83%97%E3%83%AD%E3%82%B7%E3%83%BC%E3%82%B8%E3%83%A3%E5%89%8A%E9%99%A4)**
- **[パラメータグループ内のパラメータ変更](/hmatsu47/articles/aurora-mysql3-004-ref-aws-02#%E3%83%91%E3%83%A9%E3%83%A1%E3%83%BC%E3%82%BF%E3%82%B0%E3%83%AB%E3%83%BC%E3%83%97%E5%86%85%E3%81%AE%E3%83%91%E3%83%A9%E3%83%A1%E3%83%BC%E3%82%BF%E5%A4%89%E6%9B%B4)**

#### Aurora 並列クエリの最適化対象が拡大

- **[新しいパラレルクエリの最適化](https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.MySQL80.html#AuroraMySQL.8.0-features-pq)**

LOB 系カラムやパーティショニングされたテーブル、`HAVING`句の中での集計関数に対応しました。

#### バックトラック未サポート

現時点ではバックトラックを使用中のクラスタで作成したスナップショットからの復元ができません。

今後のマイナーバージョンでサポート予定です。

#### Aurora Serverless v1 クラスタ非サポート

2022/04/21 GA の Aurora Serverless v2 クラスタでサポートされています（バージョン 3.02.0 以降）。

#### `mysql.lambda_async`ストアドプロシージャ削除

非同期関数`lambda_async`で代替します。

#### パラメータグループ内のパラメータ変更

`lower_case_table_names`パラメータの値はクラスタ作成時の指定で固定されるので、デフォルト値以外に変更する場合はアップグレード前にカスタムパラメータグループを作成・設定します。

その他の項目の変更については、前掲のこちら ↓ を確認してください。

- **[Aurora MySQL バージョン 3 に対する包括的な言語変更](https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/AuroraUserGuide/Aurora.AuroraMySQL.Compare-v2-v3.html#AuroraMySQL.8.0-inclusive-language)**

### その他

対応インスタンスクラスの変更があり、db.r3・db.r4・t3.small・t2 が使用できなくなりました。

## まとめ

- **Aurora MySQL v3 は v1・v2 と比べると独自仕様の実装が減る一方、本家 MySQL からの実装をそのまま利用する機能が増えた**
  - インスタント DDL・ハッシュ結合など
- **とはいえ本家 MySQL 8.0 との相違点はある**
- **クエリキャッシュの削除や、文字セット・照合順序など細部の違いが動作やパフォーマンスに影響を与える可能性がある**
- **現状では未サポートの Aurora 独自機能がある**
  - バックトラック・Serverless など

---

次回からは Oracle 公式の MySQL 関連ドキュメントを中心に見ていく予定です。

- **[Amazon Aurora MySQL v1（5.6 互換）→ v3（8.0 互換）移行を計画する（5）Oracle 公式ドキュメントを読む](/hmatsu47/articles/aurora-mysql3-005-ref-ora-01)**

に続きます。
