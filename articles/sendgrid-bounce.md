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

### 1. Domain Authentication / Link Branding の設定

![](/images/sendgrid-bounce/sender_authentication_01.png)

左側メニューから **「Settings - Sender Authentication」** を選択します。

![](/images/sendgrid-bounce/sender_authentication_02.png)

**Authenticate Your Domain** をクリックします。

![](/images/sendgrid-bounce/sender_authentication_03.png)

① レコードを登録する DNS host（どれにも当てはまらない場合は **「Other Host (Not Listed)」**）を選択します。
② メール送信時に送信元ドメインを「sendgrid.net」ではなくカスタムドメインに書き換える場合は **「Yes」** を選択します。

その後 **「Next」** をクリックします。

![](/images/sendgrid-bounce/sender_authentication_04.png)

送信元ドメインを入力し、必要に応じて **「Advanced Settings」** の各項目を選択・入力します。

その後 **「Next」** をクリックします。

![](/images/sendgrid-bounce/sender_authentication_05.png)

① 表示された CNAME レコードを DNS に登録します。

（↓ は AWS の Route 53 の例）

![](/images/sendgrid-bounce/sender_authentication_06.png)

登録したら SendGrid の画面に戻り、

![](/images/sendgrid-bounce/sender_authentication_07.png)

② **「I've added these records.」** にチェックを入れて **「Verify」** をクリックします。

![](/images/sendgrid-bounce/sender_authentication_08.png)

以上で Domain Authentication および Link Branding の作業（設定）は完了です。

### 2. API Keys の作成

メール送信用の API キーを作成します。

![](/images/sendgrid-bounce/api_keys_01.png)

左側メニューから **「Settings - API Keys」** を選択します。

![](/images/sendgrid-bounce/api_keys_02.png)

**「Create API Key」** をクリックします。

![](/images/sendgrid-bounce/api_keys_03.png)

キーの名前（任意の名前）を入力し、**「Restricted Access」** を選択して **「Mail Send」** の詳細項目のほうの **「Mail Send」** のみ有効にします。

![](/images/sendgrid-bounce/api_keys_04.png)

その後 **「Create & View」** をクリックします。

![](/images/sendgrid-bounce/api_keys_05.png)

表示された API キー（「SG.」で始まる文字列）をクリックするとクリップボードにコピーされます。

どこかに API キーを控え、**「Done」** をクリックして完了です。

::: message
Event Webhook は Bounce Event 取得用の API Gateway 作成後に登録します。
:::

## AWS 側の設定

### 1. Dynamo DB テーブルの作成

#### 1-1. メール送信用テーブルの作成

#### 1-2. メール送信履歴用テーブルの作成

#### 1-3. Bounce Event 用テーブルの作成

### 2. Lambda / API Gateway / IAM Role / KMS Key の作成・設定

#### 2-1. SendGrid Python SDK を Lambda レイヤーとして登録

#### 2-2. メール送信用 Lambda 関数の作成

#### 2-3. メール送信用 API Gateway の作成

#### 2-4. Bounce Event 取得用 Lambda 関数の作成

#### 2-5. Bounce Event 取得用 API Gateway の作成

#### 2-6. Lambda 関数用 KMS Key の作成

## SendGrid 側の設定 (2)

### 1. Event Webhook の設定

- Settings - Mail Settings

- Event Settings - Event Webhook
