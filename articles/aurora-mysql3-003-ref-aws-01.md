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

- **[Aurora MySQL v1 から直接 v3 へはアップグレードできない](/hmatsu47/articles/aurora-mysql3-003-ref-aws-01#aurora-mysql-v1-%E3%81%8B%E3%82%89%E7%9B%B4%E6%8E%A5-v3-%E3%81%B8%E3%81%AF%E3%82%A2%E3%83%83%E3%83%97%E3%82%B0%E3%83%AC%E3%83%BC%E3%83%89%E3%81%A7%E3%81%8D%E3%81%AA%E3%81%84)**
- **[Aurora MySQL v1 → v2 はインプレースアップグレードできるが、処理時間が掛かる](/hmatsu47/articles/aurora-mysql3-003-ref-aws-01#aurora-mysql-v1-%E2%86%92-v2-%E3%81%AF%E3%82%A4%E3%83%B3%E3%83%97%E3%83%AC%E3%83%BC%E3%82%B9%E3%82%A2%E3%83%83%E3%83%97%E3%82%B0%E3%83%AC%E3%83%BC%E3%83%89%E3%81%A7%E3%81%8D%E3%82%8B%E3%81%8C%E3%80%81%E5%87%A6%E7%90%86%E6%99%82%E9%96%93%E3%81%8C%E6%8E%9B%E3%81%8B%E3%82%8B)**
  - 特に v1.22.3 より前のバージョンからアップグレードする場合
- **[クローン・スナップショットからの復元や、binlog レプリケーションを組み合わせた Blue/Green デプロイも使える](/hmatsu47/articles/aurora-mysql3-003-ref-aws-01#%E3%82%AF%E3%83%AD%E3%83%BC%E3%83%B3%E3%83%BB%E3%82%B9%E3%83%8A%E3%83%83%E3%83%97%E3%82%B7%E3%83%A7%E3%83%83%E3%83%88%E3%81%8B%E3%82%89%E3%81%AE%E5%BE%A9%E5%85%83%E3%82%84%E3%80%81%E3%83%AC%E3%83%97%E3%83%AA%E3%82%B1%E3%83%BC%E3%82%B7%E3%83%A7%E3%83%B3%E3%82%92%E7%B5%84%E3%81%BF%E5%90%88%E3%82%8F%E3%81%9B%E3%81%9F-blue%2Fgreen-%E3%83%87%E3%83%97%E3%83%AD%E3%82%A4%E3%82%82%E4%BD%BF%E3%81%88%E3%82%8B)**
- **[Aurora MySQL v2 → v3 はクローン・インプレースアップグレードできない](/hmatsu47/articles/aurora-mysql3-003-ref-aws-01#aurora-mysql-v2-%E2%86%92-v3-%E3%81%AF%E3%82%AF%E3%83%AD%E3%83%BC%E3%83%B3%E3%83%BB%E3%82%A4%E3%83%B3%E3%83%97%E3%83%AC%E3%83%BC%E3%82%B9%E3%82%A2%E3%83%83%E3%83%97%E3%82%B0%E3%83%AC%E3%83%BC%E3%83%89%E3%81%A7%E3%81%8D%E3%81%AA%E3%81%84)**
  - スナップショットからの復元でアップグレードする
- **[v1 → v2 の際に`engine`属性が`aurora`から`aurora-mysql`に変わる](/hmatsu47/articles/aurora-mysql3-003-ref-aws-01#v1-%E2%86%92-v2-%E3%81%AE%E9%9A%9B%E3%81%ABengine%E5%B1%9E%E6%80%A7%E3%81%8Caurora%E3%81%8B%E3%82%89aurora-mysql%E3%81%AB%E5%A4%89%E3%82%8F%E3%82%8B)**
  - CLI や API で処理を自動化している場合に注意
- **[パラメータグループに注意](/hmatsu47/articles/aurora-mysql3-003-ref-aws-01#%E3%83%91%E3%83%A9%E3%83%A1%E3%83%BC%E3%82%BF%E3%82%B0%E3%83%AB%E3%83%BC%E3%83%97%E3%81%AB%E6%B3%A8%E6%84%8F)**
  - デフォルトとは別のパラメータグループを使っている場合や、CLI・API で処理を自動化している場合は特に注意

:::message
**2022/2/27 追記：**
CLI を使った具体的なアップグレード操作については取り上げていませんが、以下のリンク先に例示があります。

- **[Aurora MySQL インプレースアップグレードのチュートリアル](https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.Updates.MajorVersionUpgrade.html#AuroraMySQL.Upgrading.Tutorial)**
- **[Aurora MySQL バージョン 1 からバージョン 3 へのアップグレードの例](https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.MySQL80.html#AuroraMySQL.mysql80-upgrade-example-v1-v3)**
:::

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

v3 が GA になる前の記事なので v3 アップグレードへの言及はありません。

クローン後に差分を binlog レプリケーションして Green 環境を構築する話が出ていますが、v1 → v3 に応用するなら、

1. 一旦 v1 → v2 をクローンまたはスナップショットからの復元で実行
2. v2 のスナップショットを取って v3 指定で復元
3. v1 → v3 レプリケーションでデータの差分を反映して Green 環境を構築
4. 動作確認した後 Blue 環境に昇格
5. 後始末

となるでしょうか。

ただし MySQL 5.6 から 8.0 への binlog レプリケーションは非推奨なので、安全に進めるなら v1 → v2 のアップグレードを Blue/Green 方式で実施した後で、あらためて別日程で v2 → v3 のアップグレードを実施すれば良いでしょう。

:::message
以前の MySQL でよく行われていた 3 つ（以上）のバージョンをまたぐ多段レプリケーションも、現在の MySQL では非推奨となっています。
:::

この記事では binlog 形式として`MIXED`形式を推奨していますが、`ROW`形式でも大丈夫だと思います。

:::message
`ROW`形式との比較で、`MIXED`形式のほうが、

- 容量を節約できる
- 何か問題が生じた場合に原因となった SQL 文を突き止めやすい

などのメリットがあります。ただし MySQL 8.0 のデフォルトは`ROW`形式です。
また、データ移行やレプリケーションに DMS（Database Migration Service）を使いたい場合は`ROW`形式を使います。
:::

## Aurora MySQL v2 → v3 はクローン・インプレースアップグレードできない

今後クローン・インプレースアップグレード対応になる可能性はありますが、現時点ではスナップショットからの復元でアップグレードすることになります。

この関係で、グローバルデータベース構成にしている場合は v3 に移行する難易度が高いと思います（できなくはないと思いますが）。

クラスタ数やインスタンス数が多い場合も厳しそうですね（並行して新旧バージョンのクラスタを用意する場合はクォータに引っ掛かりそうなので上限緩和申請が必要かもしれません）。

## v1 → v2 の際に`engine`属性が`aurora`から`aurora-mysql`に変わる

マネジメントコンソールだけ使っている場合には関係ないのですが、CLI や SDK などを使って API を呼び出すコードを書いている場合、パラメータとして指定する属性（`engine`）が変わるので修正が必要になります。

## パラメータグループに注意

Aurora MySQL v1 と v2、v3 でクラスタとデータベースのパラメータグループが違います。

カスタムパラメータグループを使っている場合は、アップグレード前にあらかじめ v3 用のカスタムパラメータグループを作っておきます。

CLI や SDK などを使って API を呼び出すコードを書いている場合、パラメータ指定も変える必要があります。

なお、ここではまだ詳細には触れませんが MySQL 5.6 と 8.0 ではサーバのパラメータにかなり違いがあり、Aurora のパラメータグループの項目や初期値も違います。さらに**同じパラメータ名の`engine-default`が指す値が v1 と v3 で異なるケースなど、パラメータグループの差分が表示されなくても値の書き換えが必要な場合があります。**

# まとめ

- **Aurora MySQL v1 から v3 には直接移行できない**
- **そのため、グローバルデータベースや多クラスタ環境では一旦 v2 までのアップグレードに止めたほうが良さそう**
- **アップグレードには時間が掛かるのでレプリケーションを利用した Blue/Green 方式の移行を検討したほうが良い**
- **インフラの自動化・コード化を実装している場合はパラメータとして指定する属性値の修正が必要になる**
- **Aurora MySQL v1・v2・v3 の間でパラメータグループの項目や初期値、デフォルト値が違うので移行時に差分の確認が必要**

:::message
レプリケーションを使って移行する場合、DMS の使用を検討したほうが良いかもしれません。その場合、binlog は`ROW`形式を使います。
- https://docs.aws.amazon.com/ja_jp/dms/latest/userguide/Welcome.html
- https://docs.aws.amazon.com/ja_jp/dms/latest/userguide/CHAP_Source.MySQL.html
- https://docs.aws.amazon.com/ja_jp/dms/latest/userguide/CHAP_Target.MySQL.html
:::

---

- **[Amazon Aurora MySQL v1（5.6 互換）→ v3（8.0 互換）移行を計画する（4）AWS 公式ドキュメントを読む（2）](/hmatsu47/articles/aurora-mysql3-004-ref-aws-02)**

に続きます。