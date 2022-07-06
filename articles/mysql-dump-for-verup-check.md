---
title: "小ネタ／MySQL でバージョンアップ時のデータ整合確認に mysqldump を使う"
emoji: "😺"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["mysql", "移行", "バージョンアップ", "mysqldump"]
published: false
---

データ移行作業そのものではなく、データ移行作業時の整合チェックに `mysqldump` を使う小ネタです。

## いつ使う？

MySQL（または RDS/Aurora などの互換サービス）でバージョンアップなどを行う際に、binlog あるいは DMS（CDC）などによってデータを旧バージョン → 新バージョンにレプリケーションし、新バージョンへの切り替え時のダウンタイム短縮をはかるケースがよくあります。

その際、（ダウンタイムとのトレードオフで）ある程度データの整合性チェックをした上で新バージョンへ切り替えたい場合もあると思います。

同じバージョン間のテーブル整合性確認には `CHECKSUM TABLE` コマンドが使えますが、バージョンを跨ぐ場合は内部的なデータフォーマットの変更によってチェックサムに差異が生じることがありますし、例えば文字セットを 3 バイトの `utf8` から `utf8mb4` に変更するケースではこの方法が使えません。

そんなときに今回の小ネタを使うと良いかもしれません。

## どう使う？

- サイズが小さいテーブル、重要度が高いテーブルを `mysqldump` で出力
  - `mysqldump -u ユーザ名 -h エンドポイント -p --default-character-set=デフォルト文字セット -t --skip-comments --max_allowed_packet=1G --hex-blob データベース名 テーブル名 > 出力ファイル`
- サイズが大きく、重要度がそれほど高くないテーブルは `SELECT * ... ORDER BY nn DESC LIMIT xxx` と `SELECT COUNT(*)`
- 最終的にこれらの結果のハッシュ値を MD5 や SHA2 で計算して新旧データの整合性確認

## 注意点

- MySQL 5.6 ⇔ MySQL 8.0 の場合、8.0 の `mysqldump` で 5.6 サーバのデータをダンプできない
  - 一部不一致行が含まれるので 8.0 側を `mysqldump -u ユーザ名 -h エンドポイント -p --default-character-set=デフォルト文字セット -t --skip-comments --max_allowed_packet=1G --hex-blob データベース名 テーブル名 | sed -e '/^SET @/d' -e '5s/50503/40101/' > 出力ファイル` とする
