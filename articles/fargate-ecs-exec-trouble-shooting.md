---
title: "ECS ExecでFARGATE環境のタスクに接続できない時の対応"
emoji: "🐈"
type: "tech"
topics: ["aws", "ecs", "fargate"]
published: true
---

AWSのFARGATEはOSの管理もしないでよいマネージドサービスなのでとても便利ですよね。

FARGATEで本番環境アプリケーションを稼働させているところも多いのではないでしょうか？

今回は `ECS Exec` に接続できない時の対応について公式が手厚く準備してくださっているので、そのご紹介もかねて手順をご記載させていただきます。

公式
https://dev.classmethod.jp/articles/ecs-exec/


※ 公式の言う通りの手順で行っているだけですし、何番煎じだよって記事です

## 前提条件

- 公式の前提条件に記載されている条件を満たしている
  - タスク・AWS環境側で準備すること
      - タスクロールに必要なIAM権限を与える(タスク実行ロールではありませんのでご注意を)
      - FARGATEの場合 プラットフォームバージョン1.4.0以上を導入している
      - プライベートサブネットの場合必要なVPCエンドポイントを導入している
        - `com.amazonaws.ap-northeast-1.ssmmessages` が必要です
      - 稼働しているECSタスクの `enableExecuteCommand` プロパティが trueになっている
  - 接続元のマシン(ローカル環境MACなど)で準備すること
    - AWS CLI用の Session Managerプラグインをインストールする
    - https://docs.aws.amazon.com/ja_jp/systems-manager/latest/userguide/session-manager-working-with-install-plugin.html

必読: 公式の手順

https://docs.aws.amazon.com/ja_jp/AmazonECS/latest/userguide/ecs-exec.html#ecs-exec-prerequisites

## 結論

上記を準備しているはずなのにうまく接続できない場合は、これまた公式で提供してくださっている `Amazon ECS Exec Checker` を利用しましょう。

https://github.com/aws-containers/amazon-ecs-exec-checker

事前に `jq` と `aws cli` (バージョンも↑リポジトリのREADMEを参照)を準備しておきましょう。

上記リポジトリから cloneしてきて、 シェルスクリプトを叩くだけで何が足りないのか教えてくれます。

```bash
./check-ecs-exec.sh クラスター名 タスクのID
```

私の場合以下のように VPCエンドポイントが足りていなかったので教えてくれました。

```
-------------------------------------------------------------
Prerequisites for check-ecs-exec.sh v0.7
-------------------------------------------------------------
  jq      | OK (/opt/homebrew/bin/jq)
  AWS CLI | OK (/usr/local/bin/aws)

-------------------------------------------------------------
Prerequisites for the AWS CLI to use ECS Exec
-------------------------------------------------------------
  AWS CLI Version        | OK (aws-cli/2.1.32 Python/3.8.8 Darwin/20.6.0 exe/x86_64 prompt/off)
  Session Manager Plugin | OK (1.2.295.0)
...(省略)
 SSM PrivateLink "com.amazonaws.ap-northeast-1.ssmmessages" not found. You must ensure your task has proper outbound internet connectivity.
```

