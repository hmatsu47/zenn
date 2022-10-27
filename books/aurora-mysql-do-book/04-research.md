---
title: "情報収集・調査のフェーズ"
---

## この章について

移行ターゲットの仮決定から情報収集、移行対象のシステムに必要な情報の絞り込みに掛けての作業について振り返ります。

## 移行ターゲットの仮決定（2022/2）

本書のタイトルどおり、移行先をまずは Aurora MySQL v3（MySQL 8.0 互換）に定めて情報収集を開始しました。

v3 を選んだ理由は前掲のこの記事、

https://qiita.com/hmatsu47/items/ef48b8673f4775bef4a6#%E3%81%AA%E3%81%9C-aurora-mysql-v2-%E3%81%A7%E3%81%AF%E3%81%AA%E3%81%8F-v3-%E3%81%AB%E7%A7%BB%E8%A1%8C%E3%81%97%E3%81%9F%E3%81%AE%E3%81%8B

で触れていますが、主に

- 本家 MySQL 5.7 の EoL が迫る中、**一旦 v2 を挟むと最悪の場合 2 年で 2 回バージョンアップが必要になる**
- 私個人が **[MySQL 8.0 の薄い本](https://github.com/hmatsu47/mysql80_no_usui_hon)** の製作・頒布を MySQL 8.0.24 の頃まで続けていたので**バージョン間差分の情報をそこそこ把握していた**

です。

対象サービスが使用している Java や Web フレームワーク（自社製の軽量フレームワークを含む）、ミドルウェアなどには EoL が近いと予想される古いバージョンが混在しており、これをバージョンアップしたり、アプリケーションごとモダンなものに置き換えていく時間を捻出するには、DB のバージョンアップを早めに済ませる必要があります。

提供するサービスの状況を考えると、新機能の提供や既存機能の改良を継続して進める必要があり、バージョンアップだけで全ての時間を費やすわけにもいきません。

とはいえ、過去のメジャーバージョンアップと比べて差分が大きい Aurora MySQL v1（MySQL 5.6 互換）→ v3（同 8.0 互換）です。

調査開始時点で何もまとまった情報がなければ、リスク回避のため段階的なバージョンアップ（今回は v1 → v2、翌年 v2 → v3）を選択せざるをえませんでした。

やはり **[MySQL 8.0 の薄い本](https://github.com/hmatsu47/mysql80_no_usui_hon)** を作っていたことが、いきなり v3 へ移行する上で大きな後押しになりました。

https://github.com/hmatsu47/mysql80_no_usui_hon

## 情報収集（2022/2 〜 3）

先の章で触れたとおり、

- Aurora MySQL v1 vs v3 差分
- MySQL 5.6 vs 8.0 差分
- その他 Aurora 固有の機能
- 移行方法に関連する情報

について情報を集め、順次 Zenn の記事として公開、バージョン間差分については GitHub リポジトリで公開しました。

https://zenn.dev/hmatsu47/articles/aurora-mysql3-001-top
https://zenn.dev/hmatsu47/articles/aurora-mysql3-002-ref-material
https://zenn.dev/hmatsu47/articles/aurora-mysql3-003-ref-aws-01
https://zenn.dev/hmatsu47/articles/aurora-mysql3-004-ref-aws-02
https://zenn.dev/hmatsu47/articles/aurora-mysql3-005-ref-ora-01
https://zenn.dev/hmatsu47/articles/aurora-mysql3-006-research-01
https://github.com/hmatsu47/aurora_mysql1to3diff

:::message
このあたりは「1 日 1 記事」を目指して進めていました。
:::

## 対象システムの個人調査を実施（2022/3）

続いて、移行対象のシステムに対して、必要な情報の絞り込みと並行して（これも先の章で触れたとおりですが）個人で以下の作業を行いました。

- 開発環境でコード全体を調査し修正点（候補）をピックアップ
- 実際に修正して開発環境で動作確認
- 動作確認で見つけた不具合を修正
- DMS（CDC）を軽く試用

開発環境の構築にあたって、設定値（パラメーターグループ）について 2 点の調整が必要になりました。

### `innodb_strict_mode`のデフォルトが **厳密モード** に

https://dev.mysql.com/doc/refman/8.0/ja/innodb-parameters.html#sysvar_innodb_strict_mode

一部の SQL 文の実行で、従来は警告（Warning）で済ませていたのをエラーで止める動作に変わりました。ほぼ問題にならない見込みでしたが、念のため従来の設定に合わせて厳密モードを OFF にしました。

### `innodb_autoinc_lock_mode`のデフォルトが`1`から`2`に

https://dev.mysql.com/doc/refman/8.0/ja/innodb-parameters.html#sysvar_innodb_autoinc_lock_mode

`2`のほうが性能上有利なのですが、MySQL Connector/J の **[`rewriteBatchedStatements`オプション](https://dev.mysql.com/doc/connector-j/8.0/en/connector-j-connp-props-performance-extensions.html#cj-conn-prop_rewriteBatchedStatements)** を使ったバッチ`INSERT`と **[LAST_INSERT_ID()](https://dev.mysql.com/doc/refman/8.0/ja/information-functions.html#function_last-insert-id)** を組み合わせた処理を行っている部分があったので、従来と同じ`1`に戻しました。

これらをもとに Zenn の記事として

https://zenn.dev/hmatsu47/articles/aurora-mysql3-007-research-02
https://zenn.dev/hmatsu47/articles/aurora-mysql3-008-ope-check

を公開しました。

また、見つけた問題点を個別の記事として Qiita に公開しました。

https://qiita.com/hmatsu47/items/d3f34f39c28a4b802966

:::message
正直、この時点では「ソート順が変わる問題への対処は面倒臭そうだけど、個人でここまで調査と対処を進められるレベルだから、意外と楽勝だな」と思っていました（フラグ？）。
:::

## 記事を再構成して Zenn の本を公開（2022/4）

複数の記事を公開したのは良いのですが、冗長になり少し読みづらかったので、構成を見直して Zenn の本を公開しました。

https://zenn.dev/hmatsu47/books/aurora-mysql3-plan-book
