---
title: "ALBとCognitoを利用してWebアプリケーションにアクセス制限を設ける"
emoji: "🦔"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["terraform","cognito"]
published: true
---

私的に使いこなせたら強そうなAWSのサービスシリーズにCognitoがランキング上位にあります。

認証・認可って大体のアプリケーションに必要ですし、AWSのマネージドサービスに投げられれば非常に楽ですよね。

まぁ私は全然使いこなせてないのですが。

私はCognitoの認証・認可の機能を利用するには、Webアプリケーション側でのコード変更が必須と考えていましたが、こちらの記事を見て、ALBとCognitoを連携させて簡単に認証・認可の機能を利用できることを知りました。

https://docs.aws.amazon.com/ja_jp/elasticloadbalancing/latest/application/listener-authenticate-users.html

以下のようなユースケースですっと導入できれば、できるエンジニア感が出せるなと思いました。

- ALBの背後でウェブアプリケーションを動作させている
- 開発環境やステージング環境で、アクセス制限を導入したい
  - IPアドレスによるセキュリティグループでの制限が難しい
- Basic認証のように既存のウェブアプリケーションに変更を加えられない

## アプリケーション構成について

![構成図](/images/cognito_alb/system_configuration.png)

ALBの背後にECSでWebアプリケーションを動かします。

このアプリケーションは私がよくサンプルとして利用するテキトーなものです。
(適当ではない)

## 実装

今回はTerraformで環境構築を行いました。

やってみたリポジトリはこちらです。

https://github.com/bun913/cognito_alb_practice

全てを紹介することは難しいので、実装のポイントとなった部分を紹介します。

【参考にした記事】

https://oji-cloud.net/2021/11/08/post-6634/

### 事前に押さえておきたいポイント

公式に記載してくれていますので、確認しましょう。

https://docs.aws.amazon.com/ja_jp/elasticloadbalancing/latest/application/listener-authenticate-users.html#cognito-requirements

私的に重要だと思ったのが、以下のポイントです

- 独自ドメインが必要
  - `hoge.com` のようにALBに独自ドメインを付与
  - `auth.hoge.com` のようにCognitoで利用するサブドメインを利用する
    - `hoge.com` に対してRoute53でDNSレコードを設定できる必要がある
      - ALBに設定するドメイン(`hoge.com`)に対して、AレコードかつエイリアスレコードでDNS設定が必要であるため
- HTTPSで通信できる必要がある
  - ACMで独自ドメインに対しての証明書を作成するのが楽
  - `*.hoge.com` および `hoge.com` は任意のリージョンで(私は ap-northeast-1を利用)
  - `auth.hoge.com` は `us-east-1` で証明書をリクエストする必要がある
    - Cognitoで利用する際にこのリージョンで作成しておく必要があるとのこと

### ACMの証明書を作成・Route53で検証用のレコードを作成

modules/cert

```tf
# auth.hoge.comのサブドメインはus-east-1のリージョンで証明書を発行する
provider "aws" {
  alias  = "virginia"
  region = "us-east-1"
}

# ルートドメイン(hoge.com)の証明書発行
resource "aws_acm_certificate" "cert" {
  domain_name               = "*.${var.root_domain}"
  subject_alternative_names = [var.root_domain]
  validation_method         = "DNS"
  lifecycle {
    create_before_destroy = true
  }
}

# サブドメイン(auth.hoge.com)の証明書発行
resource "aws_acm_certificate" "cert_sub" {
  domain_name       = "auth.${var.root_domain}"
  validation_method = "DNS"
  provider          = aws.virginia
  lifecycle {
    create_before_destroy = true
  }
}

```

modules/dns

```tf
# ACMのドメイン検証
resource "aws_route53_record" "cert_validation_main" {
  for_each = {
    for dvo in var.acm_main_domain_valid_options : dvo.domain_name => {
      name   = dvo.resource_record_name
      record = dvo.resource_record_value
      type   = dvo.resource_record_type
    }
  }

  allow_overwrite = true
  name            = each.value.name
  records         = [each.value.record]
  type            = each.value.type
  ttl             = "300"

  zone_id = var.host_zone_id
}

resource "aws_route53_record" "cert_validation_sub" {
  for_each = {
    for dvo in var.acm_sub_domain_valid_options : dvo.domain_name => {
      name   = dvo.resource_record_name
      record = dvo.resource_record_value
      type   = dvo.resource_record_type
    }
  }

  allow_overwrite = true
  name            = each.value.name
  records         = [each.value.record]
  type            = each.value.type
  ttl             = "300"

  zone_id = var.host_zone_id
}
```

### Cognitoの各種リソース作成

modules/auth

```tf
resource "aws_cognito_user_pool" "main" {
  name = "${var.prefix}-auth"

  # パスワード認証だけで良いなら OFFにする
  mfa_configuration = "ON"
  software_token_mfa_configuration {
    enabled = true
  }

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
      email_subject = "${var.tags.Project}(ステージング環境)への招待"
      sms_message   = "{username}さん、あなたの初期パスワードは {####} です。初回ログインの後パスワード変更が必要です。"
    }
  }

  # ユーザー名の他にemailでの認証を許可
  alias_attributes = ["email"]

  tags = var.tags
}

resource "aws_cognito_user_pool_domain" "main" {
  domain          = "auth.${var.root_domain}"
  certificate_arn = var.acm_sub_arn
  user_pool_id    = aws_cognito_user_pool.main.id
}

resource "aws_cognito_user_pool_client" "main" {
  name            = "${var.prefix}-client"
  user_pool_id    = aws_cognito_user_pool.main.id
  generate_secret = true
  # CallBackUrlにALBのドメイン + oauth2/idpresponseの付与が必要
  callback_urls = [
    "https://${var.root_domain}/oauth2/idpresponse"
  ]
  allowed_oauth_flows = ["code"]
  explicit_auth_flows = [
    "ALLOW_REFRESH_TOKEN_AUTH",
    "ALLOW_USER_PASSWORD_AUTH",
  ]
  supported_identity_providers = [
    "COGNITO"
  ]
  allowed_oauth_scopes                 = ["openid"]
  allowed_oauth_flows_user_pool_client = true
}

```

### ALBの作成とリスナーの作成

modles/web_app

```tf
resource "aws_lb" "app" {
  name               = "${var.prefix}-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.ingress_all.id]
  subnets            = var.public_subnets

  # NOTE: デモ用なので一時的に削除保護を無効にしている
  enable_deletion_protection = false

  # NOTE: デモ用なのでアクセスログは吐き出さない
  /* access_logs { */
  /*   bucket  = aws_s3_bucket.lb_logs.bucket */
  /*   prefix  = "test-lb" */
  /*   enabled = true */
  /* } */
  tags = var.tags
}

resource "aws_lb_listener" "http_blue" {
  load_balancer_arn = aws_lb.app.arn
  port              = "80"
  protocol          = "HTTP"
  default_action {
    type = "redirect"
    redirect {
      port        = "443"
      protocol    = "HTTPS"
      status_code = "HTTP_301"
    }
  }
  tags = merge({ "Name" : "${var.prefix}-blue" }, var.tags)
  # BGデプロイで動的にtgを入れ替えるため変更を無視
  lifecycle {
    ignore_changes = [
      default_action
    ]
  }
}

resource "aws_lb_listener" "https_blue" {
  load_balancer_arn = aws_lb.app.arn
  port              = "443"
  protocol          = "HTTPS"
  ssl_policy        = "ELBSecurityPolicy-2016-08"
  certificate_arn   = var.acm_arn

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.app_blue.arn
  }
}

resource "aws_lb_listener_rule" "auth" {
  listener_arn = aws_lb_listener.https_blue.arn
  priority     = 100
  action {
    type = "authenticate-cognito"

    authenticate_cognito {
      user_pool_arn       = var.user_pool_arn
      user_pool_client_id = var.cognito_client_id
      user_pool_domain    = var.cognito_domain
    }
  }
  action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.app_blue.arn
  }
  condition {
    path_pattern {
      values = ["/*"]
    }
  }
}

resource "aws_lb_target_group" "app_blue" {
  name                 = "${var.prefix}-tg-blue"
  deregistration_delay = 60
  port                 = 8080
  protocol             = "HTTP"
  target_type          = "ip"
  vpc_id               = var.vpc_id
  health_check {
    healthy_threshold   = 2
    interval            = 30
    matcher             = 200
    path                = "/"
    port                = "traffic-port"
    protocol            = "HTTP"
    timeout             = 5
    unhealthy_threshold = 2
  }
  tags = var.tags
}

```

## 結果

これで `hoge.com` に対してブラウザでアクセスを行うと、以下のようにCognitoの認証用のUIが表示されます。

![CognitoのUI](/images/cognito_alb/cognito_ui.png)

あとは適当に Cognitoユーザープールでユーザーを作成して、そのユーザーでログインすることで、パスワードの変更後、無事ALBの配下で動作しているECSのアプリケーションにアクセスすることができました。

![アプリ](/images/cognito_alb/color_app.png)

正直Cognitoの細かいパラメーター・そもそもの認証・認可に対する知識が全く足りていないと思ったので、別途学習しながら記事にしたいと思います。