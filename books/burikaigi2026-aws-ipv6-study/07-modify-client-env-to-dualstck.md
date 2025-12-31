---
title: "クライアント環境を IPv6 対応にする（デュアルスタック編）"
---

続いて、クライアント環境を IPv6 対応にします。

クライアント環境を IPv6 対応にする場合、おおまかに 2 つのパターンがありますが、まずはデュアルスタック化を実施します。

- VPC に IPv6 CIDR ブロックを追加（割り当て）
- プライベートサブネット（2 個）に IPv6 CIDR ブロックを追加（割り当て）
- Egress Only インターネットゲートウェイを作成
- プライベートサブネットのルートテーブル（2 個）に IPv6 のデフォルトルートを追加
- EC2（クライアント）に IPv6 アドレスを追加
  - EC2 を停止
  - IPv6 アドレス（GUA）を割り当て
  - EC2 を起動（開始）
- `curl`コマンドで接続テスト

この段階の作業が完了すると、図のような構成になります。

![](/images/burikaigi2026-aws-ipv6-study/IPv6-03-client-ds.drawio.png)

作業対象となるクライアント環境部分を拡大すると以下のとおりです。

![](/images/burikaigi2026-aws-ipv6-study/IPv6-03.png)

## AWS 環境上での作業

### VPC に IPv6 CIDR ブロックを追加

- クライアント用 VPC「cl-ipv4-to-dualstack-vpc」画面 → **「アクション」** → **「CIDR の編集」** から

![](/images/burikaigi2026-aws-ipv6-study/014001-cl-add-ipv6-cidr-to-vpc.png)

- **「新しい IPv6 CIDR を追加」** をクリック
- 「Amazon 提供の IPv6 CIDR ブロック」を選択
- **「CIDR を選択」** をクリック

### プライベートサブネットに IPv6 CIDR ブロックの一部を割り当て

- クライアント用 VPC のプライベートサブネット（AZ-a）画面 → **「アクション」** → **「IPv6 CIDR の編集」** から

- **「IPv6 CIDR を追加」** をクリック

![](/images/burikaigi2026-aws-ipv6-study/014002-cl-add-ipv6-cidr-to-subnet-1.png)

- サブネットの CIDR ブロック : VPC の CIDR ブロックの先頭`/64`（プレフィックスの最後が「`00`」）を選択
  - 下向き「>」を 2 回クリック
- **「保存」** をクリック

AZ-b 側も同様に割り当てます。

- クライアント用 VPC のプライベートサブネット（AZ-b）画面 → **「アクション」** → **「IPv6 CIDR の編集」** から

- **「IPv6 CIDR を追加」** をクリック

![](/images/burikaigi2026-aws-ipv6-study/014003-cl-add-ipv6-cidr-to-subnet-2.png)

- サブネットの CIDR ブロック : VPC の CIDR ブロックの 2 番目の`/64`（プレフィックスの最後が「`01`」）を選択
  - 下向き「>」を 2 回クリック後、右向き「>」を 1 回クリック
- **「保存」** をクリック

### Egress Only インターネットゲートウェイを作成

- **「VPC」** メニュー → Egress-only インターネットゲートウェイから
- **「Egress Only インターネットゲートウェイを作成」** をクリック

![](/images/burikaigi2026-aws-ipv6-study/014004-cl-create-eigw.png)

- 名前 : `cl-eigw-001`
- VPC : クライアント用 VPC
- **「Egress Only インターネットゲートウェイを作成」** をクリック

### プライベートサブネットのルートテーブルに IPv6 のデフォルトルートを追加

- クライアント用 VPC のプライベートルートテーブル画面（AZ-a 側） → **「ルートを編集」** から

![](/images/burikaigi2026-aws-ipv6-study/014005-cl-add-route-to-eigw-1.png)

- **「ルートを追加」** で以下のルートを追加
  - 送信先 : `::/0`
  - ターゲット : Egress Only インターネットゲートウェイ（既存の`eigw-XXXXXXXX`）
- **「変更を保存」** をクリック

AZ-b 側も同様に編集します。

- クライアント用 VPC のプライベートルートテーブル画面（AZ-b 側） → **「ルートを編集」** から

![](/images/burikaigi2026-aws-ipv6-study/014006-cl-add-route-to-eigw-2.png)

- **「ルートを追加」** で以下のルートを追加
  - 送信先 : `::/0`
  - ターゲット : Egress Only インターネットゲートウェイ（既存の`eigw-XXXXXXXX`）
- **「変更を保存」** をクリック

### EC2（クライアント）に IPv6 アドレスを追加

#### EC2 を停止

- **EC2** メニュー → インスタンスから

![](/images/burikaigi2026-aws-ipv6-study/015001-cl-stop-ec2-1.png)

- 「client-001a」を選択 → **「インスタンスの状態」** → **「インスタンスを停止」**

![](/images/burikaigi2026-aws-ipv6-study/015002-cl-stop-ec2-2.png)

- **「停止」** をクリック

#### IPv6 アドレス（GUA）を割り当て

- 「client-001a」を選択 → **「アクション」** → **「IP アドレスの管理」** から

![](/images/burikaigi2026-aws-ipv6-study/015003-cl-add-ipv6addr-to-ec2-1.png)

- 「IPv6 アドレス」の **「新しい IP アドレスの割り当て」** をクリック

![](/images/burikaigi2026-aws-ipv6-study/015004-cl-add-ipv6addr-to-ec2-2.png)

- **「確認」** をクリック

#### EC2 を起動（開始）

![](/images/burikaigi2026-aws-ipv6-study/015005-cl-start-ec2.png)

- 「client-001a」を選択 → **「インスタンスの状態」** → **「インスタンスを開始」**

以上でクライアント環境のデュアルスタック化が完了しました。

サーバー環境へ接続してテストしてみましょう。

### `curl`コマンドで接続テスト

EC2（クライアント）に接続し、サーバー環境の ALB に`curl`コマンドでアクセスしてみます。

- 「client-001a」を選択 → **「接続」**

![](/images/burikaigi2026-aws-ipv6-study/016001-cl-connect-4.png)

- 接続タイプ : プライベート IP を使用して接続
- EC2 Instance Connect エンドポイント : 「cl-ec2-connect」に相当するエンドポイントを選択
- **「接続」** をクリック

先ほどの履歴を「↑」キーなどで探して ALB に対して`curl`コマンドを実行します。

```sh:
$ curl -v4 http://ipv4-to-dualstack-XXXXXXXX.ap-northeast-3.elb.amazonaws.com/index2.php
* Host ipv4-to-dualstack-XXXXXXXX.ap-northeast-3.elb.amazonaws.com:80 was resolved.
* IPv6: (none)
* IPv4: 15.XX.XX.95, 15.XX.XX.67
*   Trying 15.XX.XX.95:80...
* Connected to ipv4-to-dualstack-XXXXXXXX.ap-northeast-3.elb.amazonaws.com (15.XX.XX.95) port 80
* using HTTP/1.x
> GET /index2.php HTTP/1.1
> Host: ipv4-to-dualstack-XXXXXXXX.ap-northeast-3.elb.amazonaws.com
> User-Agent: curl/8.11.1
> Accept: */*
>
* Request completely sent off
< HTTP/1.1 200 OK
< Date: Thu, 25 Dec 2025 16:09:09 GMT
< Content-Type: text/html; charset=UTF-8
< Transfer-Encoding: chunked
< Connection: keep-alive
< Server: nginx/1.28.0
<
<html><h1>HTTP_X_FORWARDED_FOR: 56.XX.XX.235</html></h1>
* Connection #0 to host ipv4-to-dualstack-XXXXXXXX.ap-northeast-3.elb.amazonaws.com left intact
```

結果が返ってきました。こちらは IPv4 です。

IPv6 でもアクセスしてみましょう。

```sh:
$ curl -v6 http://ipv4-to-dualstack-XXXXXXXX.ap-northeast-3.elb.amazonaws.com/index2.php
* Host ipv4-to-dualstack-XXXXXXXX.ap-northeast-3.elb.amazonaws.com:80 was resolved.
* IPv6: 2406:XX:XX:6401:42fc:d75a:1696:583c, 2406:XX:XX:6400:16de:a21e:34f6:e1ab
* IPv4: (none)
*   Trying [2406:XX:XX:6401:42fc:d75a:1696:583c]:80...
* Connected to ipv4-to-dualstack-XXXXXXXX.ap-northeast-3.elb.amazonaws.com (2406:XX:XX:6401:42fc:d75a:1696:583c) port 80
* using HTTP/1.x
> GET /index2.php HTTP/1.1
> Host: ipv4-to-dualstack-XXXXXXXX.ap-northeast-3.elb.amazonaws.com
> User-Agent: curl/8.11.1
> Accept: */*
>
* Request completely sent off
< HTTP/1.1 200 OK
< Date: Thu, 25 Dec 2025 16:09:21 GMT
< Content-Type: text/html; charset=UTF-8
< Transfer-Encoding: chunked
< Connection: keep-alive
< Server: nginx/1.28.0
<
<html><h1>HTTP_X_FORWARDED_FOR: 2406:XX:XX:4b00:ba1d:1fff:c7e0:a236</html></h1>
* Connection #0 to host ipv4-to-dualstack-XXXXXXXX.ap-northeast-3.elb.amazonaws.com left intact
```

IPv6 のアドレスが表示されました。

## 作業内容に関する補足説明

### クライアント環境のデュアルスタック化

クライアント環境をデュアルスタック化して IPv6 対応した場合、クライアントからの通信は以下のように行われます。

- クライアント自身が IPv4・IPv6 それぞれの通信スタックを使って相手のホストと通信する
- インターネット上にある IPv4 ホスト宛ての通信は NAT ゲートウェイに流す
  - ルートテーブルで IPv4 のデフォルトルート（`0.0.0.0/0`）を NAT ゲートウェイに向ける
- インターネット上にある IPv6 ホスト宛ての通信は Egress Only インターネットゲートウェイに流す
  - ルートテーブルで IPv6 のデフォルトルート（`::/0`）を Egress Only インターネットゲートウェイに向ける

前のチャプターで説明したとおり、IPv6 アドレス同士の NAT は非推奨であり AWS の NAT ゲートウェイでもサポートしていないため、IPv6 インターネットへのアクセスが必要なホストには GUA を割り当てることになります。

ただ、普通に IPv6 の通信をインターネットゲートウェイに流してしまうと、EC2 に割り当てた GUA を宛先として、インターネット側から EC2 へのアクセスも可能になります。

もちろんセキュリティグループでそのような通信を止めることは可能ですが、IPv4 サブネットにおける NAT ゲートウェイと同じような使い勝手で IPv6 サブネットを「外からアクセスさせない」形に保つのが、Egress Only インターネットゲートウェイです。

### EC2 への IPv6 アドレスの割り当て

IPv4 同様に、ユーザーが固定的に割り当てる方法と、DHCP(v6) を使う方法があります。

IPv6 の仕組み上は、他にも

- SLAAC : サブネットプレフィックスなどをルータから受信し、インターフェース識別子（IID）を自動生成する
  - IID を EUI-64（MAC アドレスを使う方式）で生成するのが標準だったが、MAC アドレスが露出するプライバシー問題があるので現在は非推奨で、一時的な IID をランダムに生成して一定時間で切り替えていく Temporary ID 方式が使われることが多い

などの方法があり、また DHCP を使う方法にも「ステートレス」「ステートフル」があるなど IPv4 よりは複雑な仕様になっています。

### クライアントがデュアルスタックの場合、IPv4・IPv6 のどちらを優先する？

デュアルスタック ALB の場合、通信の「始点」はクライアントであり、

- クライアントが IPv4 でアクセスしてきたら IPv4 でレスポンスを返す
- クライアントが IPv6 でアクセスしてきたら IPv6 でレスポンスを返す

という「受け身」の対応で十分でした。

:::message
ターゲットグループ側はあらかじめ IPv4・IPv6 のどちらで通信するかを指定する形になっています。
:::

クライアントがデュアルスタックの場合は IPv4・IPv6 のどちらを優先すべきでしょうか？

IPv6 の利用が始まった当初は「まず IPv6 でアクセスしてみて、ダメだったら IPv4 でアクセスする」という流れでの「IPv6 優先」が一般的でしたが、この場合、IPv4 しか対応していないホストへのアクセスが遅延してしまい、特に当時は IPv6 対応が進んでいなかったこともあり利用者の体験を大きく損なう結果となっていました。

現在は、「IPv4・IPv6 の両方（ほぼ）同時にアクセスを試行して、最初に応答したほうを選択する」、通称「Happy Eyeballs」などの仕組みが使われています。

:::message
現時点で標準化され広く使われているのは [RFC 8305](https://datatracker.ietf.org/doc/html/rfc8305) で規定されているバージョン 2 です。

IPv4・IPv6 へのアクセスを並行で試行しますが、厳密には IPv6 が優先されます。

一方で QUIC 対応を意図した[バージョン 3](https://datatracker.ietf.org/doc/draft-ietf-happy-happyeyeballs-v3/) の策定も進んでいます。
:::

## このチャプターのまとめ

- クライアント環境をデュアルスタック化すると、クライアント自身が IPv4・IPv6 の通信を使い分けることになる
- IPv4 は NAT ゲートウェイへ、IPv6 は Egress Only インターネットゲートウェイに流す形でルーティングする
- Egress Only インターネットゲートウェイは 6to6 の NAT が使えない状況で IPv4 における NAT ゲートウェイと同等の使い勝手を提供する
- デュアルスタック環境のクライアントは「Happy Eyeballs」などの仕組みを使って IPv4・IPv6 のどちらを使うかを決める
