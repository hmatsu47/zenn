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

2022/2/28 現在、以下のような進捗状況です。

随時追加・更新していきます。

## 完了

- **[予約語](https://github.com/hmatsu47/aurora_mysql1to3diff/blob/main/mysql57_80_reserved.md)**
  - 計 30 個

## 進行中

- **[オペレータ・ビルトイン関数](https://github.com/hmatsu47/aurora_mysql1to3diff/blob/main/mysql57_80_func_oper.md)**
  - MySQL 8.0 で 64 ビットを超えるビット演算に対応したことによる非互換