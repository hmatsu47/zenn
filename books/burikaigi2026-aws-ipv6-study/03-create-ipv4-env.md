---
title: "IPv4 環境の作成（初期状態）"
---

IPv6 対応を進める前に、まずは IPv4 環境を作成します。

![](/images/burikaigi2026-aws-ipv6-study/IPv6-01-v4only.drawio.png)

- サーバー用 VPC および関連リソース
  - VPC・ALB・EC2（Web サーバー）など
- クライアント用 VPC および関連リソース
  - VPC・VPC エンドポイント・EC2（クライアント）など

の 2 種類を作成します。

なお、この資料では大阪リージョン（ap-northeast-3）で作成する場合を想定しています。東京リージョン（ap-northeast-1）で作成する場合は、AZ-b・AZ-c をそれぞれ AZ-c・AZ-d に読み替えてください。

:::message
必要なリソース・要素がわかりやすいように、この資料では（動作確認を除き）すべてマネジメントコンソールで操作していきます。
:::

## サーバー用 VPC および関連リソース作成

前掲の図のうち、この部分を作成します。

![](/images/burikaigi2026-aws-ipv6-study/IPv6-01-sv.png)

- VPC・パブリックサブネット 2 個・プライベートサブネット 2 個・インターネットゲートウェイ（ほか関連リソース）
- ALB・ターゲットグループ（同上）
- EC2（Web サーバー : nginx + PHP で`HTTP_X_FORWARDED_FOR`を返すもの）2 個（同上）

インターネットから ALB に HTTP アクセスすると、Web サーバーがアクセス元 IP アドレスを示す`HTTP_X_FORWARDED_FOR`を返します。

### サーバー用 VPC を作成

- **VPC** メニューから

![](/images/burikaigi2026-aws-ipv6-study/001001-ipv4-to-dualstack-create-vpc-1.png)

- 名前タグの自動生成 : （チェックが入った状態のまま）`sv-ipv4-to-dualstack-vpc`
- IPv4 CIDR ブロック : `10.0.0.0/16`

![](/images/burikaigi2026-aws-ipv6-study/001002-ipv4-to-dualstack-create-vpc-2.png)

- VPC エンドポイント : 今回のシナリオでは S3 ゲートウェイを使わないので「なし」も可
- **「VPC を作成」** をクリック

### EC2 インスタンスを起動（AZ-a）

- **EC2** メニューから

![](/images/burikaigi2026-aws-ipv6-study/002001-create-server-1.png)

- 名前 : `web-server-001a`

![](/images/burikaigi2026-aws-ipv6-study/002002-create-server-2.png)

- キーペア名 : キーペアなしで続行

![](/images/burikaigi2026-aws-ipv6-study/002003-create-server-3.png)

- ネットワーク設定 : **「編集」** をクリック

![](/images/burikaigi2026-aws-ipv6-study/002004-create-server-4.png)

- VPC : 先ほど作成したサーバー用 VPC
- サブネット : AZ-a パブリックサブネット
- セキュリティグループ名 : `web-server-001a`
- 説明 : `web-server-001a`
- インバウンドセキュリティグループのルール : SSH を削除してから HTTP（ソース : `0.0.0.0/0`）を追加

![](/images/burikaigi2026-aws-ipv6-study/002005-create-server-5.png)

- 「高度な詳細」セクションをクリックして展開

![](/images/burikaigi2026-aws-ipv6-study/002006-create-server-6.png)

- ユーザーデータ : 以下の内容

```sh:
#!/bin/bash
dnf install -y nginx php php-fpm
systemctl start nginx
systemctl enable nginx
systemctl start php-fpm
systemctl enable php-fpm
echo "<html><h1>HTTP_X_FORWARDED_FOR: ""<?php echo \$_SERVER['HTTP_X_FORWARDED_FOR']; ?>""</html></h1>" >> /usr/share/nginx/html/index2.php
```

:::message
こちらは AWS の IPv6 ハンズオン（日本語版）のものを利用させていただきました。
https://catalog.us-east-1.prod.workshops.aws/workshops/025ae486-39d3-4de4-a12b-049970983d18/ja-JP
:::

![](/images/burikaigi2026-aws-ipv6-study/002007-create-server-7.png)

- **「インスタンスを起動」** をクリック

起動するのを待ちます。

### EC2 インスタンスを起動（AZ-b）

- AZ-a 用のインスタンス画面を開いて

![](/images/burikaigi2026-aws-ipv6-study/002008-create-server-8.png)

- 「アクション」→「イメージとテンプレート」→「同様のものを起動」

![](/images/burikaigi2026-aws-ipv6-study/002009-create-server-9.png)

- 名前 : `web-server-001b`
- キーペア名 : キーペアなしで続行

![](/images/burikaigi2026-aws-ipv6-study/002010-create-server-10.png)

- サブネット : AZ-b パブリックサブネット

![](/images/burikaigi2026-aws-ipv6-study/002011-create-server-11.png)

- **「インスタンスを起動」** をクリック

### ALB を作成

- **EC2** メニュー → ロードバランサーから

- 「ロードバランサーの作成」の右側「△」をクリック →「Application Load Balancer を作成」

![](/images/burikaigi2026-aws-ipv6-study/003001-create-alb-1.png)

- ロードバランサー名 : `ipv4-to-dualstack`
- VPC : 先ほど作成したサーバー用 VPC
- アベイラビリティゾーンとサブネット : AZ-a・AZ-b にチェックし、それぞれに属するサブネットを選択

- 「新しいセキュリティグループを作成」リンクからセキュリティグループを作成

![](/images/burikaigi2026-aws-ipv6-study/003002-create-alb-sg.png)

- セキュリティグループ名 : `ipv4-alb`
- 説明 : `ipv4-alb`
- インバウンドルール : 「ルールを追加」で「HTTP」・ソース「Anywhere-IPv4」を追加
- **「セキュリティグループを作成」** をクリック

![](/images/burikaigi2026-aws-ipv6-study/003003-create-alb-2.png)

- セキュリティグループ : 右側のリロードボタンをクリックした後「ipv4-alb」を選択
- 「ターゲットグループを作成」リンクをクリック

![](/images/burikaigi2026-aws-ipv6-study/003004-create-alb-tg-1.png)

- ターゲットグループ名 : `ipv4-server`
- VPC : 先ほど作成したサーバー用 VPC

![](/images/burikaigi2026-aws-ipv6-study/003005-create-alb-tg-2.png)

- **「次へ」** をクリック

![](/images/burikaigi2026-aws-ipv6-study/003006-create-alb-tg-3.png)

- 2 つの EC2 インスタンスにチェックを入れて **「保留中として以下を含める」** をクリック
- ターゲットに EC2 インスタンスが登録されたことを確認
- **「次へ」** をクリック

![](/images/burikaigi2026-aws-ipv6-study/003007-create-alb-tg-4.png)

- **「ターゲットグループの作成」** をクリック

![](/images/burikaigi2026-aws-ipv6-study/003008-create-alb-3.png)

- ターゲットグループ : 右側のリロードボタンをクリックした後「ipv4-server」を選択

![](/images/burikaigi2026-aws-ipv6-study/003009-create-alb-4.png)

- **「ロードバランサーの作成」** をクリック

- 作成されたばかりのターゲットグループ「ipv4-server」画面から

![](/images/burikaigi2026-aws-ipv6-study/003010-init-alb-tg-1.png)

![](/images/burikaigi2026-aws-ipv6-study/003011-init-alb-tg-2.png)

- 「ヘルスチェック」が「Healthy」になるのを待つ

### ブラウザからアクセスして動作確認

- ALB「ipv4-to-dualstack」画面から

![](/images/burikaigi2026-aws-ipv6-study/003012-check-alb-1.png)

- DNS 名に記されたパスをコピー

![](/images/burikaigi2026-aws-ipv6-study/003013-check-alb-2.png)

- 「http://【先ほどコピーしたパス】/index2.php」をブラウザで開いたとき、アクセス元の IPv4 アドレスが表示されれば OK

以上でサーバー環境（IPv4 のみサポート）の構築は完了です。

## クライアント用 VPC および関連リソース作成

前掲の図のうち、この部分を作成します。

![](/images/burikaigi2026-aws-ipv6-study/IPv6-01-cl.png)

- VPC・プライベートサブネット 2 個・インターネットゲートウェイ・NAT ゲートウェイ（ほか関連リソース）
- VPC エンドポイント（EC2 インスタンス接続用／同上）
- EC2（クライアント : `curl`コマンド実行用／同上）

マネジメントコンソールから EC2 に接続して`curl`コマンドを実行し、サーバー環境やインターネットへの接続テストを行います。

### クライアント用 VPC を作成

- **VPC** メニューから

![](/images/burikaigi2026-aws-ipv6-study/010001-cl-ipv4-to-dualstack-create-vpc-1.png)

- 名前タグの自動生成 : （チェックが入った状態のまま）`cl-ipv4-to-dualstack-vpc`
- IPv4 CIDR ブロック : `10.1.0.0/16`
- パブリックサブネットの数 : 0
- NAT ゲートウェイ : リージョナル - 新規

最近作成できるようになったリージョナル NAT ゲートウェイを作成することで、従来は必要だった NAT ゲートウェイ用のパブリックサブネットが不要になります。

![](/images/burikaigi2026-aws-ipv6-study/010002-cl-ipv4-to-dualstack-create-vpc-2.png)

- VPC エンドポイント : なし
- **「VPC を作成」** をクリック

### VPC エンドポイント（EC2 インスタンス接続用）を作成

- **VPC** メニュー → エンドポイントから

![](/images/burikaigi2026-aws-ipv6-study/011001-cl-create-ep-1.png)

- 名前タグ : `cl-ec2-connect`
- タイプ : EC2 インスタンス接続エンドポイント
- VPC : 先ほど作成したクライアント用 VPC
- セキュリティグループ : 「default」のみ選択されているのを確認（選択されていなければ「default」のみ選択）
- サブネット : AZ-a プライベートサブネット
- IP アドレスタイプ : 「IPv4」が選択されているのを確認（選択されていなければ「IPv4」を選択）

![](/images/burikaigi2026-aws-ipv6-study/011002-cl-create-ep-2.png)

- **「エンドポイントを作成」** をクリック

### EC2 インスタンスを起動（AZ-a）

- **EC2** メニューから

![](/images/burikaigi2026-aws-ipv6-study/012001-cl-create-client-1.png)

- 名前 : `client-001a`

- 「新しいキーペアの作成」リンクをクリック

![](/images/burikaigi2026-aws-ipv6-study/012002-cl-create-client-2.png)

- キーペア名 : `client-001`
- キーペアのタイプ : ED25519
- **「キーペアを作成」** をクリック

![](/images/burikaigi2026-aws-ipv6-study/012003-cl-create-client-3.png)

- キーペア名 : リロードボタンをクリック後「client-001」を選択
- VPC : 先ほど作成したクライアント用 VPC
- サブネット : AZ-a プライベートサブネット
- セキュリティグループ名 : `client-sg`

![](/images/burikaigi2026-aws-ipv6-study/012005-cl-create-client-5.png)

- **「インスタンスを起動」** をクリック

### `curl`コマンドで接続テスト

EC2（クライアント）に接続し、サーバー環境の ALB に`curl`コマンドでアクセスしてみます。

- 作成したクライアント用 EC2 の画面から **「接続」** をクリック

![](/images/burikaigi2026-aws-ipv6-study/013001-cl-connect-1.png)

- 接続タイプ : プライベート IP を使用して接続
- EC2 Instance Connect エンドポイント : 「cl-ec2-connect」に相当するエンドポイントを選択
- **「接続」** をクリック

別のウィンドウを開いて、ALB「ipv4-to-dualstack」画面を確認します。

![](/images/burikaigi2026-aws-ipv6-study/013002-cl-connect-2.png)

- DNS 名に記されたパスをコピー

元の画面で`curl`コマンドを使って ALB に IPv4 接続してみます。

```sh:
$ curl -v4 http://【先ほどコピーしたパス】/index2.php
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
< Date: Thu, 25 Dec 2025 15:44:37 GMT
< Content-Type: text/html; charset=UTF-8
< Transfer-Encoding: chunked
< Connection: keep-alive
< Server: nginx/1.28.0
<
<html><h1>HTTP_X_FORWARDED_FOR: 56.XX.XX.235</html></h1>
* Connection #0 to host ipv4-to-dualstack-XXXXXXXX.ap-northeast-3.elb.amazonaws.com left intact
```

「HTTP_X_FORWARDED_FOR」に（クライアント環境 NAT ゲートウェイの）IPv4 アドレスが表示されれば OK です。

以上でクライアント環境（IPv4 のみサポート）の構築は完了です。
