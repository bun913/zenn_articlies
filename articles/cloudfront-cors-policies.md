---
title: "CloudFrontでCORS設定をするための3つのポリシーについて"
emoji: "👋"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["aws","cloudfront","terraform"]
published: false
---

先日業務でCloudFrontを活用していて、単純ですがCORSエラーが出ないための設定で一瞬ハマりかけたことがあるため皆さんに共有いたします。

CloudFront(というよりCDN)はキャッシュ戦略といい、奥が深いなぁということを感じました(小並感)

## やること

- TerraformでS3をオリジンとしてCloudFrontを立てる
- 特定のサイトからだけクロスオリジンリクエストを許可する様にS3とCloudFrontにルールを設定する
- curlで動作を確認していく


## 結論

先に結論をご説明いたしますと、AWSの公式で提示してくださっています。

https://aws.amazon.com/jp/premiumsupport/knowledge-center/no-access-control-allow-origin-error/

必要なことを列挙すると以下の様になりますが、一つ一つ説明して行きたいと思います。

- CloudFrontのオリジン（今回はS3)でCORS用のルールを設定する
- CloudFront側で以下3つのポリシーを設定する
  - キャッシュポリシー
  - オリジンリクエストポリシー
  - レスポンスヘッダーポリシー

![全体図](/images/cloudfront_policies.png)