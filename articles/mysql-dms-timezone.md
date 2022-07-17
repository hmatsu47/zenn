---
title: "AWS"
emoji: "⌚"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["mysql", "移行", "aws", "dms"]
published: false
---

AWS の DMS でソース・ターゲットの両方に MySQL（RDS / Aurora MySQL）を指定する場合、タイムゾーンの設定に注意が必要です。

## 端的にいうと

- AWS DMS（Data Migration Service）で MySQL（RDS / Aurora MySQL を含む）から MySQL（同）にデータを移行するときは、 **ソース・ターゲットともタイムゾーンを設定しない**
- タイムゾーンを指定した場合、MySQL の`timestamp`型の値（内部的に`UTC`で保持）を指定タイムゾーンの値と認識してしまい、移行先（ターゲット）でタイムゾーンがずれる
- MySQL の`datetime`型にはタイムゾーンは保持していないので、ソースとターゲットに同じタイムゾーンを指定していれば移行先（ターゲット）でタイムゾーンがずれない
- 結果として、`timestamp`型も`datetime`型もタイムゾーンがずれないようにするには、ソース・ターゲットのタイムゾーンを`UTC`（デフォルトのまま）にしておくのが良い
