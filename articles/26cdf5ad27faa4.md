---
title: "[偉い！]JSで書かれたOSSライブラリをよくするための改善ステップをご紹介します"
emoji: "🛤️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["testrail","postman", "newman"]
published: true
publication_name: "moneyforward"
---

こんにちは。ダイの大冒険ガチ勢の[bun913](https://x.com/bun7623514)と申します。

先日以下のように、パブリックアーカイブになっていたOSSライブラリを引き継いでnpmパッケージとして公開したことを記事にしました。

https://zenn.dev/moneyforward/articles/5452d526d695d0

公開直後の週で30ダウンロードされておりますが、ほとんど私か社内のごく一部の方々だと思われます。

![package_weekly_downloads](/images/newman_reporter/newman-weekly-download.png)

今回は~~私の偉い活動~~、JSで書かれたライブラリをリファクタリングするために行った具体的な改善ステップを紹介します。

型がなく、詳しい仕様がわからないパッケージに対するリファクタリングステップとして、多少なりとも参考になれば幸いです。

## まずは成果を見てください

元々パブリックアーカイブになっていたライブラリはテストが動かない状態であり、そのためにテストを動かすための環境を整えることから始めました。

興味のある方は以下のリポジトリをご覧ください。

https://github.com/bun913/newman-reporter-testrail-neo

### パッケージ公開直後（以前の記事の時点）

Branch Coverage（分岐網羅）を63.07%まで引き上げてパッケージを公開していました。

```
---------------------|---------|----------|---------|---------|----------------------
File                 | % Stmts | % Branch | % Funcs | % Lines | Uncovered Line #s    
---------------------|---------|----------|---------|---------|----------------------
All files            |   58.99 |    63.07 |   53.33 |   58.99 |                      
 TestrailReporter.js |   54.08 |       60 |      75 |   54.08 | ...3,186-199,203-254 
 environment.js      |     100 |      100 |     100 |     100 |                      
 index.js            |       0 |        0 |       0 |       0 | 1                    
 logger.js           |   64.28 |    41.66 |     100 |   64.28 | ...32-35,51-57,60-65 
 testRailApi.js      |   55.17 |    72.72 |   45.45 |   55.17 | ...2-84,87-92,96-102 
---------------------|---------|----------|---------|---------|----------------------
```

ただしこのパッケージの機能の重要な部分を占める `TestrailReporter.js` に対して分岐網羅が60%とちょっと不安な状態であることがわかります。

カバレッジだけで品質を判断することはできませんが、この規模のパッケージに対して「このオプションが指定されたときはこういう挙動をする」とうテストが不足しているのはちょっと心配です。

### パッケージ公開後（現在）

後に記載するステップを経て、Branch Coverage（分岐網羅）を約90%まで引き上げました。

```
----------------------------|---------|----------|---------|---------|----------------------------------------------------------------
File                        | % Stmts | % Branch | % Funcs | % Lines | Uncovered Line #s
----------------------------|---------|----------|---------|---------|----------------------------------------------------------------
All files                   |   83.56 |    89.94 |   78.94 |   83.56 |
 TestrailReporter.js        |   53.16 |     92.3 |      50 |   53.16 | 9-12,14-21,25-32,44-49,53-58,60,62-65
 environment.js             |     100 |      100 |     100 |     100 |
 index.js                   |     100 |      100 |     100 |     100 |
 logger.js                  |   90.14 |    75.75 |     100 |   90.14 | 52-58
 newmanResultModifier.js    |     100 |    95.23 |     100 |     100 | 23
 testRailApi.js             |    61.2 |    80.76 |   54.54 |    61.2 | ...1-32,35-36,48-49,54-57,65-71,78-80,82-84,87-88,91-92,96-102
 testRailResultProcessor.js |   98.17 |    93.18 |     100 |   98.17 | 78,80-81
 tsetRailResultSender.js    |   96.34 |    97.36 |     100 |   96.34 | 3-5
----------------------------|---------|----------|---------|---------|----------------------------------------------------------------
```

これに加えて、いくつかのパッケージ全体としての挙動を確認するためのシステムテストにあたる仕組みを追加しました。

また、当然テストを追加するだけでなく、以下のようにコードの実装自体も変更しています。

~~私の活動の尊さ、~~ 成果の一部となりますが、メインの複雑だったコードを他のクラスに分割した結果のdiffが以下のとおりです。

![diff](/images/newman_reporter/diff.png)

私のできる範囲で「動作を変更することなく、内部の実装を変更容易にしていく」という目標に向けて順調に進めることができていると思います。

## Step1: 課題の整理

まずはリリース当初の課題（願望含む）を以下のように整理しました。

- 前述のとおり、なんとかテストを追加したもののまだまだ機能の挙動を網羅できていない
- if文のネストや一つの関数の中に持つロジックが多めで、リファクタリングをしなければ継続的なメンテナンスが難しいと感じた
- リファクタリングをするためにもテストが必要
  - だが、コードを変更しないとテストが書きにくいというジレンマを抱えていました
- JSは素敵だがこの手の外部システムとやりとりをするライブラリは特に型があった方が良い
  - 要するに TypeScript に移行したい

TypeScriptに移行していくにも既存のライブラリの挙動を変えたくないので、以下のようにマイルストンを設定して、それぞれのステップを進めていきました。

- [x] TestRailReporterのテストで分岐を90%程度網羅する
- [x] その後にJavaScriptのコードのまま、リファクタリングを行う
- [ ] TypeScriptに移行する

上記のようにこの記事執筆時点ではまだTypeScriptに移行していませんが、そのための準備をある程度まで進めることができていると思います。

## Step2: テストケースを追加する

テストケースを追加してく流れで、CodeCovというサービスを導入してカバレッジが嫌でも目に入るようにしました。

https://zenn.dev/link/comments/18b3137eb3f8e4

### 少し苦しかったポイント

既存の処理の大部分を担っている大きめのクラスには2つのメソッドがありました

- `jsonifyResults`
  - newmanの実行結果を受け取って、TestRailに送信するための形式に変換する
- `pushToTestrail`
  - `jsonifyResults` で変換した結果をTestRailに送信する

これだけ見るととてもシンプルなのですが、実際には以下のような複雑な挙動を持っていました。

- `jsonifyResults`
  - newmanから受けとった実行結果を必要に応じて直接変更する（副作用がある）
  - 環境変数に応じてこの時点で`TestRail`にアクセスして、情報を取得する
  - 一つのnewmanの実行結果に対して、複数のTestRailのテストケースに対応する場合の処理
  - etc
- `pushToTestrail`
  - 結果をプッシュする前に実は情報を加工している部分がある
    - 環境変数に依存
  - パラメーターによりTestRailのどのエンドポイントにアクセスするかを変更する
  - etc

いずれの関数の中でも、環境変数に依存する部分が多く、テストを書くためにもフェイク（スタブ・モック）を差し込むのも苦労する状況でした。

また関数が小さな単位で分かれていないため、以下のような一塊のステップを都度テストするような形になってしまいました。

- `jsonifyResults`
  - 処理A（関数化されていない）
  - 処理B（関数化されていない）
  - 処理C (関数化されている)
  - if (環境変数による分岐)
    - 処理D（関数化されていない）
    - 処理E（関数化されていない）
  - 処理F（関数化されていない）
  - etc

- リファクタリングをするためにテストを追加する
- リファクタリングをしていないからテストが書きにくい
- だから先にコードを変更したくなる

というジレンマに陥っていましたが、以下のようにZennのスクラップにその時の気持ちを書きながらテストを追加していきました。

https://zenn.dev/link/comments/c88655563c8ec6

適宜システムテストも実行しながら、挙動が変わっていないことを確かめながら進めていきました。

### テストケースを追加する際のポイント

以下のように極力、「何の機能は」「どんな時に」「こう動く」ということを整理しながらBDD（振る舞い駆動開発）の形式を意識してテストを書いていきました。

```
✓ jsonifyResults (13)
       ✓ converts newwman test results to TestRail-compatible JSON format (3)
         ✓ converts a test result to one TestRail test run
         ✓ converts two mapped test case to two TestRail testruns
         ✓ converts two test cases to two TestRail test runs
       ✓ failed test case handling (1)
         ✓ converts a failed test case to a failed TestRail test run
       ✓ skipped test case handling (1)
         ✓ converts a skipped test case to a skipped TestRail test run
       ✓ umarked test case handling (1)
         ✓ skip when not including test caseid top of the test case title
       ✓ TESTRAIL_STEPS env value is (3)
         ✓ true: reports multiple assertions as a steps if they have same test case ids (2)
           ✓ reports as an failed test case when one step fails, even if others succeed
           ✓ resports as an success test case when all steps are success
         ✓ false (default value): reports multipe assertions as multiple test results (1)
           ✓ reports as two test results
       ✓ User wants to establish connection by title not by case id (2)
         ✓ When TESTRAIL_TITLE_MATCHING env value is `true` (2)
           ✓ Newman assertions with title `TestTitle` matched to TestRail test case with `TestTitle`
           ✓ Newman assertions with title `NoMatchedTitle` not matched to TestRail test case with `TestTitle
```

なお、このユニットテストのケースの命名については、以前以下のような発表をしています。

@[speakerdeck](08d47a5589034b70b5b4b3dd68f6fbdd)

## Step3: リファクタリングを行う

分岐網羅率も90%程度になり、「挙動というか仕様がある程度わかってきた」と自信を持ててきたので、リファクタリングを開始しました。

### リファクタリングのポイント

- 「リファクタリング」はここでは、「**機能の挙動は変更せずに、コードの実装を変更する**」という定義をはっきりと意識していました
- 一つの関数に集中する処理を丁寧に分割する
  - まずはリファクタリング対象のクラスにメソッドを追加し、そのメソッドに処理を移していく
  - テストの実行結果が変わらないことを確認しながら、別のクラスに処理を移していく
- クラスを分割する際には、DI（Dependency Injection）を意識して、フェイクを差し込みやすい形にする

例えば初期の関数では以下のようになっていたところを改善していきました。

```javascript
// Before のイメージ
class TestRailReporter {
    constructor() {
        // 処理
    }

    jsonifyResult() {
        // ここで初期化する
        this.testRailApi = new TestRailApi();
        // そのまま一気に処理を書いていくためフェイクを差し込むのが難しい
        // その他環境変数に依存する処理などの問題があった
    }
}

// 途中のイメージ
class TestRailReporter {
    constructor(testRailApi) {
        this.testRailApi = testRailApi;
    }

    jsonifyResult() {
        this.newmanResultProcess()
    }

    // まずは自分のクラスに処理を分割する
    newmanResultProcess() {
        // 関数に分けた処理を書いていく
    }
}

// After のイメージ
class TestRailReporter {
    jsonifyResult() {
        this.newmanResultProcessor.process();
        const newmanModifier = new NewmanResultModifier(this.testRailApi);

        newmanModifier.process();
    }
}

// 意図がわかるクラスに処理を分割する
class NewmanResultProcessor {

    process() {
        // 関数に分けた処理を呼び出すmain的な役割
    }

    matchTestCaseByTitle() {
        // 関数に分けた処理を書きながら、処理の意図を名前で表現する
    }
}
```

この進め方により、どんどん責務が分割され、フェイクを差し込みやすいクラスができていくので、テストのカバレッジはむしろ上がっていきました。

この段階に入ってクラスを分けていくことで、「えっ？Loggerっていう名前のクラスが結果に影響を与えるの？」と驚きながらも、それが分かるようになっていくのが楽しかったです。

## まとめ

- 社内でも多くの人が触るというよりも、一部の人が触る、ないと困るだろうなというライブラリを地道に改善しています
  - ここまで来るのにPull Requestを30件ほど一人で出しています（業務時間外）
  - 追加したテストケース数は80件くらいです
  - ~~もっと褒めてくれても全然OKですよ~~ 自身のSDET（Software Development Engineer in Test）としての成長に繋がっているので、楽しく取り組んでいます
- 「リファクタリング」という作業を通しながら、テストが先か構造の変更が先かというジレンマを感じながら真剣に取り組めました
  - 特に動的型付け言語はリファクタリングのやりがいがありますね

いかがでしたでしょうか？社内の方の笑顔を意識しながら、改善を進めていきました。~~もっとスタートかいいねとかつけてくれてもいいんですよ。~~

ここまでくればかなり良くなったので私が無理しなくても誰かが触りやすくはなった気がしています（まだまだ改善したいところはあります）。

どこまでやれるかわかりませんが、今後ともやれる範囲で色々な勉強を兼ねた活動を行いたいと思います。

以上、最後までお読みいただきありがとうございました。
