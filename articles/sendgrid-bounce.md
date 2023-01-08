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
Event Webhook は Bounce Event 取得用 API Gateway 作成後に登録します。
:::

## AWS 側の設定

### 1. Dynamo DB テーブルおよび KMS キーの作成

DynamoDB テーブルをいくつか作成していきます。

#### 1-1. KMS キーの作成

最初に DynamoDB テーブル暗号化用 KMS キーを作成します。

::: message
Amazon 所有キーを使う場合はキーの作成は不要です。
:::

![](/images/sendgrid-bounce/kms_key_01.png)

**「KMS」 - 「カスタマー管理型のキー」** 画面で **「キーの作成」** をクリックします。

![](/images/sendgrid-bounce/kms_key_02.png)

**「次へ」** をクリックします。

![](/images/sendgrid-bounce/kms_key_03.png)

エイリアス（キーの名前）を入力して **「次へ」** をクリックします。

![](/images/sendgrid-bounce/kms_key_04.png)

キーの管理者を選択して **「次へ」** をクリックします。

![](/images/sendgrid-bounce/kms_key_05.png)

**「次へ」** をクリックします。

::: message
キーの使用者（ユーザー）は各 Lambda 関数作成後に追加します。
:::

![](/images/sendgrid-bounce/kms_key_06.png)

最後に **「完了」** をクリックして作成完了です。

#### 1-2. メール送信用テーブルの作成

![](/images/sendgrid-bounce/dynamodb_mail_sender_01.png)

**「DynamoDB」**（**「ダッシュボード」** または **「テーブル」**）画面で **「テーブルの作成」** をクリックします。

![](/images/sendgrid-bounce/dynamodb_mail_sender_02.png)

テーブル名 : 任意（`testMailSender`など）、パーティションキー : `id`（文字列）を入力し、**「設定をカスタマイズ」** を選択します。

![](/images/sendgrid-bounce/dynamodb_mail_sender_03.png)

**「キャパシティーモード」** は **「オンデマンド」** を、**「暗号化キーの管理」** は **「アカウントからのキー」** で先ほど作成した KMS キーを選択します。

::: message
Amazon 所有キーを使う場合は **「Amazon DynamoDB が所有」** を選択します。
:::

![](/images/sendgrid-bounce/dynamodb_mail_sender_04.png)

最後に **「テーブルの作成」** をクリックして作成完了を待ちます。

::: message
DynamoDB Streams のトリガーは後で作成します。
:::

#### 1-3. メール送信履歴用テーブルの作成

引き続きメール送信履歴用テーブルを作成します。

- テーブル名 : 任意（`testMailSentLog`など）
- パーティションキー : `messageId`（文字列）

ほかの項目はメール送信用テーブルと同じです。

#### 1-4. Bounce Event 用テーブルの作成

最後に Bounce Event 用テーブルを作成します。

- テーブル名 : 任意（`testMailBounce`など）
- パーティションキー : `fullMessageId`（文字列）

ほかの項目はメール送信用テーブルと同じです。

### 2. Lambda 関数 / API Gateway / IAM Role および KMS キーの作成・設定

#### 2-1. SendGrid Python SDK を Lambda レイヤーとして登録

Amazon Linux 2 の EC2（Cloud9）を用意し、 **[SendGrid Python SDK](https://github.com/sendgrid/sendgrid-python)** を（依存パッケージを含めて）`.zip`化します。

```sh:
mkdir python
pip3 install sendgrid -t python
zip sendgrid-sdk.zip -r python
```

**「Lambda」 - 「レイヤー」** 画面で **「レイヤーの作成」** をクリックします。

![](/images/sendgrid-bounce/sendgrid_layers_01.png)

- 名前 : 任意（`sendgrid`など）
- 互換性のあるアーキテクチャ : x86_64・arm64 ともチェック
- ランタイム : Python 3.9

先ほど作った`.zip`ファイルをアップロードし、 **「作成」** ボタンをクリックします。

#### 2-2. メール送信用 Lambda 関数の作成

**「Lambda」 - 「関数」** 画面で **「関数の作成」** をクリックします。

![](/images/sendgrid-bounce/lambda_sender_01.png)

- 関数名 : 任意（`testMailSender`など）
- ランタイム : Python 3.9
- アーキテクチャ : どちらでも可（ここでは arm64 を選択）
- 実行ロール : 基本的な Lambda アクセス権限で新しいロールを作成

**「関数の作成」** ボタンをクリックします。

![](/images/sendgrid-bounce/lambda_sender_02.png)

##### 「コード」 - 「レイヤー」 - 「レイヤーの追加」ボタン :

![](/images/sendgrid-bounce/lambda_sender_04.png)

- レイヤーソース : **「カスタムレイヤー」** を選択
- カスタムレイヤー : 2-1. で作成したものを選択
  - 画面は「バージョン 2」になっているが再作成等をしていない場合は「バージョン 1」

**「追加」** ボタンをクリックします。

##### 「コード」 - 「コードソース」 : `lambda_function.py`

https://github.com/hmatsu47/sendgrid-test/blob/5c6c03beb108dcc675646f3a4a459cfe3f379acc/lambda/testMailSender/lambda_function.py

![](/images/sendgrid-bounce/lambda_sender_03.png)

**「Deploy」** ボタンをクリックします。

##### 「設定」 - 「トリガー」 - 「トリガーを追加」ボタン :

![](/images/sendgrid-bounce/lambda_sender_05.png)

- トリガーの設定 : **「DynamoDB」** を選択
- DynamoDB テーブル : メール送信用テーブルを選択
- バッチサイズ : 100

**「追加」** ボタンをクリックします。

![](/images/sendgrid-bounce/lambda_sender_06.png)

##### 「設定」 - 「一般設定」 - 「編集」ボタン :

![](/images/sendgrid-bounce/lambda_sender_07.png)

- タイムアウト : 3 分 0 秒
- 既存のロール : **「View the 【ロール名】」** のリンクをクリック

![](/images/sendgrid-bounce/lambda_sender_08.png)

ポリシーの **「編集」** ボタンをクリックします。

![](/images/sendgrid-bounce/lambda_sender_09.png)

DynamoDB に対する権限を追加（または追加されていることを確認）します。

- アクション : ListStreams／リソース : \*
- アクション : GetItem・Query・PutItem／リソース : メール送信履歴用テーブルの ARN
- アクション : DescribeStream・GetRecords・GetShardIterator／リソース : メール送信用テーブルの Stream の ARN

**「ポリシーの確認」** ボタンをクリックします。

![](/images/sendgrid-bounce/lambda_sender_10.png)

**「変更の保存」** ボタンをクリックします。

Lambda 関数の **「基本設定を編集」** 画面に戻って **「保存」** ボタンをクリックします。

:::message
環境変数は暗号化用の KMS キーを作成してから設定します。
:::

#### 2-3. Lambda 関数（メール送信）用 KMS キーの作成

1-1. と同様にキーを作成します。

- エイリアス : 任意の名前
- キーの使用者（ユーザー） : メール送信用 Lambda 関数の実行ロール

:::message
環境変数の暗号化にデフォルトのキーを使う場合は作成不要です。
:::

#### 2-4. メール送信用 Lambda 関数の環境変数を設定

メール送信用 Lambda 関数の画面で **「設定」 - 「環境変数」 - 「編集」** ボタンをクリックします。

![](/images/sendgrid-bounce/lambda_sender_11.png)

- キー :
  - `SENDGRID_API_KEY` : SendGrid 側の設定 (1) の 2. で発行した API キー
  - `TABLE_SENT_LOG` : メール送信履歴用テーブルの名前
- 転送時の暗号化 : **「転送時の暗号化に使用するヘルパーの有効化」** にチェック
- 保管時に暗号化する AWS KMS キー : **「カスタマーマスターキーの使用」** を選択
- カスタマーマスターキー : 2-3. で作成したキーを選択

キー「`SENDGRID_API_KEY`」の右側にある **「暗号化」** ボタンで値の暗号化を行います。

![](/images/sendgrid-bounce/lambda_sender_12.png)

先ほどと同じ AWS KMS キーを選択し、**「暗号化」** ボタンをクリックします。

前の画面に戻ったら **「保存」** ボタンをクリックして完了です。

#### 2-5. Bounce Event 取得用 Lambda 関数の作成

関数を作成します。

- 関数名 : 任意（`testBounceReceiver`など）
- ランタイム : Python 3.9
- アーキテクチャ : どちらでも可（ここでは arm64 を選択）
- 実行ロール : 基本的な Lambda アクセス権限で新しいロールを作成
- レイヤーの追加 : カスタムレイヤー（2-1. で作成したものを選択）
- コード : `lambda_function.py`

https://github.com/hmatsu47/sendgrid-test/blob/5c6c03beb108dcc675646f3a4a459cfe3f379acc/lambda/testBounceReceiver/lambda_function.py

- 一般設定 - タイムアウト : 30 秒
- アクセス権限 : 選択されているポリシーに DynamoDB に関する権限を追加
  - アクション : GetItem／リソース : メール送信履歴用テーブルの ARN
  - アクション : PutItem／リソース : Bounce Event 用テーブルの ARN
- 環境変数 :
  - `TABLE_BOUNCE` : Bounce Event 用テーブルの名前
  - `TABLE_SENT_LOG` : メール送信履歴用テーブルの名前

::: message
トリガーは API Gateway 作成時に設定します（Lambda 統合）。
:::

#### 2-6. Bounce Event 取得用 API Gateway の作成

![](/images/sendgrid-bounce/api_hook_receiver_01.png)

**「API Gateway」 - 「API」** 画面で **「API を作成」** をクリックします。

![](/images/sendgrid-bounce/api_hook_receiver_02.png)

**「REST API」** の **「構築」** ボタンをクリックします。

![](/images/sendgrid-bounce/api_hook_receiver_03.png)

任意の API 名（例 : 「testEventHookReceiver」を入力して **「API の作成」** ボタンをクリックします。

![](/images/sendgrid-bounce/api_hook_receiver_04.png)

**「リソース」 - 「アクション」** ボタンのメニューで **「リソースの作成」** を選択します。

![](/images/sendgrid-bounce/api_hook_receiver_05.png)

任意のリソース名（悪用されにくいようランダムで長いもの）を入力して **「リソースの作成」** ボタンをクリックします。

![](/images/sendgrid-bounce/api_hook_receiver_06.png)

同じリソースの **「アクション」** ボタンのメニューで **「メソッドの作成」** を選択します。

![](/images/sendgrid-bounce/api_hook_receiver_07.png)

**POST** メソッドを選択します。

- 統合タイプ : Lambda 関数
- Lambda プロキシ統合の使用 : チェック
- Lambda 関数 : Bounce Event 取得用 Lambda 関数の名前

**「保存」** ボタンをクリックします。

![](/images/sendgrid-bounce/api_hook_receiver_08.png)

**「OK」** ボタンをクリックします。

![](/images/sendgrid-bounce/api_hook_receiver_09.png)

**「メソッドレスポンス」** のリンクをクリックします。

![](/images/sendgrid-bounce/api_hook_receiver_10.png)

**「レスポンスの追加」** をクリックします。

![](/images/sendgrid-bounce/api_hook_receiver_11.png)

ステータス **500** のレスポンスを追加（作成）します（**チェックアイコン** をクリック）。

![](/images/sendgrid-bounce/api_hook_receiver_12.png)

**「500 のレスポンス本文」** を登録します。

- コンテンツタイプ : application/json
- モデル : Empty

**チェックアイコン** をクリックして登録（保存）します。

![](/images/sendgrid-bounce/api_hook_receiver_13.png)

同じリソースの **「アクション」** ボタンのメニューで **「API のデプロイ」** を選択します。

![](/images/sendgrid-bounce/api_hook_receiver_14.png)

- デプロイされるステージ : \[新しいステージ\]
- ステージ名 : v1

**「デプロイ」** ボタンをクリックして API をデプロイします。

#### 2-7. Dynamo DB テーブル用 KMS キーにユーザー（IAM Role）を追加

## SendGrid 側の設定 (2)

### 1. Event Webhook の設定

- Settings - Mail Settings

- Event Settings - Event Webhook
