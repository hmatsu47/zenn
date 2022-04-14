---
title: "調査・検証と計画策定に有用な参考資料"
---
## この章について

調査・検証段階、および計画策定に有用な参考資料を列挙します。

:::message
著者自身がまとめた v1 → v3 変更点の情報は Chapter 05 以降で扱います。
:::

## 公式資料

### AWS 公式の Aurora 関連ドキュメント

#### Aurora MySQL メジャーバージョンのアップグレード

https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.Updates.MajorVersionUpgrade.html

#### Aurora MySQL v3 での本家 MySQL 8.0 機能のサポート状況について

https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.MySQL80.html

### Oracle 公式の MySQL 関連ドキュメント

#### リリースノート

https://dev.mysql.com/doc/relnotes/mysql/8.0/en/

#### リファレンスマニュアル

https://dev.mysql.com/doc/refman/8.0/ja/

※機械翻訳が怪しい部分は、右上のセレクタで「8.0 English」を選択して英語版に切り替えて確認します。一部切り替えリンクが表示されないページがありますが、URL 中の`ja`を`en`に書き換えると表示されます。

#### MySQL 5.7 リファレンスマニュアル

https://dev.mysql.com/doc/refman/5.7/en/mysql-nutshell.html

※新機能のページです。MySQL 5.7 では基本的に GA 後の機能廃止がないので、MySQL 5.6 → 5.7 の差分はこちらを見ておけば十分でしょう。

#### バージョン間差異のリファレンス

https://dev.mysql.com/doc/mysqld-version-reference/en/

## MySQL 5.7 → 8.0 移行体験記など

本家（コミュニティ版）MySQL と Aurora MySQL にはいくつか仕様の違いがあるので、その差分を意識しながら参考にします。

#### サイボウズのブログ記事

https://blog.cybozu.io/entry/2021/05/24/175000

#### 海外のブログ記事

https://severalnines.com/database-blog/moving-mysql-57-mysql-80-what-you-should-know

※「What is Deprecated in MySQL 8.0?」以降が重要なポイントです。

#### スマートスタイルの OSS ニュース

https://www.s-style.co.jp/mysql_news/mysql

※8.0.12 以降のリリースノートの日本語訳（主要部分）があります。

## 調査の注意点

### MySQL 8.0 でのリリースモデルの変更

Chapter 01 で触れたとおり、MySQL 8.0 では **GA 後のマイナーバージョンで新規の機能追加や破壊的な動作変更、機能削除（廃止）が行われる**方針に変わったため、Aurora MySQL のリリースモデルも v2 までとは違って **MySQL 8.0 のマイナーバージョンに追従していく**ことになりました。

したがって、

- MySQL 8.0.24 以降に動作仕様が変わった機能と削除（廃止）された機能
- MySQL 5.6.11 以降に非推奨（Deprecated）になり、まだ削除（廃止）されていない機能

にも配慮する必要があります。

これらについては、

- v1 → v3 移行の準備段階で修正しておく
- v1 → v3 移行後に修正する
- とりあえず放置して必要になったら修正する

のうち、どの対応を取るのか決める必要があります。

変更点の影響調査によって対処が必要な点がいくつ見つかるか、どのタイミングなら対処できるか次第で取りうる選択肢が変わるのを意識して調査を進めましょう。