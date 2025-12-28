---
title: "IPv4 環境の作成（初期状態）"
---

IPv6 対応を進める前に、まずは IPv4 環境を作成します。

![](/images/burikaigi2026-aws-ipv6-study/IPv6-01-v4only.drawio.png)

- サーバー用 VPC および関連リソース
- クライアント用 VPC および関連リソース

の 2 種類を作成します。

なお、この資料では大阪リージョン（ap-northeast-3）で作成する場合を想定しています。東京リージョン（ap-northeast-1）で作成する場合は、AZ-b・AZ-c をそれぞれ AZ-c・AZ-d に読み替えてください。

:::message
必要なリソース・要素がわかりやすいように、この資料では（動作確認を除き）すべてマネジメントコンソールで操作していきます。
:::

## サーバー用 VPC および関連リソース作成

### サーバー用 VPC を作成

- **VPC** メニューから

![](/images/burikaigi2026-aws-ipv6-study/001001-ipv4-to-dualstack-create-vpc-1.png)

- 名前タグの自動生成 : （チェックが入った状態のまま）`sv-ipv4-to-dualstack-vpc`
- IPv4 CIDR ブロック : `10.0.0.0/16`

![](/images/burikaigi2026-aws-ipv6-study/001002-ipv4-to-dualstack-create-vpc-2.png)

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

**「インスタンスを起動」** をクリック

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

**「インスタンスを起動」** をクリック

### ALB を作成

### ブラウザからアクセスして動作確認

## クライアント用 VPC および関連リソース作成

### クライアント用 VPC を作成

### VPC エンドポイント（EC2 インスタンス接続用）を作成

### EC2 インスタンスを起動（AZ-a）

### `curl`コマンドで接続テスト
