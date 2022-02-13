---
title: '結局Githubに学習履歴を統一した方が諸々良かった'
emoji: '🙌'
type: 'idea' # tech: 技術記事 / idea: アイデア
topics: ["github","githubactions","ポエム"]
published: true
---

私はアウトプットを大事にしたいと思いつつも以下のような悩みを持っておりました。

- 毎日何かしらの学習をしているので、毎日学習履歴を残したい
- エンジニアたるもの、どうせならGithub に草を生やしたい
- コーディング系の学習はしていなくても、読書したり資格の学習をしている日があるが、Github のプロフィールだけ見ると何もしていないように見える
- また資格学習のために外部サービスを使って勉強しているけど同じく何も勉強していないように見える

そこで １年前くらいにGithub でアウトプットを残すのに「TIL」リポジトリを利用するという考え方を知ったので、自分もやってみることにしました。

https://qiita.com/nemui_/items/239335b4ed0c3c797add

やってみた結果、自分なりに良い感じにモチベーションを保てる方法が固まってきたので、誰得ですが共有しておきたいと思います。

## 学習の方法とアウトプットしやすさ

以下は私が主に行う学習の方法と、そのアウトプットのしやすさについてです。

| 学習の方法                               | Github へのアウトプットしやすさ | 備考                                   |
| ---------------------------------------- | ------------------------------- | -------------------------------------- |
| コーディング・サービス開発               | ○                               | 書いたコードがそのままアウトプット     |
| 読書                                     | △                               | 要約や自分のまとめがアウトプットになる |
| 動画視聴                                 | △                               | 要約や自分のまとめがアウトプットになる |
| 外部サービスによる問題演習等(資格学習等) | ×                               | 外部サービス側に学習履歴は残るが・・・ |

改めて説明する必要もないのですが、本や動画サービスによるインプットに関してはマークダウン形式でまとながら行うため、そこまでアウトプットが苦ではありません。

逆に外部サービスを使った資格学習のための問題演習などは少し手間です。
読書や動画サービスのようにマークダウンにまとめながらアウトプットしてもよいのですが、資格系の問題演習は移動時間や隙間時間に利用することも多いので、都度Githubにコミットするのは難しいです。

なんとか作業を自動化したいので以下のような方法を利用するようにしてみました。

### 学習履歴のデータを取得する

例えばStudyplusではAPIが提供されています。

https://info.studyplus.co.jp/contact/studyplus-api

利用しているサービスによっては、このようにAPIを提供してくれていたりするので、これを利用してデータを取得します。

またサービスの利用規約を確認して、常識的な範囲で自身の学習履歴のデータをスクリプトを組んで取得するのもありだと思います。

※ 今回個別のサービスのデータ取得用のスクリプトなどは記載しておりません

### 1日1回自動でアウトプットする

上記でデータを取得する処理を自動化するスクリプトを記載できたら、以下の一連の作業を自動化します。

1. スクリプトでデータ取得
2. 取得したデータを整形・アウトプット(ファイルに学習履歴などを出力)
3. commit, pushを毎日自動実行する

私の場合GithubActionsを利用しております。

設定ファイルは以下のような形です。

```yml
name: CommitStudyHistory

on:
  # 手動でもWolkflowを利用できるようにしておく
  workflow_dispatch:
  schedule:
    # JSTで8:45に毎日１回実行
    - cron: '45 23 * * *'

jobs:
  output_study_history:

    runs-on: ubuntu-latest

    steps:
    # Pythonスクリプトを利用
    - uses: actions/checkout@v2
    - name: Set up Python 3.10
      uses: actions/setup-python@v2
      with:
        python-version: "3.10"
    - name: exec aggregate
      env:
          # コマンドを実行するための環境変数をセット
          GIT_USER: ${{ secrets.GIT_USER }}
          GIT_MAIL: ${{ secrets.GIT_MAIL }}
          TZ: 'Asia/Tokyo'
      run: |
        pip install requests
        # データ取得・取得したデータを整形してファイルに出力
        python index.py
        git config --local user.name ${GIT_USER}
        git config --local user.email ${GIT_MAIL}
        git add -N .
        # 差分があった時だけcommit,pushする
        if ! git diff --exit-code --quiet
        then
          git add .
          git commit -m 'Update Record Data'
          git push
        fi
```

ちなみに `${{ secrets.HOGE }}` などど記載しているものは Githubのシークレットの機能を利用してセキュアに環境変数に機密情報などを渡しております。

https://docs.github.com/ja/actions/security-guides/encrypted-secrets

## 結果

私の場合ほぼ毎日、読書またはコーディング系の学習を行なっているのですがどうしても時間や気力がなくても、隙間時間に外部サービスを利用して勉強したことがちゃんとGithubの草が生えるようになりました。

![Githubの草](/images/github_study_history/field.png)

これにより「学習の方法」に拘らずGithubに学習履歴が「活動」として草を生やしてくれるため、問題演習などの敷居の低さが下がって、学習のモチベーション向上に繋がったように感じています！
