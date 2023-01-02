---
title: "SendGrid で送信したメールの Bounce Event を取得するサンプル"
emoji: "🐐"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["sendgrid", "aws", "lambda", "python"]
published: false
---

AWS Lambda（Python）で SendGrid を使ってメールを送信し、その Bounce Event（`Bounced`および`Dropped`）を取得するサンプルです。

## サンプルの内容

「メール送信時に送信履歴用の DynamoDB テーブルに記録した情報」と「SendGrid の Event Webhook で取得した情報」を突合し、Bounce Event に関する情報を取得するサンプルです。

今回のサンプルでは、メール送信部分を

- メール送信用の DynamoDB テーブルに送信するメールの情報をレコード登録
- DynamoDB Streams 経由でメール送信用の Lambda 関数を呼び出し
- 同 Lambda 関数でメールを送信し送信履歴テーブルにレコード登録

という流れで処理していますが、この部分はメール送信を行うアプリケーションで実装したほうが良いでしょう（わざわざ DynamoDB Streams を使って実装する必要はないと思います）。

:::message
AWS のマネジメントコンソール操作だけでテスト実行できるように、このようなサンプル実装にしています。
:::

そして、

- SendGrid の Event Webhook によって Bounce Event 取得用 API Gateway を呼び出し
- 同 API Gateway 経由で Bounce Event 取得用 Lambda 関数を実行
- 同 Lambda 関数 で受信した`Bounced`および`Dropped`のイベント情報と DynamoDB 送信履歴テーブルのレコードを突合
- 突合した結果を DynamoDB Bounce Event 用テーブルにレコード登録

という流れで Bounce Event を記録します。

[こちらの GitHub リポジトリ](https://github.com/hmatsu47/sendgrid-test) に実際のコードを置いてあります。

https://github.com/hmatsu47/sendgrid-test

## SendGrid 側の設定 (1)

### Settings - Sender Authentication（Link Branding を含む）

#### Domain Authentication / Link Branding

### Settings - API Keys

#### Create API Key

::: message
Event Webhook は Bounce Event 取得用の API Gateway 作成後に登録します。
:::

## AWS 側の設定

### Dynamo DB テーブル作成

#### メール送信用テーブル

#### メール送信履歴用テーブル

#### Bounce Event 用テーブル

### Lambda / API Gateway / IAM Role / KMS Key 作成

#### SendGrid PythonSDK を Lambda レイヤーとして登録

#### メール送信用 Lambda 関数作成

#### メール送信用 API Gateway 作成

#### Bounce Event 取得用 Lambda 関数作成

#### Bounce Event 取得用 API Gateway 作成

#### Lambda 関数用 KMS Key 作成

## SendGrid 側の設定 (2)

### Settings - Mail Settings

#### Event Settings - Event Webhook
