---
title: "M1マックでTerraformでエラーが出たり成功したりする件の解消"
emoji: "👏"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["terraform", "mac"]
published: true
---

タイトルままですが、私の環境（以下に記載）でTerraformの `plan` `apply` `destroy` を行うと以下のようなエラーが起こることが頻発していました。

環境

- PC
    - M1 Mac
    - OS: Big Sur
- Terraform
  - tfenv
  - terraform: v1.1.7

エラー内容

```
Error: Failed to load plugin schemas
│ 
│ Error while loading schemas for plugin components: Failed to obtain provider schema: Could not load the schema for provider
│ registry.terraform.io/hashicorp/aws: failed to instantiate provider "registry.terraform.io/hashicorp/aws" to obtain schema: Unrecognized
│ remote plugin message: 
│ 
│ This usually means that the plugin is either invalid or simply
│ needs to be recompiled to support the latest protocol...


Error: Plugin did not respond
│ 
│   with provider["registry.terraform.io/hashicorp/aws"],
│   on main.tf line 1, in provider "aws":
│    1: provider "aws" {
│ 
│ The plugin encountered an error, and failed to respond to the plugin.(*GRPCProvider).ValidateProviderConfig call. The plugin logs may
│ contain more details.

```

この現象の嫌なところは、エラーが出ないこともあれば出る時もあるということでした・・・

## 解決した方法

結論から申し上げますと以下環境変数を設定することでエラーが出なくなりました。

```
GODEBUG=asyncpreemptoff=1
```

`zsh` を利用している場合は `~/.zshrc` に以下の通り追記します。

```bash
export GODEBUG=asyncpreemptoff=1
```

すでに立ち上がっているシェルで適用する場合はこちらの実行も忘れずに。　`source ~/.zshrc`

## 解決に至るまで

エラー文で検索してもなかなか答えが見つけられませんでした。

諸々Issueを探しているとこちらのIssue中のコメントを発見しました。

https://github.com/hashicorp/terraform/issues/27350#issuecomment-751213404

> I've had a similar issue on my M1 on Big Sur creating a few basic AWS resources. It isn't 100% reproducible yet, sometimes the apply/destroy works, sometimes it does not.

「あ。俺と同じで上手く動作する時もあるんだ」

ということで同じIssue内のコメントにて解決策を見つけることができました。

https://github.com/hashicorp/terraform/issues/27350#issuecomment-751269666

