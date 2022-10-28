---
title: "調査（1）設定パラメータ関連の変更点"
---

## この章について

著者が調査した v1 → v3 の変更点のうち、主に Aurora MySQL クラスタ・インスタンス設定（パラメータグループ等）に関するものを記します。

なお、サーバ設定（サーバ変数）をアプリケーションから参照・上書きしている場合は、アプリケーションの改修対象になる可能性があります。

## AWS 公式の Aurora 関連ドキュメントより

### v1 → v2 で`engine`属性が`aurora`から`aurora-mysql`に

マネジメントコンソールだけ使っている場合には関係ないのですが、CLI や SDK などを使って API を呼び出すコードを書いている場合、パラメータとして指定する属性（`engine`）が変わるので修正が必要になります。

### パラメータグループ

Aurora MySQL v1 と v2、v3 でクラスタとデータベースのパラメータグループが違います。

カスタムパラメータグループを使っている場合は、アップグレード前にあらかじめ v3 用のカスタムパラメータグループを作っておきます。

CLI や SDK などを使って API を呼び出すコードを書いている場合、パラメータ指定も変える必要があります。

MySQL 5.6 と 8.0 ではサーバのパラメータにかなり違いがあり、Aurora のパラメータグループの項目や初期値も違います。さらに**同じパラメータ名の`engine-default`が指す値が v1 と v3 で異なるケースなど、パラメータグループの差分が表示されなくても値の書き換えが必要な場合があります。**

## Oracle 公式の MySQL 関連ドキュメントより

パラメータ差分は行数が多いのでこちらを参照してください。

https://github.com/hmatsu47/aurora_mysql1to3diff/blob/main/aurora-mysql1_3_param_diff.csv

実際に使用しているパラメータグループ（クラスタ・DB）と比較して、対応が必要な点を見付けます。

### 対応が必要になりそうな変更点

#### `innodb_strict_mode`のデフォルトが厳密モードに

https://dev.mysql.com/doc/refman/8.0/ja/innodb-parameters.html#sysvar_innodb_strict_mode

Aurora MySQL v2（MySQL 5.7）の時点でデフォルトが変わっています。

一部の SQL 文の実行で、従来は警告（Warning）で済ませていたのをエラーで止める動作に変わりました。多くの場合は問題にならないはずですが、列数の多いテーブルを`CREATE`・`ALTER`する必要があるなど、`OFF`にせざるを得ないケースもあります。

:::message
関連する項目に **[`sql-mode`](https://dev.mysql.com/doc/refman/8.0/ja/sql-mode.html)** があります。
こちらは本家 MySQL のデフォルトとは違い、Aurora MySQL のデフォルトパラメータグループでは v1 から v3 まで`0`（本家 MySQL では **無指定** に相当）です。
:::

#### `innodb_autoinc_lock_mode`のデフォルトが`1`から`2`に

**`AUTO_INCREMENT`値** 生成時のロックモードです。

https://dev.mysql.com/doc/refman/8.0/ja/innodb-parameters.html#sysvar_innodb_autoinc_lock_mode

`2`のほうが性能上有利なのですが、例えば MySQL Connector/J の **[`rewriteBatchedStatements`オプション](https://dev.mysql.com/doc/connector-j/8.0/en/connector-j-connp-props-performance-extensions.html#cj-conn-prop_rewriteBatchedStatements)** を使ったバッチ`INSERT`と **[LAST_INSERT_ID()](https://dev.mysql.com/doc/refman/8.0/ja/information-functions.html#function_last-insert-id)** を組み合わせた処理を行っているようなケースでは、動作の問題が生じる可能性があるので従来と同じ`1`に戻します。

#### `tx_isolation`が`transaction-isolation`に

これはパラメータグループというよりも、アプリケーションから DB に接続する部分の実装やライブラリのオプション設定に関わる問題ですが、以前触れた **[サイボウズのブログ記事](https://blog.cybozu.io/entry/2021/05/24/175000#%E3%83%88%E3%83%A9%E3%83%B3%E3%82%B6%E3%82%AF%E3%82%B7%E3%83%A7%E3%83%B3%E5%88%86%E9%9B%A2%E3%83%AC%E3%83%99%E3%83%AB%E3%82%92%E8%A8%AD%E5%AE%9A%E3%81%99%E3%82%8B%E5%A4%89%E6%95%B0%E5%90%8D%E3%81%AE%E5%A4%89%E6%9B%B4)** ではこの変更の影響を受けていました。

#### `internal_tmp_mem_storage_engine`の追加（内部テンポラリテーブル変更）

:::message
`CREATE TEMPORARY TABLE`ではなく、SQL 文の実行時に内部的に使用されるテンポラリテーブルの話です。
:::

内部テンポラリテーブルでは、MyISAM ストレージエンジンが廃止されアーキテクチャが変わりました。

- **[内部 (黙示的) 一時テーブルのストレージエンジン](https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/AuroraUserGuide/ams3-temptable-behavior.html#ams3-temptable-behavior-engine)**

動作調整のために`internal_tmp_mem_storage_engine`パラメータが追加されました。

特に Aurora の Reader インスタンスでは設定値を超える容量の内部テンポラリテーブルを作ることができないため、設定値のチューニングに注意が必要です。

https://aws.amazon.com/jp/blogs/database/use-the-temptable-storage-engine-on-amazon-rds-for-mysql-and-amazon-aurora-mysql/
