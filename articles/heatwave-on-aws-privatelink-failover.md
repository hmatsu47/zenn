---
title: "HeatWave on AWS で PrivateLink 接続インバウンドレプリケーション時に Aurora のフェイルオーバーに対応する"
emoji: "🔁"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["mysql", "heatwave", "aws", "oci"]
published: true
---

[こちらの記事](https://zenn.dev/hmatsu47/articles/heatwave-on-aws-privatelink)の続きです。

https://zenn.dev/hmatsu47/articles/heatwave-on-aws-privatelink

記事の最後に、

> また、ソース DB のフェイルオーバーに追従するには、Aurora クラスターのフェイルオーバーイベントを SNS トピック経由で受け取ってターゲットの IP アドレス（Writer）再登録を行う処理を Lambda などで実装する必要があります。

と書きましたが、これを実装してみました。

:::message
前記事からの流れで非 GTID 形式（匿名トランザクション）でレプリケーションしていますが、フェイルオーバー後のデータ不整合の可能性を考えて、GTID 形式でのレプリケーションをお勧めします。
:::

## 実装の前に…なぜ PrivateLink の NLB 標準のヘルスチェック機能ではダメなのか？

NLB にはターゲットグループ内のターゲットに対して定期的にヘルスチェックを行い、正常なターゲットにリクエストを振り分ける機能があります。

通常ならこれを使いたくなるところですが、**NLB は MySQL プロトコルを喋れません。**

Oracle が公式ドキュメントで提示している設定手順でも（HeatWave on AWS 側で「異常な接続を繰り返している」と判定されて接続拒否されないよう）ヘルスチェック用のポートを TCP:3306 ではなく TCP:40000 など **MySQL に無関係のポート番号で上書きする**指定になっています（つまり、ヘルスチェック自体が有効に作動しないようにしている）。

そのため、NLB のヘルスチェック機能とは別の仕組みを用意してターゲットの切り替え（入れ替え）を行う必要があります。

## PrivateLink 用のターゲットグループ属性を編集

PrivateLink 用のターゲットグループの画面で「属性」タブを開いて右端の「編集」をクリックします。

![](/images/heatwave-on-aws-privatelink-failover/heatwave-on-aws-privatelink-failover_001.png)

以下の設定項目を変更して「変更内容の保存」をクリックします。

##### ターゲットの異常な状態の管理

- ターゲットが異常になったときに接続を終了する : オフ（無効）

##### ターゲットの登録解除管理

- 登録解除時に接続を終了 : オン（有効）
- 登録解除の遅延（ストリーミング間隔） : 0 秒

##### ターゲット選択設定

- クロスゾーン負荷分散 : オン

![](/images/heatwave-on-aws-privatelink-failover/heatwave-on-aws-privatelink-failover_002.png)

## ソース DB（Aurora）のクラスターに Reader を追加

（RDS のメニューで）ソース DB（Aurora）のクラスターの画面から、「アクション」メニューの「リーダーの追加」を実施します。

## ターゲット入れ替え用の Lambda 関数を作成

Aurora クラスターのフェイルオーバーイベントを受信した際に、PrivateLink 用のターゲットグループのターゲットを新しい Writer に入れ替えるための Lambda 関数です。

Lambda の「関数の作成」画面で以下を入力・選択して関数を作成します。

- 一から作成
- 関数名 : 任意
- ランタイム : Python 3.12
- アーキテクチャ : 任意（arm64 を推奨）

関数が作成されたら、「コード」タブでコードソースを入力します（`lambda_function.py`）。

:::details lambda_function.py

```python:lambda_function.py
import boto3
import json
import os

client = boto3.client('elbv2')
target_group_arn = os.getenv('TARGET_GROUP_ARN')
ip_dict = json.loads(os.getenv('IP_DICT'))

# インスタンスIDを取得
def get_instance_id(event):
    records = event.get('Records', [])
    if len(records) == 0:
        print('Recordsの要素が見つかりません')
        return None
    sns = records[0].get('Sns', '')
    if sns == '':
        print('Snsが見つかりません')
        return None
    message_json = sns.get('Message', '')
    if message_json == '':
        print('Messageが見つかりません')
        return None
    message = json.loads(message_json)
    source_id = message.get('Source ID', '')
    if source_id == '':
        print('Source IDが見つかりません')
        return None
    event_message = message.get('Event Message', '')
    if event_message == '':
        print('Event Messageが見つかりません')
        return None
    index = event_message.rfind(': ')
    if index == -1:
        print('Instance IDが見つかりません')
        return None
    return event_message[index + 2:]

# ターゲットとして設定されているIPアドレスを取得
def get_target_ip_address(target_group_arn):
    response = client.describe_target_health(TargetGroupArn=target_group_arn)
    target_health_descriptions = response.get('TargetHealthDescriptions', [])
    if len(target_health_descriptions) == 0:
        print('ターゲットが見つかりません')
        return None
    target = target_health_descriptions[0].get('Target')
    if target is None:
        print('Targetが見つかりません')
        return None
    ip_address = target.get('Id')
    if ip_address is None:
        print('ターゲットとして設定されているIPアドレスが見つかりません')
        return None
    return ip_address

# 指定IPアドレスをターゲットから登録解除
def deregist_target_ip_address(ip_address):
    response = client.deregister_targets(
        TargetGroupArn=target_group_arn,
        Targets=[
            {
                'Id': ip_address,
                'Port': 3306
            },
        ]
    )
    return

# 指定IPアドレスをターゲットとして登録
def regist_target_ip_address(ip_address):
    response = client.register_targets(
        TargetGroupArn=target_group_arn,
        Targets=[
            {
                'Id': ip_address,
                'Port': 3306
            },
        ]
    )
    return

def lambda_handler(event, context):
    # インスタンスID取得
    instance_id = get_instance_id(event)
    if instance_id is None:
        # インスタンスIDなし→処理打ち切り
        return
    # 変更後のIPアドレスを取得
    new_ip_address = ip_dict.get(instance_id)
    if new_ip_address is None:
        print('ターゲット候補の中にフェイルオーバー先のインスタンスIDが見つかりません : (%s)' % (instance_id))
        return

    # 変更前のIPアドレスを取得
    old_ip_address = get_target_ip_address(target_group_arn)
    if old_ip_address is None:
        return
    # 変更前後でターゲットIPアドレスが変わるか？
    if new_ip_address == old_ip_address:
        print('ターゲットIPアドレスの変更はありません : (%s)' % (new_ip_address))
        return
    # 古いターゲットIPアドレスを登録解除
    deregist_target_ip_address(old_ip_address)
    print('古いターゲットIPアドレスを登録解除しました : (%s)' % (old_ip_address))
    # 新しいターゲットIPアドレスを登録
    regist_target_ip_address(new_ip_address)
    print('新しいIPアドレスをターゲットに登録しました : (%s)' % (new_ip_address))
```

:::

「設定」タブの「一般設定」でタイムアウトを 10 秒程度に設定しておきます。

通常は 2 秒以内に処理が終わるので、デフォルトの 3 秒のままでも支障はありませんが、念のため少し延長しておきます。

![](/images/heatwave-on-aws-privatelink-failover/heatwave-on-aws-privatelink-failover_011.png)

「環境変数」で以下の環境変数を登録します。

- `IP_DICT` :
  - `{"【1つ目のインスタンスID】": "【1つ目のインスタンスのIPアドレス】", "【2つ目のインスタンスID】": "【2つ目のインスタンスのIPアドレス】"}`
- `TARGET_GROUP_ARN` :
  - 前の記事で作成したターゲットグループの ARN】

:::message
2 つ以上の Reader インスタンスをクラスターに追加する場合は、フェイルオーバー対象になるすべてのインスタンス ID と IP アドレスの組み合わせを列挙します。
:::

![](/images/heatwave-on-aws-privatelink-failover/heatwave-on-aws-privatelink-failover_012.png)

ここで一旦「コード」タブに戻って「Deploy」をクリックしておきます。

### Lambda 関数実行用のロールに権限を追加

「設定」タブの「アクセス権限」でロール名のリンクをクリックし、ロール（IAM）画面の「許可」タブで「許可を追加」→「インラインポリシーを作成」をクリックします。

![](/images/heatwave-on-aws-privatelink-failover/heatwave-on-aws-privatelink-failover_021.png)

以下のポリシーを JSON 形式で入力して「次へ」をクリックします。

:::message
【アカウント ID】の部分を正しいアカウント ID に置き換えてください。
:::

:::details ポリシー（JSON）

```json:ポリシー
{
	"Version": "2012-10-17",
	"Statement": [
		{
			"Sid": "VisualEditor0",
			"Effect": "Allow",
			"Action": [
				"elasticloadbalancing:DescribeLoadBalancers",
				"elasticloadbalancing:DescribeTargetGroupAttributes",
				"elasticloadbalancing:DescribeTargetHealth",
				"elasticloadbalancing:DescribeTargetGroups"
			],
			"Resource": "*"
		},
		{
			"Sid": "VisualEditor1",
			"Effect": "Allow",
			"Action": [
				"elasticloadbalancing:RegisterTargets",
				"elasticloadbalancing:DeregisterTargets"
			],
			"Resource": "arn:aws:elasticloadbalancing:ap-northeast-1:【アカウントID】:targetgroup/*/*"
		}
	]
}
```

:::

![](/images/heatwave-on-aws-privatelink-failover/heatwave-on-aws-privatelink-failover_022.png)

「ポリシーの作成」をクリックします。

![](/images/heatwave-on-aws-privatelink-failover/heatwave-on-aws-privatelink-failover_023.png)

## SNS トピックとサブスクリプションを作成

Aurora クラスターのフェイルオーバーイベントを受けて Lambda 関数を呼び出すための SNS トピックとサブスクリプションを作成します。

### SNS トピックを作成

SNS のメニューから「トピック」を選択し、「トピックの作成」をクリックします。

![](/images/heatwave-on-aws-privatelink-failover/heatwave-on-aws-privatelink-failover_031.png)

以下を入力・選択して「トピックの作成」をクリックします。

##### 詳細

- タイプ : スタンダード
- 名前 : 任意

![](/images/heatwave-on-aws-privatelink-failover/heatwave-on-aws-privatelink-failover_032.png)

### SNS サブスクリプションを作成

SNS のメニューから「サブスクリプション」を選択し、「サブスクリプションの作成」をクリックします。

![](/images/heatwave-on-aws-privatelink-failover/heatwave-on-aws-privatelink-failover_041.png)

以下を入力・選択して「サブスクリプションの作成」をクリックします。

##### 詳細

- トピック ARN : 直前に作成したトピックを選択
- プロトコル : AWS Lambda
- エンドポイント : 先に作成した Lambda 関数を選択

##### サブスクリプションフィルターポリシー - オプション

- サブスクリプションフィルターポリシー : オン（有効化）
- フィルターポリシーのスコープ : メッセージ本文
- JSON エディタ : 以下の JSON を入力

::::details サブスクリプションフィルターポリシー（JSON）

```json:サブスクリプションフィルターポリシー
{
"Event ID": [
    {
    "suffix": "RDS-EVENT-0072"
    },
    {
    "suffix": "RDS-EVENT-0073"
    }
]
}
```

:::message
フェイルオーバーに追従してターゲットを切り替えるタイミングをできるだけ早くするため、「フェイルオーバー開始」イベントを拾って Lambda 関数を起動しています。
「フェイルオーバー完了」イベントを拾う場合は`RDS-EVENT-0071`を指定します。
:::

::::

![](/images/heatwave-on-aws-privatelink-failover/heatwave-on-aws-privatelink-failover_042.png)

## Aurora（RDS）のイベントサブスクリプションを作成

RDS のメニューから「イベントサブスクリプション」を選択し、「イベントサブスクリプションの作成」をクリックします。

![](/images/heatwave-on-aws-privatelink-failover/heatwave-on-aws-privatelink-failover_051.png)

以下を入力・選択して「作成」をクリックします。

##### 詳細

- 名前 : 任意

##### ターゲット

- 次への通知の送信 : Amazon リソースネーム（ARN）
- ARN : 先ほど作成した SNS トピックの ARN を選択

##### ソース

- ソースタイプ : クラスター
- クラスターを含みます : 特定のクラスターを選択
- 特定のクラスター : ソース DB（Aurora）のクラスターを選択
- イベントカテゴリを含みます : 特定のイベントカテゴリを選択
- 特定のイベントカテゴリ : failover

![](/images/heatwave-on-aws-privatelink-failover/heatwave-on-aws-privatelink-failover_052.png)

## フェイルオーバーを試してみる

実際にソース DB（AUrora）でフェイルオーバーを発生させて、レプリケーションが追従するか試してみます。

### ソース DB でテスト用の DB スキーマ・テーブル・データ行を追加

簡単なテーブルを作成してデータ行を入れてみます。

```sql:ソースDBでDBスキーマ・テーブル・データ行追加
mysql> CREATE DATABASE hoge;
Query OK, 1 row affected (0.00 sec)

mysql> USE hoge;
Database changed
mysql> CREATE TABLE fuga (id int PRIMARY KEY AUTO_INCREMENT, val int);
Query OK, 0 rows affected (0.02 sec)

mysql> INSERT INTO fuga SET val = 100;
Query OK, 1 row affected (0.01 sec)

mysql> INSERT INTO fuga SET val = 150;
Query OK, 1 row affected (0.00 sec)
```

### レプリカ DB で複製確認

レプリカ DB（HeatWave on AWS）で確認してみます。

先ほどソース DB で追加した DB スキーマ・テーブル・データ行が複製されています。

![](/images/heatwave-on-aws-privatelink-failover/heatwave-on-aws-privatelink-failover_061.png)

### フェイルオーバーを実行

ソース DB（Aurora）の画面から、フェイルオーバーを実行します。

![](/images/heatwave-on-aws-privatelink-failover/heatwave-on-aws-privatelink-failover_071.png)

「フェイルオーバー」をクリックします。

![](/images/heatwave-on-aws-privatelink-failover/heatwave-on-aws-privatelink-failover_072.png)

しばらく待ってフェイルオーバーが実行されると、クラスターの「最近のイベント」にフェイルオーバー関連のイベントが表示されます。

![](/images/heatwave-on-aws-privatelink-failover/heatwave-on-aws-privatelink-failover_073.png)

PrivateLink 用のターゲットグループのターゲットが入れ替わりました。

![](/images/heatwave-on-aws-privatelink-failover/heatwave-on-aws-privatelink-failover_081.png)

レプリカ DB（HeatWave on AWS）の Channel 状態が「Needs Attention」に変わりました。

![](/images/heatwave-on-aws-privatelink-failover/heatwave-on-aws-privatelink-failover_091.png)

その後、4 ～ 5 分ほど経過して「Active」に戻りました。

インバウンドレプリケーションが復活したようです。

![](/images/heatwave-on-aws-privatelink-failover/heatwave-on-aws-privatelink-failover_092.png)

### ソース DB でデータ行を追加

インバウンドレプリケーションの状態を（ステータスの表示だけではなく）実際の環境で確かめるために、ソース DB（Aurora）でデータ行を追加してみます。

```sql:データ行追加
mysql> INSERT INTO fuga SET val = 200;
ERROR 2013 (HY000): Lost connection to MySQL server during query
No connection. Trying to reconnect...
Connection id:    55
Current database: hoge

Query OK, 1 row affected (0.04 sec)

mysql> INSERT INTO fuga SET val = 250;
Query OK, 1 row affected (0.01 sec)
```

### レプリカ DB でフェイルオーバー後の複製確認

再びレプリカ DB（HeatWave on AWS）で確認してみます。

先ほどソース DB で追加したデータ行が複製されています。

![](/images/heatwave-on-aws-privatelink-failover/heatwave-on-aws-privatelink-failover_101.png)

## 感想など

ちょっと実装が大変ですね。

また、フェイルオーバーイベントが短い間隔で発生（またはイベント通知が重複発火）し、イベント通知が連続する・順序が入れ替わる場合などエッジケースでテストしてみないと実用に耐えるかどうかわかりません。

- イベント通知に含まれる新しい Writer インスタンス（ID）をそのまま「信用」してターゲットに加える仕様で大丈夫か？
  - インバウンドレプリケーションの復活が少し（数十秒～ 1 分程度）遅れることになっても「フェイルオーバー完了」イベントを拾って、その時点の Aurora クラスターの Writer インスタンスを[`describe_db_clusters`](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/rds/client/describe_db_clusters.html)を使って確認した上でターゲットに加えるべき？
- 古いターゲット（IP アドレス）の登録解除を行う際、「ターゲットグループにはターゲットが 1 つしかない」と決めつけて良いのか？
  - 運用上 Writer インスタンスの IP アドレスしか含まれない想定だが、イベント通知の連続によりたまたま複数の IP アドレスがターゲットとして登録されてしまう事態を想定すべき？
- イベント通知ではなく定期的に Aurora クラスターの Writer インスタンスとターゲットの「差分」が出ていないかを確認してターゲットを切り替えたほうが良くないか？
  - イベント通知を利用する方法と比べると、どうしても切り替えのタイミングが遅くなってしまうし確認処理の分コストが掛かってしまうが、イベント処理の取りこぼしや順序の逆転などで誤った状態になるぐらいなら定期確認を採用すべき？

**インバウンドレプリケーションが復活（自動復旧）するまで 4 ～ 5 分掛かってしまう**のも気になりますね。

MySQL 8.0.22 で実装された[（非同期）レプリケーション接続フェイルオーバー機能](https://blog.s-style.co.jp/2020/10/6828/)が使えると良いのに…と思いつつ、その場合は Egress PrivateLink をあらかじめ複数のソース DB インスタンスに接続しておく必要がありそう（注：現状の仕様ではどちらも不可）なので、コスト増が気になります。

https://blog.s-style.co.jp/2020/10/6828/

色々悩ましいですね。
