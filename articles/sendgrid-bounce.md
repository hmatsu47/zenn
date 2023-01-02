---
title: "SendGrid で送信したメールの Bounce Event を取得するサンプル"
emoji: "🐐"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["sendgrid", "aws", "lambda", "python"]
published: false
---

AWS Lambda（Python）で SendGrid を使ってメールを送信し、その Bounce Event（`Bounced`および`Dropped`）を取得するサンプルです。

## サンプルの内容

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

#### 送信履歴用テーブル

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
