---
title: "Cognitoに認証リクエストを送りまくってThrottleCountを確認するためのスクリプト"
emoji: "🦔"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["cognito","locust"]
published: true
---

AWSでCognitoを利用されていますか？

Cognitoで監視をする際、以下のようなサイトを参考に [ThrottleCount](https://docs.aws.amazon.com/ja_jp/cognito/latest/developerguide/metrics-for-cognito-user-pools.html) などのメトリクスを監視しようとしたことはありませんか？私はあります。

https://tech.quickguard.jp/posts/monitoring-aws-cognito/

要するに「CognitoのRate Limitを超えるような大量アクセスが届くなら、クオータの上限緩和の申請をしなきゃ」ということに気付けるようにCloudWatch Alarmで通知を受け取りたいという願望です。

ということで今回[Locust](https://locust.io/)という負荷試験に向いているツールを使って、サクッとCognitoのAPIレート制限まで認証リクエストを送るスクリプトを書いてみました。

なお、今回記載する内容はあくまでメトリクスの挙動を確かめることを目的としております。

また、今回のCognitoの利用用途はM2M認証のように、クライアントシークレットを利用するタイプを想定しており、ブラウザのログイン画面からログインするタイプを想定しておりません。

## 私の環境

### OSなど

| 項目   | 値                   |
| ------ | -------------------- |
| OS     | Mac(Apple M1) 13.3.1 |
| Python | 3.11.6               |
| Locust | 2.21.0               |

### Cognito ユーザープール アプリケーションクライアントの設定

| 項目                     | 値                       |
| ------------------------ | ------------------------ |
| アプリケーションタイプ   | 秘密クライアント         |
| クライアントシークレット | 生成する                 |
| 認証フロー               | ALLOW_REFRESH_TOKEN_AUTH |

その他リソースサーバーなどの設定を行った上で、`/oauth2/token` のエンドポイントへリクエストを送信します。

また、特にリクエストレートのクォータ引き上げのリクエストを行っていないため、120RPS程度で制限に引っかかるアカウント環境となっています。

https://docs.aws.amazon.com/ja_jp/cognito/latest/developerguide/limits.html

## Locust のスクリプト

早速ですが、以下のスクリプトでリクエストを送信しました。

```python:app.py
from locust import TaskSet, HttpUser, task, constant
from pprint import pprint
import base64


class CheckThrottle(TaskSet):
    count = 0

    def on_start(self):
        # UserPoolのドメインを参照
        self.url = "<url>"
        # 設定したアプリケーションクライアントのクライアントIDとクライアントシークレット
        self.client_id = "<client_id>"
        self.client_secret = "<client_secret>"

    def token_request(self):
        # 以下を参考にリクエストのパラメーターを作成
        # https://docs.aws.amazon.com/ja_jp/cognito/latest/developerguide/token-endpoint.html
        credentials = base64.b64encode(
            f'{self.client_id}:{self.client_secret}'.encode('utf-8')).decode('utf-8')
        headers = {
            'Content-Type': 'application/x-www-form-urlencoded',
            'Authorization': f'Basic {credentials}'
        }
        data = f"grant_type=client_credentials&client_id={self.client_id}&client_secret={self.client_secret}"
        res = self.client.post(self.url, data=data, headers=headers)
        print(res.text)

    """
    シナリオの実行
    """
    @task(1)
    def scenario(self):
        self.token_request()


class MyUser(HttpUser):
    wait_time = constant(1)  # 1秒おきにリクエストを送る
    tasks = [CheckThrottle]
```


あとはLocustで実行するさいに、`Number of users` を300、`Spawn rate` を100にでも設定することでものの数秒で以下のようにHTTPステータスコード429でレスポンスが帰ってくるようになりました。

```bash
locust -f app.py -u 300
```

```
{"error":"Rate exceeded"}
```

また、少し見えにくいですが `ThrottleCount` のメトリクスも計上されていることを確認できました。

![メトリクスの確認](/images/cognito_throttle_count/throttlecount.jpg)

今回はだいぶ省略していますが、このメトリクスを監視対象とした CloudWatch Alarm などによる通知もバッチリ受け取れました。

以上小ネタ感が非常に強い中で、最後までご覧いただきましてありがとうございました。

