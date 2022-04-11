---
title: "AWSで使った方が良いセキュリティ系サービスをTerraformで有効化(一部手動あり)"
emoji: "📑"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["aws", "terraform"]
published: true
---

前回CloudFrontとWAFをTerraformでプロビジョニングしてみました。

https://zenn.dev/bun913/articles/waf-cloudfront-by-terraform

今回もクラスメソッド様のこちらのブログ(Youtube)を参考にして、有効化しておいた方が良いセキュリティ系のサービスをTerraformで設定するためのテンプレのようなものを作ってみました

ブログ

https://dev.classmethod.jp/articles/aws-security-all-in-one-2021/

Youtube

https://www.youtube.com/watch?v=Ci_4dhUCZm8&list=TLGGviofTcMM0FMxMTA0MjAyMg

内容的には中級編にあたるかと思います。

- CloudTrail
- Config
- GuardDuty

これらを有効化して Security Hub に集約した結果を、ChatBotを活用してSlack通知するような形です。

Configを有効化するのにCloudTrailを有効化する必要があり、GuardDutyもCloudTrailのログを機械学習で監視してくれているはずなので、以下のようなイメージになると思っています。

![構成図](/images/securityhub_terraform/system_archi.png)

## 今回のスコープ

### 前提条件

- シングルアカウントでの検知・通知を実装します
  - マルチアカウントで１つのアカウントに各アカウントのSecurity Hubの結果を集約するには、別途作業が必要になります
- 一部手動作業があります
  - ChatBotの作成
    - 2022.4.12時点でTerraformでChatBotリソースを利用できないため
      - このような方法もありはする
      - https://zenn.dev/shonansurvivors/articles/894cae91806052
    - AWS SDK(Go言語)でChatBotが対応すればTerraformでも利用できるようになると思われるため、それを見越してLambda関数で代替していない
  - SecurityHubの一部設定
    - TerraformではSecurityHubの有効化だけしかしていない
    - どのような基準を適用するかなどは手動で設定が必要
- Config等では全てのルールを全リージョンで有効化します
  - コストが嵩みますので、必要に応じて適用するルールを制限するなどの対応を行う必要があります

### すること

- Terraformで以下リソース群とそれに付随するリソースを起動
  - CloudTrail
  - Config
  - GuardDuty
  - SecurityHub
    - EventBridgeルール
    - SNSトピック

## 実装内容

今回も全ての実装はZennには書かず、ポイントとなりそうな部分を抜粋します。

ソースを見たい場合は、こちらのリポジトリをご参照ください。

https://github.com/bun913/aws_security_hub


### CloudTrail

```tf:main.tf
/*
  CloudTrailの有効化
*/

module "cloudtrail" {
  source = "./modules/cloudtrail/"

  prefix = local.default_prefix
  region = var.region
}

```

モジュール側では以下のようにCloudTrailを全リージョンで有効化しています。

```tf:modules/cloudtrail/main.tf
resource "aws_cloudtrail" "main" {
  name                          = "${var.prefix}-trail"
  s3_bucket_name                = aws_s3_bucket.cloudtrail.id
  include_global_service_events = true
  is_multi_region_trail         = true
  kms_key_id                    = aws_kms_key.cloudtrail.arn
  s3_key_prefix                 = "cloudtrail"
}
```

### Config

Config・GuardDutyは全リージョンで有効化する必要があるため、少々変わった呼び出し方をしております。

このやり方はBASE様のブログを参考にさせていただきました。

https://devblog.thebase.in/entry/guardduty-chatbot-terraform

```tf:config.tf
/* Configを各リージョンで有効化 */
module "config_ap-northeast-1" {
  source = "./modules/config/"

  prefix         = local.default_prefix
  bucket_name    = aws_s3_bucket.config.bucket
  is_main_region = true
  config_role    = aws_iam_role.config.arn

}
module "config_us-east-1" {
  source = "./modules/config/"

  prefix         = local.default_prefix
  bucket_name    = aws_s3_bucket.config.bucket
  is_main_region = false
  config_role    = aws_iam_role.config.arn

  providers = {
    aws = aws.us-east-1
  }
}
# ...省略していますが全てのリージョン分有効化しています
```

また Configのベストプラクティスによると、Globalリソースの監視は1つのリージョンのみで有効化した方が良いとのことなので `is_main_region` という変数で制御しております。

https://aws.amazon.com/jp/blogs/news/aws-config-best-practices/


```tf:modules/config/main.tf
resource "aws_config_configuration_recorder" "main" {
  name     = "${var.prefix}-config-recorder"
  role_arn = var.config_role

  recording_group {
    all_supported                 = true
    # メインのリージョンでのみグローバルリソースの監視を行う
    include_global_resource_types = var.is_main_region ? true : false
  }
}

resource "aws_config_configuration_recorder_status" "main" {
  name       = aws_config_configuration_recorder.main.name
  is_enabled = true
  depends_on = [aws_config_delivery_channel.main]
}

resource "aws_config_delivery_channel" "main" {
  name           = "${var.prefix}-config-delivery-channel"
  s3_bucket_name = var.bucket_name
  s3_key_prefix  = "config"
  depends_on     = [aws_config_configuration_recorder.main]
}
```

### GuardDuty

GuardDutyもConfigと同じように、全てのリージョンで有効化できるようにリージョンごとにモジュールを呼び出しています。

```tf:guardduty.tf
module "guardduty-us-east-1" {
  source = "./modules/guardduty/"

  providers = {
    aws = aws.us-east-1
  }
}

module "guardduty-us-east-2" {
  source = "./modules/guardduty/"

  providers = {
    aws = aws.us-east-2
  }
}
# ...省略していますが全リージョンで有効化しています
```

```tf:modules/guardduty/main.tf
resource "aws_guardduty_detector" "guardduty" {
  enable = true
}
```

### SecurityHub

Terraformでは有効化だけおこないます

```tf:securityhub.tf
resource "aws_securityhub_account" "main" {}
```

### EventBridgeルール・SNSトピックなど作成

Terraformではなく手動で作成するChatBotで利用するSNSトピックなどを作成します。

```tf:modules/notification/main.tf
/* CloudWatchEventの作成 */
# CRITICAL・HIGHの検出かつ、新規のものだけ拾う
resource "aws_cloudwatch_event_rule" "securityhub" {
  name        = "${var.prefix}-securityhub"
  description = "SecurutiHub High and Critical Events Event"

  event_pattern = <<EOF
{
  "source": [
    "aws.securityhub"
  ],
  "detail-type": [
    "Security Hub Findings - Imported"
  ],
  "detail": {
    "findings":
      {
        "Compliance": {
          "Status": [
            {
              "anything-but": "PASSED"
            }
          ]
        },
        "Severity": {
           "Label": [
             "CRITICAL",
             "HIGH"
           ]
        },
        "Workflow": {
          "Status": [
            "NEW"
          ]
        },
        "RecordState": [
          "ACTIVE"
        ]
      }
  }
}
EOF
}

resource "aws_cloudwatch_event_target" "sns-publish" {
  rule      = aws_cloudwatch_event_rule.securityhub.name
  target_id = aws_sns_topic.chatbot.name
  arn       = aws_sns_topic.chatbot.arn
}


/* SNS Topicの作成 */

# マネジメントコンソールからChatBotを利用する際に指定する
resource "aws_sns_topic" "chatbot" {
  name              = "${var.prefix}-topic"
  kms_master_key_id = aws_kms_key.for_encrypt_sns_topic.id
}

resource "aws_sns_topic_policy" "chatbot" {
  arn    = aws_sns_topic.chatbot.arn
  policy = data.aws_iam_policy_document.chatbot.json
}
# ...省略
```

## Terraform適用・動作確認までの手順

### Terraform適用

以下手順でTerraformでリソース群を作成します

```bash
cd infra/global
terraform init
terraform plan
terraform apply
```

### ChatBotの有効化(手動)

手動手順に関しては、他のブログや記事で丁寧なものがたくさんありますので、今回は以下を参考にSlack通知のための設定を行いました

こちらのブログ 「Chatbotの設定」の手順どおりにChatbotを作成しました。

AWSマネジメントコンソールから作業を行います。

https://blog.serverworks.co.jp/2021/12/27/210000

SNSトピックは modules/notification で作成されたものを指定するようにしてください。

※ なおSlack側で事前に通知先チャンネルの作成を行う必要があります

### SecurityHubの設定(手動)

SecurityHubの基準などを設定します。

AWSマネジメントコンソールから作業を行います。

- マネジメントコンソールからSecurity Hubのサービスへ移動
- Security Hub に移動　をクリック

![セキュリティハブの有効化](/images/securityhub_terraform/securityhub.png)

- 適用したい基準を有効化

![基準の有効化](/images/securityhub_terraform/securityhub2.png)

今回のハンズオンに関しては以下基準の有効化のみで十分です

- AWS 基礎セキュリティのベストプラクティス v1.0.0 を有効化

### 確認

SecurityHubで `Critical` もしくは `High` の問題が新たに検出されれば以下のようにSlackに通知が届きます。

テストする場合は一時的にSecurityGroupを作成し、インバウンドルールでSSH接続をフルオープン(0.0.0.0/0)などに設定すれば検出してくれます。
**もちろん作ったSGはテストの後に削除しましょう**

![Slack通知](/images/securityhub_terraform/slack.png)

すでに `Critical` または `High` のルールが検出されている場合は、「ワークフローのステータス」を「新規」に設定することでSlackに通知が届きます。

![新規に変更](/images/securityhub_terraform/securityhub-changed.png)
