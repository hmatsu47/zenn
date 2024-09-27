---
title: "HeatWave on AWS で PrivateLink 接続を使ってインバウンドレプリケーションを試す"
emoji: "🏜"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["mysql", "heatwave", "aws", "oci"]
published: true
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

ついでにバイナリログのログ（ファイル）名とログポジション（ファイルサイズ）を確認しておきます（後で使います）。

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

あわせて、ソース DB クラスターの Writer エンドポイントの IP アドレスを（CloudShell などで）調べておきます（後で使います）。

```sh:ソースDBクラスターWriterエンドポイントIPアドレス確認
$ ping aurora-source.cluster-XXXXXXXXXXXX.ap-northeast-1.rds.amazonaws.com
PING aurora-source-instance-1.XXXXXXXXXXXX.ap-northeast-1.rds.amazonaws.com (172.31.124.155) 56(84) bytes of data.
^C
--- aurora-source-instance-1.XXXXXXXXXXXX.ap-northeast-1.rds.amazonaws.com ping statistics ---
3 packets transmitted, 0 received, 100% packet loss, time 2081ms
```

## PrivateLink 用のターゲットグループを設定

AWS マネジメントコンソールで EC2 のメニューから「ターゲットグループ」を選択し、「ターゲットグループの作成」をクリックします。

![](/images/heatwave-on-aws-privatelink/heatwave-on-aws-privatelink_011.png)

- ターゲットタイプ : IP アドレス
- ターゲットグループ名 : 任意
- プロトコル・ポート : TCP・3306
- IP アドレスタイプ : IPv4
- VPC : Aurora クラスター作成先の VPC

![](/images/heatwave-on-aws-privatelink/heatwave-on-aws-privatelink_012.png)

- ヘルスチェックプロトコル : TCP
- ヘルスチェックポート : 上書き・**3306 以外**の番号（例：40000）

「次へ」をクリックします。

![](/images/heatwave-on-aws-privatelink/heatwave-on-aws-privatelink_013.png)

ターゲットを登録します。

- ネットワーク : Aurora クラスター作成先の VPC
- IPv4 アドレス : 先に調べておいたソース DB クラスターの Writer エンドポイントの IP アドレス
- ポート : 3306

「保留中として以下を含める」をクリックします。

![](/images/heatwave-on-aws-privatelink/heatwave-on-aws-privatelink_014.png)

ターゲットを確認して「ターゲットグループを作成」をクリックします。

![](/images/heatwave-on-aws-privatelink/heatwave-on-aws-privatelink_015.png)

作成したターゲットグループを開き、指定した IP アドレスがターゲットとして登録されたのを確認します。

なお、ヘルスチェックは成功しない状態（Unhealthy）のままで問題ありません。

![](/images/heatwave-on-aws-privatelink/heatwave-on-aws-privatelink_016.png)

## PrivateLink 用の NLB を作成

引き続き EC2 のメニューから「ロードバランサー」を選択し、「ロードバランサーの作成」をクリックします。

![](/images/heatwave-on-aws-privatelink/heatwave-on-aws-privatelink_021.png)

Network Load Balancer の「作成」をクリックします。

![](/images/heatwave-on-aws-privatelink/heatwave-on-aws-privatelink_022.png)

- ロードバランサー名 : 任意
- スキーム : 内部
- ロードバランサーの IP アドレスタイプ : IPv4

![](/images/heatwave-on-aws-privatelink/heatwave-on-aws-privatelink_023.png)

- VPC : Aurora クラスター作成先の VPC
- アベイラビリティゾーン : サブネットグループで選択したサブネットのゾーンをチェック
  - サブネット : サブネットグループで選択したサブネット
  - プライベート IPv4 アドレス : CIDR（選択したサブネットのアドレス空間）から割り当て済み

![](/images/heatwave-on-aws-privatelink/heatwave-on-aws-privatelink_024.png)

「新しいセキュリティグループを作成」をクリックします。

![](/images/heatwave-on-aws-privatelink/heatwave-on-aws-privatelink_025.png)

- セキュリティグループ名 : 任意
- 説明 : 任意
- VPC : Aurora クラスター作成先の VPC
- インバウンドルール : 空のまま
- アウトバウンドルール : すべてのトラフィックを 0.0.0.0/0 へ送信（デフォルトのまま）

「セキュリティグループを作成」をクリックします。

![](/images/heatwave-on-aws-privatelink/heatwave-on-aws-privatelink_026.png)

セキュリティグループを作成したら、元の画面に戻って続きの設定を行います。

- セキュリティグループ : 直前に作成したものを選択（右側のリロードボタンをクリックしてから）
- リスナー TCP:3306 : デフォルトアクションとして先に作成した PrivateLink 用のターゲットグループを選択

![](/images/heatwave-on-aws-privatelink/heatwave-on-aws-privatelink_027.png)

設定内容を確認して「ロードバランサーの作成」をクリックします。

![](/images/heatwave-on-aws-privatelink/heatwave-on-aws-privatelink_028.png)

ロードバランサー作成完了を待ちます。

![](/images/heatwave-on-aws-privatelink/heatwave-on-aws-privatelink_029.png)

完了したら、「セキュリティ」タブを開いて「編集」ボタンをクリックします。

![](/images/heatwave-on-aws-privatelink/heatwave-on-aws-privatelink_030.png)

「PrivateLink トラフィックにインバウンドルールを適用する」のチェックを外して「変更内容の保存」をクリックします。

![](/images/heatwave-on-aws-privatelink/heatwave-on-aws-privatelink_031.png)

## ソース DB 用のセキュリティグループにインバウンドルールを追加

ソース DB のインスタンスの画面で「接続とセキュリティ」タブから VPC セキュリティグループのリンクをクリックします。

続いて、「インバウンドリンク」のタブで「インバウンドルールの編集」をクリックします。

![](/images/heatwave-on-aws-privatelink/heatwave-on-aws-privatelink_041.png)

以下のルールを追加します。

- TCP:3306（MYSQL/Aurora） ソース : 先ほど作成・編集したセキュリティグループ

追加したら「ルールを保存」をクリックします。

![](/images/heatwave-on-aws-privatelink/heatwave-on-aws-privatelink_042.png)

## PrivateLink 用のエンドポイントサービスを作成

VPC のメニューから「エンドポイントサービス」を選択し、「エンドポイントサービスの作成」をクリックします。

![](/images/heatwave-on-aws-privatelink/heatwave-on-aws-privatelink_051.png)

- 名前 : 任意
- ロードバランサーのタイプ : ネットワーク
- ロードバランサー : 先ほど作成した NLB を選択
- 承諾が必要 : チェック
- プライベート DNS 名をサービスに関連付ける : **チェックを外す**
- サポートされている IP アドレスタイプ : IPv4

入力・選択したら「作成」をクリックします。

![](/images/heatwave-on-aws-privatelink/heatwave-on-aws-privatelink_052.png)

エンドポイントサービスの作成が完了したら、「サービス名」を確認します（後で使います）。

![](/images/heatwave-on-aws-privatelink/heatwave-on-aws-privatelink_053.png)

「アクション」から「プリンシパルを許可」を選択します。

![](/images/heatwave-on-aws-privatelink/heatwave-on-aws-privatelink_054.png)

「追加するプリンシパル」に HeatWave on AWS アカウントの ARN を入力します（固定値です）。

- ARN : arn:aws:iam::612981981079:root

入力したら「プリンシパルを許可」をクリックします。

![](/images/heatwave-on-aws-privatelink/heatwave-on-aws-privatelink_055.png)

## レプリカ DB（HeatWave on AWS）作成

ここからは HeatWave on AWS コンソールで作業します。

Resources タブ → DB Systems タブで「Create DB System」をクリックします。

![](/images/heatwave-on-aws-privatelink/heatwave-on-aws-privatelink_061.png)

- Display name : 任意
- Username・Password : 任意
- Shape : 任意（ここでは MySQL.2.16GB を選択）
- Data storage size : 任意（同じく 32 を指定）
- Availability Zone : ソース DB を作成した AZ に合わせる
- MySQL Configuration : デフォルトで OK だがカスタムでタイムゾーンを調整しておいたほうが良い

![](/images/heatwave-on-aws-privatelink/heatwave-on-aws-privatelink_062.png)

- MySQL version : ソース DB より後で出たバージョン（かつメジャーバージョン番号が同等以上・ここでは 8.4.0）を選択
- Maintenance window : 任意
- enable inbound connectivity from allowed public IP address ranges : チェックしない
- Port : 3306
- X Plugin Port : 空欄
- Backup policy : 任意
- IAM roles : 無指定

入力・選択したら「Next」をクリックします。

![](/images/heatwave-on-aws-privatelink/heatwave-on-aws-privatelink_063.png)

- Provision HeatWave Cluster : チェック
- Display name : 任意
- Enable HeatWave Lakehouse : チェックしない
- Shape : 任意（ここでは HeatWave.16GB を選択）
- Node count : 任意（同じく 1 を指定）

入力・選択したら「Create」をクリックします。

![](/images/heatwave-on-aws-privatelink/heatwave-on-aws-privatelink_064.png)

作成完了を待ちます。

## Egress PrivateLink 作成

Resources タブ → PrivateLinks タブで「Create PrivateLink」をクリックします。

![](/images/heatwave-on-aws-privatelink/heatwave-on-aws-privatelink_071.png)

- Display name : 任意
- PrivateLink type : Egress

入力・選択したら「Next」をクリックします。

![](/images/heatwave-on-aws-privatelink/heatwave-on-aws-privatelink_072.png)

- Service name : 先ほど作成したエンドポイントのサービス名
- Source :
  - Hostname : 空欄のまま
  - Port : 3306 のまま
  - Target DB System : 先ほど作成したレプリカ DB を選択

入力・選択したら「Create」をクリックします。

![](/images/heatwave-on-aws-privatelink/heatwave-on-aws-privatelink_073.png)

作成完了を待ちます。

State が Active になったら完了です。

完了したら、「Default hostname」を確認します（後で使います）。

![](/images/heatwave-on-aws-privatelink/heatwave-on-aws-privatelink_074.png)

## AWS 側でエンドポイント接続リクエストを承諾

一旦 AWS マネジメントコンソールに移って作業を進めます。

先ほど作成したエンドポイントサービスの画面で「エンドポイント接続」タブを開きます。

![](/images/heatwave-on-aws-privatelink/heatwave-on-aws-privatelink_081.png)

HeatWave on AWS 側で作成したエンドポイント接続が一覧に Pending acceptance 状態で表示されるのでそれを選択し、「アクション」から「エンドポイント接続リクエストの承諾」を選択します。
![](/images/heatwave-on-aws-privatelink/heatwave-on-aws-privatelink_082.png)

内容を確認し、確認フィールドに「承諾」と入力して「承諾」（ボタン）をクリックします。

![](/images/heatwave-on-aws-privatelink/heatwave-on-aws-privatelink_083.png)

対象エンドポイント接続が Available 状態になります。

![](/images/heatwave-on-aws-privatelink/heatwave-on-aws-privatelink_084.png)

## チャネルを作成

Resources タブ → Channels タブで「Create Channel」をクリックします。

![](/images/heatwave-on-aws-privatelink/heatwave-on-aws-privatelink_091.png)

- Display name : 任意
- Enable channel replication upon creation : チェック
- Target DB System : レプリカ DB を選択
- Channel over PrivateLink を選択
- Egress PrivateLink : 先ほど作成した Egress PrivateLink を選択
- Hostname : 先ほど確認した Egress PrivateLink の Default hostname の内容を入力
- Port : 3306
- Username・Password : 最初のほうで作成したレプリカ DB からの接続用ユーザーとパスワードを入力
- SSL mode : Required

![](/images/heatwave-on-aws-privatelink/heatwave-on-aws-privatelink_092.png)

- Do not use GTID auto-positioning を選択
- UUID specification : Same UUID as target DB system を選択
- Binary log file name : 最初のほうで確認したバイナリログファイル名を入力
- Binary log offset : 同様にログポジションを入力
- Replication details : デフォルトのまま
- Tables without primary key : お好みで選択
- Channel filters : AWS Aurora MySQL v3 (8.0) を選択（実質フィルタなし）

入力・選択したら「Create」をクリックします。

![](/images/heatwave-on-aws-privatelink/heatwave-on-aws-privatelink_093.png)

作成完了を待ちます。

State が Active になったら完了です。

![](/images/heatwave-on-aws-privatelink/heatwave-on-aws-privatelink_094.png)

:::message
この記事ではインバウンドレプリケーション用の設定を扱います。そのため、AWS アカウントにある EC2・ECS などから HeatWave on AWS に接続するための、逆方向の PrivateLink については扱いません。

逆方向の PrivateLink については、こちら ↓ の記事を参考にすると良いでしょう。

https://blog.s-style.co.jp/2024/09/12560/
:::

## データレプリケーションをテスト

環境構築が完了したら、実際にデータを投入してレプリケーションの動作を試してみます。

### Aurora MySQL へサンプルデータを投入

以前[こちらの記事](https://qiita.com/hmatsu47/items/0979f877ad596cf3cf67#%E3%82%B5%E3%83%B3%E3%83%97%E3%83%AB%E3%83%87%E3%83%BC%E3%82%BF%E3%83%99%E3%83%BC%E3%82%B9%E6%A7%8B%E7%AF%89)で使ったサンプルデータをソース DB（Auora）に投入します。

https://qiita.com/hmatsu47/items/0979f877ad596cf3cf67#%E3%82%B5%E3%83%B3%E3%83%97%E3%83%AB%E3%83%87%E3%83%BC%E3%82%BF%E3%83%99%E3%83%BC%E3%82%B9%E6%A7%8B%E7%AF%89

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

しばらく待ってから、HeatWave on AWS コンソールの Workspace タブで「Connect to DB System」をクリックし、レプリカ DB に接続します。

Query Editor タブで以下の SQL 文を実行し、結果が正しく返ればデータは正常にレプリケーションされています。

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

### HeatWave エンジン有効化

ついでに HeatWave エンジンも有効化して試してみます。

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

先ほどと同じ結果が短い時間で返って来れば成功です。

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

## 気になる点

これは HeatWave on AWS というよりは AWS の PrivateLink の仕様上仕方がないことなのですが、ターゲットの指定が IP アドレスに限定されるので、（ソース DB である）Aurora のインスタンスタイプ変更などによってソース DB（Writer）の IP アドレスが変わると、新しい IP アドレスでのターゲット再作成が必要になります。

また、ソース DB のフェイルオーバーに追従するには、Aurora クラスターのフェイルオーバーイベントを SNS トピック経由で受け取ってターゲットの IP アドレス（Writer）再登録を行う処理を Lambda などで実装する必要があります。

---

**2024/9/27 追記：**
[続き](https://zenn.dev/hmatsu47/articles/heatwave-on-aws-privatelink-failover)を書きました。

https://zenn.dev/hmatsu47/articles/heatwave-on-aws-privatelink-failover
