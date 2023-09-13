---
title: "公開プレビュー中の Amazon Aurora MySQL と Amazon Redshift の ゼロ ETL 統合を試してみた"
emoji: "🔺"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["aws", "aurora", "redshift", "etl"]
published: true
---

Amazon Aurora MySQL と Amazon Redshift の ゼロ ETL 統合が公開プレビュー中なので、試してみました。

https://aws.amazon.com/jp/about-aws/whats-new/2023/06/amazon-aurora-mysql-zero-etl-integration-redshift-public-preview/

:::message alert
この記事は 2023/8/24 現在の情報をもとに構成しています。
ゼロ ETL 統合はプレビュー中の機能です。GA の際に仕様や制限事項が変更になる可能性があります。
（おそらく制限事項のうちのいくつかは GA 時に撤廃または緩和されるはずです）
:::

## ゼロ ETL 統合とは

Aurora などのリレーショナルデータベース上のデータは、通常 ETL（Extract（抽出）/ Transform（変換）/ Load（ロード）の略）の流れを経て Redshift などのデータウェアハウスや分析基盤への流し込みを行います。

Aurora と Redshift のゼロ ETL 統合は、**この部分の構築・設定を簡素化し、ほぼリアルタイムでデータの流し込み（複製）を可能にする機能** です。

結果として、Redshift を使用した分析や機械学習がほぼリアルタイムで実行できます。

https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/AuroraUserGuide/zero-etl.html

:::message
プレビュー中のユーザーガイドですので、GA の際にリンク先ページがなくなる可能性があります（以降同様）。
:::

![](/images/aurora-rs-zeroetl-preview/zero-etl-integrations.png)

（出典：前掲のユーザーガイドのページ）

### プレビューの制限事項

2023/8/24 現在で、次のリンク先ページに示されている制限事項があります。

https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/AuroraUserGuide/zero-etl.html#zero-etl.reqs-lims

特に、サポートされない（Aurora MySQL の）データ型が存在する一方で、そのデータ型を持つテーブルのフィルタができない点については、実利用上大きな障害になりそうですね。

少し脱線しますが、こちら ↓ の記事から始まる一連の記事に、競合となる MySQL HeatWave の制限事項をまとめています（現在進行中）。
MySQL HeatWave にも`BLOB`型などサポートされないデータ型がありますが、HeatWave 側にロードされないようフィルタする機能があります（テーブル単位または列単位で）。

https://qiita.com/hmatsu47/items/5bf7b37f694e56f3dc82

## 試してみた

先に記しておきますが、ゼロ ETL 自体の設定部分の挙動が不安定でした。

### ソース Aurora 設定

#### DB クラスターパラメータグループ作成

以下の制約があるため、先にクラスターのパラメータグループを設定しておきます。

:::details Aurora 設定上の制約
![](/images/aurora-rs-zeroetl-preview/zeroetl-before-001.png)
:::

https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/AuroraUserGuide/zero-etl.setting-up.html#zero-etl.parameters

まずは作成します。

::: details パラメータグループ作成
![](/images/aurora-rs-zeroetl-preview/zeroetl-before-002.png)
:::

指定のパラメータを変更します。

:::details パラメータをデフォルト値から変更
![](/images/aurora-rs-zeroetl-preview/zeroetl-before-003.png)
:::

パラメーター間の矛盾を指摘するアラートが表示されるので、追加で変更します。

:::details 不整合パラメータを変更
![](/images/aurora-rs-zeroetl-preview/zeroetl-before-004.png)
:::

アラートは消えませんが気にせず先に進みます。

:::details 確認
![](/images/aurora-rs-zeroetl-preview/zeroetl-before-005.png)
※`temptable_max_ram`については値を変更していません。
:::

保存して完了です。

#### Aurora クラスター作成

続いて、テスト用の Aurora クラスターを設定します。

内容は以下のとおりです。

- Aurora MySQL 3.04.0（MySQL 8.0.28 互換）
- db.t4g.medium
- レプリカなし
- IAM データベース認証（必要かどうかは不明）
- 「最初のデータベース名」は（zero-ETL の設定の挙動が不安定なため）試行錯誤で入れてみたが結果的に不要
- DB クラスターのパラメータグループとして先ほど作成したものを指定
- デフォルトのキーで暗号化

:::details Aurora クラスター作成
![](/images/aurora-rs-zeroetl-preview/zeroetl-001.png)
![](/images/aurora-rs-zeroetl-preview/zeroetl-002.png)
![](/images/aurora-rs-zeroetl-preview/zeroetl-003.png)
![](/images/aurora-rs-zeroetl-preview/zeroetl-004.png)
![](/images/aurora-rs-zeroetl-preview/zeroetl-005.png)
![](/images/aurora-rs-zeroetl-preview/zeroetl-006.png)
![](/images/aurora-rs-zeroetl-preview/zeroetl-007.png)
:::

#### サンプルデータ投入

こちら ↓ の記事で使ったデータを投入します。

https://qiita.com/hmatsu47/items/0979f877ad596cf3cf67

まず EC2 でサンプルデータをダウンロードし、展開します。

:::details サンプルデータをダウンロード

```sh
$  wget https://objectstorage.ap-osaka-1.oraclecloud.com/p/seAq8Kgd4TyUqlv5M5qObMJwvsluhCPyOuHOn1L_t4HQYUle2DV-KdFeK44MS7yQ/n/idazzjlcjqzj/b/workshop/o/heatwave_workshop.zip
--2023-07-22 14:01:10--  https://objectstorage.ap-osaka-1.oraclecloud.com/p/seAq8Kgd4TyUqlv5M5qObMJwvsluhCPyOuHOn1L_t4HQYUle2DV-KdFeK44MS7yQ/n/idazzjlcjqzj/b/workshop/o/heatwave_workshop.zip
Resolving objectstorage.ap-osaka-1.oraclecloud.com (objectstorage.ap-osaka-1.oraclecloud.com)... 134.70.112.3
Connecting to objectstorage.ap-osaka-1.oraclecloud.com (objectstorage.ap-osaka-1.oraclecloud.com)|134.70.112.3|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 348382849 (332M) [application/x-zip-compressed]
Saving to: ‘heatwave_workshop.zip’

100%[======================================================================================================================>] 348,382,849 59.1MB/s   in 5.4s

2023-07-22 14:01:15 (61.3 MB/s) - ‘heatwave_workshop.zip’ saved [348382849/348382849]
```

:::

`heatwave_workshop.zip`を展開します。

:::details サンプルデータを展開

```sh
$  unzip heatwave_workshop.zip
Archive:  heatwave_workshop.zip
   creating: tpch_dump/
  inflating: tpch_dump/@.json
  inflating: tpch_dump/tpch.json
（中略）
  inflating: tpch_offload.sql
  inflating: tpch_queries_mysql.sql
  inflating: tpch_queries_rapid.sql
```

:::

MySQL Shell のロードダンプユーティリティを使って、サンプルデータを Aurora へロードします。

:::details サンプルデータを Aurora へロード

```sh
$ mysqlsh -u 【ユーザー名】 -h 【クラスターエンドポイントの URL】 -p
Please provide the password for '【ユーザー名】@【クラスターエンドポイントの URL】': 【パスワードを入力】
Save password for '【ユーザー名】@【クラスターエンドポイントの URL】'? [Y]es/[N]o/Ne[v]er (default No): N
MySQL Shell 8.0.34

Copyright (c) 2016, 2023, Oracle and/or its affiliates.
（中略）
No default schema selected; type \use <schema> to set one.
JS > util.loadDump("tpch_dump", {dryRun: false, resetProgress:true, ignoreVersion:true})
Loading DDL and Data from 'tpch_dump' using 4 threads.
Opening dump...
NOTE: Dump format has version 1.0.2 and was created by an older version of MySQL Shell. If you experience problems loading it, please recreate the dump using the current version of MySQL Shell and try again.
（中略）
0 warnings were reported during the load.
```

:::

### Redshift サーバーレスのプレビュー版ワークグループ作成

ゼロ ETL 統合（プレビュー）の対象となる Redshift には制限があるので、今回はサーバーレスでワークグループを作成します。

https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/AuroraUserGuide/zero-etl.setting-up.html#zero-etl-setting-up.data-warehouse

通常メニューではなく、サーバーレスダッシュボード画面上方の「Create preview workgroup」ボタンより、**プレビュー版を作成** します。

:::details Redshift サーバーレスダッシュボード画面
![](/images/aurora-rs-zeroetl-preview/zeroetl-008.png)
:::

- ベース RPU 容量はデフォルトの 128
- Aurora クラスターと同じリージョンを選択
  - 念のため VPC も同じに
- 名前空間も合わせて作成
- 管理者ユーザー認証情報は一旦デフォルトのまま
  - ゼロ ETL の設定がうまくいかずワークグループを作り直す前にユーザー名とパスワードを設定してみたが、影響があったかどうかは不明
- IAM ロールは新規作成
  - 一旦広めに「任意の S3 バケット」を指定しておいたが、おそらく今回の試用には無関係

:::details Redshift プレビュー版ワークグループ作成
![](/images/aurora-rs-zeroetl-preview/zeroetl-009.png)
![](/images/aurora-rs-zeroetl-preview/zeroetl-010.png)
![](/images/aurora-rs-zeroetl-preview/zeroetl-011.png)
![](/images/aurora-rs-zeroetl-preview/zeroetl-012.png)
![](/images/aurora-rs-zeroetl-preview/zeroetl-013.png)
:::

### Redshift データウェアハウスの大文字・小文字識別を有効にする

こちら ↓ に書かれているとおり大文字・小文字識別を有効にする必要があるので、Cloudshell から実行します。

https://docs.aws.amazon.com/ja_jp/redshift/latest/mgmt/zero-etl-using.setting-up.html#zero-etl-setting-up.case-sensitivity

:::details Cloudshell から実行

```sh
aws redshift-serverless update-workgroup \
        --workgroup-name zeroetl-workgroup \
        --config-parameters parameterKey=enable_case_sensitive_identifier,parameterValue=true
```

:::

成功するとレスポンスが JSON 形式で返ってきます（ここでは省略）。

### ゼロ ETL 統合設定

#### ゼロ ETL 統合用の IAM ロールを作成

こちら ↓ のページを参考にして IAM ロールを作成します。

https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/AuroraUserGuide/zero-etl.creating.html#zero-etl.create-permissions

::: details ゼロ ETL 統合用の IAM ロール
![](/images/aurora-rs-zeroetl-preview/zeroetl-014.png)
:::

### Redshift 名前空間のリソースポリシー設定

「Authorized principals」に ↑ で作成した IAM ロールの ARN、「Authorized integration sources」にソースの Aurora クラスターの ARN を指定します。

::: details Redshift 名前空間のリソースポリシー
![](/images/aurora-rs-zeroetl-preview/zeroetl-015.png)
:::

### ゼロ ETL 統合作成

いろいろな画面から作成できますが、ここではソース Aurora クラスターの画面から作成してみます。

- Aurora MySQL DB クラスターは自動で選択される
- Amazon Redshift データウェアハウスとして先に作成した名前空間を指定

::: details Aurora クラスター画面からゼロ ETL 統合を作成
![](/images/aurora-rs-zeroetl-preview/zeroetl-016.png)
![](/images/aurora-rs-zeroetl-preview/zeroetl-017.png)
:::

すると、一見うまく作成されたように見えますが、

:::details ゼロ ETL 統合作成完了？
![](/images/aurora-rs-zeroetl-preview/zeroetl-018.png)
:::

作成したゼロ ETL 統合のリンクを開くとエラーが表示されます。

:::details ゼロ ETL 統合作成エラー (1)
![](/images/aurora-rs-zeroetl-preview/zeroetl-019.png)
:::

Redshift 側から表示すると、「No database」となっていますが（右下の赤字）、この画面のようなエラーメッセージになってしまうと手詰まりです。

:::details ゼロ ETL 統合作成エラー (2)
![](/images/aurora-rs-zeroetl-preview/zeroetl-020.png)
:::

仕方がないので、一旦ゼロ ETL 統合とワークグループを削除して再作成します。

::: details ゼロ ETL 統合とワークグループを削除
![](/images/aurora-rs-zeroetl-preview/zeroetl-021.png)
![](/images/aurora-rs-zeroetl-preview/zeroetl-022.png)
:::

再作成後、Redshift 側から表示すると「No database」は変わりませんが、エラーメッセージに「Create database from integration」ボタンが表示されています。

:::details ゼロ ETL 統合再作成後
![](/images/aurora-rs-zeroetl-preview/zeroetl-024.png)
:::

ボタンをクリックしてゼロ ETL 統合用のデータベースを作成します。

:::details ゼロ ETL 統合用のデータベース作成
![](/images/aurora-rs-zeroetl-preview/zeroetl-025.png)
:::

なぜかエラーは表示されたままですが、データは Redshift に複製されたようです。

:::details ゼロ ETL 統合による複製完了
![](/images/aurora-rs-zeroetl-preview/zeroetl-026.png)
:::

「Redshift query editor v2」でクエリも実行可能です。

:::details クエリ実行結果
![](/images/aurora-rs-zeroetl-preview/zeroetl-027.png)
:::

### わざと複製エラーを起こしてみる

#### Aurora に`BLOB`列ありのテーブルを作成

ソースの Aurora に`BLOB`列があるテーブルを作成してみます。

こちら ↓ で試したのと同じテーブルです。

https://qiita.com/hmatsu47/items/a9667762fb5ecdd66e75#blob%E5%88%97%E3%82%92%E9%99%A4%E5%A4%96%E3%81%9B%E3%81%9A%E3%81%AB-heatwave-%E3%81%AB%E3%83%AD%E3%83%BC%E3%83%89%E3%81%97%E3%81%A6%E3%81%BF%E3%82%8B

:::details Aurora に BLOB 投入

```sql
mysql> CREATE DATABASE type_test;
Query OK, 1 row affected (0.00 sec)

mysql> USE type_test;
Database changed
mysql> CREATE TABLE blob_test (id INT NOT NULL AUTO_INCREMENT, val INT NOT NULL, bindata BLOB, PRIMARY KEY(id));
Query OK, 0 rows affected (0.03 sec)

mysql> INSERT INTO blob_test SET val = 100, bindata = 'aaaaa';
Query OK, 1 row affected (0.00 sec)

mysql> INSERT INTO blob_test SET val = 200, bindata = 'bbbbb';
Query OK, 1 row affected (0.01 sec)
```

:::

想定どおり、Aurora → Redshift の複製が停止しました。

:::details 複製が停止
![](/images/aurora-rs-zeroetl-preview/zeroetl-028.png)
![](/images/aurora-rs-zeroetl-preview/zeroetl-029.png)
:::

#### `BLOB`列のみ削除

同テーブルの`BLOB`列だけ削除してみました。

:::details BLOB 列削除

```sql
mysql> ALTER TABLE blob_test DROP COLUMN bindata;
Query OK, 0 rows affected (0.06 sec)
Records: 0  Duplicates: 0  Warnings: 0
```

:::

結果は変わらず複製は停止したままでした。

#### 同テーブル削除

テーブルごと削除してみました（SQL 文は省略）。
状態が変化し、対象テーブルが「Deleted」になりました。

:::details テーブル削除
![](/images/aurora-rs-zeroetl-preview/zeroetl-030.png)
:::

#### 複製再開確認のため新規テーブル作成

今度は`TEXT`列を持つテーブルを作成しました。

:::details TEXT 列を持つテーブル作成

```sql
mysql> CREATE TABLE text_test (id INT NOT NULL AUTO_INCREMENT, val INT NOT NULL, textdata TEXT, PRIMARY KEY(id));
Query OK, 0 rows affected (0.03 sec)

mysql> INSERT INTO text_test SET val = 100, textdata = 'aaaaa';
Query OK, 1 row affected (0.01 sec)

mysql> INSERT INTO text_test SET val = 200, textdata = 'bbbbb';
Query OK, 1 row affected (0.01 sec)
```

:::

相変わらずエラーの表示は消えませんが、正しく複製されたようです。

:::details 複製再開
![](/images/aurora-rs-zeroetl-preview/zeroetl-031.png)
![](/images/aurora-rs-zeroetl-preview/zeroetl-032.png)
:::

「Redshift query editor v2」でテーブルを見ると行が複製されているのがわかります。

:::details クエリ実行結果
![](/images/aurora-rs-zeroetl-preview/zeroetl-033.png)
:::

## 感想

正直、どうすればエラーが解消されるのかよくわからないところが多くて試すのが大変でした。

また、冒頭に記したとおり GA の際にいくつかの制限事項は撤廃または緩和されると見ていますが、GA 時またはそれ以降の改善に期待、です。
