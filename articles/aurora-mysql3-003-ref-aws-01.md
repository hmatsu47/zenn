---
title: "Amazon Aurora MySQL v1（5.6 互換）→ v3（8.0 互換）移行を計画する（3）AWS 公式ドキュメントを読む（1）"
emoji: "📘"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["aws", "aurora", "mysql"]
published: true
---

これは

- **[Amazon Aurora MySQL v1（5.6 互換）→ v3（8.0 互換）移行を計画する（2）調査の進め方と参考資料](/hmatsu47/articles/aurora-mysql3-002-ref-material)**

の続きです。

今回は AWS 公式の Aurora 関連ドキュメントのうち、クラスタの移行に関する部分を取り上げてみます。

# 「Aurora MySQL DB クラスターのメジャーバージョンのアップグレード」を読む

https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.Updates.MajorVersionUpgrade.html

より、重要な情報を拾っていきます。

:::message
2022/2/25 時点で公開されている情報をもとに書いています。
:::

- **[Aurora MySQL v1 から直接 v3 へはアップグレードできない](/hmatsu47/articles/aurora-mysql3-003-ref-aws-01#Aurora%20MySQL%20v1%20から直接%20v3%20へはアップグレードできない)**
- **[Aurora MySQL v1 → v2 はインプレースアップグレードできるが、処理時間が掛かる](/hmatsu47/articles/aurora-mysql3-003-ref-aws-01#Aurora%20MySQL%20v1%20→%20v2%20はインプレースアップグレードできるが、処理時間が掛かる)**
  - 特に v1.22.3 より前のバージョンからアップグレードする場合
- **[クローン・スナップショットからの復元や、binlog レプリケーションを組み合わせた Blue/Green デプロイも使える](/hmatsu47/articles/aurora-mysql3-003-ref-aws-01#クローン・スナップショットからの復元や、レプリケーションを組み合わせた%20Blue/Green%20デプロイも使える)**
- **[Aurora MySQL v2 → v3 はクローン・インプレースアップグレードできない](/hmatsu47/articles/aurora-mysql3-003-ref-aws-01#Aurora%20MySQL%20v2%20→%20v3%20はクローン・インプレースアップグレードできない)**
  - スナップショットからの復元でアップグレードする
- **[v1 → v2 の際に`engine`属性が`aurora`から`aurora-mysql`に変わる](/hmatsu47/articles/aurora-mysql3-003-ref-aws-01#v1%20→%20v2%20の際に`engine`属性が`aurora`から`aurora-mysql`に変わる)**
  - CLI や API で処理を自動化している場合に注意
- **[パラメータグループに注意](/hmatsu47/articles/aurora-mysql3-003-ref-aws-01#パラメータグループに注意)**
  - デフォルトとは別のパラメータグループを使っている場合や、CLI・API で処理を自動化している場合は特に注意

## Aurora MySQL v1 から直接 v3 へはアップグレードできない

一旦 v2 に移行してから v3 に移行しましょう、という話です。

スナップショットからの復元でも、v1 から v3 に直接アップグレードはできないようです。

![](https://storage.googleapis.com/zenn-user-upload/7a8593395976-20220225.png)

## Aurora MySQL v1 → v2 はインプレースアップグレードできるが、処理時間が掛かる

インプレースアップグレードは同一リージョンで同時に処理できる数に限りがありそうです。以前、どこかの AWS ユーザさんが複数のクラスタを同時にアップグレードしようとして計画時間内に全然進まなかったという話をされていました。

移行作業は特定のメンテナンス日に一気に行うのではなく、可能なら binlog レプリケーションを併用して段階的に準備を進める方法を取るのが良さそうです。

なお、v1.22.3 より前のバージョンからアップグレードする場合は、内部的に 2 段階のアップグレードになるため特に時間が掛かるようです。

## クローン・スナップショットからの復元や、レプリケーションを組み合わせた Blue/Green デプロイも使える

こちらのブログ記事が紹介されています。

https://aws.amazon.com/jp/blogs/database/performing-major-version-upgrades-for-amazon-aurora-mysql-with-minimum-downtime/

v3 が GA になる前の記事なので v3 へのアップグレードへの言及はありません。

クローン後に差分を binlog レプリケーションして Green 環境を構築する話が出ていますが、v1 → v3 に応用するなら、

1. 一旦 v1 → v2 をクローンまたはスナップショットからの復元で実行
2. v2 のスナップショットを取って v3 指定で復元
3. v1 → v3 レプリケーションでデータの差分を反映して Green 環境を構築
4. 動作確認した後 Blue 環境に昇格
5. 後始末

となるでしょうか。

この記事では binlog 形式として`MIXED`を推奨していますが、`ROW`形式でも大丈夫だと思います。

## Aurora MySQL v2 → v3 はクローン・インプレースアップグレードできない

今後クローン・インプレースアップグレード対応になる可能性はありますが、現時点ではスナップショットからの復元でアップグレードすることになります。

この関係で、グローバルデータベース構成にしている場合は v3 に移行する難易度が高いと思います（できなくはないと思いますが）。

## v1 → v2 の際に`engine`属性が`aurora`から`aurora-mysql`に変わる

マネジメントコンソールだけ使っている場合には関係ないのですが、CLI や SDK などを使って API を呼び出すコードを書いている場合、パラメータとして指定する属性（`engine`）が変わるので修正が必要になります。

## パラメータグループに注意

Aurora v1 と v2、v3 でクラスタとデータベースのパラメータグループが違います。

カスタムパラメータグループを使っている場合は、アップグレード前にあらかじめ v3 用のカスタムパラメータグループを作っておきます。

CLI や SDK などを使って API を呼び出すコードを書いている場合、パラメータ指定も変える必要があります。

なお、ここではまだ詳細には触れませんが MySQL 5.6 と 8.0 ではサーバのパラメータにかなり違いがあり、Aurora のパラメータグループの項目や初期値も違います。さらに**同じパラメータ名の`engine-default`が指す値が v1 と v3 で異なるケースなど、パラメータグループの差分が表示されなくても値の書き換えが必要な場合があります。**

---

- Amazon Aurora MySQL v1（5.6 互換）→ v3（8.0 互換）移行を計画する（4）AWS 公式ドキュメントを読む（2）

に続きます。