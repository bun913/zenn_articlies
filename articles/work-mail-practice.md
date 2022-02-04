---
title: "AWS WorkMailで複数人でメールの受信・返信を試してみた"
emoji: "👋"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["aws","workmail"]
published: true
---

こんにちは。

今回ビジネス上の以下のようなユースケースを想定し、AWS WorkMailを練習がてら試してみたら、とても楽しかったので記事にいたします。

- メールマガジンなどでSESを利用して一斉にメール送信
- それを特定のドメインでメール受信する(WorkMail)
- 複数人の担当者（グループ）で任意のメールクライアントを使って返信できるようにする

お問い合わせの対応を行うイメージでしょうか。

この辺はたくさん先人がいましたので、以下のような記事を参考にさせていただきました。

https://dev.classmethod.jp/articles/amazon-ses-amazon-workmail-send-and-receive/

https://qiita.com/ysKey2/items/2b019337772f8499beec


## WorkMailで組織の作成

まずWorkMailは以下のリージョンでしか利用できない(2022.2.4現在)ため、今回は バージニアリージョンで試してみました。

- バージニア北部
- オレゴン
- アイルランド

WorkSpaceに入るとまず組織の作成をおこないます

![組織の作成](/images/workmail/organization_setting.png)

名前は適当に `test-bun` として、今回利用したい独自ドメインを入力しました(`support.hoge.com`とします)

組織が作成できたら、その組織を選択します。

## ドメインの追加

組織のサイドバーから `Domains` を選択します。

`support.hoge.com` を選択します。

するとWorkMailからDNSの登録の指示がたくさん出ています。

- Domain ownership
    - そもそも本当にそのドメインの所有者か検証(所有者ならDNSレコード登録できるよね？)
- WorkMail configuration
    - WorkMailの設定
    - MXレコードなどを指定される
- Improved security
    - SPFのための設定など

これら全てをDNSサーバーで登録します（私の場合お名前.comを今回利用しましたが、Route53で大丈夫です)

なお、SPFなどについては他のサイトにたくさんわかりやすい記事がありますので、ご参照ください

https://sendgrid.kke.co.jp/blog/?p=10121


## グループ・ユーザーの作成

ドメインの検証が終わったら、いよいよグループ・ユーザーを作成します。

まずサイドバーから `Groups` を選び、 `Create group` を押下します。

`support` という名前のGroupを作成します。

![グループ作成](/images/workmail/group_create.png)

その後に ユーザーを2名作成します。

同じようにサイドバーから `Users` を選び `Create user` を押下します。

![ユーザー作成](/images/workmail/user_create.png)

ユーザーを試しに2名作成しました。

- owner
  - 名前は `test hanako`
- company1
  - 名前は `test tar`

![user](/images/workmail/users.png)


次にまた サイドバーから `Groups` を選び、 Add memberで先ほど作成した2名のユーザーを追加します。

![](https://storage.googleapis.com/zenn-user-upload/972f843bcb33-20220204.png)

## WorkMailにユーザーでログインしてみる

次にサイドバーから `Organization` を選択すると、下の方に `User login` と書いてある項目があり、その中に `Amazon WorkMail web application` という項目でログイン用のURLが記載されているためクリックします。

以下のような画面になるため、先ほど作成した ownerのログイン情報でログインしてみます

![ログイン](/images/workmail/login.png)

試しに `owner` のメールアドレスに対して、自分のmailクライアントから送信してみると、`My Mail` の `Inbox` にメールがとどいています!

![](/images/workmail/workspace.png)

## グループ宛のメールを確認

次に自身のメールクライアントから グループ宛にメールを送ってみます。

`support` と名前をつけたグループのメールアドレス宛に送ってみるとメンバーである `owner` はちゃんとメールを受け取れました

![メール](/images/workmail/group_mail.png)

ちゃんと返信もばっちりできました！

![reply](/images/workmail/reply.png)

その後、もう一人のユーザーである `company1` でWorkMailにログインしてみると、ちゃんと `support` グループ宛に送ったメールを受け取れていました！

![メール2](/images/workmail/user2.png)


想像以上に楽にメールの送受信、グループの作成ができました。

WorkMail本当に便利です!!

もっと遊んでみようかと思います！！