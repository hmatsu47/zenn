---
title: "AWS DMS（CDC）で MySQL to MySQL 移行時の TIMESTAMP 不一致問題"
emoji: "❌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["mysql", "移行", "aws", "dms"]
published: true
---

AWS の DMS（CDC）で、

- ターゲットが MySQL（RDS / Aurora MySQL）
- `MEDIUMBLOB`または`LONGBLOB`列がある
- `TIMESTAMP`（または`DATETIME`）列があり、`ON UPDATE CURRENT_TIMESTAMP`が指定されている

場合、当該`TIMESTAMP`（`DATETIME`）列の値が **DMS（CDC）でレプリケーションされた時刻で上書きされる** ので注意が必要です。

## 実験環境

- DMS 3.4.7（3.4.6 でも実験済み）
  - 完全 LOB モード（チャンクサイズは 64KB）
- Aurora MySQL 3.02.0

## 実験内容

### ソースの準備

まずは 1 つ目の Aurora クラスタ／インスタンスを立てます。このときパラメータグループを作成し、`binlog_format`を`ROW`にしたものを指定します。

そして`BLOB`列と`ON UPDATE CURRENT_TIMESTAMP`付きの`TIMESTAMP`列を持つテーブルを作成し、10 行ほどデータを入れておきます。

```sql:ソースにデータ投入
mysql> CREATE DATABASE dmstest;
Query OK, 1 row affected (0.01 sec)

mysql> USE dmstest;
Database changed
mysql> CREATE TABLE blobtest (id INT PRIMARY KEY AUTO_INCREMENT, contents BLOB, updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP);
Query OK, 0 rows affected (0.02 sec)

mysql> INSERT INTO blobtest SET contents = "aaaaaaaaaa";
Query OK, 1 row affected (0.01 sec)
（中略）

mysql> SELECT * FROM blobtest;
+----+------------------------+---------------------+
| id | contents               | updated_at          |
+----+------------------------+---------------------+
|  1 | 0x61616161616161616161 | 2022-08-14 14:22:51 |
|  2 | 0x61616161616161616161 | 2022-08-14 14:22:52 |
|  3 | 0x61616161616161616161 | 2022-08-14 14:22:53 |
|  4 | 0x61616161616161616161 | 2022-08-14 14:22:54 |
|  5 | 0x61616161616161616161 | 2022-08-14 14:22:54 |
|  6 | 0x61616161616161616161 | 2022-08-14 14:22:55 |
|  7 | 0x61616161616161616161 | 2022-08-14 14:22:55 |
|  8 | 0x61616161616161616161 | 2022-08-14 14:22:56 |
|  9 | 0x61616161616161616161 | 2022-08-14 14:22:56 |
| 10 | 0x61616161616161616161 | 2022-08-14 14:22:57 |
+----+------------------------+---------------------+
10 rows in set (0.01 sec)
```

念のためバイナリログの保持期間の設定をしてからバイナリログポジションを確認しておきます。

```sql:バイナリログポジション確認
mysql> call mysql.rds_set_configuration('binlog retention hours', 168);
Query OK, 0 rows affected (0.02 sec)

mysql> SHOW BINARY LOGS;
+----------------------------+-----------+-----------+
| Log_name                   | File_size | Encrypted |
+----------------------------+-----------+-----------+
| mysql-bin-changelog.000001 |       156 | No        |
| mysql-bin-changelog.000002 |       440 | No        |
| mysql-bin-changelog.000003 |      3762 | No        |
+----------------------------+-----------+-----------+
3 rows in set (0.01 sec)
```

### ターゲットの準備

ターゲットとなる Aurora クラスタ／インスタンスをソースからクローンします。

クローン後、ソースとデータが一致していることを確認します。

```sql:ターゲットのデータ確認
mysql> USE dmstest;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> SELECT * FROM blobtest;
+----+------------------------+---------------------+
| id | contents               | updated_at          |
+----+------------------------+---------------------+
|  1 | 0x61616161616161616161 | 2022-08-14 14:22:51 |
|  2 | 0x61616161616161616161 | 2022-08-14 14:22:52 |
|  3 | 0x61616161616161616161 | 2022-08-14 14:22:53 |
|  4 | 0x61616161616161616161 | 2022-08-14 14:22:54 |
|  5 | 0x61616161616161616161 | 2022-08-14 14:22:54 |
|  6 | 0x61616161616161616161 | 2022-08-14 14:22:55 |
|  7 | 0x61616161616161616161 | 2022-08-14 14:22:55 |
|  8 | 0x61616161616161616161 | 2022-08-14 14:22:56 |
|  9 | 0x61616161616161616161 | 2022-08-14 14:22:56 |
| 10 | 0x61616161616161616161 | 2022-08-14 14:22:57 |
+----+------------------------+---------------------+
10 rows in set (0.00 sec)
```

### ソースでデータ追加

ソースでデータ行を 10 行追加します。

```sql:ソースでデータ追加
mysql> INSERT INTO blobtest SET contents = "bbbbbbbbbb";
Query OK, 1 row affected (0.01 sec)
（中略）

mysql> SELECT * FROM blobtest;
+----+------------------------+---------------------+
| id | contents               | updated_at          |
+----+------------------------+---------------------+
|  1 | 0x61616161616161616161 | 2022-08-14 14:22:51 |
|  2 | 0x61616161616161616161 | 2022-08-14 14:22:52 |
|  3 | 0x61616161616161616161 | 2022-08-14 14:22:53 |
|  4 | 0x61616161616161616161 | 2022-08-14 14:22:54 |
|  5 | 0x61616161616161616161 | 2022-08-14 14:22:54 |
|  6 | 0x61616161616161616161 | 2022-08-14 14:22:55 |
|  7 | 0x61616161616161616161 | 2022-08-14 14:22:55 |
|  8 | 0x61616161616161616161 | 2022-08-14 14:22:56 |
|  9 | 0x61616161616161616161 | 2022-08-14 14:22:56 |
| 10 | 0x61616161616161616161 | 2022-08-14 14:22:57 |
| 11 | 0x62626262626262626262 | 2022-08-14 14:39:57 |
| 12 | 0x62626262626262626262 | 2022-08-14 14:39:57 |
| 13 | 0x62626262626262626262 | 2022-08-14 14:39:58 |
| 14 | 0x62626262626262626262 | 2022-08-14 14:39:59 |
| 15 | 0x62626262626262626262 | 2022-08-14 14:39:59 |
| 16 | 0x62626262626262626262 | 2022-08-14 14:40:00 |
| 17 | 0x62626262626262626262 | 2022-08-14 14:40:01 |
| 18 | 0x62626262626262626262 | 2022-08-14 14:40:01 |
| 19 | 0x62626262626262626262 | 2022-08-14 14:40:02 |
| 20 | 0x62626262626262626262 | 2022-08-14 14:40:02 |
+----+------------------------+---------------------+
20 rows in set (0.00 sec)
```

### DMS（CDC）レプリケーションを設定し実行

こんな感じで移行タスクを作成して、手動で CDC レプリケーションを開始します。

![](/images/mysql-dms-cdc-timestamp-mismatch/dms_task_01.png)
![](/images/mysql-dms-cdc-timestamp-mismatch/dms_task_02.png)
![](/images/mysql-dms-cdc-timestamp-mismatch/dms_task_03.png)
![](/images/mysql-dms-cdc-timestamp-mismatch/dms_task_04.png)
![](/images/mysql-dms-cdc-timestamp-mismatch/dms_task_05.png)

### ターゲットでデータ確認

ターゲットで確認してみると、CDC で複製された 11 行目以降も`updated_at`列の値は一致していました。

```sql:ターゲットでデータ確認
mysql> SELECT * FROM blobtest;
+----+------------------------+---------------------+
| id | contents               | updated_at          |
+----+------------------------+---------------------+
|  1 | 0x61616161616161616161 | 2022-08-14 14:22:51 |
|  2 | 0x61616161616161616161 | 2022-08-14 14:22:52 |
|  3 | 0x61616161616161616161 | 2022-08-14 14:22:53 |
|  4 | 0x61616161616161616161 | 2022-08-14 14:22:54 |
|  5 | 0x61616161616161616161 | 2022-08-14 14:22:54 |
|  6 | 0x61616161616161616161 | 2022-08-14 14:22:55 |
|  7 | 0x61616161616161616161 | 2022-08-14 14:22:55 |
|  8 | 0x61616161616161616161 | 2022-08-14 14:22:56 |
|  9 | 0x61616161616161616161 | 2022-08-14 14:22:56 |
| 10 | 0x61616161616161616161 | 2022-08-14 14:22:57 |
| 11 | 0x62626262626262626262 | 2022-08-14 14:39:57 |
| 12 | 0x62626262626262626262 | 2022-08-14 14:39:57 |
| 13 | 0x62626262626262626262 | 2022-08-14 14:39:58 |
| 14 | 0x62626262626262626262 | 2022-08-14 14:39:59 |
| 15 | 0x62626262626262626262 | 2022-08-14 14:39:59 |
| 16 | 0x62626262626262626262 | 2022-08-14 14:40:00 |
| 17 | 0x62626262626262626262 | 2022-08-14 14:40:01 |
| 18 | 0x62626262626262626262 | 2022-08-14 14:40:01 |
| 19 | 0x62626262626262626262 | 2022-08-14 14:40:02 |
| 20 | 0x62626262626262626262 | 2022-08-14 14:40:02 |
+----+------------------------+---------------------+
20 rows in set (0.00 sec)
```

その後`BLOB`列に 65535 バイトのデータを投入してみましたが、`update_at`の値は一致していました。

### `MEDIUMBLOB`列で再試行

今度は`MEDIUMBLOB`列でやり直してみます。

#### テーブル再作成・データ再投入

テーブルを作り直し、`MEDIUMBLOB`列に 65535 バイトのデータを計 15 行分入れます。

```sql:MEDIUMBLOBでソース再準備
mysql> CREATE TABLE blobtest (id INT PRIMARY KEY AUTO_INCREMENT, contents MEDIUMBLOB, updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP);
Query OK, 0 rows affected (0.03 sec)

mysql> INSERT INTO blobtest SET contents = REPEAT('a', 65535);
Query OK, 1 row affected (0.01 sec)
（中略）

mysql> SELECT id, LENGTH(contents), updated_at FROM blobtest;
+----+------------------+---------------------+
| id | LENGTH(contents) | updated_at          |
+----+------------------+---------------------+
|  1 |            65535 | 2022-08-15 01:47:02 |
|  2 |            65535 | 2022-08-15 01:47:06 |
|  3 |            65535 | 2022-08-15 01:47:08 |
|  4 |            65535 | 2022-08-15 01:47:11 |
|  5 |            65535 | 2022-08-15 01:47:18 |
|  6 |            65535 | 2022-08-15 02:01:38 |
|  7 |            65535 | 2022-08-15 02:01:39 |
|  8 |            65535 | 2022-08-15 02:01:41 |
|  9 |            65535 | 2022-08-15 02:01:42 |
| 10 |            65535 | 2022-08-15 02:01:44 |
| 11 |            65535 | 2022-08-15 02:29:39 |
| 12 |            65535 | 2022-08-15 02:29:40 |
| 13 |            65535 | 2022-08-15 02:29:41 |
| 14 |            65535 | 2022-08-15 02:29:41 |
| 15 |            65535 | 2022-08-15 02:29:42 |
+----+------------------+---------------------+
15 rows in set (0.00 sec)
```

#### クローンしソースにデータを追加

これをクローンしてターゲットを再作成した後、あらためてソースに 5 行分のデータを投入します。

```sql:データを追加
mysql> INSERT INTO blobtest SET contents = REPEAT('d', 65535);
Query OK, 1 row affected (0.01 sec)
（中略）

mysql> SELECT id, LENGTH(contents), updated_at FROM blobtest;
+----+------------------+---------------------+
| id | LENGTH(contents) | updated_at          |
+----+------------------+---------------------+
|  1 |            65535 | 2022-08-15 01:47:02 |
|  2 |            65535 | 2022-08-15 01:47:06 |
|  3 |            65535 | 2022-08-15 01:47:08 |
|  4 |            65535 | 2022-08-15 01:47:11 |
|  5 |            65535 | 2022-08-15 01:47:18 |
|  6 |            65535 | 2022-08-15 02:01:38 |
|  7 |            65535 | 2022-08-15 02:01:39 |
|  8 |            65535 | 2022-08-15 02:01:41 |
|  9 |            65535 | 2022-08-15 02:01:42 |
| 10 |            65535 | 2022-08-15 02:01:44 |
| 11 |            65535 | 2022-08-15 02:29:39 |
| 12 |            65535 | 2022-08-15 02:29:40 |
| 13 |            65535 | 2022-08-15 02:29:41 |
| 14 |            65535 | 2022-08-15 02:29:41 |
| 15 |            65535 | 2022-08-15 02:29:42 |
| 16 |            65535 | 2022-08-15 02:45:30 |
| 17 |            65535 | 2022-08-15 02:45:31 |
| 18 |            65535 | 2022-08-15 02:45:32 |
| 19 |            65535 | 2022-08-15 02:45:32 |
| 20 |            65535 | 2022-08-15 02:45:33 |
+----+------------------+---------------------+
20 rows in set (0.00 sec)
```

#### DMS（CDC）レプリケーションを再設定・実行しターゲットで確認

CDC レプリケーションを行い、ターゲットで確認してみると、今度は`updated_at`の値が一致しません（16 行目〜）。

```sql:ターゲットで不一致確認
mysql> SELECT id, LENGTH(contents), updated_at FROM blobtest;
+----+------------------+---------------------+
| id | LENGTH(contents) | updated_at          |
+----+------------------+---------------------+
|  1 |            65535 | 2022-08-15 01:47:02 |
|  2 |            65535 | 2022-08-15 01:47:06 |
|  3 |            65535 | 2022-08-15 01:47:08 |
|  4 |            65535 | 2022-08-15 01:47:11 |
|  5 |            65535 | 2022-08-15 01:47:18 |
|  6 |            65535 | 2022-08-15 02:01:38 |
|  7 |            65535 | 2022-08-15 02:01:39 |
|  8 |            65535 | 2022-08-15 02:01:41 |
|  9 |            65535 | 2022-08-15 02:01:42 |
| 10 |            65535 | 2022-08-15 02:01:44 |
| 11 |            65535 | 2022-08-15 02:29:39 |
| 12 |            65535 | 2022-08-15 02:29:40 |
| 13 |            65535 | 2022-08-15 02:29:40 |
| 14 |            65535 | 2022-08-15 02:29:41 |
| 15 |            65535 | 2022-08-15 02:29:42 |
| 16 |            65535 | 2022-08-15 02:49:33 |
| 17 |            65535 | 2022-08-15 02:49:33 |
| 18 |            65535 | 2022-08-15 02:49:33 |
| 19 |            65535 | 2022-08-15 02:49:33 |
| 20 |            65535 | 2022-08-15 02:49:33 |
+----+------------------+---------------------+
20 rows in set (0.01 sec)
```

※これ以外のパターンもいくつか試し、例えば LOB のチャンクサイズを上げてみても結果は同じでした。

## 実験結果まとめ

列に投入するデータサイズに関わらず、以下のようになります。

- `TINYBLOB`・`BLOB`列は行本体と同時にレプリケーションされるので **`ON UPDATE CURRENT_TIMESTAMP`付きの`TIMESTAMP`・`DATETIME`列はソースとターゲットで一致する**
- `MEDIUMBLOB`・`LONGBLOB`列は行本体がレプリケーションされた後に`UPDATE`でターゲットに反映されるので、そのタイミングで **`ON UPDATE CURRENT_TIMESTAMP`付きの`TIMESTAMP`・`DATETIME`列が上書きされる**
  - 結果として **ターゲットの列値がレプリケーション実行時刻** になる（不正な値）

## 回避策（またはワークアラウンド）

おそらく次のような選択肢が考えられると思います。

### ターゲットで一旦`ON UPDATE CURRENT_TIMESTAMP`を外しておき、データ移行後にあらためて`ON UPDATE CURRENT_TIMESTAMP`を付ける

移行後のダウンタイム（サービス停止時間）が長くなるデメリットがあります。

ダウンタイムを短縮する目的でレプリケーションを使う場合は本末転倒になりかねないので、対象テーブルが少ないケース以外は採用しづらい選択肢です。

### アプリケーションを改修して`ON UPDATE CURRENT_TIMESTAMP`が不要な状態にしてから外す

アプリケーション改修の手間が掛かりますが、改修箇所が少ない場合は有力な選択肢になるでしょう。

### データ移行後に対象列を正しい値で上書きする

対象テーブル・対象データ行の両方が少ない場合にのみ考えられる選択肢です。

### 他の移行方法に切り替える

例えば Aurora MySQL のバージョンアップでデータ移行を行う場合、スナップショットからの復元によるバージョンアップと結果的にダウンタイムが大差ない可能性があります。

また、MySQL 5.6 → 5.7 → 8.0 のような多段 binlog レプリケーションはサポート外ではあるものの、状況によっては問題なくデータ移行が可能かもしれません。
