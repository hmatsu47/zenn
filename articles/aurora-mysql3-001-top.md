---
title: "Amazon Aurora MySQL v1（5.6 互換）→ v3（8.0 互換）移行を計画する（1）はじめに"
emoji: "🌌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["aws", "aurora", "mysql", "移行", "バージョンアップ"]
published: true
---

Amazon Aurora MySQL 互換エディション（以降「Aurora MySQL」と表記）のバージョン 1（MySQL 5.6 互換・以降「v1」）の EoL が発表されました。

https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/AuroraUserGuide/Aurora.MySQL56.EOL.html

ここで **「v2（MySQL 5.7 互換）への移行でお茶を濁す」** ことも考えられますが、本家 MySQL 5.7 の EoL も 2023/10/21 に迫っています。

https://endoflife.software/applications/databases/mysql

仮に今回の v1（MySQL 5.6）同様だとすると Aurora MySQL v2 の EoL は本家の 2 年後の 2025/10 あたりでしょうか。ちょっと微妙ですね。

:::message
こちらを見ると、2024/2/29 より前に EoL になることはなさそうですが、v1 の日付から類推すると 2024/7/31 あたりに EoL になりそう、とも…？

- https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/AuroraUserGuide/Aurora.VersionPolicy.html#Aurora.VersionPolicy.MajorVersionLifetime
  :::

というわけで、**これから数週間の予定で Aurora MySQL v3（MySQL 8.0 互換）移行に必要そうな情報をかき集めてみます。**

:::message
2022/2/23 現在、Aurora MySQL v3 の MySQL ベースバージョンは 8.0.23 です。
:::

:::message
**こちらの記事群を本にまとめました。**
**[Aurora MySQL v1 → v3（3.02.1 移行計画編）](https://zenn.dev/hmatsu47/books/aurora-mysql3-plan-book)**
:::

## そもそも Aurora MySQL v3 に移行するメリットは？

Aurora MySQL v2 の EoL 予想時期が微妙だとして、それだけの理由でいきなり v3 への移行を進めようとすると、

- まだ 3 年以上あるなら大丈夫じゃない？
- いきなりバージョン飛ばしするのはリスク大きいんじゃない？

という声に負けて頓挫するかもしれません。

そのような声に負けないように（？）あらかじめ知っておくと良さそうな点をまとめてみました。

:::message
本家 MySQL 8.0 で追加・強化された機能については以下の資料にまとめてあります。
:::
https://speakerdeck.com/hmatsu47/mysql-8-dot-0hefalseyi-xing-wokao-eru
https://github.com/hmatsu47/mysql80_no_usui_hon

### 新機能が追加されている

代表的なものを挙げます。

- ウィンドウ関数
  - https://qiita.com/hmatsu47/items/6cc0e69f3895f3e4a486
- 共通テーブル式（CTE・`WITH`句）
  - https://qiita.com/hmatsu47/items/01211556089b19913d05
- `CHECK`制約
  - https://qiita.com/hmatsu47/items/7526b5a4bfdc346b158c
- ロールによる権限管理
  - https://qiita.com/hmatsu47/items/e4a49d32685220d492a9

### 既存の機能が強化されている

同様に、いくつか挙げます。

- インデックス関連
  - 降順インデックス
    - https://qiita.com/hmatsu47/items/8c5e7abe204f7ecc5084
  - 関数＆式インデックス
  - 不可視インデックス
- JSON 機能
  - `JSON_TABLE()`関数
  - 複数値インデックス
    - https://qiita.com/hmatsu47/items/3e49a473bc36aeefc706
  - 行更新の内部処理効率化
  - ~~ドキュメントストアとしての利用~~
    - ~~https://qiita.com/hmatsu47/items/2de98cd0c9472e72a52a~~
    - Aurora MySQL v3 は X プラグインをサポートしていませんでした…残念
- GIS 機能
  - SRID 対応
    - https://qiita.com/hmatsu47/items/97839fd9c3db1d2e9557

### （ワークロードによっては）高速化が期待できる

実行計画のバリエーションが増えたため、ワークロードによっては高速化する可能性があります。

- （本家 MySQL 8.0 ベースの）ハッシュジョイン対応
  - https://qiita.com/hmatsu47/items/e9d3d4396fea42c8960e
- アンチジョインの効率化
- ヒストグラム対応
  - https://qiita.com/hmatsu47/items/3cfc6762bca766c5d9a1

また、後日続きの記事で触れる予定ですが、Aurora 独自実装のパラレルクエリの適用範囲が広がっています。

## v3 移行で気を付けなければならないポイントは？

v1（MySQL 5.6）→ v2（MySQL 5.7）と比べても**要配慮な変更点が多い**です。

現時点で頭の中に浮かんでいるものだけを挙げても、

- 過去のバージョンとの非互換
  - デフォルト設定の変更
    - 例：SQL モード・文字セット・照合順序・認証プラグイン
  - デフォルト動作の変更
    - 例：暗黙のソート順・`GROUP BY`でのソート
  - 機能追加の影響
    - 例：予約語の追加 → 既存アプリケーションで使用しているデータベース（スキーマ）名・テーブル名・列名などとのバッティング
      - https://qiita.com/hmatsu47/items/a1da0e06f0597acd6502
  - 機能削除（または非推奨）の影響
    - 例：クエリキャッシュ廃止
  - 新機能への移行による影響
    - 例：ロック確認用`INFORMATION_SCHEMA`スキーマ・`sys`スキーマ内テーブル＆ビュー変更
      - https://qiita.com/hmatsu47/items/607d176e885f098262e8
- 性能の変化
  - CPU コア数が少ないインスタンスでは以前のバージョンより性能が劣化する可能性がある
    - スケール重視の設計に変わってきているため
  - 過去バージョンでクエリキャッシュが有効に働いていたケースでは性能低下の可能性がある
  - 認証方式と暗号化の見直しを行う場合はコネクションのオーバーヘッド増加で性能低下の可能性がある
- Aurora 固有の事情・制約
  - パラメータグループの初期値・項目の変更がある
  - 現時点では v3 へのインプレースアップグレードができない
  - 現時点ではバックトラックが使えない

などがあります。

今後の記事で順次触れていく予定です。

:::message
必ずしも ↑ に記した順番に触れるわけではありません。
また、ここに挙げていない項目についても取り上げます。
:::

---

- **[Amazon Aurora MySQL v1（5.6 互換）→ v3（8.0 互換）移行を計画する（2）調査の進め方と参考資料](/hmatsu47/articles/aurora-mysql3-002-ref-material)**

に続きます。
