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

---

**【参考資料】**

- [Internet Number Resource Status Report As of 30 September 2025](https://www.nro.net/wp-content/uploads/NRO-Number-Resource-Status-Report-Q3-2025-FINAL.pdf)（[NRO RIR Statistics](https://www.nro.net/about/rirs/statistics/)）
- [IP addresses through 2024](https://blog.apnic.net/2025/01/13/ip-addresses-through-2024/)（[APNIC Blog](https://blog.apnic.net/)）
- [AWS IPv6 Hands On (japanese)](https://catalog.us-east-1.prod.workshops.aws/workshops/025ae486-39d3-4de4-a12b-049970983d18/ja-JP)
  - ほかに EKS などをカバーする[英語版ワークショップ（Get hands-on with IPv6）](https://catalog.workshops.aws/ipv6-on-aws/en-US)もあります
