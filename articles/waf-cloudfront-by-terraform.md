---
title: "AWSのWAFとCloudFrontをTerraformで導入してみました"
emoji: "🗂"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["aws","terraform"]
published: true
---

以前は以下のようにAWSにおける踏み台の構成を2つ考えて、Terraformで構築してみました。

https://zenn.dev/bun913/articles/aws-bastion-fargate-and-ec2

今回は引き続き以下クラスメソッド様が公開してくれている「2021年版 AWSセキュリティ対策全部盛り 初級から上級まで」の初級編で紹介されていた WAFとCloudFrontを経由して ALB配下で動作しているアプリケーション及び静的なファイルを配置しているS3にアクセスが届くように構築してみました。

ブログ版

https://dev.classmethod.jp/articles/aws-security-all-in-one-2021/

Yotube版

https://youtu.be/Ci_4dhUCZm8

## 概要

構成図は以下のような形になります。

![システム構成図](/images/waf_cloudfront/system_configuration.png)

- まずWAF経由でパスされた通信のみCloudFrontに届く
- CloudFrontで静的なコンテンツを示すパス /static/* に関してはS3をOriginとする
- その他のパスは ALBをOriginとする
- ALB・S3のコンテンツはCloudFrontを経由したアクセスのみ許可する
  - 直接ALBやS3のドメイン名を指定してアクセスさせない

この構成のメリット

- まずWAFで怪しいリクエストを拒否できる
- ALBに到達する前にCloudFrontを経由するのでDDOS攻撃に強い
  - 以下参考サイト(AWS公式)をご参照ください

【参考】

https://aws.amazon.com/jp/blogs/news/improve-your-website-availability-with-amazon-cloudfront/

この構成で考慮すべきポイント

- WAFのWebACLにどのようなルールを設定するか
  - 今回はAWSが用意してくれているマネージドルールのみを採用
  - サービスの種類によって、あまりルールを適用できないケースもあるか
- CloudFrontのキャッシュ戦略

## 実装概要

一部のポイントとなりそうなコードのみをZennには掲載しますので、コード全文がみたい場合はこちらのリポジトリをご参照ください。

https://github.com/bun913/aws_cloudfront_and_waf

### 事前準備

CloudFront・ALBに適用するドメインを取得する必要があります。

今回では `hoge.com` というドメインを取得していて

CloudFront: `cdn.hoge.com`
ALB: `alb.hoge.com`

というサブドメインを割り当てる前提で構築しております。

Route53で独自ドメインを取得すれば手早いですが、今回は別のドメイン取得サービスで取得したドメインを使っていて、NameServerにRoute53を利用して各種DNSレコードを作成しております。

### ALB・S3のリソース作成

事前準備で記載したように、CloudFront・ALBに適用するドメインのためにACMで証明書を発行します

```tf:modules/cert/main.tf
provider "aws" {
  alias  = "virginia"
  region = "us-east-1"
}

# 証明書発行リクエスト
resource "aws_acm_certificate" "cert" {
  domain_name               = "*.${var.root_domain}"
  subject_alternative_names = [var.root_domain]
  validation_method         = "DNS"
  lifecycle {
    create_before_destroy = true
  }
}

resource "aws_acm_certificate" "cert_cloudfront" {
  domain_name       = "cdn.${var.root_domain}"
  validation_method = "DNS"
  provider          = aws.virginia
  lifecycle {
    create_before_destroy = true
  }
}

```

また、ACMで証明書を発行する際に本当にドメインの所有者か検証が必要です。

今回は検証方法に「AWSが指定するDNSレコードを設定する方式」を選んでいるため、以下のように検証用のRoute53のレコードを作成しております。

```tf:modules/cert/route53.tf
# ACMのドメイン検証
resource "aws_route53_record" "cert_validation_main" {
  for_each = {
    for dvo in aws_acm_certificate.cert.domain_validation_options : dvo.domain_name => {
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
# cert_cloudfrontも同様に検証
```

ALBリソースの作成の際には↑で作成したACMの証明書のARNを指定する必要があります

```tf:modules/web_app/alb.tf
resource "aws_lb" "app" {
  name               = "${var.prefix}-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.ingress_all.id]
  subnets            = var.alb_subnets

  # テストようなので保護機能を有効にしない
  enable_deletion_protection = false
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
}

resource "aws_lb_listener" "https_blue" {
  load_balancer_arn = aws_lb.app.arn
  port              = "443"
  protocol          = "HTTPS"
  ssl_policy        = "ELBSecurityPolicy-2016-08"
  # NOTE: ACMで作成した署名書のARN
  certificate_arn   = var.acm_arn

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.app_blue.arn
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
}
```

最低限の対策を入れつつ、S3バケットを作成します

```tf:modules/web_app/s3.tf
resource "aws_s3_bucket" "static" {
  bucket = "${var.prefix}-static"
}

resource "aws_s3_bucket_versioning" "static" {
  bucket = aws_s3_bucket.static.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "static" {
  bucket = aws_s3_bucket.static.bucket

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

resource "aws_s3_bucket_public_access_block" "static" {
  bucket = aws_s3_bucket.static.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# CloudFront経由のアクセスに制限
resource "aws_s3_bucket_policy" "static" {
  bucket = aws_s3_bucket.static.id
  policy = templatefile("${path.module}/policy/bucket_policy.json", {
    OAI        = aws_cloudfront_origin_access_identity.static.iam_arn
    BUCKET_ARN = aws_s3_bucket.static.arn
  })
}

# テスト用にファイルを配置
resource "aws_s3_object" "object" {
  bucket = aws_s3_bucket.static.id
  key    = "static/test.json"
  source = "${path.module}/object/test.json"
}
```

なおS3のバケットポリシーは以下のようなテンプレートを利用して、CloudFront経由でのみアクセスをさせるようにOAIからのアクセスのみ許容しております。

```json
{
  "Statement": {
    "Sid": "1",
    "Effect": "Allow",
    "Principal": {
      "AWS": "${OAI}"
    },
    "Action": "s3:GetObject",
    "Resource": "${BUCKET_ARN}/*"
  }
}
```

### CloudFrontリソースの作成

CloudFrontのリソースを以下のように作成します。

```tf:modules/web_app/cloufront.tf
# Cloudfront
resource "aws_cloudfront_distribution" "asset" {
  enabled = true
  aliases = [
    "cdn.${var.root_domain}"
  ]
  # NOTE: WAFのWebACLの適用
  web_acl_id = aws_wafv2_web_acl.main.arn
  # NOTE: Origin設定(ALB)
  origin {
    domain_name = "alb.${var.root_domain}"
    origin_id   = aws_lb.app.id
    custom_origin_config {
      http_port                = 80
      https_port               = 443
      origin_keepalive_timeout = 5
      origin_protocol_policy   = "https-only"
      origin_read_timeout      = 60
      origin_ssl_protocols     = ["TLSv1", "TLSv1.1", "TLSv1.2"]
    }
  }

  # NOTE: Origin設定(S3)
  origin {
    domain_name = aws_s3_bucket.static.bucket_regional_domain_name
    origin_id   = aws_s3_bucket.static.id

    s3_origin_config {
      origin_access_identity = aws_cloudfront_origin_access_identity.static.cloudfront_access_identity_path
    }
  }

  viewer_certificate {
    cloudfront_default_certificate = false
    acm_certificate_arn            = var.cloudfront_acm_arn
    minimum_protocol_version       = "TLSv1"
    ssl_support_method             = "sni-only"
  }
  # ALBのキャッシュ戦略
  default_cache_behavior {
    allowed_methods  = ["HEAD", "DELETE", "POST", "GET", "OPTIONS", "PUT", "PATCH"]
    cached_methods   = ["HEAD", "GET", "OPTIONS"]
    target_origin_id = aws_lb.app.id

    forwarded_values {
      query_string = true
      cookies {
        forward = "all"
      }
      headers = ["*"]
      /* headers = ["Accept", "Accept-Language", "Authorization", "CloudFront-Forwarded-Proto", "Host", "Origin", "Referer", "User-agent"] */
    }

    viewer_protocol_policy = "redirect-to-https"
    min_ttl                = 0
    default_ttl            = 10
    max_ttl                = 60
  }

  # S3のキャッシュ戦略
  ordered_cache_behavior {
    path_pattern     = "/static/*"
    allowed_methods  = ["GET", "HEAD", "OPTIONS"]
    cached_methods   = ["GET", "HEAD", "OPTIONS"]
    target_origin_id = aws_s3_bucket.static.id

    forwarded_values {
      query_string = false
      headers      = ["Origin"]
      cookies {
        forward = "none"
      }
    }

    min_ttl                = 0
    default_ttl            = 300
    max_ttl                = 600
    compress               = true
    viewer_protocol_policy = "redirect-to-https"
  }

  restrictions {
    geo_restriction {
      restriction_type = "whitelist"
      locations        = ["JP"]
    }
  }
}

resource "aws_cloudfront_origin_access_identity" "static" {}
```

WAFのWebACLは以下のように作成しました。


### WAFのリソース作成

とりあえず無料で利用できるAWSのマネージドルールを詰め込んだらコードが縦に長くなってしまいました。

```tf:modules/web_app/waf.tf
resource "aws_wafv2_web_acl" "main" {
  name        = "${var.prefix}-app-acl"
  description = "Web ACL for ${var.prefix}-app"
  scope       = "CLOUDFRONT"
  provider    = aws.virginia

  default_action {
    allow {}
  }

  # 無料で使えるManagedRuleを追加
  rule {
    name     = "AWSManagedRulesCommonRuleSet"
    priority = 10

    override_action {
      none {}
    }

    statement {
      managed_rule_group_statement {
        name        = "AWSManagedRulesCommonRuleSet"
        vendor_name = "AWS"
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "AWSManagedRulesCommonRuleSetMetric"
      sampled_requests_enabled   = false
    }
  }


  rule {
    name     = "AWSManagedRulesKnownBadInputsRuleSet"
    priority = 20

    override_action {
      none {}
    }

    statement {
      managed_rule_group_statement {
        name        = "AWSManagedRulesKnownBadInputsRuleSet"
        vendor_name = "AWS"
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "AWSManagedRulesKnownBadInputsRuleSetMetric"
      sampled_requests_enabled   = false
    }
  }

  rule {
    name     = "AWSManagedRulesAmazonIpReputationList"
    priority = 30

    override_action {
      none {}
    }

    statement {
      managed_rule_group_statement {
        name        = "AWSManagedRulesAmazonIpReputationList"
        vendor_name = "AWS"
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "AWSManagedRulesAmazonIpReputationListMetric"
      sampled_requests_enabled   = false
    }
  }

  rule {
    name     = "AWSManagedRulesAnonymousIpList"
    priority = 40

    override_action {
      none {}
    }

    statement {
      managed_rule_group_statement {
        name        = "AWSManagedRulesAnonymousIpList"
        vendor_name = "AWS"
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "AWSManagedRulesAnonymousIpListMetric"
      sampled_requests_enabled   = false
    }
  }

  rule {
    name     = "AWSManagedRulesSQLiRuleSet"
    priority = 50

    override_action {
      none {}
    }

    statement {
      managed_rule_group_statement {
        name        = "AWSManagedRulesSQLiRuleSet"
        vendor_name = "AWS"
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "AWSManagedRulesSQLiRuleSetMetric"
      sampled_requests_enabled   = false
    }
  }

  rule {
    name     = "AWSManagedRulesLinuxRuleSet"
    priority = 60

    override_action {
      none {}
    }

    statement {
      managed_rule_group_statement {
        name        = "AWSManagedRulesLinuxRuleSet"
        vendor_name = "AWS"
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "AWSManagedRulesLinuxRuleSetMetric"
      sampled_requests_enabled   = false
    }
  }

  rule {
    name     = "AWSManagedRulesUnixRuleSet"
    priority = 70

    override_action {
      none {}
    }

    statement {
      managed_rule_group_statement {
        name        = "AWSManagedRulesUnixRuleSet"
        vendor_name = "AWS"
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "AWSManagedRulesUnixRuleSetMetric"
      sampled_requests_enabled   = false
    }
  }

  rule {
    name     = "AWSRateBasedRule"
    priority = 1

    action {
      block {}
    }

    statement {
      rate_based_statement {
        limit              = 500
        aggregate_key_type = "IP"

        scope_down_statement {
          geo_match_statement {
            country_codes = ["US", "NL"]
          }
        }
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "AWSRateBasedRuleMetric"
      sampled_requests_enabled   = false
    }
  }

  visibility_config {
    cloudwatch_metrics_enabled = true
    metric_name                = "TerraformWebACLMetric"
    sampled_requests_enabled   = false
  }
}

# NOTE: アクセス数がそれなりのサービスだとCloudWatchLogsに出力するとコストが嵩む
# Firehose経由でS3への配信も考慮が必要
resource "aws_wafv2_web_acl_logging_configuration" "main" {
  log_destination_configs = [aws_cloudwatch_log_group.waf.arn]
  provider                = aws.virginia
  resource_arn            = aws_wafv2_web_acl.main.arn
  redacted_fields {
    uri_path {}
  }
}
```

## 動作確認

以下コマンドで動作確認をしていきます。

### WAFが適用されているか確認

```bash
# これならステータスコード200でレスポンス
curl --location --request GET 'https://cdn.hoge.com' \
--header 'User-Agent: test'
# これはステータスコード403でレスポンス(User-Agentが空)
curl --location --request GET 'https://cdn.hoge.com' -H 'User-Agent: '
```

### ALBへのアクセスはCloudFront経由でしかできないことを確認

```bash
# CloudFront経由はアクセスできる
curl --location --request GET 'https://cdn.hoge.com' \
--header 'User-Agent: test'

# ALBのドメイン名直はアクセスできない
curl --location --request GET 'https://alb.hoge.com' \
--header 'User-Agent: test'
```

### S3へのアクセスはCDN経由でしかできないことを確認

```bash
# CloudFront経由はアクセスできる
curl --location --request GET https://hoge.com/static/test.json
# S3直接はアクセスできない
curl --location --request GET https://${S3のDNS名}/static/test.json
```

ブラウザでも以下のようにCDNのドメイン名を指定したアクセスはできます。

こちらはALBの配下で稼働しているテスト用のアプリケーションのコンテンツを取得できている時に表示されるHTMLです。

`https://cdn.hoge.dom` へアクセス

![アプリのイメージ](/images/waf_cloudfront/browser_alb.png)

続いて `https://cdn.hoge.com/static/test.json` にアクセスするとこちらのファイルを取得できます。

```json
{
  "test": true
}
```

