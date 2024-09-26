---
title: "HeatWave on AWS で PrivateLink 接続インバウンドレプリケーション時に Aurora のフェイルオーバーに対応する"
emoji: "🔁"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["mysql", "heatwave", "aws", "oci"]
published: false
---

[こちらの記事](https://zenn.dev/hmatsu47/articles/heatwave-on-aws-privatelink)の続きです。

https://zenn.dev/hmatsu47/articles/heatwave-on-aws-privatelink

![](/images/heatwave-on-aws-privatelink-failover/heatwave-on-aws-privatelink-failover_001.png)

![](/images/heatwave-on-aws-privatelink-failover/heatwave-on-aws-privatelink-failover_002.png)

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

![](/images/heatwave-on-aws-privatelink-failover/heatwave-on-aws-privatelink-failover_011.png)

- `IP_DICT` :
  - `{"aurora-source-instance-1": "172.31.124.155", "aurora-source-instance-2": "172.31.128.232"}`
- `TARGET_GROUP_ARN` :
  - 前の記事で作成したターゲットグループの ARN】

![](/images/heatwave-on-aws-privatelink-failover/heatwave-on-aws-privatelink-failover_012.png)

![](/images/heatwave-on-aws-privatelink-failover/heatwave-on-aws-privatelink-failover_021.png)

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

![](/images/heatwave-on-aws-privatelink-failover/heatwave-on-aws-privatelink-failover_022.png)

![](/images/heatwave-on-aws-privatelink-failover/heatwave-on-aws-privatelink-failover_023.png)

![](/images/heatwave-on-aws-privatelink-failover/heatwave-on-aws-privatelink-failover_031.png)

![](/images/heatwave-on-aws-privatelink-failover/heatwave-on-aws-privatelink-failover_032.png)

![](/images/heatwave-on-aws-privatelink-failover/heatwave-on-aws-privatelink-failover_041.png)

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

![](/images/heatwave-on-aws-privatelink-failover/heatwave-on-aws-privatelink-failover_042.png)

![](/images/heatwave-on-aws-privatelink-failover/heatwave-on-aws-privatelink-failover_051.png)

![](/images/heatwave-on-aws-privatelink-failover/heatwave-on-aws-privatelink-failover_052.png)

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

![](/images/heatwave-on-aws-privatelink-failover/heatwave-on-aws-privatelink-failover_061.png)

![](/images/heatwave-on-aws-privatelink-failover/heatwave-on-aws-privatelink-failover_071.png)

![](/images/heatwave-on-aws-privatelink-failover/heatwave-on-aws-privatelink-failover_072.png)

![](/images/heatwave-on-aws-privatelink-failover/heatwave-on-aws-privatelink-failover_073.png)

![](/images/heatwave-on-aws-privatelink-failover/heatwave-on-aws-privatelink-failover_081.png)

![](/images/heatwave-on-aws-privatelink-failover/heatwave-on-aws-privatelink-failover_091.png)

![](/images/heatwave-on-aws-privatelink-failover/heatwave-on-aws-privatelink-failover_092.png)

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

![](/images/heatwave-on-aws-privatelink-failover/heatwave-on-aws-privatelink-failover_101.png)
