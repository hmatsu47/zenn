---
title: "Amazon Aurora MySQL v1（5.6 互換）→ v3（8.0 互換）移行を計画する（2）調査の進め方と参考資料"
emoji: "📚"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["aws", "aurora", "mysql", "移行", "バージョンアップ"]
published: true
---

これは

- **[Amazon Aurora MySQL v1（5.6 互換）→ v3（8.0 互換）移行を計画する（1）はじめに](/hmatsu47/articles/aurora-mysql3-001-top)**

の続きです。

## 調査をどう進めるか？

大雑把に分けると、

1. テスト環境などを先に作って動作確認し、動かなくなった（または変な動きをする）箇所だけ原因調査して改修する
2. 事前に資料や事例などを収集し、問題になりそうな箇所の「あたり」を付けてから実物での調査に入る

の 2 通りの進め方があると思いますが、MySQL 5.7 以前から MySQL 8.0 への移行では**それ以前の 5.x 間の移行よりも注意すべき点が多く、またそのまま移行しただけでは動かなくなる可能性が低くない**ので、小規模な利用かつ稼働率や性能などの要件が緩い環境でなければ、事前の情報収集をお勧めします。

## 参考資料① MySQL 5.7 以前→ 8.0 移行体験記など

探してもかなり少ないと思います。

当たり前かもしれませんが、細部を参考にするというよりも雰囲気を掴むための参考資料ですね。

https://dupont.hatenablog.jp/entry/mysql_migration_55_80

2019 年、Windows での MySQL 5.5 → 8.0（8.0.17？）移行事例です。翌日`GROUP BY ASC/DESC`問題修正の記事もポストされています。

https://blog.cybozu.io/entry/2021/05/24/175000

こちらは（2020 〜？）2021 年のサイボウズさんの MySQL 5.7 → 8.0 移行事例です。有名な記事なのでご覧になられた方も多いと思います。

https://severalnines.com/database-blog/moving-mysql-57-mysql-80-what-you-should-know

海外の「MySQL 5.7 → 8.0 移行に際して知っておくべきこと」の記事です。「What is Deprecated in MySQL 8.0?」以降が重要なポイントです。

これ以外に Facebook（Meta）社の事例も見つかりますが、Facebook で主力として利用しているのは InnoDB とは別の MyRocks というストレージエンジンですので（すべてではないと思いますが）、混乱を避けるためにここでは割愛します。

## 参考資料② AWS 公式の Aurora 関連ドキュメント

[前回の記事](/hmatsu47/articles/aurora-mysql3-001-top)で示した

https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/AuroraUserGuide/Aurora.MySQL56.EOL.html

から辿っていくと、

https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.Updates.MajorVersionUpgrade.html

https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.MySQL80.html

あたりの情報が参考になりそうです。

## 参考資料③ Oracle 公式の MySQL 関連ドキュメント

こちらにまとまっています。

https://dev.mysql.com/doc/

特に、

- **リリースノート（英語）**

https://dev.mysql.com/doc/relnotes/mysql/8.0/en/

- **リファレンスマニュアル（日本語）**

https://dev.mysql.com/doc/refman/8.0/ja/

※機械翻訳が怪しい部分は、右上のセレクタで「8.0 English」を選択して英語版に切り替えて確認します。

- **バージョン間差異のリファレンス（英語）**

https://dev.mysql.com/doc/mysqld-version-reference/en/

が役に立ちそうです。

その他、すでにアーカイブ扱いになっていますが、

https://dev.mysql.com/blog-archive/

こちらも補助的に使えそうです。

## 参考資料④ その他個人ブログなど

Oracle の中の人や Oracle ACE（MySQL）のみなさまの個人ブログにも参考になる記事がありそうです。

※ここではリストアップしませんが、次回以降の記事の中で適宜リンクしていく予定です。

---

- **[Amazon Aurora MySQL v1（5.6 互換）→ v3（8.0 互換）移行を計画する（3）AWS 公式ドキュメントを読む（1）](/hmatsu47/articles/aurora-mysql3-003-ref-aws-01)**

に続きます。

参考資料①は雰囲気を掴む目的で目を通しておけば良さそうなのでここでは省略し、次回は参考資料②の AWS 公式 Aurora 関連ドキュメントから情報を拾っていきます。