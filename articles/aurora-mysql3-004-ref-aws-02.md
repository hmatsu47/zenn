---
title: "Amazon Aurora MySQL v1（5.6 互換）→ v3（8.0 互換）移行を計画する（4）AWS 公式ドキュメントを読む（2）"
emoji: "📙"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["aws", "aurora", "mysql"]
published: false
---

これは

- **[Amazon Aurora MySQL v1（5.6 互換）→ v3（8.0 互換）移行を計画する（3）AWS 公式ドキュメントを読む（1）](/hmatsu47/articles/aurora-mysql3-003-ref-aws-01)**

の続きです。

今回は AWS 公式の Aurora 関連ドキュメントのうち、

- **ベースとなる MySQL のバージョンアップ（5.6 → 8.0）による差異**
- **本家 MySQL 8.0 と Aurora MySQL v3 の相違点**
- **Aurora 独自機能の変更点**

を中心に取り上げてみます。

:::message
[前回の記事](/hmatsu47/articles/aurora-mysql3-003-ref-aws-01)と重複する点は省略します。また、v1 → v3 の移行に直接関係しない点も省略します。
:::

# 「Aurora MySQL バージョン 3 は MYSQL 8.0 との互換性があります。」を読む

https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.MySQL80.html

より拾っていきます。

:::message
2022/2/26 時点で公開されている情報をもとに書いています。
:::

## ベースとなる MySQL のバージョンアップ（5.6 → 8.0）による差異

- **インスタント DDL をサポート**
- **`Administrator`権限の分割**
- **クエリキャッシュの削除**
- **ハッシュ結合（Hash join）を実装**
- **デフォルト文字セットが`latin1`から`utf8mb4`に変更**
- **参照専用（Reader）インスタンスでの`CREATE TEMPORARY TABLE (AS SELECT)`挙動変化**
- **内部テンポラリテーブル変更**

### インスタント DDL をサポート

MySQL 8.0 では新たにインスタント DDL がサポートされました。

https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.Managing.FastDDL.html#AuroraMySQL.mysql80-instant-ddl

これに伴い、Aurora MySQL v1 で提供されていた（ラボモードの）高速 DDL が廃止されました。

カラムやインデックスの追加・変更・削除など、DDL の実行時の所要時間とロックの有無などが変わる可能性があります。

### `Administrator`権限の分割

MySQL 8.0 では以前のバージョンと比べて管理者権限が細分化されています。

- **[ロールベースの特権モデル](https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.MySQL80.html#AuroraMySQL.privilege-model)**

Aurora MySQL v1 で実行していた SQL 文などが権限エラーになる場合は、実行に必要な権限を確認してください。

また、MySQL 8.0 ではロールベースの権限管理が導入されています。

https://dev.mysql.com/doc/refman/8.0/ja/roles.html

これにあわせて`rds_superuser_role`という管理者権限（特権）を持つロールが追加されています。

### クエリキャッシュの削除

クエリキャッシュは I/O の低減に寄与する一方で並列スレッドのロック競合を引き起こすなどの問題があり、MySQL 5.6 の時点ですでに非推奨になっていましたが、MySQL 8.0 で廃止されました。

これに合わせて、Aurora MySQL v3 でも Aurora 独自仕様のクエリキャッシュが廃止されました。

Aurora MySQL v1 でクエリキャッシュを使用していた場合は、パフォーマンスの変化に気を付ける必要があります。

### ハッシュ結合（Hash join）を実装

MySQL 8.0 に ハッシュ結合（Hash join）が実装され、Aurora 独自仕様のハッシュ結合が廃止されました。

https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/AuroraUserGuide/aurora-mysql-parallel-query.html#aurora-mysql-parallel-query-enabling-hash-join

あわせて、ハッシュ結合の有効化・無効化の指定方法が変わりました。

### デフォルト文字セットが`latin1`から`utf8mb4`に変更

https://dev.mysql.com/doc/refman/8.0/ja/charset-unicode-utf8mb4.html

あわせて、デフォルトの照合順序（Collation）も変更されています。

https://dev.mysql.com/doc/refman/8.0/ja/charset-charsets.html

地味ですが、文字セットと照合順序は

- 文字化け
- 文字の照合問題（どの文字を同一と見做すか？）
- パフォーマンス

に関係します。

以前紹介した **[サイボウズのブログ記事](https://blog.cybozu.io/entry/2021/05/24/175000#utf8mb4-%E3%81%AE%E7%85%A7%E5%90%88%E9%A0%86%E5%BA%8F%E3%81%AE%E3%83%87%E3%83%95%E3%82%A9%E3%83%AB%E3%83%88%E5%80%A4%E3%81%AE%E5%A4%89%E6%9B%B4)** でも`utf8mb4`のデフォルト照合順序が変わった問題について触れていました。

パフォーマンス面では、

https://yoku0825.blogspot.com/2018/12/utf8mb40900aici.html

のような違いがあります。

:::message
さらに MySQL 8.0.17 で`utf8mb4_0900_bin`が追加されています。
https://qiita.com/hmatsu47/items/d66830c8a00c21f5edad
:::

### 参照専用（Reader）インスタンスでの`CREATE TEMPORARY TABLE (AS SELECT)`挙動変化

`innodb_read_only`が`1`のときに InnoDB ストレージエンジンで一時（テンポラリ）テーブルが作成できなくなったのに伴い、Reader インスタンスで`CREATE TEMPORARY TABLE … ENGINE=InnoDB`を実行する場合は SQL モード`NO_ENGINE_SUBSTITUTION`を無効にする必要があります。

ただし`AS SELECT`付きで実行する場合は SQL モードにかかわらずエラーになります。

そのため、Reader インスタンスで`CREATE TEMPORARY TABLE`を実行する場合は`ENGINE=InnoDB`を削除しておくのが良さそうです。

- **[リーダー DB インスタンスの一時テーブル](https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.MySQL80.html#AuroraMySQL.mysql80-temp-tables-readers)**

### 内部テンポラリテーブル変更

:::message
`CREATE TEMPORARY TABLE`ではなく、SQL 文の実行時に内部的に使用されるテンポラリテーブルの話です。
:::

MyISAM ストレージエンジンが廃止され、アーキテクチャが変わりました。

- **[内部一時テーブルのストレージエンジン](https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.MySQL80.html#AuroraMySQL.mysql80-internal-temp-tables-engine)**

動作調整のために`internal_tmp_mem_storage_engine`パラメータが追加されました。

## 本家 MySQL 8.0 と Aurora MySQL v3 の相違点

- **初期リリースは MySQL 8.0.26 ベースだが、一部 8.0.26 からバックポートしているキーワードが存在**
- **Percona XtraBackup ツールからの物理バックアップ復元は未対応**
- **パラメータ`innodb_flush_log_at_trx_commit`変更不可に**
- **一部ステータス変数が非適応**
- デフォルトで binlog が OFF
- リソースグループは非対応
- **UNDO テーブルスペースの処理方法相違**
- TLS 1.3 非対応
- **プラグインの設定変更不可**

### 初期リリースは MySQL 8.0.26 ベースだが、一部 8.0.26 からバックポートしているキーワードが存在

「Master」「Slave」などの用語の読み替えがバックポートされています。

- **[Aurora MySQL バージョン 3 に対する包括的な言語変更](https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.MySQL80.html#AuroraMySQL.8.0-inclusive-language)**

### Percona XtraBackup ツールからの物理バックアップ復元は未対応

今後のマイナーバージョンでサポート予定です。

### パラメータ`innodb_flush_log_at_trx_commit`変更不可に

`1`で固定です。

### 一部ステータス変数が非適応

- **[Aurora MySQL に適応されない MySQL ステータス変数](https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.Reference.html#AuroraMySQL.Reference.StatusVars.Inapplicable)**

:::message
Aurora 独自のステータス変数を含みます。
:::

### UNDO テーブルスペースの処理方法相違

そもそも本家 MySQL と Aurora ではストレージ層のアーキテクチャが異なりますので、本家 MySQL 8.0 の UNDO テーブル関連機能は使えません。

> - Aurora MySQL は名前付きテーブルスペースをサポートしていません。
> - `innodb_undo_log_truncate`構成設定はオフになっており、オンにできません。Aurora には、ストレージスペースを再利用するための独自のメカニズムがあります。
> - Aurora MySQL には`CREATE UNDO TABLESPACE`、`ALTER UNDO TABLESPACE ... SET INACTIVE`、および`DROP UNDO TABLESPACE`ステートメントがありません。
> - Aurora は、UNDO テーブルスペースの数を自動的に設定し、それらのテーブルスペースを自動的に管理します。

### プラグインの設定変更不可

ドキュメントストア向けの X プラグイン、クエリリライトプラグインなどが使えません。

## Aurora 独自機能の変更点

重複する点は省略します。

- **Aurora 並列クエリの最適化対象が拡大**
- **バックトラック未サポート**
- **Aurora Serverless v1 クラスタ非サポート**
- **`mysql.lambda_async`ストアドプロシージャ削除**
- **パラメータグループ内のパラメータ変更**

### Aurora 並列クエリの最適化対象が拡大

- **[新しい並列クエリの最適化](https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.MySQL80.html#AuroraMySQL.8.0-features-pq)**

LOB 系カラムやパーティショニングされたテーブル、`HAVING`句の中での集計関数に対応しました。

### バックトラック未サポート

現時点ではバックトラックを使用中のクラスタで作成したスナップショットからの復元ができません。

今後のマイナーバージョンでサポート予定です。

### Aurora Serverless v1 クラスタ非サポート

現在プレビュー中の Aurora Serverless v2 クラスタでサポート予定です。

### `mysql.lambda_async`ストアドプロシージャ削除

非同期関数`lambda_async`で代替します。

### パラメータグループ内のパラメータ変更

`lower_case_table_names`パラメータの値はクラスタ作成時の指定で固定されるので、デフォルト値以外に変更する場合はアップグレード前にカスタムパラメータグループを作成・設定します。

## その他

対応インスタンスクラスの変更があり、db.r3・db.r4・t3.small・t2 が使用できなくなりました。

# まとめ

- **Aurora MySQL v3 は v1・v2 と比べると独自仕様の実装が減る一方、本家 MySQL からの実装をそのまま利用する機能が増えた**
  - インスタント DDL・ハッシュ結合など
- **とはいえ本家 MySQL 8.0 との相違点はある**
- **クエリキャッシュの削除や、文字セット・照合順序など細部の違いが動作やパフォーマンスに影響を与える可能性がある**

---

次回からは Oracle 公式の MySQL 関連ドキュメントを中心に見ていく予定です。