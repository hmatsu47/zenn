---
title: "MySQL 8.0 で SELECT COUNT(*) が失速する"
emoji: "😫"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["mysql", "aws", "aurora", "rds"]
published: true
---

MySQL 8.0 では、テーブル全件に対する`SELECT COUNT(*)`がパラレルスキャンによって高速化されましたが、MySQL 5.7 以前よりも遅くなるケースが発生したので、少し調べてみました。

:::message
これは、Aurora MySQL v1 → v3 の移行準備中、大きめのテーブルの整合性確認に`SELECT * ... LIMIT n`と`SELECT COUNT(*)`を利用している際に偶然見つけた問題です。
当初は Aurora MySQL v3（3.02.0 : MySQL 8.0.23 ベース）で調査していましたが、MySQL 8.0.23 〜 8.0.29 でも同様の状況になりそうなことがわかったので、あらためて RDS for MySQL 8.0.28 でテストし直したものです。
:::

### タイトルの元ネタ

https://atsuizo.hatenadiary.jp/entry/2019/01/23/112608

## テストの内容

- インスタンスを再起動してバッファプールをクリアした状態から、3 つあるテーブルの全件`SELECT COUNT(*)`を続けて実行し、実行時間を計測
  - ①〜④ ではテストデータのサイズ＞バッファプールのサイズ
  - ⑤ ではテストデータのサイズ＜バッファプールサイズ
- テストケースは 5 つ
  - **①：`test_short` (\*1) →`test_long` (\*2) →`test_long_uk` (\*3) の順に、`SELECT COUNT(*) FROM テーブル名`をそれぞれ 10 回以上連続で実行する**
    - `test_short`→`test_long`→`test_long_uk`を 1 セットとして 10 回繰り返すのではなく、`test_short`を 10 回繰り返してから`test_long`を 10 回繰り返す、という意味（以降同じ）
  - **②：`test_short`→`test_long_uk`→`test_long`の順に`SELECT COUNT(*) FROM テーブル名`をそれぞれ 10 回以上連続で実行する**
  - **③：`innodb_parallel_read_threads`を`1`に変更して、① と同じ順に`SELECT COUNT(*) FROM テーブル名`をそれぞれ 10 回以上連続で実行する**
    - 並列数 1 になるように設定を変更
  - **④：① と同じ順に`SELECT COUNT(*) FROM テーブル名 WHERE id > 0`をそれぞれ 10 回以上連続で実行する**
    - パラレルスキャンにならない SQL 文に変更
  - **⑤：インスタンスタイプを r6g.2xlarge に変更して ① と同じ順に`SELECT COUNT(*) FROM テーブル名`をそれぞれ 10 回以上連続で実行する**

(*1) `BLOB`を含まないテーブル
(*2) `BLOB`を含む大容量のテーブル（10GiB 超）
(\*3) `BLOB`を含みセカンダリインデックスを持つ大容量のテーブル（同上）

## テスト環境

- AWS の RDS for MySQL 8.0.28
- インスタンスタイプはテスト ①〜④ が db.r6g.large（2vCPU・Mem 16GiB）、テスト ⑤ が db.r6g.2xlarge（8vCPU・Mem 64GiB）
- ストレージは gp2 の 700GB
- Single-AZ

## テストデータ

以下の方法で生成しました。全体で約 26 GiB あります。

```sql:データ生成
CREATE DATABASE count_test;
USE count_test;

CREATE TABLE test_short (id INT PRIMARY KEY, val1 INT NOT NULL, val2 INT NOT NULL);
CREATE TABLE test_long (id INT PRIMARY KEY, val1 INT NOT NULL, val2 INT NOT NULL, str1 TEXT);
CREATE TABLE test_long_uk (id INT PRIMARY KEY, val1 INT NOT NULL, val2 INT NOT NULL, str1 TEXT, UNIQUE (val1));

INSERT INTO test_short SET id = 1, val1 = 1, val2 = FLOOR(RAND() * 10);
INSERT INTO test_short SET id = 2, val1 = 2, val2 = FLOOR(RAND() * 10);
（中略）
INSERT INTO test_short SET id = 10, val1 = 10, val2 = FLOOR(RAND() * 10);

INSERT INTO test_long SELECT id, val1, val2, REPEAT('a', FLOOR(100 + RAND() * 9901)) FROM test_short;
INSERT INTO test_long_uk SELECT * FROM test_long;

INSERT INTO test_short SELECT id + 10, val1 + 10, val2 FROM test_short ORDER BY id;
INSERT INTO test_long SELECT id + 10, val1 + 10, val2, str1 FROM test_long ORDER BY id;
INSERT INTO test_long_uk SELECT id + 10, val1 + 10, val2, str1 FROM test_long_uk ORDER BY id;

INSERT INTO test_short SELECT id + 20, val1 + 20, val2 FROM test_short ORDER BY id;
INSERT INTO test_long SELECT id + 20, val1 + 20, val2, str1 FROM test_long ORDER BY id;
INSERT INTO test_long_uk SELECT id + 20, val1 + 20, val2, str1 FROM test_long_uk ORDER BY id;
（中略）
INSERT INTO test_short SELECT id + 1310720, val1 + 1310720, val2 FROM test_short ORDER BY id;
INSERT INTO test_long SELECT id + 1310720, val1 + 1310720, val2, str1 FROM test_long ORDER BY id;
INSERT INTO test_long_uk SELECT id + 1310720, val1 + 1310720, val2, str1 FROM test_long_uk ORDER BY id;
```

## 結果（単位：秒）

| 実行回数 | ①short | ①long | ①long_uk | ②short | ②long_uk | ②long | ③short | ③long | ③long_uk | ④short | ④long | ④long_uk | ⑤short | ⑤long | ⑤long_uk |
| -------: | -----: | ----: | -------: | -----: | -------: | ----: | -----: | ----: | -------: | -----: | ----: | -------: | -----: | ----: | -------: |
|   1 回目 |   0.93 | 72.30 |    62.56 |   0.40 |    68.50 | 58.21 |   0.42 | 38.28 |    36.10 |   0.81 | 51.94 |    58.15 |   0.84 | 66.37 |    63.46 |
|   2 回目 |   0.08 |  0.33 |    45.13 |   0.08 |     0.34 | 46.88 |   0.34 |  0.85 |    77.66 |   0.64 | 80.50 |    77.20 |   0.04 |  0.18 |     0.17 |
|   3 回目 |   0.09 |  0.36 |    46.65 |   0.08 |     0.34 | 46.70 |   0.34 |  0.84 |    73.98 |   0.64 | 47.61 |    76.93 |   0.04 |  0.17 |     0.17 |
|   4 回目 |   0.08 |  0.32 |    52.16 |   0.09 |     0.33 | 49.13 |   0.34 |  0.85 |    17.55 |   0.63 | 22.79 |    44.45 |   0.04 |  0.18 |     0.17 |
|   5 回目 |   0.09 |  0.33 |    43.03 |   0.08 |     0.33 | 49.49 |   0.34 |  0.84 |    10.92 |   0.63 | 14.38 |    43.64 |   0.04 |  0.18 |     0.18 |
|   6 回目 |   0.08 |  0.35 |    40.14 |   0.09 |     0.33 | 51.86 |   0.34 |  0.84 |    10.23 |   0.63 | 14.28 |    44.40 |   0.04 |  0.17 |     0.17 |
|   7 回目 |   0.08 |  0.34 |    42.18 |   0.08 |     0.33 | 51.62 |   0.34 |  0.84 |     8.96 |   0.63 | 14.18 |    44.06 |   0.05 |  0.17 |     0.17 |
|   8 回目 |   0.08 |  0.33 |    43.05 |   0.09 |     0.32 | 57.11 |   0.34 |  0.85 |     1.32 |   0.64 | 14.26 |    44.60 |   0.04 |  0.17 |     0.17 |
|   9 回目 |   0.10 |  0.33 |    42.12 |   0.08 |     0.33 | 61.55 |   0.34 |  0.83 |     1.01 |   0.63 | 14.42 |    53.88 |   0.05 |  0.17 |     0.17 |
|  10 回目 |   0.11 |  0.33 |    39.24 |   0.09 |     0.33 | 55.76 |   0.34 |  0.84 |     1.00 |   0.63 | 14.27 |    51.60 |   0.04 |  0.18 |     0.18 |

全部まとめて見るのは難しいので順に見ていきます。

### パラレルスキャンかつ「テストデータのサイズ＞バッファプールのサイズ」の場合

| 実行回数 | ①short | ①long | ①long_uk | ②short | ②long_uk | ②long |
| -------: | -----: | ----: | -------: | -----: | -------: | ----: |
|   1 回目 |   0.93 | 72.30 |    62.56 |   0.40 |    68.50 | 58.21 |
|   2 回目 |   0.08 |  0.33 |    45.13 |   0.08 |     0.34 | 46.88 |
|   3 回目 |   0.09 |  0.36 |    46.65 |   0.08 |     0.34 | 46.70 |
|   4 回目 |   0.08 |  0.32 |    52.16 |   0.09 |     0.33 | 49.13 |
|   5 回目 |   0.09 |  0.33 |    43.03 |   0.08 |     0.33 | 49.49 |
|   6 回目 |   0.08 |  0.35 |    40.14 |   0.09 |     0.33 | 51.86 |
|   7 回目 |   0.08 |  0.34 |    42.18 |   0.08 |     0.33 | 51.62 |
|   8 回目 |   0.08 |  0.33 |    43.05 |   0.09 |     0.32 | 57.11 |
|   9 回目 |   0.10 |  0.33 |    42.12 |   0.08 |     0.33 | 61.55 |
|  10 回目 |   0.11 |  0.33 |    39.24 |   0.09 |     0.33 | 55.76 |

① では、`table_short`および`table_long`については 1 回目だけが遅く、2 回目以降が速い（かつ、ほぼ同じ所要時間）です。所要時間より 2 回目以降の`SELECT COUNT(*)`がバッファプールからの読み出しなのが分かります。

:::message
表にはありませんが、それぞれの所要時間は MySQL 5.7 以前と比べて 1 回目こそわずかな高速化にとどまるものの、2 回目以降は 2 倍以上高速化しています。
:::

ところが、`table_long_uk`については 2 回目にわずかに速くなったもののその後は横ばいでそれ以上高速化していません。CloudWatch のモニタリングメトリクスで確認してみたところ、2 回目以降もストレージからの読み出しが発生しており、

- **`table_long_uk`のデータページを読み出す途中でバッファプールがいっぱいになった**
- **`SELECT COUNT(*)`の処理で`table_long_uk`をバッファプールに載せる処理に何らかの問題が生じているので 2 回目以降もストレージからの読み出しが必要になっている**

ことが推察できます。

:::message
MySQL 5.7 以前ではこのような現象は発生しません。ストレージからの読み出しになる 1 回目が遅く、バッファプールに乗っている 2 回目以降は速くなります（かつ、ほぼ同じ所要時間）。
:::

続いて ② で順番を変えて`table_long`より前に`table_long_uk`を`SELECT COUNT(*)`したところ、今度は`table_long_uk`が 2 回目以降も高速化した（バッファプールからの読み出しになった）一方で、後から実行した`table_long`のほうで 2 回目以降が十分に高速化しない事象が発生しました。
やはり`table_long`・`table_long_uk`の構造に起因する問題ではなく、バッファプールがいっぱいになるタイミング以降に問題が生じていると見て間違いなさそうです。

どうやら、 **MySQL 8.0 ではテーブル全件に対する`SELECT COUNT(*)`の処理で、バッファプールがいっぱいになり古いデータページを追い出す必要がある状況下の処理に問題を抱えている様子** です。

なお、`table_long_uk`にはセカンダリインデックスがあります。MySQL 5.7 以前の動作ではセカンダリインデックスがある場合はセカンダリインデックスを全件スキャンする仕様で、 **MySQL 8.0 でも`EXPLAIN`で見る限りではプライマリインデックスの全件スキャン（＝パラレルスキャンの対象）ではなくセカンダリインデックスの全件スキャン（＝パラレルスキャンの対象外）** になっていましたが、セカンダリインデックスの有無に関わらず、同様の結果になりました。

### パラレルスキャンをやめてみる

前述のとおり「セカンダリインデックスがある`table_long_uk`についても後から`SELECT COUNT(*)`すると遅い」時点ですでに怪しい雰囲気はあるのですが、念のためパラレルスキャンをやめるとどうなるのかを確かめてみます。

パラレルスキャンをやめる方法として、

- **③：`innodb_parallel_read_threads`を`1`にする**
- **④：`innodb_parallel_read_threads`を変えずに`WHERE id > 0`を追加する**

の 2 とおりで試してみた結果が以下の表です。

| 実行回数 | ③short | ③long | ③long_uk | ④short | ④long | ④long_uk |
| -------: | -----: | ----: | -------: | -----: | ----: | -------: |
|   1 回目 |   0.42 | 38.28 |    36.10 |   0.81 | 51.94 |    58.15 |
|   2 回目 |   0.34 |  0.85 |    77.66 |   0.64 | 80.50 |    77.20 |
|   3 回目 |   0.34 |  0.84 |    73.98 |   0.64 | 47.61 |    76.93 |
|   4 回目 |   0.34 |  0.85 |    17.55 |   0.63 | 22.79 |    44.45 |
|   5 回目 |   0.34 |  0.84 |    10.92 |   0.63 | 14.38 |    43.64 |
|   6 回目 |   0.34 |  0.84 |    10.23 |   0.63 | 14.28 |    44.40 |
|   7 回目 |   0.34 |  0.84 |     8.96 |   0.63 | 14.18 |    44.06 |
|   8 回目 |   0.34 |  0.85 |     1.32 |   0.64 | 14.26 |    44.60 |
|   9 回目 |   0.34 |  0.83 |     1.01 |   0.63 | 14.42 |    53.88 |
|  10 回目 |   0.34 |  0.84 |     1.00 |   0.63 | 14.27 |    51.60 |

微妙な結果になりました。

③ については`table_long_uk`の 8 回目の`SELECT COUNT(*)`からバッファプールを上手く使える状態になっていますが、2 回目と 3 回目が 1 回目よりも遅くなっています。

④ については`table_long`も遅くなってしまいましたし、`table_long_uk`の速度が改善することはありませんでした。

:::message
Aurora MySQL v3 では`innodb_parallel_read_threads`が設定できないので ③ の方法は使えません。
（一応、接続中に`SET`はできるのですが有効に機能しないはずです）
:::

### パラレルスキャンかつ「テストデータのサイズ＜バッファプールのサイズ」の場合

最後に、インスタンスタイプを大きくして、バッファプールを十分な容量に増やしてみました。

| 実行回数 | ⑤short | ⑤long | ⑤long_uk |
| -------: | -----: | ----: | -------: |
|   1 回目 |   0.84 | 66.37 |    63.46 |
|   2 回目 |   0.04 |  0.18 |     0.17 |
|   3 回目 |   0.04 |  0.17 |     0.17 |
|   4 回目 |   0.04 |  0.18 |     0.17 |
|   5 回目 |   0.04 |  0.18 |     0.18 |
|   6 回目 |   0.04 |  0.17 |     0.17 |
|   7 回目 |   0.05 |  0.17 |     0.17 |
|   8 回目 |   0.04 |  0.17 |     0.17 |
|   9 回目 |   0.05 |  0.17 |     0.17 |
|  10 回目 |   0.04 |  0.18 |     0.18 |

必要なデータページが全部バッファプールに載る場合は何の問題もありませんでした。

:::message
結果は載せていませんが、このインスタンスタイプでもデータ容量を 4 倍にすると同じ問題が発生します。
:::

## 現時点での結論

「全データのサイズ＜バッファプールのサイズ」で運用するのは現実的ではないので、有効な回避策が見つからず困っています。

「10 GiB を超えるようなテーブルの全件`SELECT COUNT(*)`は取るな」と言われればそのとおりなのですが。

## 余談

パラレルスキャンの`SELECT COUNT(*)`については以前からいくつかのバグが見つかって修正されています。

### [MySQL 8.0.24 リリースノート](https://dev.mysql.com/doc/relnotes/mysql/8.0/en/news-8-0-24.html#mysqld-8-0-24-bug)より

> - **_InnoDB:_** On Windows, stalls were caused by concurrent SELECT COUNT(\*) queries where the number of parallel read threads exceeded the number of machine cores. **(Bug #32224707, Bug #101789)**

### [MySQL 8.0.26 リリースノート](https://dev.mysql.com/doc/relnotes/mysql/8.0/en/news-8-0-26.html#mysqld-8-0-26-bug)より

> - **_InnoDB:_** Stalls were caused by concurrent SELECT COUNT(\*) queries where the number of parallel read threads exceeded the number of machine cores. A patch for this issue was provided for Windows builds in MySQL 8.0.24. The MySQL 8.0.26 patch addresses the same issue on other affected platforms. **(Bug #32678019)**

ただし、まだ Fix されていないバグも残っているようです（2022/07/22 現在）。

https://bugs.mysql.com/bug.php?id=99717
