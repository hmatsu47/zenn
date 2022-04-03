---
title: "調査・検証に有用な参考資料"
---
# この章について

調査・検証段階で有用な参考資料を列挙します。

:::message
著者自身がまとめた v1 → v3 差分情報はチャプター 5 以降で扱います。
:::

# 公式資料

## AWS 公式の Aurora 関連ドキュメント

### Aurora MySQL メジャーバージョンのアップグレード

https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.Updates.MajorVersionUpgrade.html

### Aurora MySQL v3 での本家 MySQL 8.0 機能のサポート状況について

https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.MySQL80.html

## Oracle 公式の MySQL 関連ドキュメント

### リリースノート

https://dev.mysql.com/doc/relnotes/mysql/8.0/en/

### リファレンスマニュアル

https://dev.mysql.com/doc/refman/8.0/ja/

※機械翻訳が怪しい部分は、右上のセレクタで「8.0 English」を選択して英語版に切り替えて確認します。一部切り替えリンクが表示されないページがありますが、URL 中の`ja`を`en`に書き換えると表示されます。

https://dev.mysql.com/doc/mysqld-version-reference/en/

# MySQL 5.7 → 8.0 移行体験記など

本家（コミュニティ版）MySQL と Aurora MySQL にはいくつか仕様の違いがあるので、その差分を意識しながら参考にします。

### サイボウズのブログ記事

https://blog.cybozu.io/entry/2021/05/24/175000

### 海外のブログ記事

https://severalnines.com/database-blog/moving-mysql-57-mysql-80-what-you-should-know

※「What is Deprecated in MySQL 8.0?」以降が重要なポイントです。