---
title: "JMerterの設定TIPS(随時更新)"
emoji: "🐈"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["jmeter"]
published: true
---

最近業務でAWSが用意してくださっているテンプレートを活用して負荷テストの準備をしております。

https://aws.amazon.com/jp/solutions/implementations/distributed-load-testing-on-aws/

シンプルに特定のURLにリクエストを送り続けるだけなら良いのですが、扱っているサービスの性質上、ログイン後に見ることができる個別ユーザーごとの負荷を計測する必要がありました。

そのためカスタムJMerterスクリプトを作成する必要があり、自分が設定の時に困ったポイントについてまとめていきたいと思います。
（随時更新していきます）

## デバッグ

例えば後述する正規表現抽出などで、レスポンスヘッダーやレスポンスBodyから特定の値を抽出して、その値を参照させたい時などにはDebugSamplerが利用できます。

1. デバッグをしたいスレッドグループにおいて右クリック
2. 追加 -> サンプラー -> Debugサンプラーを選択
3. 追加 -> リスナー -> 結果をツリーで表示を選択

試しにリクエストを送ることで結果をツリーで表示のところに各種変数の値などが出力されます。

![デバッグサンプラー](/images/jmeter/debug_sampler.png)

## ログイン周り

### HTTPレスポンスヘッダーから任意の値を抽出

よくあるのがBearer認証のためのトークンをレスポンスヘッダーに含めていて、リクエスト時にそれを送るパターンですね。

以下のようにして抽出できました。

1. ヘッダーから値を抽出したいHTTPリクエストサンプラーの位置で右クリック
2. 追加を選択
3. 後処理から正規表現抽出を選択

![正規表現サンプラー](/images/jmeter/add_regexp.png)

例えばヘッダーから `access-token` を取得したい以下のように設定

- Fileld to check: `ResponseHeaders` を選択
- 参照名: 今回は `accessToken`
  - 変数名の命名のようなもの
  - `${accessToken}` のように値を参照できる
- 正規表現
  - `access-token:\s+(.+)` とする
  - 例えば clientを取得したい時には `client:\s+(.+)` でOK
- テンプレート: `$1$` とする

https://jmeter.apache.org/usermanual/regular_expressions.html

正規表現だけでなくリクエストBodyから JSONPath形式の記述で任意の値を抽出する `JsonExtractor` というものもあります。

https://jmeter.apache.org/usermanual/component_reference.html#JSON_Extractor

### 抽出した値をリクエストで利用する

`${変数名}` の形式で割とどこにでも組み込めます。

例えば HTTPリクエストのパラメーターとして利用する場合には、以下のように値の欄で指定するだけです。

![変数を参照](/images/jmeter/refer_valiables.png)

HTTPヘッダーに組み込みたい場合などは、HTTPヘッダーマネージャーを追加して同様の方法で値を参照させることができます。

![ヘッダマネージャー](/images/jmeter/header_manager.png)
