---
title: "Terraformを使ったAWS CognitoUserPoolによるGoogle認証の導入"
emoji: "✨"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["aws", "cognito", "terraform"]
published: true
---

前回ALBとCognitoを利用してWebアプリケーションにアクセス制限を設けることに関して記事を書きました。

https://zenn.dev/bun913/articles/alb-cognito-auth

これにより私の中での「Cognitoすげー！」が増大していき、今度はよくあるOAuthによるソーシャルIDログインをできる基盤を作りたいと思い、Terraformにより実装してみました。

![イメージ](/images/cognito_google_auth/system_configuration.png)

今回すること

- GCPでGoogleのOAuthに必要なリソースの作成(手動)
    - Terraformで利用できるようなAPIがないようであるため
- TerraformでCognitoユーザープールのリソースを作成
- ホストされたUIでGoogleログインができることを確認


今回しないこと

- クライアントアプリで独自UIを作成して、Google認証を実装
- CognitoのIDプールを活用して、AWSリソースへの認可
  - 別途記事を作成する予定

## 下準備(GoogleのOAuth準備)

こちらについて、早速丸投げですが、クラスメソッド様の記事が非常にわかりやすかったです。

以下記事の「Google認証設定」の作業を行なって、ClientIDとClientSecretを取得してください。

https://dev.classmethod.jp/articles/amazon-cognito-google-social-signin/#toc-2

## CognitoUserPoolの作成

以下、Terraformを利用してCognitoユーザープールのリソースを作成します。

ポイントとなる箇所だけ抜粋しますので、全コードはこちらのリポジトリを参照ください。

https://github.com/bun913/cognito_id_pool_practice


```tf:modules/auth/main.tf
resource "aws_cognito_user_pool" "main" {
  name = "${var.prefix}-auth"

  mfa_configuration = "OFF"

  account_recovery_setting {
    recovery_mechanism {
      name     = "verified_phone_number"
      priority = 1
    }
  }

  admin_create_user_config {
    allow_admin_create_user_only = true
    invite_message_template {
      email_message = "{username}さん、あなたの初期パスワードは {####} です。初回ログインの後パスワード変更が必要です。"
      email_subject = "${var.tags.Project}(開発環境)への招待"
      sms_message   = "{username}さん、あなたの初期パスワードは {####} です。初回ログインの後パスワード変更が必要です。"
    }
  }

  # ユーザー名の他にemailでの認証を許可
  alias_attributes = ["email"]

  tags = var.tags
}

resource "aws_cognito_user_pool_domain" "main" {
  # 今回独自ドメインは利用せず、 `ap-northeast-1.amazoncognito.com` のドメインを利用
  domain       = "test-cogn-social-login-bun"
  user_pool_id = aws_cognito_user_pool.main.id
}

resource "aws_cognito_user_pool_client" "main" {
  name            = "${var.prefix}-client"
  user_pool_id    = aws_cognito_user_pool.main.id
  generate_secret = true
  # CallBackUrlにローカルで別途動かしているアプリケーションを指定
  callback_urls = [
    "http://localhost:8080/"
  ]
  allowed_oauth_flows = ["code"]
  explicit_auth_flows = [
    "ALLOW_REFRESH_TOKEN_AUTH",
    "ALLOW_USER_SRP_AUTH",
  ]
  supported_identity_providers = [
    "COGNITO",
    "Google"
  ]
  allowed_oauth_scopes                 = ["openid"]
  allowed_oauth_flows_user_pool_client = true
}

resource "aws_cognito_identity_provider" "google_provider" {
  user_pool_id  = aws_cognito_user_pool.main.id
  provider_name = "Google"
  provider_type = "Google"

  provider_details = {
    authorize_scopes = "email"
    client_id        = var.google_client_id
    client_secret    = var.google_client_secret
  }

  attribute_mapping = {
    email    = "email"
    username = "sub"
  }
}

```

variables.tf に変数の情報を記載しておりますが

特に `google_client_id` および `google_client_secret` の指定を忘れないようにお願いします。

私は `terraform.tfvars` というファイルを作成して、そこに上記のGoogle認証の準備作業で取得したクライアントIDとクライアントシークレットの情報を格納しています。
(機密情報のためGit管理外)


## ホストされたUIから確認

以下手順で、Terraformを実行してCongitoのリソースを作成します。

```bash
terraform init
terraform plan
terraform apply
```

次にAWSマネジメントコンソールに移動し、以下手順でホストされたUIを確認します。
（2022年3月23日現在の新UIでの手順）

- Cognitoのサービス画面から上記で作成したユーザープールを選択
- ユーザープールの詳細画面の「アプリケーションの統合」タブを選択
- 「アプリケーションクライアントのリスト」から上記で作成したアプリケーションクライアントを選択
- ホストされたUIから「ホストされたUIを表示」をクリック

![ホストされたUI](/images/cognito_google_auth/host_ui.png)

以下のようにホストされたUIが表示されるはずなので、GoogleのIDで認証できるか確かめましょう

![Google認証確認](/images/cognito_google_auth/google_ui.png)