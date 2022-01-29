---
title: "CloudFrontでCORS設定をするための3つのポリシーについて"
emoji: "👋"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["aws","cloudfront","terraform"]
published: true
---

先日業務でCloudFrontを活用していて、単純ですがCORSエラーが出ないための設定で一瞬ハマりかけたことがあるため皆さんに共有いたします。

CloudFront(というよりCDN)はキャッシュ戦略といい、奥が深いなぁということを感じました(小並感)

## やること

- TerraformでS3をオリジンとしてCloudFrontを立てる
- 特定のサイトからだけクロスオリジンリクエストを許可する様にS3とCloudFrontにルールを設定する
- curlで動作を確認していく

## 前提知識

- CloudFrontがどのようなサービスか概要を知っている
- CORSの基本的な知識

CORSについてはたくさん良さそうな記事がありますので、検索してみてください。

https://qiita.com/att55/items/2154a8aad8bf1409db2b

## 結論

先に結論をご説明いたしますと、AWSの公式で提示してくださっています。

https://aws.amazon.com/jp/premiumsupport/knowledge-center/no-access-control-allow-origin-error/

必要なことを列挙すると以下の様になりますが、一つ一つ説明して行きたいと思います。

- CloudFrontのオリジン（今回はS3)でCORS用のルールを設定する
- CloudFront側で以下3つのポリシーを設定する
  - キャッシュポリシー
  - オリジンリクエストポリシー
  - レスポンスヘッダーポリシー

![全体図](/images/cloudfront_cors/cloudfront_policies.png)

## 設定内容

今回の設定に関しては以下のリポジトリに格納しておりますので、気になる方はご参照ください。

https://github.com/bun913/aws_network_practice/tree/main/cloudfront_cors_setting

### S3の設定

作成するS3バケットにCORS用のルールを設定していきます。

以下の様に `https://hoge.com` と `https://fuga.com` からの `GET` と `HEAD`メソッドを許可します。

```json
[
    {
        "AllowedHeaders": [
            "*"
        ],
        "AllowedMethods": [
            "GET",
            "HEAD"
        ],
        "AllowedOrigins": [
            "https://hoge.com",
            "https://fuga.com"
        ],
        "ExposeHeaders": [],
        "MaxAgeSeconds": 3000
    }
]
```

Terraformで作成する場合この様にいたします。

```tf:
# oai作成
resource "aws_cloudfront_origin_access_identity" "oai" {
  comment = "${var.project}-oai"
}

# create bucket
resource "aws_s3_bucket" "origin" {
  bucket = "${var.project}-origin"
  acl    = "private"
  policy = templatefile(
    "${path.module}/policy.json",
    {
      oai_arn     = aws_cloudfront_origin_access_identity.oai.id
      bucket_name = "${var.project}-origin"
    }
  )

  # cors_ruleを設定
  cors_rule {
    allowed_headers = ["*"]
    allowed_methods = ["GET", "HEAD"]
    allowed_origins = var.allowed_origins
    expose_headers  = []
    max_age_seconds = 3000
  }

  tags = var.tags
}

```

### CloudFrontのキャッシュポリシーを設定する

キャッシュポリシーとは何かについてはクラスメソッドさんの記事がわかりやすかったです。

https://dev.classmethod.jp/articles/cloudfront-cache-key-policy/

私個人が人に話すときには、以下の様にざっくり理解していることをお伝えします。

- CloudFrontはCDNだから、オリジン(今回でいえばS3)の情報をキャッシュしています
- 同じ形のリクエストが来た場合は、わざわざS3にリクエストを流さずにCloudFrontにキャッシュしている情報で返事してくれるわけです
- そのときに `何を持って同じ形のリクエスト` と見なしてキャッシュするかを設定するのがキャッシュポリシーです
- 例えば リクエストヘッダーの `Origin` が異なれば、別のリクエストしてキャッシュするような設定を入れることができます

などとお伝えします。

これを設定し忘れると 例えば以下の様なリクエストが `https://hoge.com` と `https://fuga.com` からされた場合、同じリクエストとしてCloudFrontがみなしてしまいかねません。

```
リクエスト先: https://example.cloudfront.net/sample_obj
# リクエストヘッダー
Method: GET
Origin: https://hoge.com
```

```
リクエスト先: https://example.cloudfront.net/sample_obj
# リクエストヘッダー
Method: GET
Origin: https://fuga.com
```

ここの設定を漏らすと、後で出てくるレスポンスヘッダーポリシーなどを設定していても意図どおりに動作しません。。。

![キャッシュポリシー未設定](/images/cloudfront_cors/cloudfront_cache_policy.png)

たまたま先に `https://hoge.com` がCloudFrontにリクエストを送っていたため、その情報でキャッシュが残っており、後から帰ってきた `https://fuga.com` からのリクエストも同じキャッシュの情報で返却します。

どこのOriginからCloudFrontにリクエストが来てもリクエストBodyが一緒であれば同じリクエストとしてみなされてしまうため、`https://hoge.com` 以外のサイトはCORSエラーでコンテンツを取得できないのです・・・

そのため、キャッシュの情報に Originヘッダーも見てくれるようにキャッシュポリシーを設定します。

以下の様にAWSはマネージドでポリシーを用意してくれています。

![キャッシュポリシー](/images/cloudfront_cors/cloudfront_managed_cache_policy.png)

https://docs.aws.amazon.com/ja_jp/AmazonCloudFront/latest/DeveloperGuide/using-managed-cache-policies.html


1からルールを作っても良いのですが、今回は省略して Originヘッダーを条件に含んでいる `Elemental-MediaPackage` を利用させていただきます。

```tf
data "aws_cloudfront_cache_policy" "asset" {
  name = "Managed-Elemental-MediaPackage"
}
```

### CloudFrontのオリジンリクエストポリシーを設定する

先ほどのクラスメソッドさんの記事にオリジンリクエストポリシーについても記載されていました。

https://dev.classmethod.jp/articles/cloudfront-cache-key-policy/


こちらも自分が他の方にざっくり説明すると

- CloudFrontでキャッシュにヒットしなかった時に、オリジン(今回はS3)にリクエストを流してくれますよね?
- このときにリクエスト元から来ていたリクエストヘッダーのうち、何をS3にも送るかなどをオリジンリクエストポリシーで設定できます(クッキーとかクエリパラメーターの条件とかも)
- 今回で言えば, 折角リクエスト元が Originヘッダーを送ってくれるので、それをS3に送る様に設定しておく必要がありますね

という感じで説明します。

![オリジンリクエスト](/images/cloudfront_cors/cloudfront_origin_request_policy.png)

こちらもAWSがマネージドポリシーを用意してくれています。

![マネージドオリジンポリシー](/images/cloudfront_cors/cloudfront_managed_origin_request_policy.png)

今回は `Managed-CORS-S3Origin` を利用させてもらいます。

どのような情報を含んでいるかはこちらをご覧ください。

https://docs.aws.amazon.com/ja_jp/AmazonCloudFront/latest/DeveloperGuide/using-managed-origin-request-policies.html#:~:text=%E5%90%8D%E5%89%8D%3A-,CORS%2DS3Origin,-ID%3A%2088a5eaf4%2D2fd4

```tf
# オリジンリクエストポリシー
data "aws_cloudfront_origin_request_policy" "asset" {
  name = "Managed-CORS-S3Origin"
}
```

### レスポンスヘッダーポリシー

こちらはわかりやすいですね。

CloudFrontがレスポンスを返却する際にヘッダーとして返却を許容するものを指定できるものと理解しております。

https://aws.amazon.com/jp/blogs/news/amazon-cloudfront-introduces-response-headers-policies/

こちらもAWSがマネージドポリシーを用意してくださっています。

![オリジンレスポンスヘッダーポリシー](/images/cloudfront_cors/cloudfront_managed_response_header_policy.png)

https://docs.aws.amazon.com/ja_jp/AmazonCloudFront/latest/DeveloperGuide/using-managed-response-headers-policies.html

今回は 特定のOriginにだけ許可したいため、以下の様に作成します。

```tf
# レスポンスヘッダーポリシー
resource "aws_cloudfront_response_headers_policy" "asset" {
  name    = "${var.project}-response-headers-policy"
  comment = "Allow from limited origins"

  cors_config {
    access_control_allow_credentials = false

    access_control_allow_headers {
      items = ["*"]
    }

    access_control_allow_methods {
      items = ["GET", "HEAD"]
    }

    access_control_allow_origins {
      items = var.allowed_origins
    }

    origin_override = true
  }
}
```

### TerraformでCloudFrontリソース作成

これまでを踏まえてCloudFrontDestributionを作成します。

```tf
# Cloudfront
resource "aws_cloudfront_distribution" "asset" {
  enabled = true
  origin {
    domain_name = aws_s3_bucket.origin.bucket_regional_domain_name
    origin_id   = aws_s3_bucket.origin.id
    s3_origin_config {
      origin_access_identity = aws_cloudfront_origin_access_identity.oai.cloudfront_access_identity_path
    }
  }
  default_cache_behavior {
    allowed_methods        = ["GET", "HEAD", "OPTIONS"]
    cached_methods         = ["GET", "HEAD"]
    target_origin_id       = aws_s3_bucket.origin.id
    viewer_protocol_policy = "allow-all"

    min_ttl     = 0
    default_ttl = 3600
    max_ttl     = 86400

    # 3つのポリシーを適用
    cache_policy_id            = data.aws_cloudfront_cache_policy.asset.id
    origin_request_policy_id   = data.aws_cloudfront_origin_request_policy.asset.id
    response_headers_policy_id = aws_cloudfront_response_headers_policy.asset.id
  }
  restrictions {
    geo_restriction {
      restriction_type = "whitelist"
      locations        = ["JP"]
    }
  }
  viewer_certificate {
    cloudfront_default_certificate = true
  }
}
```

## 動作確認

これでCloudFrontディトリビューションの作成ができましたので、`curl`コマンドで動作確認をしてみます。
先にS3へ `test.json` というファイルを適当に格納しております。

```bash
# hoge.comをOriginとして初めてリクエスト
curl --head  -H "Origin: https://hoge.com" https://example.cloudfront.net/test.json
> x-cache: Miss from cloudfront #キャッシュになかったのでS3までリクエストが届いている
> access-control-allow-origin: https://hoge.com

# fuga.comをOriginとして初めてリクエスト
curl --head  -H "Origin: https://fuga.com" https://example.cloudfront.net/test.json
> x-cache: Miss from cloudfront
> access-control-allow-origin: https://fuga.com

# hoge.comをOriginとして何回かリクエストを送ってみる
curl --head  -H "Origin: https://hoge.com" https://example.cloudfront.net/test.json
> x-cache: Hit from cloudfront #CloudFrontにキャッシュが残っていた
> access-control-allow-origin: https://hoge.com

# fuga.comをOriginとして何回かリクエストを送ってみる
curl --head  -H "Origin: https://fuga.com" https://example.cloudfront.net/test.json
> x-cache: Hit from cloudfront
> access-control-allow-origin: https://fuga.com

```

このようにそれぞれ許可されたOrigin(hoge.com, fuga.com)についてAccess-Control-Allow-Origin ヘッダーが返却されていることを確認しました。

ちなみに許可されていないOriginの場合以下の様に Access-Control-Allow-Originヘッダーが返却されませんので、非同期通信でコンテンツを取得しようとしてもCORSエラーになります。

```
curl --head  -H "Origin: https://example22.com" https://example.cloudfront.net/test.json

# 以下レスポンスヘッダー
content-type: application/json
content-length: 16
date: Sat, 29 Jan 2022 05:27:32 GMT
last-modified: Sat, 29 Jan 2022 04:43:52 GMT
etag: "e4d5bca64e471cdb3e490f3eb0258c66"
accept-ranges: bytes
server: AmazonS3
vary: Origin
x-cache: Miss from cloudfront
via: 1.1 example.cloudfront.net (CloudFront)
x-amz-cf-pop: NRT12-C2
x-amz-cf-id: xcnosRfjgUMSyZUTc_XZYqb3YRI3L0o8Eb5d3OkT0Fhwdd5K9Tf4qA==
```

## まとめ

- CloudFrontにてCORS設定をする際は3つのポリシーに留意
- Origin(今回はS3)側でもCORSルールの設定をお忘れずに

私の理解が間違っていたり、おかしな点があれば是非ご教示ください!