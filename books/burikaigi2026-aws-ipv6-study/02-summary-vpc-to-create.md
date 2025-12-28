---
title: "セッションで作成する VPC について"
---

以下のように作成・変更していきます。

## 全体の流れ

1. （初期状態）IPv4 VPC を サーバー用（Web）・クライアント用の 2 種類作成し、EC2（Web サーバー・クライアント）などの必要なリソースを配置します
   ![](/images/burikaigi2026-aws-ipv6-study/IPv6-01-v4only.drawio.png)
2. サーバー用 VPC のパブリックサブネットに IPv6（GUA）アドレスブロックを割り当てるとともに、ALB をデュアルスタックに変更します
   ![](/images/burikaigi2026-aws-ipv6-study/IPv6-02-server-ds.drawio.png)
3. クライアント用 VPC のプライベートサブネットに IPv6（GUA）アドレスブロックを割り当てます
   ![](/images/burikaigi2026-aws-ipv6-study/IPv6-03-client-ds.drawio.png)
4. クライアント用 VPC に IPv6（GUA）アドレスブロックのみを持つプライベートサブネットを作成するとともに EC2（クライアント）を配置し、NAT64/DNS64 を使ってインターネットにアクセスします
   ![](/images/burikaigi2026-aws-ipv6-study/IPv6-04-client-nat64-dns64.drawio.png)
5. （オプション）サーバー用 VPC に IPv6（ULA）アドレスブロックのみを持つプライベートサブネットを作成し、EC2（Web サーバー）を配置して ALB のターゲットグループを IPv6（ULA）プライベートサブネット側に変更します
   ![](/images/burikaigi2026-aws-ipv6-study/IPv6-05-server-tg-ipv6-ula-only.drawio.png)
