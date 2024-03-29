---
title: "対象システムの概要と全体の流れ"
---

## この章について

対象システムの概要を紹介するとともに、実際に行った移行作業について、全体の流れを振り返ります。

## 対象システムの概要

以下のような構成です。

- Java のレガシーな Web アプリケーションが複数動作
- Web サーバが 8 〜 10 台規模（EC2 / ECS Fargate）
- Aurora DB は 2 クラスタ 5 インスタンス
  - db.r5.2xlarge ✖️ 3 インスタンスのクラスタ（容量は 800GB 程度）
  - db.r5.xlarge ✖️ 2 インスタンスのクラスタ（容量は 250GB 程度）
- その他 S3・DynamoDB なども使用（移行作業とは無関係なので詳細は省略）
- バッチ処理やそれ以外の定時処理が深夜・早朝に多数

## 移行スケジュール（実績）

当初の計画では 2022/9 中旬の移行を予定していましたが、アプリケーションの修正箇所が多く、予定より 1 ヶ月遅れの 2022/10 に移行作業を実施しました。

初期の情報収集と情報の取りまとめ、AWS インフラ面の対応を SRE チーム（筆者を含む）、アプリケーションの修正を部署全体（複数の開発チームから構成）で行いました。

また、当日の移行作業は SRE チームと各開発チームメンバーの混成部隊で対応しました。

### 2022/2 〜 情報収集

- 移行ターゲットを v3 に仮決定
- Aurora MySQL v1 vs v3 差分調査
- MySQL 5.6 vs 8.0 差分調査
- その他 Aurora 固有の機能の情報収集
- 移行方法に関する情報収集

#### その他

- 集めた情報を順次 Zenn の記事として公開（2/23 〜 3/23 公開）
- バージョン間差分については GitHub リポジトリで公開

### 2022/3 〜 手元での個人確認とまとめ

- 開発環境でコード全体を調査し修正点（候補）をピックアップ
- 実際に修正して開発環境で動作確認
- 動作確認で見つけた不具合を修正
- DMS（CDC）を軽く試用

#### その他

- 開発チームにコーディング規約への反映を働きかけ
- Zenn の記事を本として再構成（4/5 公開）

### 2022/4 〜 SRE チーム内で動作確認・見つかった不具合を修正

- テスト環境で v1 vs v3 の動作比較ができる状態に
- 小規模（3 〜 4 人）で動作確認
- 動作確認で見つけた不具合を修正

#### その他

- ここから随時 Zenn の記事と本に加筆修正
- 一部の事象は新規の記事として Qiita・Zenn の記事として公開

### 2022/5 〜 アプリケーション修正の全社展開

- 調査・確認のポイントと、その時点で発見済みの修正箇所と修正方法を部署全体に共有
- `ORDER BY`が一意でない箇所が大量 → 修正／見送りの線引きを検討

### 2022/6 〜 アプリケーション修正着手・並行して移行手順の検討と手順書の仮作成

- 部署全体（各開発チーム）で修正点の再調査と修正
  - 本格的な修正は 2022/8 頃からスタート
- 移行に必要な作業をリストアップ
  - 先にまとめた情報をもとに
- DMS（CDC）をテスト環境で試用
- データ整合確認用の`mysqldump`スクリプト作成
- プレリハーサルを行い、その結果を受けて移行手順書を仮作成

### 2022/8 〜 リハーサル 1 回目・一部手順変更および手順書修正

- SRE チームメンバーのみでリハーサル 1 回目を実施
  - リハーサル前にテスト環境のデータで試したときには見つからなかった、DMS（CDC）レプリケーション時のデータ不整合問題が発覚
    - 手順を見直し（binlog レプリケーションに変更）、移行手順書を修正

### 2022/9 〜 リハーサル 2 回目・性能試験

- 実際の作業メンバーを集めてリハーサル 2 回目を実施
- 本番相当のデータ・負荷をベースに性能試験を実施
  - 性能面の不安 → 対処を検討

### 2022/10 修正版アプリケーションリリース・移行作業実施

- 修正版アプリケーションは移行作業に先行してリリース
- 移行のタイミングで、2 つあるうちの一方の DB クラスタはスケールアップ、もう一方の DB クラスタは Reader 活用で性能問題に対処
