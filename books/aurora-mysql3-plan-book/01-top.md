---
title: "はじめに"
---
## 本書の目的

Amazon Aurora MySQL 互換エディション（以降「Aurora MySQL」と表記）のバージョン 1（MySQL 5.6 互換・以降「v1」）の EoL が発表されました。

https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/AuroraUserGuide/Aurora.MySQL56.EOL.html

v2（MySQL 5.7 互換）への移行も考えられますが、本家 MySQL 5.7 の EoL も 2023 年 10 月 21 日に迫っています。

https://endoflife.software/applications/databases/mysql

そして AWS の Aurora バージョン情報によると、最短で 2024 年前半に EoL になる可能性があります。

https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/AuroraUserGuide/Aurora.VersionPolicy.html#Aurora.VersionPolicy.MajorVersionLifetime

このような状況のもと、Aurora MySQL v1 から v3 への移行計画に必要な情報を提供します。

## 注意事項など

- 本書は以下の Zenn 記事を再構成したものです。
  - **[Amazon Aurora MySQL v1（5.6 互換）→ v3（8.0 互換）移行を計画する（1）はじめに](https://zenn.dev/hmatsu47/articles/aurora-mysql3-001-top)**
  - **[Amazon Aurora MySQL v1（5.6 互換）→ v3（8.0 互換）移行を計画する（2）調査の進め方と参考資料](https://zenn.dev/hmatsu47/articles/aurora-mysql3-002-ref-material)**
  - **[Amazon Aurora MySQL v1（5.6 互換）→ v3（8.0 互換）移行を計画する（3）AWS 公式ドキュメントを読む（1）](https://zenn.dev/hmatsu47/articles/aurora-mysql3-003-ref-aws-01)**
  - **[Amazon Aurora MySQL v1（5.6 互換）→ v3（8.0 互換）移行を計画する（4）AWS 公式ドキュメントを読む（2）](https://zenn.dev/hmatsu47/articles/aurora-mysql3-004-ref-aws-02)**
  - **[Amazon Aurora MySQL v1（5.6 互換）→ v3（8.0 互換）移行を計画する（5）Oracle 公式ドキュメントを読む](https://zenn.dev/hmatsu47/articles/aurora-mysql3-005-ref-ora-01)**
  - **[Amazon Aurora MySQL v1（5.6 互換）→ v3（8.0 互換）移行を計画する（6）詳細調査について](https://zenn.dev/hmatsu47/articles/aurora-mysql3-006-research-01)**
  - **[Amazon Aurora MySQL v1（5.6 互換）→ v3（8.0 互換）移行を計画する（7）パラメータ・コード確認](https://zenn.dev/hmatsu47/articles/aurora-mysql3-007-research-02)**
  - **[Amazon Aurora MySQL v1（5.6 互換）→ v3（8.0 互換）移行を計画する（8）動作確認](https://zenn.dev/hmatsu47/articles/aurora-mysql3-008-ope-check)**
- 本書記載の内容は無保証です。本書の利用により生じた一切の損害等を著者は負わないものとします。
- 本書記載の内容は著者個人の調査等によるものであり、所属する組織とは無関係です。
- 本書の内容は 2022 年 3 月現在の情報をもとに構成しています。
- 移行先の Aurora MySQL のマイナーバージョンは 3.01.0 〜 3.02.0 を想定しています。
  - マイナーバージョン 3.01.0 は MySQL 8.0.23 をベースに開発されています。
    - 2022 年 4 月 21 現在、3.01.1 および 3.02.0 がリリースされています。いずれも MySQL 8.0.23 がベースであり移行方法は本書の説明内容と変わりません。
  - Aurora MySQL v3 は、ベースとなる MySQL 8.0 の Continuous Delivery Model（継続提供モデル）に倣い、**マイナーバージョンごとの機能追加および変更・削除が想定されています。**
  - そのため、マイナーバージョンが上がると、対応が必要な変更点が増える可能性があります。

:::message
Aurora MySQL v3 の現時点のリリース（3.01.0 〜 3.02.0：MySQL 8.0.23 ベース）は LTS ではありません。
今後のマイナーバージョンで LTS リリースが設けられる予定です。
:::