---
title: "Amazon Aurora MySQL v1（5.6 互換）→ v3（8.0 互換）移行・実施事例（1）はじめに"
emoji: "🏃"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["aws", "aurora", "mysql", "移行", "バージョンアップ"]
published: false
---

これは

- **[Amazon Aurora MySQL v1（5.6 互換）→ v3（8.0 互換）移行を計画する（8）動作確認](/hmatsu47/articles/aurora-mysql3-008-ope-check)**

の続きです。

これまでの記事では移行の計画に必要な情報をまとめてきました。ここから先は、移行計画に沿って実施する中で実際に出てきた諸問題について、一つの事例として記して行きます。

:::message alert

- これは一つの事例であって、すべての移行ケースで生じるであろう問題をカバーするものではありません。
- 一部、過去の記事の内容と重複します。

ご了承ください。
:::
