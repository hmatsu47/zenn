---
title: "IP アドレス（グローバルアドレス）利用の現況について"
---

IPv6 について学ぶ前に、まずは IP アドレス、特にインターネットの通信に利用するグローバルアドレスの現況について簡単に触れます。

## IPv4 アドレスの現況

### IPv4 新規割り当て用の IPv4 アドレスブロックはほぼ枯渇

2011 年 2 月 3 日、IP アドレスをグローバルに管理する ICANN（IANA）から各地域の RIR（地域インターネットレジストリ：Regional Internet Registry）への最後のアドレスブロック割り当てが行われ、新規に割り当て可能な IPv4 アドレスブロックがなくなりました。

各地域の RIR（ARIN / RIPE NCC / APNIC / LACNIC / AFRINIC）でも在庫の消費が進み、[2025 年 9 月 30 日時点の NRO（Number Resource Organization）の報告資料](https://www.nro.net/wp-content/uploads/NRO-Number-Resource-Status-Report-Q3-2025-FINAL.pdf)（P.4）によると、`/8`換算で

- AFRINIC が 0.05（IP アドレス約 84 万個分）
- APNIC が 0.19（同・約 319 万個分）

となっています（ARIN / RIPE NCC / LACNIC は 0）。

![](/images/burikaigi2026-aws-ipv6-study/ipv4-rir.jpg)

（出典は同資料：https://www.nro.net/wp-content/uploads/NRO-Number-Resource-Status-Report-Q3-2025-FINAL.pdf）

:::message
他に「予約済み」アドレスがありますがそちらは省略します。
:::

### 現在は新規割り当てではなく移転が中心

現在は、新たな IPv4 アドレスブロックの割り当てを求める組織と、「割り当て済みだが未使用（不使用）」のアドレスブロックを持つ組織の間での「移転」が中心になっています（[前述資料](https://www.nro.net/wp-content/uploads/NRO-Number-Resource-Status-Report-Q3-2025-FINAL.pdf)の P.6 〜 9）。

当初、移転は同じ RIR 内に限られていましたが、現在は RIR 間移転も行われています（[同](https://www.nro.net/wp-content/uploads/NRO-Number-Resource-Status-Report-Q3-2025-FINAL.pdf)・P.9）。

![](/images/burikaigi2026-aws-ipv6-study/inter-rir-ipv4-trans.jpg)

（出典は同資料：https://www.nro.net/wp-content/uploads/NRO-Number-Resource-Status-Report-Q3-2025-FINAL.pdf）

数年前には「AWS が IPv4 アドレスブロックを買い漁っている」ことが話題になりました。

**【参考】**

- [Amazon が保有している IP アドレスだけで 2500 億円以上の資産価値がある（Gigazine）](https://gigazine.net/news/20201022-amazon-ipv4-address/)
- [AWS とか他のパブリッククラウドプロバイダーって、どうやってあんなにたくさんのパブリック IPv4 アドレスを手に入れてるんだろう？（Reddit）](https://www.reddit.com/r/aws/comments/ds06go/where_aws_or_other_public_cloud_provider_gets_so/?tl=ja)

### 「割り当て済みだが未使用（不使用）」の IPv4 アドレスの状況は？

そうなると「すでに IPv4 アドレスを大量に割り当てられている組織で、値段をつりあげるため『売り惜しみ』をしているのでは？」と考えたくなるところですが、そのあたりの考察が [APNIC の記事](https://blog.apnic.net/2025/01/13/ip-addresses-through-2024/)で触れられており、

- ルート未広告の IPv4 アドレスブロックは 2024 年末の Amazon の広告ルート追加により大幅に減少
  ![](/images/burikaigi2026-aws-ipv6-study/addr24-7.png)
- 公表されている移転価格は 2022 年あたりにピークになってその後低下（ただしばらつきが大きくなった）
  ![](/images/burikaigi2026-aws-ipv6-study/addrfig9.png)

などの結果が示されています。

（図の出典はいずれも同記事：https://blog.apnic.net/2025/01/13/ip-addresses-through-2024/）

広告ルートに載ったとしてもそれがそのまま「使う意思がある」ことにつながるわけではありませんが、移転についても今後少なくとも IPv6 の利用が一般的になるまではあまり期待できない状況ですね。

:::message
その他、APNIC の[同記事](https://blog.apnic.net/2025/01/13/ip-addresses-through-2024/)には「実験用に確保されたクラス E のアドレス空間（240.0.0.0 〜 255.255.255.255）について何度か開放が議論されては立ち消えになっている話が触れられています。
:::

## IPv6 アドレスの現況

### IPv6 アドレスの割り当て状況

というわけで、「次世代 IP」として期待されている IPv6 の利用状況ですが、こちらも[前述の NRO の資料](https://www.nro.net/wp-content/uploads/NRO-Number-Resource-Status-Report-Q3-2025-FINAL.pdf)（P.10 〜 16）に示されています。

RIR ごとの地域差はかなりありますが、P.15 〜 16 を見ると意外と IPv6 の割り当ても進んでいることがわかります。

![](/images/burikaigi2026-aws-ipv6-study/ipv6-rir-48s.jpg)

全 RIR 合計で 1,500 万個以上の `/48` IPv6 アドレスブロックが割り当て済みです。その大半を LACNIC が占めています。

![](/images/burikaigi2026-aws-ipv6-study/ipv6-rir-pct.jpg)

（出典はいずれも同資料：https://www.nro.net/wp-content/uploads/NRO-Number-Resource-Status-Report-Q3-2025-FINAL.pdf）

### 肝心の利用状況は？

あまり実感はないのですが、見えないところですでに利用が進んでいます。

日本の場合、固定回線・モバイル回線ともに IPv6 が利用可能なサービスが増えていますが、モバイル回線については、特に意識しないうちに利用しているケースもかなりありそうです。

実際、今回のセッションの準備のために IPv6 サーバー環境（HTTP アクセスするとアクセス元の IP アドレスを表示）を立てて手元のモバイル回線からアクセスしてみたところ、

- 主要キャリア A 社 : IPv6 アドレスを表示
- 主要キャリア B 社のサブキャリア : 同上
- 主要キャリア B 社の MVNO : IPv4 アドレスを表示

という結果になりました。
