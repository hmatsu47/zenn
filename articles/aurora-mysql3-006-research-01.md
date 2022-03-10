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

2022/3/10 現在、以下のような進捗状況です（各項目の調査を完了し整理中）。

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
  - `OLD_PASSWORD()`廃止（MySQL 5.7 で）
  - `IDENTIFIED BY PASSWORD()`などパスワード関連の一部機能の廃止
  - `PROCEDURE ANALYSE()`廃止
  - `ROUND()`・`TRUNCATE()`戻り値の型決定方法の非互換
  - `INSERT ... ON DUPLICATE KEY UPDATE`で`UPDATE`句の`VALUES()`が非推奨に
  - `WAIT_UNTIL_SQL_THREAD_AFTER_GTID`が非推奨に
- **[サーバ変数とオプション（パラメータ）](https://github.com/hmatsu47/aurora_mysql1to3diff/blob/main/aurora-mysql1_3_param.md)**
  - 厳密モードのデフォルト化と`AUTO_INCREMENT`値のロックモードの変更以外はそれほど気を付ける点はなさそう
  - デフォルトのパラメータグループから変える必要がある項目が減った印象

- **マニュアル全体の差分調査**
  - ステートメントの非互換・構文解析の非互換・その他
    - 64 文字を超える外部キー制約名を持つテーブルは NG に
    - `\N`は`NULL`のシノニムではなくなった
    - `CHANGE MASTER TO`→`CHANGE REPLICATION SOURCE TO`
    - `CHECK`制約の有効化
      - 過去に無視された制約の定義が有効あるいはエラーに
    - Connector を対応バージョンに入れ替え
    - `CREATE TABLE ... SELECT`のトランザクションの扱いが変更（8.0.21）
      - 行ベースレプリケーションで 1 つのトランザクションとして記録
    - `CREATE TEMPORARY TABLE`での`TABLESPACE = {innodb_file_per_table | innodb_temporary}`非推奨
    - `DELAYED`廃止（InnoDB では元から使えず）
    - `EXPLAIN`の`EXTENDED`・`PARTITIONS`キーワード削除（常に有効）
    - `FLOAT`・`DECIMAL`・`DOUBLE`型（シノニム含む）の`UNSIGNED`属性非推奨
    - `FLOAT`・`DOUBLE`型（シノニム含む）の`AUTO_INCREMENT`非推奨
    - `FLOAT(M,D)`・`DOUBLE(M,D)`（シノニム含む）非推奨
    - `FLUSH HOSTS`が非推奨に（8.0.23）
    - `GRANT`で暗黙のユーザ作成およびユーザ属性のみの変更を廃止
    - `GRANT`操作の読み取りロックの変更（8.0.22）
    - `GROUP BY ASC/DESC`廃止（8.0.13）
      - グループ化関数を使用した`ORDER BY`サポート（8.0.12）
    - GTID レプリケーションの非互換
    - `KEY`パーティショニングのカラムインデックス接頭辞が非推奨に（8.0.21）
    - `LOCK TABLES ... WRITE`による明示的テーブルロック時の`innodb_table_locks=0`無効化
    - `ORDER BY 【列番号】`・カッコで囲まれたクエリー式内で発生し外部クエリーにも適用される`ORDER BY`（動作不定）が非推奨に
    - `RESET SLAVE`→`RESET REPLICA`
    - `RIGHT JOIN`の実行結果が非互換の可能性（8.0.22）
      - 以前の結果が不正確な可能性あり
    - `SELECT ... INTO ... FROM`が非推奨に（8.0.22）
    - `SELECT`・`UNION`パーサールールの変更
      - ロック句を含む `SELECT`ステートメントにはカッコが必要に
    - `SHOW ENGINE INNODB MUTEX`一旦廃止後再導入（仕様変更に注意）
    - `SHOW GRANTS`で（動的権限導入により）`ALL PRIVILEGES`が表示されなくなった
    - `SHOW SLAVE STATUS`→`SHOW REPLICA STATUS`
    - SQL モードの非推奨・削除
      - `ERROR_FOR_DIVISION_BY_ZERO`, `PAD_CHAR_TO_FULL_LENGTH`（非推奨）
      - `DB2`, `MAXDB`, `MSSQL`, `MYSQL323`, `MYSQL40`, `ORACLE`, `POSTGRESQL`, `NO_FIELD_OPTIONS`, `NO_KEY_OPTIONS`, `NO_TABLE_OPTIONS`, `NO_AUTO_CREATE_USER`（削除）
    - `START SLAVE`→`START REPLICA`
    - `TABLE ... UNION (TABLE)`の挙動の変更
    - `UNION DISTINCT`・`UNION ALL`の挙動の変更
    - `UNION`の制限変更
    - `utf8mb3`が非推奨に
      - 現時点では MySQL 8.0 含め`utf8`は`utf8mb3`のシノニム
    - `utf8mb4`のデフォルト照合順序が`utf8mb4_0900_ai_ci`へ
    - `YEAR(2)`型廃止・`YEAR(4)`非推奨→`YEAR`へ
    - アトミック DDL 導入によるレプリケーションの挙動変化
      - `IF EXISTS`が付かない`DROP TABLE`・`DROP VIEW`のレプリケーション差異
    - クライアントの`--ssl`・`--ssl-verify-server-cert`オプションが削除
    - デフォルト認証が変わったことにより、Aurora MySQL v3 で新規ユーザを作成した場合に既存アプリケーションから接続できない可能性がある
      - `CREATE USER`時に`mysql_native_password`を指定する
    - プリペアドステートメント内のユーザー変数への参照のタイプの決定タイミング変更（8.0.22）
    - レプリケーション設定の不整合（ギャップ）にかかわる設定項目の変更（8.0.19）
    - 一時テーブルの`ROW`・`MIXED`形式バイナリログ記録の変更（5.7.25・8.0.4）
    - 管理者権限の分割（Aurora MySQL v1 → v3 変更点でもピックアップ）
    - 個々の ENUM または SET カラム要素の長さが 255 文字または 1020 バイトを超えるテーブルまたはストアドプロシージャは NG に
    - 数値データ型の`ZEROFILL`属性が非推奨に
    - 内部一時テーブルの変更（Aurora MySQL v1 → v3 変更点でもピックアップ）
    - 明示的に定義されたカラム名が 64 文字を超えるビューは NG に
    - 文字列データ型の`BINARY`属性が非推奨に

## 調査対象外

- **`INFORMATION_SCHEMA`・パフォーマンススキーマ・`sys`スキーマの各テーブル・ビュー**
  - 大幅に変更されており、利用対象のテーブル等をピンポイントで調べたほうが効率が良いため
    - https://dev.mysql.com/doc/refman/8.0/ja/information-schema.html
    - https://dev.mysql.com/doc/refman/8.0/ja/performance-schema.html
    - https://dev.mysql.com/doc/refman/8.0/ja/sys-schema.html
  - なお、Aurora MySQL 3.01 時点では、パフォーマンスインサイト（パフォーマンススキーマの情報を利用）で取得可能なデータに異常？がある可能性も
    - https://qiita.com/hmatsu47/items/9db2ad8e8f41e44a54b7#%E3%83%91%E3%83%95%E3%82%A9%E3%83%BC%E3%83%9E%E3%83%B3%E3%82%B9%E3%82%A4%E3%83%B3%E3%82%B5%E3%82%A4%E3%83%88%E3%81%8C
