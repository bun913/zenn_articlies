---
title: "Q: How to name the test title for non-developer people?"
free: true
---

[jp]

## 要約: ここだけ見れば全部わかる

- テストタイトルは非開発者も理解できるように以下の情報を含むと素敵です
- 含んでくれると嬉しい情報
  - テストの対象 (System Under the Test)
  - テストの条件・いつ ( When )
  - テスト対象の振る舞い (Behavior)
    - `should` や `must` などの助動詞は入れず、いきなり振る舞い（動詞）から書き始めてOKです
- 以下みたいにテストの実行結果を貼るだけで何をテストしているかわかると素敵です
  - この結果を「こういう検証までしてるよ」という情報を渡すだけで、QAEやPOなどの非開発者もきっと喜びます
  - 特にQAEとしては「じゃあシステムテストではこの領域に力を入れよう」などの時間削減につながり、逆に「このケースは開発者テストで入れてくれません？」とより安心できるテストを書くことにつながるかもしれません
```bash
ServiceNameValidator
    ValidSpacingRule
      ✓ judge OK when service name is "Security Hub"
      ✓ judge NG when service name is "SecurityHub"
    ValidPrefixRule
      ✓ judge OK when service name is "AWS Security Hub"
      ✓ judge NG when service name is "Amazon Security Hub"
    ErrorHandling
      ✓ diplay error message when service name is empty
```

## 背景・思想

### 揺れる、揺れるテストタイトル

実はテストのタイトルは開発者によって揺れがちです。なんなら同じ人が書いても揺れがちですよね。

以下のような状況は誰しも見たことがあるのではないでしょうか？

```ts
// 開発者Iさん
// 日本語で書いている。抽象度高めのタイトル。（itよりtestを使う派）
describe("ServiceNameValidator", () => {
    describe("ValidSpacingRule", () => {
        test("必要なスペースが入っていなければfalseを返す", () => {
            const validator = new ServiceNameValidator("SecurityHub")
            // 省略
            expect(result).toBe(false)
        })
    })
})

// 開発者Mさん
// 英語で書いている。it派。shouldとかmustとか入れている
describe("ServiceNameValidator", () => {
    describe("ValidSpacingRule", () => {
        it("should return false when necessary space is not included", () => {
            const validator = new ServiceNameValidator("SecurityHub")
            // 省略
            expect(result).toBe(false)
        })
    })
})

// 開発者Aさん
// 開発者たるもの不要なネストを作らない。
test("ServiceNameValidator, validSpacingRule, judgeOK", () => {
    const validator = new ServiceNameValidator("Security Hub")
    // 省略
    expect(result).toBe(true)
})
```

安心してください。みなさんテストを書いている時点で、素晴らしいです。ただ、テストのタイトルについてルールが統一されていないだけだと思います。

### おすすめのテストタイトル

開発者テストの一番の目的は「開発者が安心して継続して開発できるようにする」だと思います。

ただ、それ以外の用途にも使えるんじゃないかと思う日々です。

開発の皆さんは「関数やクラス名に意図を込める」ことをしていますよね。このテストタイトルについて、「非技術者の人にも結果だけ貼れば何をテストしているかわかるようにする」という意図を込めることができるんじゃないかと思います。

そのために必要な情報は以下の3つだと思います。

- テストの対象 (System Under the Test)
- テストの条件・いつ ( When )
- テスト対象の振る舞い (Behavior)

それもできるだけ具体的に書くことで、まるで仕様を具体化しているようでとても素敵だと思います。

### ぺたっと結果を貼るだけでいい

以下のようにPRやチケットの説明欄にテストの結果を貼ることを想像してみてください。

```bash
ServiceNameValidator
    ValidSpacingRule
      ✓ judge OK when service name is "Security Hub"
      ✓ judge NG when service name is "SecurityHub"
    ValidPrefixRule
      ✓ judge OK when service name is "AWS Security Hub"
      ✓ judge NG when service name is "Amazon Security Hub"
    ErrorHandling
      ✓ diplay error message when service name is empty
```

これを見るだけでコードレビューが捗りますよね？「あぁスペースの検証をしているのか。じゃあこういう実装をしているのかな」といった情報が一瞬で伝わります。

さらに実装コードを見る以前から「ん？この挙動受け例基準と違うんじゃない？」といった気づきを得ることができるかもしれません。

さらにさらに、QAEやPOなどの非技術者の方にも「このケースは念入りにテストされているからシステムテストでは少しでいいか」「探索的テストに力を入れよう」といった気づきをもたらすことができるかもしれません。

テストってすごいですね。「開発者に安心感をもたらす」だけでなく、「誰かに気づきを与える」という役割を持たせることができるんですね。すごい。

### 参考資料・リンク

- 以前筆者が発表した資料: [あなたはどっち派？XSpec系テストフレームワークの構造化流派について](https://speakerdeck.com/bun913/xspec-title-naming)
- 今回の考えが載っている書籍
  - [The Art of Unit Testing, Third Edition](https://www.manning.com/books/the-art-of-unit-testing-third-edition)
  - [単体テストの考え方/使い方 | マイナビブックス](https://book.mynavi.jp/ec/products/detail/id=134252)

-------------

[en]

## Summary: Everything you need to know is here

- It's nice to include the following information in the test title so that non-developers can understand it
- Information that you would be happy to include
  - System Under the Test
  - When
  - Behavior of the test target
    - It's OK to start writing from the behavior (verb) without including auxiliary verbs such as `should` or `must`
    - It's nice to be able to understand what you are testing just by pasting the test results like below
      - Just by passing on the information that "I'm verifying this," non-developers such as QAE and PO will surely be pleased
      - Especially as a QAE, it may lead to time savings such as "Let's focus on this area in system testing," and conversely, it may lead to writing tests that are more reassuring, such as "Can't you put this case in developer testing?"

```bash
ServiceNameValidator
    ValidSpacingRule
      ✓ judge OK when service name is "Security Hub"
      ✓ judge NG when service name is "SecurityHub"
    ValidPrefixRule
      ✓ judge OK when service name is "AWS Security Hub"
      ✓ judge NG when service name is "Amazon Security Hub"
    ErrorHandling
      ✓ diplay error message when service name is empty
```

## Background and Philosophy

### Unstable test titles

In fact, test titles tend to vary depending on the developer. In fact, even the same person tends to vary.

Have you ever seen a situation like this?

```ts
// Developer I
// Written in Japanese. High abstract title. (prefers test over it)
describe("ServiceNameValidator", () => {
    describe("ValidSpacingRule", () => {
        test("必要なスペースがなければfalseを返す", () => {
            const validator = new ServiceNameValidator("SecurityHub")
            // omitted
            expect(result).toBe(false)
        })
    })
})

// Developer M
// Written in English. it lover. Includes should and must
describe("ServiceNameValidator", () => {
    describe("ValidSpacingRule", () => {
        it("should return false when necessary space is not included", () => {
            const validator = new ServiceNameValidator("SecurityHub")
            // omitted
            expect(result).toBe(false)
        })
    })
})

// Developer A
// Developers don't create unnecessary nests.
test("ServiceNameValidator, validSpacingRule, judgeOK", () => {
    const validator = new ServiceNameValidator("Security Hub")
    // omitted
    expect(result).toBe(true)
})
```

Don't worry. Everyone is doing a great job just by writing tests. However, I think it's just that the rules for test titles are not unified.

### Recommended test titles

I think the main purpose of developer tests is to "make developers feel safe and continue development."

However, I think it can be used for other purposes as well.

Developers are "putting intentions into function and class names," right? I think you can put the intention of "making it possible for non-technical people to understand what you are testing just by pasting the results" into this test title.

I think the following three pieces of information are necessary to do this.

- System Under the Test
- When
- Behavior of the test target

By writing as specific as possible, it looks like you are concretizing the specifications, which I think is very nice.

### Just paste the results

Imagine pasting the test results in the PR or ticket description as follows.

```bash
ServiceNameValidator
    ValidSpacingRule
      ✓ judge OK when service name is "Security Hub"
      ✓ judge NG when service name is "SecurityHub"
    ValidPrefixRule
      ✓ judge OK when service name is "AWS Security Hub"
      ✓ judge NG when service name is "Amazon Security Hub"
    ErrorHandling
      ✓ diplay error message when service name is empty
```

Isn't code review easier just by looking at this? You can instantly understand "Oh, the feature verifying the space? Then I guess you're implementing like this."

Even before looking at the implementation code, you may be able to get a sense of "Huh? Isn't this behavior different from the acceptance criteria?".

Furthermore, you may be able to give insights to non-technical people such as QAE and PO, such as "This case is being thoroughly tested, so we don't need much in system testing" or "Let's focus on exploratory testing."

Tests are amazing. Not only do they "give developers peace of mind," but they can also "give someone insights." It's amazing.

### References and Links

- Presentation slides by the author: [Which side are you on? About the structured sect of XSpec test frameworks](https://speakerdeck.com/bun913/xspec-title-naming)
- Books that contain the author's thoughts
  - [The Art of Unit Testing, Third Edition](https://www.manning.com/books/the-art-of-unit-testing-third-edition)
  - [Unit Testing Principles, Practices, and Patterns: Effective testing styles, patterns, and reliable automation for unit testing, mocking, and integration testing with examples in C#](https://www.amazon.co.jp/Unit-Testing-Principles-Practices-Patterns/dp/1617296279)

