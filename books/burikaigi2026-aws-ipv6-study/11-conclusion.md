---
title: "おわりに"
---

AWS での IPv6 利用について、VPC の構成を変えながらここまで見てきましたが、「意外と簡単！」と感じられた方が多かったのではないでしょうか？

おそらく「IPv6 同士の NAT がない」点以外についてはすんなり受け入れられたのではないかと思います。

ただし AWS のサービスでは「生身の IPv6 の難しいところ」をうまく隠蔽している面もあります。

例えば、途中で触れたホストへの IPv6 アドレスの割り当て方は IPv4 と比べて複雑ですし、

- 利用可能なルーターの探索
- 近隣ホストの探索（ARP の代わりに ICMPv6 を使う）

などなど、AWS が隠蔽している部分にはほかにもいくつか難しいポイントが含まれています。

:::message
難しいからこそ「いい感じに隠蔽した」のでしょうけれど。
:::

特にオンプレ環境との接続やハイブリッドな構成が必要になった場合にそのあたりの知識も必要になりそうですが、まずは当セッションを「入口」として活用していただければ幸いです。

## 環境の削除

試しに作った環境を放置してしまうと料金が発生し続けるので、不要になったら削除してください。

- 各 EC2 インスタンスを削除
- （オプションまで進めた場合は）AMI と EBS スナップショットを削除
- ALB（ロードバランサー）を削除
- 同・ターゲットグループを削除
- NAT ゲートウェイを削除
- Elastic IP アドレスを削除
- VPC を削除
  - 削除時にエラーが出たときは画面の指示にしたがって削除の妨げになっているリソースを先に削除
- （オプションまで進めた場合は）IP Address Manager（IPAM）とそのプールを（階層的に）削除

---

**【参考資料】**

- [Internet Number Resource Status Report As of 30 September 2025](https://www.nro.net/wp-content/uploads/NRO-Number-Resource-Status-Report-Q3-2025-FINAL.pdf)（[NRO RIR Statistics](https://www.nro.net/about/rirs/statistics/)）
- [IP addresses through 2024](https://blog.apnic.net/2025/01/13/ip-addresses-through-2024/)（[APNIC Blog](https://blog.apnic.net/)）
- [AWS IPv6 Hands On (japanese)](https://catalog.us-east-1.prod.workshops.aws/workshops/025ae486-39d3-4de4-a12b-049970983d18/ja-JP)
  - ほかに EKS などをカバーする[英語版ワークショップ（Get hands-on with IPv6）](https://catalog.workshops.aws/ipv6-on-aws/en-US)もあります
