---
title: "HeatWave on AWS で PrivateLink 接続を使ってインバウンドレプリケーションを試す"
emoji: "🏜"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["mysql", "heatwave", "aws", "oci"]
published: false
---

2024/8/22 に GA になった Egress PrivateLink を使ってインバウンドレプリケーションを試してみます。

https://dev.mysql.com/doc/heatwave-aws/en/heatwave-aws-ibr-config-privatelink.html#GUID-522AE101-2529-44B4-8306-65F171CBB099

## ソース DB（Aurora MySQL）作成

以前[パブリック IP アドレスを使ってインバウンドレプリケーションしたとき](https://qiita.com/hmatsu47/items/fc7b033f701ae8d5fb4d)とは少し変えて、今回は以下の構成で試してみます。

- Aurora MySQL 3.04 LTS（3.04.3）
- 非 GTID 形式（匿名トランザクション）
- プライベートサブネットからの接続

### DB クラスターパラメータグループ設定

最初に、ROW 形式のバイナリログを有効にした DB クラスターパラメータグループを用意します。

![](/images/heatwave-on-aws-privatelink/heatwave-on-aws-privatelink_001.png)

:::message
`log_bin_trust_function_creators`は必要に応じて設定してください。
:::

### サブネットグループ作成

プライベートサブネットで構成したサブネットグループを作成します。

![](/images/heatwave-on-aws-privatelink/heatwave-on-aws-privatelink_003.png)

### DB クラスター・インスタンス作成

前述の DB クラスターパラメータグループとサブネットグループを使って、DB クラスターとインスタンスを作成します。

作成している間に、作業中の AWS アカウントにおけるアベイラビリティゾーンとアベイラビリティゾーン ID の対応関係を確認しておきます。

![](/images/heatwave-on-aws-privatelink/heatwave-on-aws-privatelink_002.png)

DB クラスター・インスタンスが起動したら、MySQL クライアントでバイナリログの保持期間を設定します。

```sql:バイナリログ保持期間設定
mysql> call mysql.rds_set_configuration('binlog retention hours', 144);
Query OK, 0 rows affected (0.01 sec)
```

続いて、レプリカ DB からの接続用ユーザーを作成し、権限を付与します。

```sql:レプリカDBからの接続用ユーザー作成
mysql> CREATE USER rpluser001 IDENTIFIED BY '【パスワード】' REQUIRE SSL;
Query OK, 0 rows affected (0.00 sec)

mysql> GRANT REPLICATION SLAVE on *.* to rpluser001;
Query OK, 0 rows affected (0.00 sec)
```

ついでにバイナリログのログ（ファイル）名とログポジション（ファイルサイズ）を確認しておきます。

```sql:バイナリログ確認
mysql> SHOW BINARY LOGS;
+----------------------------+-----------+-----------+
| Log_name                   | File_size | Encrypted |
+----------------------------+-----------+-----------+
| mysql-bin-changelog.000001 |       157 | No        |
| mysql-bin-changelog.000002 |       441 | No        |
| mysql-bin-changelog.000003 |       677 | No        |
+----------------------------+-----------+-----------+
3 rows in set (0.00 sec)
```

非 GTID 形式（匿名トランザクション）接続時のログファイル・ログポジションの指定が

- ログファイル : `mysql-bin-changelog.000003`
- ログポジション : 677

であることがわかりました。

あわせて、ソース DB クラスターの Writer エンドポイントの IP アドレスを（CloudShell などで）調べておきます。

```sh:ソースDBクラスターWriterエンドポイントIPアドレス確認
$ ping aurora-source.cluster-XXXXXXXXXXXX.ap-northeast-1.rds.amazonaws.com
PING aurora-source-instance-1.XXXXXXXXXXXX.ap-northeast-1.rds.amazonaws.com (172.31.124.155) 56(84) bytes of data.
^C
--- aurora-source-instance-1.XXXXXXXXXXXX.ap-northeast-1.rds.amazonaws.com ping statistics ---
3 packets transmitted, 0 received, 100% packet loss, time 2081ms
```

## PrivateLink 用のターゲットグループを設定

![](/images/heatwave-on-aws-privatelink/heatwave-on-aws-privatelink_011.png)

![](/images/heatwave-on-aws-privatelink/heatwave-on-aws-privatelink_012.png)

![](/images/heatwave-on-aws-privatelink/heatwave-on-aws-privatelink_013.png)

![](/images/heatwave-on-aws-privatelink/heatwave-on-aws-privatelink_014.png)

![](/images/heatwave-on-aws-privatelink/heatwave-on-aws-privatelink_015.png)

![](/images/heatwave-on-aws-privatelink/heatwave-on-aws-privatelink_016.png)

## PrivateLink 用の NLB を作成

![](/images/heatwave-on-aws-privatelink/heatwave-on-aws-privatelink_021.png)

![](/images/heatwave-on-aws-privatelink/heatwave-on-aws-privatelink_022.png)

![](/images/heatwave-on-aws-privatelink/heatwave-on-aws-privatelink_023.png)

![](/images/heatwave-on-aws-privatelink/heatwave-on-aws-privatelink_024.png)

![](/images/heatwave-on-aws-privatelink/heatwave-on-aws-privatelink_025.png)

![](/images/heatwave-on-aws-privatelink/heatwave-on-aws-privatelink_026.png)

![](/images/heatwave-on-aws-privatelink/heatwave-on-aws-privatelink_027.png)

![](/images/heatwave-on-aws-privatelink/heatwave-on-aws-privatelink_028.png)

![](/images/heatwave-on-aws-privatelink/heatwave-on-aws-privatelink_029.png)

![](/images/heatwave-on-aws-privatelink/heatwave-on-aws-privatelink_030.png)

![](/images/heatwave-on-aws-privatelink/heatwave-on-aws-privatelink_031.png)

## ソース DB 用のセキュリティグループにインバウンドルールを追加

![](/images/heatwave-on-aws-privatelink/heatwave-on-aws-privatelink_041.png)

![](/images/heatwave-on-aws-privatelink/heatwave-on-aws-privatelink_042.png)

## PrivateLink 用のエンドポイントサービスを作成

![](/images/heatwave-on-aws-privatelink/heatwave-on-aws-privatelink_051.png)

![](/images/heatwave-on-aws-privatelink/heatwave-on-aws-privatelink_052.png)

![](/images/heatwave-on-aws-privatelink/heatwave-on-aws-privatelink_053.png)

![](/images/heatwave-on-aws-privatelink/heatwave-on-aws-privatelink_054.png)

![](/images/heatwave-on-aws-privatelink/heatwave-on-aws-privatelink_055.png)

## レプリカ DB（HeatWave on AWS）作成

![](/images/heatwave-on-aws-privatelink/heatwave-on-aws-privatelink_061.png)

![](/images/heatwave-on-aws-privatelink/heatwave-on-aws-privatelink_062.png)

![](/images/heatwave-on-aws-privatelink/heatwave-on-aws-privatelink_063.png)

![](/images/heatwave-on-aws-privatelink/heatwave-on-aws-privatelink_064.png)

## Egress PrivateLink 作成

![](/images/heatwave-on-aws-privatelink/heatwave-on-aws-privatelink_071.png)

![](/images/heatwave-on-aws-privatelink/heatwave-on-aws-privatelink_072.png)

![](/images/heatwave-on-aws-privatelink/heatwave-on-aws-privatelink_073.png)

![](/images/heatwave-on-aws-privatelink/heatwave-on-aws-privatelink_074.png)

## AWS 側でエンドポイント接続リクエストを承諾

![](/images/heatwave-on-aws-privatelink/heatwave-on-aws-privatelink_081.png)

![](/images/heatwave-on-aws-privatelink/heatwave-on-aws-privatelink_082.png)

![](/images/heatwave-on-aws-privatelink/heatwave-on-aws-privatelink_083.png)

![](/images/heatwave-on-aws-privatelink/heatwave-on-aws-privatelink_084.png)

## チャネルを作成

![](/images/heatwave-on-aws-privatelink/heatwave-on-aws-privatelink_091.png)

![](/images/heatwave-on-aws-privatelink/heatwave-on-aws-privatelink_092.png)

![](/images/heatwave-on-aws-privatelink/heatwave-on-aws-privatelink_093.png)

![](/images/heatwave-on-aws-privatelink/heatwave-on-aws-privatelink_094.png)

## データレプリケーションをテスト

### Aurora MySQL へサンプルデータを投入

```sh:サンプルデータ投入（Aurora MySQL）
$ mysqlsh -u admin -h aurora-source.cluster-XXXXXXXXXXXX.ap-northeast-1.rds.amazonaws.com -p
Please provide the password for 'admin@aurora-source.cluster-XXXXXXXXXXXX.ap-northeast-1.rds.amazonaws.com': 【パスワードを入力】
Save password for 'admin@aurora-source.cluster-XXXXXXXXXXXX.ap-northeast-1.rds.amazonaws.com'? [Y]es/[N]o/Ne[v]er (default No): 【デフォルトのままEnter】
MySQL Shell 8.0.34

Copyright (c) 2016, 2023, Oracle and/or its affiliates.
Oracle is a registered trademark of Oracle Corporation and/or its affiliates.
Other names may be trademarks of their respective owners.

Type '\help' or '\?' for help; '\quit' to exit.
Creating a session to 'admin@aurora-source.cluster-XXXXXXXXXXXX.ap-northeast-1.rds.amazonaws.com'
Fetching schema names for auto-completion... Press ^C to stop.
Your MySQL connection id is 753
Server version: 8.0.28 0a740fc6
No default schema selected; type \use <schema> to set one.
 MySQL  aurora-source.cluster-XXXXXXXXXXXX.ap-northeast-1.rds.amazonaws.com:3306 ssl  JS > util.loadDump("tpch_dump", {dryRun: false, resetProgress:true, ignoreVersion:true})
Loading DDL and Data from 'tpch_dump' using 4 threads.
Opening dump...
NOTE: Dump format has version 1.0.2 and was created by an older version of MySQL Shell. If you experience problems loading it, please recreate the dump using the current version of MySQL Shell and try again.
Target is MySQL 8.0.28. Dump was produced from MySQL 8.0.23-u2-cloud
Scanning metadata - done
Checking for pre-existing objects...
Executing common preamble SQL
Executing DDL - done
Executing view DDL - done
Starting data load
2 thds loading \ 100% (1.11 GB / 1.11 GB), 3.53 MB/s, 8 / 8 tables done
Recreating indexes - done
Executing common postamble SQL
50 chunks (8.66M rows, 1.11 GB) for 8 tables in 1 schemas were loaded in 2 min 41 sec (avg throughput 7.01 MB/s)
0 warnings were reported during the load.
```

### HeatWave on AWS でデータのレプリケーションを確認

```sql:クエリ実行（HeatWave on AWS）
USE tpch;

SELECT
  l_returnflag,
  l_linestatus,
  SUM(l_quantity) AS sum_qty,
  SUM(l_extendedprice) AS sum_base_price,
  SUM(l_extendedprice * (1 - l_discount)) AS sum_disc_price,
  SUM(l_extendedprice * (1 - l_discount) * (1 + l_tax)) AS sum_charge,
  AVG(l_quantity) AS avg_qty,
  AVG(l_extendedprice) AS avg_price,
  AVG(l_discount) AS avg_disc,
  COUNT(*) AS count_order
FROM
  lineitem
WHERE
  l_shipdate <= DATE '1998-12-01' - INTERVAL '90' DAY
GROUP BY
  l_returnflag,
  l_linestatus
ORDER BY
  l_returnflag,
  l_linestatus;
```

![](/images/heatwave-on-aws-privatelink/heatwave-on-aws-privatelink_101.png)

### HeatWave 有効化

```sql:HeatWaveエンジン有効化
ALTER TABLE tpch.customer SECONDARY_ENGINE=RAPID;
ALTER TABLE tpch.lineitem SECONDARY_ENGINE=RAPID;
ALTER TABLE tpch.nation SECONDARY_ENGINE=RAPID;
ALTER TABLE tpch.orders SECONDARY_ENGINE=RAPID;
ALTER TABLE tpch.part SECONDARY_ENGINE=RAPID;
ALTER TABLE tpch.partsupp SECONDARY_ENGINE=RAPID;
ALTER TABLE tpch.region SECONDARY_ENGINE=RAPID;
ALTER TABLE tpch.supplier SECONDARY_ENGINE=RAPID;
```

```sql:HeatWaveにデータをロード
ALTER TABLE tpch.customer SECONDARY_LOAD;
ALTER TABLE tpch.lineitem SECONDARY_LOAD;
ALTER TABLE tpch.nation SECONDARY_LOAD;
ALTER TABLE tpch.orders SECONDARY_LOAD;
ALTER TABLE tpch.part SECONDARY_LOAD;
ALTER TABLE tpch.partsupp SECONDARY_LOAD;
ALTER TABLE tpch.region SECONDARY_LOAD;
ALTER TABLE tpch.supplier SECONDARY_LOAD;
```

![](/images/heatwave-on-aws-privatelink/heatwave-on-aws-privatelink_102.png)

### HeatWave エンジンで実行

```sql:クエリ再実行（HeatWave on AWS）
USE tpch;

SELECT
  l_returnflag,
  l_linestatus,
  SUM(l_quantity) AS sum_qty,
  SUM(l_extendedprice) AS sum_base_price,
  SUM(l_extendedprice * (1 - l_discount)) AS sum_disc_price,
  SUM(l_extendedprice * (1 - l_discount) * (1 + l_tax)) AS sum_charge,
  AVG(l_quantity) AS avg_qty,
  AVG(l_extendedprice) AS avg_price,
  AVG(l_discount) AS avg_disc,
  COUNT(*) AS count_order
FROM
  lineitem
WHERE
  l_shipdate <= DATE '1998-12-01' - INTERVAL '90' DAY
GROUP BY
  l_returnflag,
  l_linestatus
ORDER BY
  l_returnflag,
  l_linestatus;
```

![](/images/heatwave-on-aws-privatelink/heatwave-on-aws-privatelink_103.png)
