---
title: "クライアント環境の IPv6 対応（NAT64/DNS64 編）"
---

クライアント環境の IPv6 対応として、もう一つの「NAT64/DNS64 を使う」方法についても実施します。

なお、このセッションでは NAT64/DNS64 の「効果」を確かめるため

- IPv6 専用サブネットを作成
  - クライアントから接続テスト
- NAT64/DNS64 が利用できるよう設定
  - 再度クライアントから接続テストして結果を比較

という 2 段階の作業を行います。

- **IPv6 専用サブネットを作成**
  - プライベートサブネットを追加し、IPv6 CIDR ブロックのみを割り当て
    - IPv4 は割り当てない
  - 新設プライベートサブネットのルートテーブルを作成
    - 新設プライベートサブネットにアタッチ
  - 新設プライベートサブネットのルートテーブルに IPv6 のデフォルトルートを追加
  - 既存の VPC エンドポイント（EC2 インスタンス接続用）を削除
    - クォーターの上限緩和申請を行っている場合は不要
  - 新設プライベートサブネット用の VPC エンドポイント（EC2 インスタンス接続用）を作成
  - EC2（クライアント）インスタンスを起動
  - `curl`コマンドで接続テスト
- **NAT64/DNS64 が利用できるよう設定**
  - 新設プライベートサブネットのルートテーブルに NAT64 用のルートを追加
  - 新設プライベートサブネットの DNS64 を有効化
  - `curl`コマンドで接続テスト

これらの作業が完了すると、図のような構成になります。

![](/images/burikaigi2026-aws-ipv6-study/IPv6-04-client-nat64-dns64.drawio.png)

作業対象となるクライアント環境部分を拡大すると以下のとおりです。

![](/images/burikaigi2026-aws-ipv6-study/IPv6-04.png)

## AWS 環境上での作業 ① IPv6 専用サブネットを作成

### プライベートサブネットを追加し、IPv6 CIDR ブロックのみを割り当て

- **「VPC」** メニュー → サブネットから

- **「サブネットを作成」** をクリック

![](/images/burikaigi2026-aws-ipv6-study/020001-cl-add-subnet-ipv6only.png)

- VPC ID : 「cl-ipv4-to-dualstack-vpc」の VPC ID
- サブネット名 : `cl-ipv4-to-dualstack-subnet-private3-ap-northeast-3c`
- アベイラビリティゾーン : AZ-c
- IPv4 CIDR ブロック : IPv4 CIDR がありません
- IPv6 CIDR ブロック : 手動入力
- IPv6 VPC CIDR ブロック : VPC の CIDR ブロックの 3 番目の`/64`（プレフィックスの最後が「`02`」）を選択
  - 下向き「>」を 2 回クリック後、右向き「>」を 2 回クリック
- **「サブネットを作成」** をクリック

### 新設プライベートサブネットのルートテーブルを作成

- **「VPC」** メニュー → ルートテーブルから

- **「ルートテーブルを作成」** をクリック

![](/images/burikaigi2026-aws-ipv6-study/020002-cl-add-rt-ipv6only.png)

- 名前 : `cl-ipv4-to-dualstack-rtb-private3-ap-northeast-3c`
- VPC : cl-ipv4-to-dualstack-vpc
- **「ルートテーブルを作成」** をクリック

#### 新設プライベートサブネットにアタッチ

- 作成したルートテーブル「cl-ipv4-to-dualstack-rtb-private3-ap-northeast-3c」画面 → **「アクション」** → **「サブネットの関連付けを編集」** から

![](/images/burikaigi2026-aws-ipv6-study/020003-cl-attach-rt-to-subnet-ipv6only.png)

- 利用可能なサブネット : 「cl-ipv4-to-dualstack-subnet-private3-ap-northeast-3c」にチェック
- **「関連付けを保存」** をクリック

### 新設プライベートサブネットのルートテーブルに IPv6 のデフォルトルートを追加

- クライアント用 VPC のプライベートルートテーブル画面（AZ-c 側） → **「ルートを編集」** から

![](/images/burikaigi2026-aws-ipv6-study/020004-cl-modify-rt-ipv6only.png)

- **「ルートを追加」** で以下のルートを追加
  - 送信先 : `::/0`
  - ターゲット : Egress Only インターネットゲートウェイ（既存の`eigw-XXXXXXXX`）
- **「変更を保存」** をクリック

### 既存の VPC エンドポイント（EC2 インスタンス接続用）を削除

デフォルトでは 1 つの VPC に対して EC2 インスタンス接続用 VPC エンドポイントは 1 つしか作成できないので、先ほど使ったものを削除します。

:::message
クォーターの上限緩和申請により 2 個以上作成できる状態になっていれば、削除をスキップしても構いません。
:::

- **VPC** メニュー → エンドポイントから「cl-ec2-connect」をチェックして

![](/images/burikaigi2026-aws-ipv6-study/021001-cl-remove-ep-1.png)

- **「アクション」** → **「VPC エンドポイントの削除」** をクリック

![](/images/burikaigi2026-aws-ipv6-study/021002-cl-remove-ep-2.png)

- フィールドに「削除」と入力
- **「削除」** をクリック

### 新設プライベートサブネット用の VPC エンドポイント（EC2 インスタンス接続用）を作成

- **VPC** メニュー → エンドポイントから

- **「エンドポイントを作成」** をクリック

![](/images/burikaigi2026-aws-ipv6-study/011001-cl-create-ep-1.png)

- 名前タグ : `cl-ec2-connect-ipv6`
- タイプ : EC2 インスタンス接続エンドポイント
- VPC : cl-ipv4-to-dualstack-vpc
- セキュリティグループ : 「default」のみ選択されているのを確認（選択されていなければ「default」のみ選択）
- サブネット : AZ-c プライベートサブネット
- IP アドレスタイプ : 「IPv6」が選択されているのを確認（選択されていなければ「IPv6」を選択）

![](/images/burikaigi2026-aws-ipv6-study/021004-cl-create-ep-ipv6-2.png)

- **「エンドポイントを作成」** をクリック

### EC2（クライアント）インスタンスを起動

- **EC2** メニューから

- **「インスタンスを起動」** をクリック

![](/images/burikaigi2026-aws-ipv6-study/022001-cl-create-client-1.png)

- 名前 : `client-001c-ipv6only`

- 「新しいキーペアの作成」リンクをクリック

![](/images/burikaigi2026-aws-ipv6-study/022002-cl-create-client-2.png)

- キーペア名 : `client-001`
- VPC : cl-ipv4-to-dualstack-vpc
- サブネット : AZ-c プライベートサブネット
- ファイアーウォール（セキュリティグループ） : 「セキュリティグループを作成」になっていなければ「セキュリティグループを作成」を選択
- セキュリティグループ名 : `client-001-ipv6-sg`
- 説明 : `client-sg`
- **「高度なネットワーク設定」** セクションをクリックして開く

![](/images/burikaigi2026-aws-ipv6-study/022003-cl-create-client-3.png)

- プライマリ IPv6 IP を割り当てる : はい

![](/images/burikaigi2026-aws-ipv6-study/022004-cl-create-client-4.png)

- **「インスタンスを起動」** をクリック

### `curl`コマンドで接続テスト

EC2（クライアント）に接続し、サーバー環境の ALB に`curl`コマンドでアクセスしてみます。

- 作成したクライアント用 EC2「client-001c-ipv6only」画面から **「接続」** をクリック

![](/images/burikaigi2026-aws-ipv6-study/023001-cl-connect-6.png)

- 接続タイプ : プライベート IP を使用して接続
- EC2 Instance Connect エンドポイント : 「cl-ec2-connect-ipv6」に相当するエンドポイントを選択
- **「接続」** をクリック

ALB に対して`curl`コマンドでアクセスしてみます（ALB の URL については以前と同様に ALB の画面からコピーしてください）。

まずは IPv6 でアクセスしてみます。

```sh:
$ curl -v6 http://ipv4-to-dualstack-XXXXXXXX.ap-northeast-3.elb.amazonaws.com/index2.php
* Host ipv4-to-dualstack-XXXXXXXX.ap-northeast-3.elb.amazonaws.com:80 was resolved.
* IPv6: 2406:XX:XX:6400:16de:a21e:34f6:e1ab
* IPv4: (none)
*   Trying [2406:XX:XX:6400:16de:a21e:34f6:e1ab]:80...
* Connected to ipv4-to-dualstack-XXXXXXXX.ap-northeast-3.elb.amazonaws.com (2406:XX:XX:6400:16de:a21e:34f6:e1ab) port 80
* using HTTP/1.x
> GET /index2.php HTTP/1.1
> Host: ipv4-to-dualstack-XXXXXXXX.ap-northeast-3.elb.amazonaws.com
> User-Agent: curl/8.11.1
> Accept: */*
>
* Request completely sent off
< HTTP/1.1 200 OK
< Date: Fri, 26 Dec 2025 03:47:16 GMT
< Content-Type: text/html; charset=UTF-8
< Transfer-Encoding: chunked
< Connection: keep-alive
< Server: nginx/1.28.0
<
<html><h1>HTTP_X_FORWARDED_FOR: 2406:XX:XX:4b02:7c0a:291e:caa4:65a8</html></h1>
* Connection #0 to host ipv4-to-dualstack-XXXXXXXX.ap-northeast-3.elb.amazonaws.com left intact
```

アクセスできました。

IPv4 ではどうでしょうか？

```sh:
$ curl -v4 http://ipv4-to-dualstack-XXXXXXXX.ap-northeast-3.elb.amazonaws.com/index2.php
* Host ipv4-to-dualstack-XXXXXXXX.ap-northeast-3.elb.amazonaws.com:80 was resolved.
* IPv6: (none)
* IPv4: 15.XX.XX.67
*   Trying 15.XX.XX.67:80...
* Immediate connect fail for 15.XX.XX.67: Network is unreachable
* Failed to connect to ipv4-to-dualstack-XXXXXXXX.ap-northeast-3.elb.amazonaws.com port 80 after 4 ms: Could not connect to server
* closing connection #0
curl: (7) Failed to connect to ipv4-to-dualstack-XXXXXXXX.ap-northeast-3.elb.amazonaws.com port 80 after 4 ms: Could not connect to server
```

先に「IPv6 は IPv4 とは互換性がなく、あるホストが IPv6 をサポートしたからといって IPv4 しかサポートしていないホストとは直接通信を行うことができません」と説明したとおり、IPv4 アドレスを持たない EC2 からはアクセスできませんね。

#### GitHub にもアクセスしてみる

ついでに GitHub にもアクセスしてみましょう。

```sh:
$ curl https://github.com/hmatsu47
curl: (7) Failed to connect to github.com port 443 after 1 ms: Could not connect to server
```

IP のバージョンを指定していないので「使えるバージョンでアクセス」を試行する形になるのですが、この EC2 は IPv6 しかサポートしていないので IPv6 でアクセスしようとしています。

しかし 2025 年 12 月現在、GitHub（`github.com`）の Web サイトは IPv6 をサポートしていないので、IPv6 環境からは接続できませんでした。

#### そのほか「IPv6 非対応（未対応）で有名なもの」といえば？

ほかにも、Gmail を除く多くのメール（SMTP）サーバーでは 2025 年 12 月現在 IPv6 アドレスでの受信をサポートしていないようです。

とはいえ「SMTP サーバーのソフトウェア（Postfix など）や SPF・DKIM・DMARC などの送信者認証に関わる仕組みが IPv6 に対応していない」わけではありません。

メール受信にとって重要な「送信側のサーバーは果たして信用できる相手なのか？」を判断するためのいわゆるレピュテーションのシステムが、広大な IPv6 アドレスでは効果的に運用するのが大変、というような IPv6 固有の事情が影響している可能性があります。

## AWS 環境上での作業 ② NAT64/DNS64 が利用できるよう設定

というわけで、「IPv6 専用 VPC から IPv4 のサイトにアクセスする」方法として、NAT64/DNS64 を試してみます。

### 新設プライベートサブネットのルートテーブルに NAT64 用のルートを追加

- クライアント用 VPC のプライベートルートテーブル画面（AZ-c 側） → **「ルートを編集」** から

![](/images/burikaigi2026-aws-ipv6-study/024001-cl-modify-rt-nat64.png)

- **「ルートを追加」** で以下のルートを追加
  - 送信先 : `64:ff9b::/96`
  - ターゲット : NAT ゲートウェイ（既存の`nat-XXXXXXXX`）
- **「変更を保存」** をクリック

### 新設プライベートサブネットの DNS64 を有効化

- クライアント用 VPC のサブネット画面（AZ-c 側） → **「アクション」** → **「サブネットの設定を編集」** から

![](/images/burikaigi2026-aws-ipv6-study/024002-cl-modify-subnet-dns64.png)

- DNS64 を有効化 : チェック
- **「保存」** をクリック

### `curl`コマンドで接続テスト

再度、EC2（クライアント）から`curl`コマンドでアクセスしてみます。

まずは IPv4 指定でサーバー環境の ALB へ。

```sh:
$ curl -v4 http://ipv4-to-dualstack-XXXXXXXX.ap-northeast-3.elb.amazonaws.com/index2.php
* Host ipv4-to-dualstack-XXXXXXXX.ap-northeast-3.elb.amazonaws.com:80 was resolved.
* IPv6: (none)
* IPv4: 15.XX.XX.67, 15.XX.XX.15
*   Trying 15.XX.XX.67:80...
* Immediate connect fail for 15.XX.XX.67: Network is unreachable
*   Trying 15.XX.XX.15:80...
* Immediate connect fail for 15.XX.XX.15: Network is unreachable
* Failed to connect to ipv4-to-dualstack-XXXXXXXX.ap-northeast-3.elb.amazonaws.com port 80 after 1 ms: Could not connect to server
* closing connection #0
curl: (7) Failed to connect to ipv4-to-dualstack-XXXXXXXX.ap-northeast-3.elb.amazonaws.com port 80 after 1 ms: Could not connect to server
```

あれ？うまくいきませんね。

続いて、指定なしで GitHub（`github.com`）にアクセスしてみます。

```sh:
$ curl https://github.com/hmatsu47
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0

<!DOCTYPE html>
<html
  lang="en"

  data-color-mode="auto" data-light-theme="light" data-dark-theme="dark"
  data-a11y-animated-images="system" data-a11y-link-underlines="true"

  >




  <head>
    <meta charset="utf-8">
  <link rel="dns-prefetch" href="https://github.githubassets.com">
  <link rel="dns-prefetch" href="https://avatars.githubusercontent.com">
  <link rel="dns-prefetch" href="https://github-cloud.s3.amazonaws.com">
  <link rel="dns-prefetch" href="https://user-images.githubusercontent.com/">
  <link rel="preconnect" href="https://github.githubassets.com" crossorigin>
  <link rel="preconnect" href="https://avatars.githubusercontent.com">

（中略）

    </div>
    <div id="js-global-screen-reader-notice" class="sr-only mt-n1" aria-live="polite" aria-atomic="true" ></div>
    <div id="js-global-screen-reader-notice-assertive" class="sr-only mt-n1" aria-live="assertive" aria-atomic="true"></div>
  </body>
</html>
```

こちらはうまくいきました。

#### NAT64/DNS64 での通信の流れ

NAT64/DNS64 を使うと、IPv6 しかサポートしないホスト（EC2）でも、以下のような流れでインターネット上の IPv4 ホストにアクセスできます。

1. ホスト（EC2）から DNS リゾルバ（Route 53 Resolver）に名前を引きに行く（正引き）
   1-1. 結果に AAAA レコードが含まれていたら、DNS リゾルバ（Route 53 Resolver）はそのまま IPv6 アドレスを返す
   1-2. ホスト（EC2）は Egress Only インターネットゲートウェイ経由で IPv4 ホストにアクセスする
2. A レコード（のみ）が返ってきたら、DNS リゾルバ（Route 53 Resolver）は IPv4 アドレスを`64:ff9b`で始まる特殊な IPv6 アドレスに変換して返す
3. ホスト（EC2）はルートテーブルに従い NAT ゲートウェイ経由でインターネットにアクセスしに行く
4. NAT Gateway は IPv6 の通信かつ宛先が`64:ff9b`で始まる特殊な IPv6 アドレスであることを検出し、IPv4 アドレスに変換して IPv4 で宛先のホストにアクセス
   4-1. レスポンスは IPv4 から IPv6 に変換してアクセス元ホスト（EC2）に返す

ただし、最初から IPv4 指定で通信を始めた場合、Route 53 Resolver は A レコードをそのまま返すため、IPv4 をサポートしない EC2 では相手先に到達できません。

（NAT64/DNS64 は「送信元 EC2（ホスト）自身は IPv6 で通信する」ことが前提となる仕組み）

先ほどの「ALB 宛ての`curl`コマンド（`-v4`措定）」がこれにあたります。

:::message
「IPv6 しかサポートしていないはずなのに、IPv4 指定の`curl`コマンドでなぜ Route 53 Resolver が応答を返せるの？」と思うかもしれませんが、VPC で IPv4 を有効にしていなくても、IPv4 リンクローカルアドレスはデフォルトで有効な状態になっており、Route 53 Resolver への問い合わせができるようになっています。

**【参考】**

- [Amazon DNS について理解する - Amazon DNS サーバー](https://docs.aws.amazon.com/ja_jp/vpc/latest/userguide/AmazonDNS-concepts.html#AmazonDNS)（Amazon Virtual Private Cloud ユーザーガイド）

> IPv6 専用サブネットでは、"AmazonProvidedDNS" が DHCP オプションセット内のネームサーバーである限り、IPv4 リンクローカルアドレス (169.254.169.253) に引き続きアクセスできます。

:::

一方で、IPv4 に限定せず通信を始めた場合、Route 53 Resolver は AAAA レコードを持たず A レコードのみ持つ相手のアドレスを`64:ff9b`で始まる特殊な IPv6 アドレスに変換して返すため、NAT ゲートウェイ経由でのアクセスが可能になります。

GitHub（`github.com`）の例はこちらにあたります。

## このチャプターのまとめ

- IPv6 専用サブネットからインターネット上の IPv4 ホストにアクセスするには NAT64/DNS64 が利用可能な設定を行う
  - ルートテーブル・サブネット
- DNS64 では A レコードの IPv4 アドレスを`64:ff9b`で始まる特殊な IPv6 アドレスに変換して返す
- NAT64 では`64:ff9b`で始まる特殊な IPv6 アドレスを宛先とする通信を IPv4 に変換して宛先に流す
- IPv6 専用の VPC サブネットに置かれた EC2 でも IPv4 リンクローカルアドレス（IMDS・リゾルバ用）は有効になっている
