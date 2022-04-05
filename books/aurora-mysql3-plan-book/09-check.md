---
title: "動作検証"
---
# この章について

パラメータグループやアプリケーション・コード修正後の動作検証について記します。

# 検証内容

- **a. 動作に不具合がないか（移行前後で動作が変わらないか）**
- **b. 性能が低下しないか**

を検証します。

また、移行作業にレプリケーション（DMS の CDC）を使う場合は、動作検証後に

- **c. レプリケーションのテスト**

も行います。

# 検証環境構築のポイント

## 実際の移行方法を意識する

検証環境構築は、

- **スナップショット復元（v1 → v2、v2 → v3）**
- **インプレースアップグレード（v1 → v2）＋スナップショット復元（v2 → v3）**

のように、実際の移行作業で使う方法を意識して進めるのが望ましいです。

a.（不具合の有無の検証）では必ずしも実容量を意識してテストデータを用意する必要はありませんが、移行の方法やパスを合わせておくことで、早い段階で作業上の問題点を顕在化させることができます。

私が関わっているサービスの場合、以下のような問題点とその原因、および対処法を見つけることができました。

- **[管理者権限が移行されない](https://qiita.com/hmatsu47/items/d3f34f39c28a4b802966#%E7%99%BA%E7%94%9F%E3%81%97%E3%81%9F%E5%95%8F%E9%A1%8C%E3%81%9D%E3%81%AE-1--%E7%AE%A1%E7%90%86%E8%80%85%E6%A8%A9%E9%99%90%E3%81%8C%E7%A7%BB%E8%A1%8C%E3%81%95%E3%82%8C%E3%81%AA%E3%81%84)**
- **[一部のユーザの権限が無効になる（アクセスが拒否される）](https://qiita.com/hmatsu47/items/d3f34f39c28a4b802966#%E7%99%BA%E7%94%9F%E3%81%97%E3%81%9F%E5%95%8F%E9%A1%8C%E3%81%9D%E3%81%AE-2--%E4%B8%80%E9%83%A8%E3%81%AE%E3%83%A6%E3%83%BC%E3%82%B6%E3%81%AE%E6%A8%A9%E9%99%90%E3%81%8C%E7%84%A1%E5%8A%B9%E3%81%AB%E3%81%AA%E3%82%8B%E3%82%A2%E3%82%AF%E3%82%BB%E3%82%B9%E3%81%8C%E6%8B%92%E5%90%A6%E3%81%95%E3%82%8C%E3%82%8B)**

なお、b.（性能検証）ではできるだけ実容量を意識してテストデータを用意します。

:::message
検証環境は、a. と b. で別々に用意しても同じ環境を使っても良いでしょう。
:::

## 実際の構成を意識する

a. では必ずしも実環境と同じサイズと個数のインスタンスを用意する必要はありませんが（アプリケーションコードがインスタンスサイズや個数に依存している場合を除いて）、Reader インスタンスを使用する構成であれば必ず Reader インスタンスを用意します。

これは、Chapter 05・08 に示したとおり、 **[内部テンポラリテーブルの仕様変更](https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.MySQL80.html#AuroraMySQL.mysql80-internal-temp-tables-engine)** や **[テンポラリテーブルの仕様変更](https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.MySQL80.html#AuroraMySQL.mysql80-temp-tables-readers)** などの影響が生じないか確認するのが目的です。

またその際、実際の AZ 構成を意識してインスタンスを配置します。

b. では実環境と同じサイズと個数のインスタンスを用意します。

## 移行前後で比較可能にする

a.・b. いずれも、移行前後の比較が可能なように、同じデータをもとに同じ構成の環境を v1 と v3 のそれぞれについて用意し、動作や性能を直接比較できるようにするのが望ましいです。

経験上、システム移行でアプリケーションの既存バグを見つけることが多いのですが、比較対象の環境を用意しているとそれが既存バグか動作の非互換かを判別しやすいです（後者の場合は同様の実装を行っている箇所の動作確認をすべき）。

性能検証の際も同じデータに対して同じ負荷を掛けることで、移行後に性能低下するかどうかを判別しやすいです。

:::message
Aurora MySQL v3 では本家 MySQL 8.0 由来のハッシュ結合が導入されており、この機能を使う場合は結合バッファを大きめに確保するのが望ましいです。
結合バッファのサイズを大きくするには、以下の 2 通りの方法があります。

- **パラメータグループで`join_buffer_size`を上げる**
  - アプリケーションの修正は不要だが、全体的にメモリの使用量が増える欠点がある
- **ハッシュ結合を使う SQL 文に`SET_VAR`ヒントを記述して個別に大きめの`join_buffer_size`を指定する**
  - アプリケーションの修正が必要だが、メモリの使用量増加を最小限に抑えられる
:::

## 動作検証を完了したら DMS レプリケーション（CDC）をテストする

DMS レプリケーション（CDC）を使ってメンテナンス停止時間の短縮をはかる場合は、アプリケーションの動作検証後、移行ステップを進める前にレプリケーションに支障がないかを必ず確認しておきます。

何らかの問題点が見つかった場合はそれを修正するか、別の方法での移行を検討します。

例えば、以下のような問題が見つかる可能性があります。

- **`time`型の列が移行されない**
- **`float`型の列が移行されない**

それぞれ、テーブルマッピングの変換ルールで該当列の型変換を指定します。

- `time`型（扱う値の範囲が 24 時間以内の場合）
  - MySQL の`time`型は DMS では`string`型として扱われるので`time`型に変換する
- `float`型（精度の指定がない場合）
  - DMS の`real4`型に変換する

また、

- **`blob`型の列の更新が無視される**

ケースでは、

- タスク設定で完全 LOB モードを指定しているか？
- アプリケーションで`blob`型単独で`UPDATE`処理をしている箇所がないか？

を確認し、問題があれば修正します。