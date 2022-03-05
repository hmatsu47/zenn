---
title: "Amazon Aurora MySQL v1（5.6 互換）→ v3（8.0 互換）移行を計画する（6）詳細調査について"
emoji: "🕵"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["aws", "aurora", "mysql"]
published: true
---

これは

- **[Amazon Aurora MySQL v1（5.6 互換）→ v3（8.0 互換）移行を計画する（5）Oracle 公式ドキュメントを読む](/hmatsu47/articles/aurora-mysql3-005-ref-ora-01)**

の続きです。

# 詳細調査用リポジトリについて

Zenn 記事を逐一追加していくのも冗長ですので、GitHub リポジトリで調査状況を公開することにしました。

https://github.com/hmatsu47/aurora_mysql1to3diff

2022/3/5 現在、以下のような進捗状況です。

随時追加・更新していきます。

## 完了

- **[予約語](https://github.com/hmatsu47/aurora_mysql1to3diff/blob/main/mysql57_80_reserved.md)**
  - 計 30 個

- **[オペレータ・ビルトイン関数](https://github.com/hmatsu47/aurora_mysql1to3diff/blob/main/mysql57_80_func_oper.md)**
  - MySQL 8.0 で 64 ビットを超えるビット演算に対応したことによる非互換
  - `&&`・`||`（`OR`のシノニム）・`!`が非推奨に
  - GIS 関数の実装変更（5.6 → 5.7 → 8.0）
    - 非互換の可能性
  - `ST_`・`MBR`が頭に付かないなどの古い名前の GIS 関数の削除
  - `BINARY`オペレータが非推奨に
    - MySQL 8.0.27 の変更点なので将来の Aurora MySQL v3 で取り込まれる可能性が高い
  - 時刻関連の関数の変更（タイムスタンプの内部処理 64 ビット化）
    - MySQL 8.0.28 の変更点なので将来の Aurora MySQL v3 で取り込まれる可能性が高い
      - `CAST(... AS BINARY)`に置換
  - `DATE_ADD()`・`DATE_SUB()`の動作非互換
    - MySQL 8.0.28 で元の動作に
      - 将来の Aurora MySQL v3 で取り込まれる可能性が高い
  - 暗号関連の関数（DES など古いもの）が非推奨に
  - `FOUND_ROWS()`・`SQL_CALC_FOUND_ROWS`が非推奨に
  - `GREATEST()`・`LEAST()`の引数の型推測（キャスト）ルール変更
  - `MASTER`→`SOURCE`読み替え（MySQL 8.0.26 からバックポート済み）
  - `MATCH()`の動作非互換
    - MySQL 8.0.28 の変更点なので将来の Aurora MySQL v3 で取り込まれる可能性が高い
  - 正規表現ライブラリ変更
    - 非互換の可能性
  - `IDENTIFIED BY PASSWORD()`などパスワード関連の一部機能の廃止
  - `PROCEDURE ANALYSE()`廃止
  - `ROUND()`・`TRUNCATE()`戻り値の型決定方法の非互換
  - `INSERT ... ON DUPLICATE KEY UPDATE`で`UPDATE`句の`VALUES()`が非推奨に
  - `WAIT_UNTIL_SQL_THREAD_AFTER_GTID`が非推奨に

## 進行中

- **[サーバ変数とオプション（パラメータ）](https://github.com/hmatsu47/aurora_mysql1to3diff/blob/main/work/aurora_param_diff.csv)**
  - Aurora MySQL v1 と v3 の差分のうち、「o」で始まるパラメータまで確認済み
    - 一旦 v2 は諦めました
