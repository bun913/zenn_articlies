---
title: "AWSによるBastion(踏み台)の構成を2つTerraformで作ってみた(ECS on Fargate と EC2)"
emoji: "📌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["aws", "terraform"]
published: true
---

私は業務でAWS環境の改善提案・Terraformを活用した環境構築をするお仕事をしております。
（とはいえ、ペーパードライバー感は否めませんが）

その中で、「あぁ・・・ちゃんとセキュリティ考えなあかんな・・・・」と考えることが多く、以下の記事・Yotubeを改めて見返しました。

ブログ版

https://dev.classmethod.jp/articles/aws-security-all-in-one-2021/

Yotube版

https://youtu.be/Ci_4dhUCZm8


まず初級編として、「パブリックサブネットにEC2を配置してSSH接続は極力やめて、SessionManager経由で接続しましょう」というようなことを教えてくれます。

さすがに業務でそのようなことはしていなかったのですが、改めてTerraformでテンプレートを作っておくことで諸々楽になるかなと思い今回Bastionの構成を2つ考えて、Terraformで作ってみました。

## 構成案

Fargate on ECSを利用する構成とEC2インスタンスを使う構成を考えてみました。

### 構成1 ECS on Fargateを使う

![構成1](/images/bastion/system_configuration1.png)

Fargate on ECSを利用する構成です。

プライベートサブネットにECSタスクを立てて、ECS Execコマンドで接続してもらう構成にしています。

この構成のメリット

- Fargateで構成しているため、OS以下のレイヤーの管理が不要
  - 踏み台マシンの脆弱性管理の範囲がぐっと狭くなる
- ssh用のポート穴あけが不要で従来のSSH接続よりセキュア
- CloudTrailsに監査用のログも残る

この構成のデメリット

- コマンドラインによるDB接続にしか対応していない
  - Dockerfileを工夫したり、接続時にPortForwadingすれば何とかできそうですが私は未対応
  - 例えばTablePlusやSequelAce なんかのDBクライアントを使ったグラフィカルなDB操作に未対応

やってみたリポジトリはこちらになります。

https://github.com/bun913/aws_fargate_bastion

※ **確認までに必要な手順などはREADMEに記載しておりますので、この記事では省略しております**

いくつかポイントとなった点だけ挙げておきます

イメージを作成するDockerfile

```Docker
FROM amazonlinux:2
# MySQLの新しいGPGキーをインストールする
RUN rpm --import https://repo.mysql.com/RPM-GPG-KEY-mysql-2022 && \
    yum install -y sudo jq awscli shadow-utils htop lsof telnet bind-utils yum-utils && \
    yum install -y yum localinstall https://dev.mysql.com/get/mysql80-community-release-el7-3.noarch.rpm && \
    yum-config-manager --disable mysql80-community && \
    yum-config-manager --enable mysql57-community && \
    yum install -y mysql-community-client
```

AmazonLinux2をベースイメージにしたのは、親切なお兄さんにECRのスキャンでAmazonLinuxベースのイメージを使えば脆弱性が検出されにくいと教わったからです。
（本当にそうでした）

AuroraMySQLをDBに使っているため、接続用のパッケージなどを導入しております。

タスク定義はこちらで作成

```tf:modules/bastion/ecs_taskdef.tf
# 今回アプリケーションは作成しないのでBastion用にクラスターを作成する
resource "aws_ecs_cluster" "bastion" {
  name = "${var.prefix}-bastion-cluster"

  setting {
    name  = "containerInsights"
    value = "enabled"
  }
}

resource "aws_ecs_task_definition" "app" {
  family        = "${var.prefix}-task-def"
  task_role_arn = aws_iam_role.ecs_task.arn
  network_mode  = "awsvpc"
  requires_compatibilities = [
    "FARGATE"
  ]
  execution_role_arn = aws_iam_role.bastion_task_exec_role.arn
  memory             = "512"
  cpu                = "256"
  container_definitions = templatefile(
    "${path.module}/taskdef/taskdef.json",
    {
      IMAGE_PREFIX      = "${var.ecr_base_uri}/${var.tags.Project}"
      BASTION_LOG_GROUP = aws_cloudwatch_log_group.bastion.name
      REGION            = var.region
    }
  )
}
```

ECSタスクロール(タスク実行ロールではない)には以下ポリシーを付与しています

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Action": [
        "ssmmessages:CreateControlChannel",
        "ssmmessages:CreateDataChannel",
        "ssmmessages:OpenControlChannel",
        "ssmmessages:OpenDataChannel"
      ],
      "Effect": "Allow",
      "Resource": "*"
    }
  ]
}
```

また今回は、NATGatewayなどは配置せずプライベートサブネットにポツンと、Fargateタスクを立ち上げるだけなので、各種VPCエンドポイントも作成しております。

```tf
vpc_endpoint = {
  "interface" : [
    # ECSからECRにイメージをpullするため
    "com.amazonaws.ap-northeast-1.ecr.dkr",
    "com.amazonaws.ap-northeast-1.ecr.api",
    # CloudWatchLogsにログ出力のため
    "com.amazonaws.ap-northeast-1.logs",
    # SSM SessionManagerで接続するため
    "com.amazonaws.ap-northeast-1.ssmmessages",
    "com.amazonaws.ap-northeast-1.ssm",
  ],
  "gateway" : [
    # ECSに必要
    "com.amazonaws.ap-northeast-1.s3"
  ]
}
```

### 構成2 EC2インスタンスを使う

![構成2](/images/bastion/system_configuration2.png)

EC2インスタンを使う構成です。

この構成のメリット

- SSM SessionManagerでsshポートを開けずにプライベートサブネットのEC2に接続できる
  - ECS Execも内部的にはこの仕組みを使っているので構成1と同等と考えています
- RDBに各開発者のDBクライアントを使って接続するのも簡単
  - EC2なので最初にキーペアの情報を渡しておくだけで良い

この構成のデメリット

- EC2を立てるのでOS以上を運用管理者で管理する必要がある
  - PatchManagerで最低限の脆弱性の自動更新は導入している

何といっても、OS以上を自分たちで管理しないといけないのがポイントですね。

ただ簡単な設定で DBクライアントを使うためのSSHトンネリングができるのは非常に魅力的です。

やってみたリポジトリはこちら

https://github.com/bun913/aws_ec2_bastion

こちらもポイントとなった点をいくつか記載します。

※ **確認までに必要な手順などはREADMEに記載しておりますので、この記事では省略しております**

EC2のメタデータサービスv1にはSSRFの脆弱性等があるので、v2を使うように設定しています。

https://dev.classmethod.jp/articles/ec2-imdsv2-release/

```tf:modules/bastion/main.tf
# 最新のAmazonLinux2のAMIのIDを取得
data "aws_ssm_parameter" "amzn2_ami" {
  name = "/aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2"
}

resource "aws_iam_instance_profile" "bastion-profile" {
  name = "${var.prefix}-profile"
  role = aws_iam_role.bastion_instance_role.name
}

resource "aws_instance" "bastion" {
  ami                    = data.aws_ssm_parameter.amzn2_ami.value
  instance_type          = "t2.micro"
  iam_instance_profile   = aws_iam_instance_profile.bastion-profile.name
  subnet_id              = var.private_subnet_id
  vpc_security_group_ids = [aws_security_group.bastion.id]
  user_data              = file("${path.module}/user_data.sh")
  # NOTE: キーペアはあらかじめ作成した名前で指定が必要
  key_name = var.key_pair_name

  metadata_options {
    # NOTE: インスタンスメタデータ取得をする場合の設定(不要ならdisabledに)
    http_endpoint = "enabled"
    # IMDSv2の利用(セキュリティ強化)
    # https://docs.aws.amazon.com/ja_jp/AWSEC2/latest/UserGuide/configuring-instance-metadata-service.html
    http_tokens = "required"
  }

  tags = merge(var.tags, { Name = "${var.prefix}-bastion" })
}
```

今回EC2のインスタンスプロファイルに渡すロールには横着してAWS管理ポリシーを付与しております。

```tf:modules/bastion/iam.tf
# ManagedRoleを取得
data "aws_iam_policy" "ssmManagedPolicy" {
  arn = "arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore"
}
# EC2 Role
resource "aws_iam_role" "bastion_instance_role" {
  name = "${var.prefix}-bastion"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Sid    = ""
        Principal = {
          Service = "ec2.amazonaws.com"
        }
      },
    ]
  })
}

# AttachRole
resource "aws_iam_role_policy_attachment" "bastion_instance_role_attach" {
  role       = aws_iam_role.bastion_instance_role.name
  policy_arn = data.aws_iam_policy.ssmManagedPolicy.arn
}
```

PatchManagerでは、AmazonLinux2用のマネージドルールを利用しています

```tf:modules/bastion/patch_manager.tf
/*
　パッチマネージャーを使うまでに作成が必要なリソース群
　https://docs.aws.amazon.com/ja_jp/systems-manager/latest/userguide/sysman-patch-cliwalk.html
*/

# 今回パッチベースラインはAmazonLinux2のマネージドルールを利用
data "aws_ssm_patch_baseline" "al2" {
  owner            = "AWS"
  name_prefix      = "AWS-"
  operating_system = "AMAZON_LINUX_2"
}

resource "aws_ssm_patch_group" "main" {
  baseline_id = data.aws_ssm_patch_baseline.al2.id
  patch_group = "${var.prefix}-patchgroup"
}

resource "aws_ssm_maintenance_window" "main" {
  name     = "${var.prefix}-for-bastion"
  # NOTE: 確認のため頻度を3分おきにしているので、実運用では設定を変更する
  schedule = "rate(3 minutes)"
  duration = 2
  cutoff   = 1
}

resource "aws_ssm_maintenance_window_target" "main" {
  window_id     = aws_ssm_maintenance_window.main.id
  name          = "${var.prefix}-target1"
  description   = "Target for ${var.prefix}-bastion Tagged Instances"
  resource_type = "INSTANCE"

  targets {
    key    = "tag:Name"
    values = ["${var.prefix}-bastion"]
  }
}

resource "aws_ssm_maintenance_window_task" "main" {
  max_concurrency = 1
  max_errors      = 1
  priority        = 1
  task_arn        = "AWS-RunPatchBaseline"
  task_type       = "RUN_COMMAND"
  window_id       = aws_ssm_maintenance_window.main.id

  targets {
    key    = "WindowTargetIds"
    values = [aws_ssm_maintenance_window_target.main.id]
  }

  task_invocation_parameters {
    run_command_parameters {
      /* output_s3_bucket     = aws_s3_bucket.example.bucket */
      /* output_s3_key_prefix = "output" */

      # ログをS3に出力する場合などは必要
      /* service_role_arn     = aws_iam_role.example.arn */
      timeout_seconds = 600

      /* notification_config { */
      /*   notification_arn    = aws_sns_topic.example.arn */
      /*   notification_events = ["All"] */
      /*   notification_type   = "Command" */
      /* } */

      parameter {
        name   = "Operation"
        values = ["Install"]
      }
    }
  }
}

```

## まとめ

結局どっちを使えば良いかというのは、 **要件による** としか言えないと思いました。（逃げ)

ただ、やはりシステム管理者の手間を減らすという意味では構成1を使った方が、面倒を見る範囲が少ないのかなと思います。

一方で踏み台がEC2としてあるというのは、システム管理者の気持ちを考えると色々と便利な気もします。

今回の構成1では、DBクライアントを使わせるためのSSHトンネリングの設定方法などのベストプラクティスが見つけられなかったので、DBへのアクセスはECS Execに接続してから、コマンドラインで操作することにしか対応しておりませんが、もっと良い方法があればご教示いただけますと助かります。
