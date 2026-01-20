---
title: "サーバー環境のターゲットを IPv6（ULA）化（オプション）"
---

:::message
このチャプターの内容はオプションです。セッションの中では扱いません。
:::

ALB がデュアルスタックであればターゲットサーバーは IPv4・IPv6 のどちらで動いていても構わないので、ALB ターゲットのサーバーを置くサブネットを IPv6 ユニークローカルユニキャストアドレス（ULA）専用にしてみます。

- 既存の EC2（Web Server 用）の AMI を作成
- IP Address Manager（IPAM）を有効化し（無料利用枠）、IPv6 ULA のプールを作成
  - `/48`→`/52`の 2 階層
- IPv6 IPAM プール（ULA）からサーバー用 VPC に`/56`を割り当て
- サーバー用 VPC に新規のサブネットを作成し、IPv6 IPAM プールから`/64`を割り当て
  - AZ-a・AZ-b の 2 個
- 同サブネット用にルートテーブルを作成
  - AZ-a・AZ-b の 2 個
- AMI から EC2（Web Server 用）を作成・新規追加サブネットに配置
  - AZ-a・AZ-b の 2 個
- IPv6（ULA）サブネット用のターゲットグループを作成
  - ターゲットとして EC2（Web Server 用・2 個）を登録
- ALB のターゲットグループを IPv4 用から IPv6（ULA）用に切り替え
- `curl`コマンドまたは IPv6 アドレスが使えるブラウザから接続テスト

これらの作業が完了すると、図のような構成になります。

![](/images/burikaigi2026-aws-ipv6-study/IPv6-05-server-tg-ipv6-ula-only.drawio.png)

作業対象となるサーバー環境部分を拡大すると以下のとおりです。

![](/images/burikaigi2026-aws-ipv6-study/IPv6-05.png)

## AWS 環境上での作業

セッションで詳細を扱わないので、こちらは後日の公開となります。

※手順は順次追記しています

### 既存の EC2（Web Server 用）の AMI を作成

![](/images/burikaigi2026-aws-ipv6-study/030001-create-ami-1.png)
![](/images/burikaigi2026-aws-ipv6-study/030002-create-ami-2.png)

### IP Address Manager（IPAM）を有効化

![](/images/burikaigi2026-aws-ipv6-study/031001-create-ipam-1.png)
![](/images/burikaigi2026-aws-ipv6-study/031002-create-ipam-2.png)

### IPv6 ULA のプールを作成

#### `/48`の階層を作成

![](/images/burikaigi2026-aws-ipv6-study/031003-create-ipam-pool-1.png)
![](/images/burikaigi2026-aws-ipv6-study/031004-create-ipam-pool-2.png)

#### `/52`の階層を作成

![](/images/burikaigi2026-aws-ipv6-study/031005-create-ipam-pool-3.png)
![](/images/burikaigi2026-aws-ipv6-study/031006-create-ipam-pool-4.png)
![](/images/burikaigi2026-aws-ipv6-study/031007-create-ipam-pool-5.png)
![](/images/burikaigi2026-aws-ipv6-study/031008-create-ipam-pool-6.png)

### IPv6 IPAM プール（ULA）からサーバー用 VPC に`/56`を割り当て

![](/images/burikaigi2026-aws-ipv6-study/032001-add-ipam-pool-to-vpc.png)

### サーバー用 VPC に新規のサブネットを作成し、IPv6 IPAM プールから`/64`を割り当て

#### AZ-a のサブネットを作成

![](/images/burikaigi2026-aws-ipv6-study/032002-create-ula-subnet-1.png)

#### AZ-b のサブネットを作成

![](/images/burikaigi2026-aws-ipv6-study/032003-create-ula-subnet-2.png)

### 同サブネット用にルートテーブルを作成

#### AZ-a のルートテーブルを作成

![](/images/burikaigi2026-aws-ipv6-study/032004-create-rt-for-ula-subnet-1.png)
![](/images/burikaigi2026-aws-ipv6-study/032005-create-rt-for-ula-subnet-2.png)

#### AZ-b のルートテーブルを作成

![](/images/burikaigi2026-aws-ipv6-study/032006-create-rt-for-ula-subnet-3.png)
![](/images/burikaigi2026-aws-ipv6-study/032007-create-rt-for-ula-subnet-4.png)

### AMI から EC2（Web Server 用）を作成・新規追加サブネットに配置

#### AZ-a の EC2（Web Server 用）インスタンスを起動

![](/images/burikaigi2026-aws-ipv6-study/033001-create-server-ipv6only-1.png)
![](/images/burikaigi2026-aws-ipv6-study/033002-create-server-ipv6only-2.png)
![](/images/burikaigi2026-aws-ipv6-study/033003-create-server-ipv6only-3.png)
![](/images/burikaigi2026-aws-ipv6-study/033004-create-server-ipv6only-4.png)
![](/images/burikaigi2026-aws-ipv6-study/033005-create-server-ipv6only-5.png)

#### AZ-b の EC2（Web Server 用）インスタンスを起動

![](/images/burikaigi2026-aws-ipv6-study/033006-create-server-ipv6only-6.png)
![](/images/burikaigi2026-aws-ipv6-study/033007-create-server-ipv6only-7.png)
![](/images/burikaigi2026-aws-ipv6-study/033008-create-server-ipv6only-8.png)
![](/images/burikaigi2026-aws-ipv6-study/033009-create-server-ipv6only-9.png)

### IPv6（ULA）サブネット用のターゲットグループを作成

![](/images/burikaigi2026-aws-ipv6-study/034001-create-tg-ipv6-1.png)
![](/images/burikaigi2026-aws-ipv6-study/034002-create-tg-ipv6-2.png)
![](/images/burikaigi2026-aws-ipv6-study/034003-create-tg-ipv6-3.png)
![](/images/burikaigi2026-aws-ipv6-study/034004-create-tg-ipv6-4.png)

### ALB のターゲットグループを IPv4 用から IPv6（ULA）用に切り替え

![](/images/burikaigi2026-aws-ipv6-study/035001-modify-alb-tg-ipv6.png)

### `curl`コマンドまたは IPv6 アドレスが使えるブラウザから接続テスト

※ブラウザからの接続テスト実行例

![](/images/burikaigi2026-aws-ipv6-study/036001-check-alb-tg-ipv6.png)

## 構成に関する補足説明

### IPv6 ユニークローカルユニキャストアドレス（ULA）とは？

IPv4 プライベートアドレスのように、組織内で利用しインターネットアクセスに使わないアドレスです。

`fc00::/7`の範囲が ULA 用に確保されていますが、現在は`fd00::/8`の範囲が実際に使われています。

### ULA と IPv4 プライベートアドレスとの違い

IPv4 プライベートアドレスは NAT でグローバルアドレスに変換して使うことが可能で、AWS でもプライベートサブネットから NAT ゲートウェイを経由してインターネットアクセスが可能です。

すでに何度か触れていますが、IPv6 では GUA と ULA の NAT は非推奨で、AWS の NAT ゲートウェイでもサポートしていません。つまり、何らかのプロキシを立てるなどしないと IPv6（ULA）専用サブネットからインターネットにアクセスすることはできません。

### ULA を使うときに問題になりそうなこと

「インターネットにアクセスできない」ことはセキュリティ上のメリットになりうるのですが、一方で制約がある分利用が大変でもあります。

インターネット上の API などには当然直接アクセスできませんし、RPM や APT などのリポジトリにもアクセスできません。

先に示した手順（概要）の最初で EC2 の AMI を使ってインスタンスを作成したのも、この「RPM リポジトリにアクセスできない」ことが理由です。

初期状態の構築では、起動（作成）時のユーザーデータで nginx と PHP を`dnf install`しているのですが、これは IPv6（ULA）専用サブネットではリポジトリにアクセスできないので正しく実行できません。

そのため、ULA だけをサブネットに割り当てるのではなく、GUA も一緒に割り当てて使う（組織内の通信にのみ ULA を使う）ことになりそうです。

:::message
[こちらのブログ](https://aws.amazon.com/jp/blogs/news/design-and-build-ipv6-internet-inspection-architectures-on-aws/)で言及されているとおり、AWS のサービス（GWLB・ELB・Network Firewall など）ではなく NAT66 または NPTv6（[RFC 6296](https://datatracker.ietf.org/doc/html/rfc6296)）をサポートしているサードパーティ検査アプライアンスを使う場合は、IPv6（ULA）専用サブネットからインターネットにアクセス可能になります。

- https://aws.amazon.com/jp/blogs/news/design-and-build-ipv6-internet-inspection-architectures-on-aws/

:::

### リンクローカルユニキャストアドレス（LLA）との違い

「組織内の通信用」といえば ULA 以外にもリンクローカルユニキャストアドレス（LLA）があります。

IPv4 のリンクローカルアドレスに相当しますが、ULA が「組織内であればルーターで中継して別サブネットと通信できる」のに対し、LLA はレイヤ 2 におけるセグメントの範囲内でのみ通信が可能で、ルーター超えはできません。

AWS では特に意識して使うことはないので、LLA についてはこの程度の説明にとどめておきます。

## このチャプターのまとめ

- AWS の VPC には GUA 以外にユニークローカルユニキャストアドレス（ULA）も割り当てることができる
- ULA を割り当てるときは IP Address Manager（IPAM）を使う
- ULA のみの IPv6 専用サブネットを作ると、セキュリティ面でメリットはあるが、一方で使い勝手が悪くなるので注意
  - 結果的に ULA 単独ではなく GUA と併用することになりそう
