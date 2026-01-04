---
title: "IP アドレス（グローバルアドレス）利用の現況について"
---

IPv6 について学ぶ前に、まずは IP アドレス、特に全世界で一意に割り当てられるグローバルアドレスの現況について簡単に触れます。

## IPv4 アドレスの現況

### 新規割り当て用のアドレスブロックはほぼ枯渇

2011 年 2 月 3 日、IP アドレスをグローバルに管理する ICANN（IANA）から各地域の RIR（地域インターネットレジストリ：Regional Internet Registry）への最後のアドレスブロック割り当てが行われ、新規に割り当て可能な IPv4 アドレスブロックがなくなりました。

各地域の RIR（ARIN / RIPE NCC / APNIC / LACNIC / AFRINIC）でも在庫の消費が進み、[2025 年 9 月 30 日時点の NRO（Number Resource Organization）の報告資料](https://www.nro.net/wp-content/uploads/NRO-Number-Resource-Status-Report-Q3-2025-FINAL.pdf)（P.4）によると、「利用可能（Available）」なアドレスは`/8`換算で

- AFRINIC（アフリカ地域を管轄）が 0.05（IP アドレス約 84 万個分）
- APNIC（アジア太平洋地域を管轄）が 0.19（同・約 319 万個分）

となっています（ARIN / RIPE NCC / LACNIC は 0）。

![](/images/burikaigi2026-aws-ipv6-study/ipv4-rir.jpg)

発行された IPv4 アドレス（`/8`換算）の推移を見ると、（2025 年は第 3 四半期までの集計値なので注意が必要ですが）徐々に少なくなってきているのがわかります。

![](/images/burikaigi2026-aws-ipv6-study/ipv4-rir-issued.jpg)

（出典はいずれも同資料：https://www.nro.net/wp-content/uploads/NRO-Number-Resource-Status-Report-Q3-2025-FINAL.pdf ）

:::message
他に「予約済み（Reserved）」アドレスがありますが、用途限定であり（利用可能（Available）なアドレスに転換されない限り）一般向けの割り当ては行われないのでそちらは省略します。
:::

### 現在は新規割り当て＜移転

現在は、RIR およびその配下のレジストリからの新規割り当てよりも、新たな IPv4 アドレスブロックの割り当てを求める組織が「割り当て済みだが不使用（未使用）」のアドレスブロックを持つ組織からの「移転」を受けることによって IPv4 アドレスブロックを確保するケースが多くなっています（[前述資料](https://www.nro.net/wp-content/uploads/NRO-Number-Resource-Status-Report-Q3-2025-FINAL.pdf)の P.6 〜 9）。

![](/images/burikaigi2026-aws-ipv6-study/intra-rir-ivp4-trans-addr.jpg)

当初、移転は同じ RIR 内に限られていましたが、現在は RIR 間移転も行われています（[同](https://www.nro.net/wp-content/uploads/NRO-Number-Resource-Status-Report-Q3-2025-FINAL.pdf)・P.9）。

![](/images/burikaigi2026-aws-ipv6-study/inter-rir-ipv4-trans.jpg)

（出典はいずれも同資料：https://www.nro.net/wp-content/uploads/NRO-Number-Resource-Status-Report-Q3-2025-FINAL.pdf ）

ただし総量としては移転も横ばい〜微減傾向ですね。

:::message
数年前には「AWS が IPv4 アドレスブロックを買い漁っている」ことが話題になりました。

**【参考】**

- [Amazon が保有している IP アドレスだけで 2500 億円以上の資産価値がある（Gigazine）](https://gigazine.net/news/20201022-amazon-ipv4-address/)
- [AWS とか他のパブリッククラウドプロバイダーって、どうやってあんなにたくさんのパブリック IPv4 アドレスを手に入れてるんだろう？（Reddit）](https://www.reddit.com/r/aws/comments/ds06go/where_aws_or_other_public_cloud_provider_gets_so/?tl=ja)

:::

### 「割り当て済みだが不使用」アドレスの状況は？

そうなると「すでに IPv4 アドレスを大量に割り当てられている組織が値段をつりあげるため『売り惜しみ』をしているのでは？」と考えたくなるところですが、そのあたりも含めた考察が [APNIC の記事](https://blog.apnic.net/2025/01/13/ip-addresses-through-2024/)で触れられており、

- ルート未広告の IPv4 アドレスブロックは 2024 年末の Amazon の広告ルート追加により大幅に減少
  ![](/images/burikaigi2026-aws-ipv6-study/addr24-7.png)

（図の出典は同記事：https://blog.apnic.net/2025/01/13/ip-addresses-through-2024/ ）

など、いくつかの結果が示されています。

広告ルートに載ったとしてもそれがそのまま「使う意思がある」ことにつながるわけではありませんが（2024 年 6 月にも一時的に大きな変動が見られますがすぐに戻っています）、移転による放出が期待できるアドレスブロックが減ったとすれば、移転についても当面は増加が期待できない雰囲気です。

:::message
その他、APNIC の[同記事](https://blog.apnic.net/2025/01/13/ip-addresses-through-2024/)には「実験用に確保されたクラス E のアドレス空間（240.0.0.0 〜 255.255.255.255）について何度か開放が議論されては立ち消えになっている話や、CGNAT（キャリアグレード NAT／文章中には「CG-NAT」と表記）についても触れられています。
:::

### AWS では？

2024 年 2 月 1 日より、特定のサービスに割り当てられているかどうかに関わらず、すべてのパブリック IPv4 アドレス（AWS 内で利用する、インターネット接続用の IPv4 グローバルアドレス）の利用に対して 1 IP アドレスあたり 0.005 USD/hour が課金されるようになりました。

料金などについてのアナウンスはこちらです。

- [新着情報 – パブリック IPv4 アドレスの利用に対する新しい料金体系を発表 / Amazon VPC IP Address Manager が Public IP Insights の提供を開始](https://aws.amazon.com/jp/blogs/news/new-aws-public-ipv4-address-charge-public-ip-insights/)

:::message
CloudFront など、サービス本体の利用料金以外にパブリック IPv4 アドレスの利用料金が掛からないサービスもあります。
:::

## IPv6 アドレスの現況

### IPv6 アドレスの割り当て状況

というわけで、「次世代 IP」として期待されている IPv6 の利用状況ですが、こちらも[前述の NRO の資料](https://www.nro.net/wp-content/uploads/NRO-Number-Resource-Status-Report-Q3-2025-FINAL.pdf)（P.10 〜 16）に示されています。

RIR ごとの地域差はかなりありますが、P.15 〜 16 を見ると意外と IPv6 の割り当ても進んでいることがわかります。

![](/images/burikaigi2026-aws-ipv6-study/ipv6-rir-48s.jpg)

全 RIR 合計で 1,500 万個以上の `/48` IPv6 アドレスブロックが割り当て済みです。その大半を LACNIC が占めています。

![](/images/burikaigi2026-aws-ipv6-study/ipv6-rir-pct.jpg)

（出典はいずれも同資料：https://www.nro.net/wp-content/uploads/NRO-Number-Resource-Status-Report-Q3-2025-FINAL.pdf ）

### 肝心の利用状況は？

あまり実感はないのですが、見えないところですでに利用が進んでいます。

日本の場合、固定回線・モバイル回線ともに IPv6 が利用可能なサービスが増えていますが、実際の統計情報を見てみましょう。

ChatGPT の力を借りて、[APNIC Labs Measurements and Data](https://labs.apnic.net/measurements/) のデータから日本の集計値を一部抜粋してみました。

![](/images/burikaigi2026-aws-ipv6-study/ipv6-stat-jp.png)

:::message
IPv6 Capable は「IPv6 に到達できた端末の割合」、IPv6 Preferred は「デュアルスタックで IPv6 が選ばれた割合」です。
:::

2023 年に 50% を超え、ここ 2 年で IPv6 Capable が 60% に迫っています。

![](/images/burikaigi2026-aws-ipv6-study/ipv6-stat-jp-mobile.png)

Softbank の IPv6 比率が低いようですが、（AS 内に含まれるモバイル回線以外の比率が高い？など）統計を取る上で何らかの問題があるのかもしれませんね。

それを差し引いても、モバイル回線については意識せずに利用しているケースがかなりありそうです。

実際、今回のセッションの準備のために IPv6 サーバー環境（HTTP アクセスするとアクセス元の IP アドレスを表示）を立てて手元のモバイル回線からアクセスしてみたところ、

- 主要キャリア（MNO）A 社・B 社 : IPv6 アドレスを表示
- 主要キャリア（MNO）C 社のサブキャリア D 社 : 同上
- MVNO E 社の C 社回線プラン : IPv4 アドレスを表示

という結果になりました。

## このチャプターのまとめ

- IPv4 アドレスブロックの新規割り当ては（RIR レベルでは）ほぼ終了していて、現在は移転の割合が大きくなっている
  - 移転も横ばい〜微減傾向
- AWS では 2024 年 2 月 1 日からすべてのパブリック IPv4 アドレスに対して 1 IP アドレスあたり 0.005 USD/hour が課金されるようになった
  - CloudFront などは除く
- IPv6 アドレスの利用は気づかないところで年々増えている
  - 日本では特にモバイル回線での利用が進んでいる
