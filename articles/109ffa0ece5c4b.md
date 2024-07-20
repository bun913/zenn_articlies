---
title: "more-itertoolsという存在を知る【AtCoder Beginner Contest 363 振り返り（茶）】"
emoji: "🧭"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["AtCoder"]
published: true
publication_name: "moneyforward"
---

こんにちは。ダイの大冒険ガチ勢の[bun913](https://x.com/bun76235104)と申します。

皆さんはAtCoderという競技プログラミングに気軽に参加できるサービスをご存知でしょうか？

https://atcoder.jp/

競プロと聞くと一見とっつきにくいですが、普段プログラミングができないなかでも「**あ〜今アルゴリムを感じている。輝いている。俺は計算量を意識している**」という気持ちになれるので、とてもおすすめです。

まったく参加したことのない方でも、以下のような記事を見るだけで簡単な問題を解けるようになりますので、興味があればぜひ見てください。

https://qiita.com/drken/items/fd4e5e3630d0f5859067

この記事は以下のような1社会人のエンジニアとして、限られたリソースの中でも復習をしつつも、同じレベル帯の方になんらか参考になることがあるかもしれないというモチベーションで書いています。

- 私の状況・現況
  - 茶色コーダーと呼ばれる下から2番目のランクづけ
    - 参考: [Algorithm部門のレーティングと業務における期待できる活躍 - AtCoderInfo](https://info.atcoder.jp/utilize/jobs/rating-business-impact)
  - 競プロ好きだけどつよくない
  - 正直数学はできない
  - 競プロに全てのリソースを割くことはできない
    - ので、公式の[AtCoderで強くなるには](https://info.atcoder.jp/more/practice/stronger)のとおり復習だけすることにしました

## [A - Piling Up](https://atcoder.jp/contests/abc363/tasks/abc363_a)

### 考えたこと

- あまりを使って効率的に解けそうだけど、今回は問題文にRが1以上299以下であることが保証されているから素直にif文で書くか
  - というようにA,B問題くらいまでは「とにかく早く提出する」という解法をとることがおおいです

```python
# -*- coding: utf-8 -*-
"""
解く前のメモ用
"""

from sys import setrecursionlimit

setrecursionlimit(10**8)


def solve():
    act()


def act():
    R = int(input())
    if R <= 99:
        print(100 - R)
    elif R <= 199:
        print(200-R)
    elif R <= 299:
        print(300-R)
    else:
        print(400-R)


solve()
```

特に悩むことなく解くことができました。

が、今回みたいにRの範囲に保証がない場合は、以下のようにあまりを使って効率的に解けます（コンテスト終了後に提出）

```python
# -*- coding: utf-8 -*-
"""
解く前のメモ用
"""

from sys import setrecursionlimit

setrecursionlimit(10**8)


def solve():
    act()


def act():
    R = int(input())
    mod = R % 100
    print(100-mod)


solve()
```

## [B - Japanese Cursed Doll](https://atcoder.jp/contests/abc363/tasks/abc363_b)

### 考えたこと

- 今回は各人の髪の長さ(L[i])が100以下であることが保証されているなぁ
- さらに目指すべき髪の長さTも100以下であることが保証されている
- さらに、さらにNも100人しかいないので **最大でも100日繰り返せばみんなが目標とするTに到達するので、素直に繰り返しながら条件を満たしている人がP人以上か見れば良さそう**


```python
# -*- coding: utf-8 -*-
"""
解く前のメモ用
"""

from sys import setrecursionlimit

setrecursionlimit(10**8)


def solve():
    act()


def act():
    N, T, P = map(int, input().split())
    L = list(map(int, input().split()))
    for n in range(100):
        cur = 0
        for l in L:
            if l >= T:
                cur += 1
        if cur >= P:
            print(n)
            return
        for i in range(N):
            L[i] += 1


solve()

```

## [C - Avoid K Palindrome 2](https://atcoder.jp/contests/abc363/tasks/abc363_c)

### 考えたこと(1回目 ~ 5回以上) 不正解

- 今回は与えられる文字列の長さが10と非常に短いなぁ
- ということは与えられる文字列Sを全て並び替える = 順列を計算したとしてお、せいぜい `O(10**6)` だよね
- であれば、その各候補に対して `O(N)` で部分文字列を切り出して、さらに回文判定をしたとしても余裕で全探索できるな！（ **フラグ** ）

ということで以下のようなコードを提出しました。

が、少なくないPythonユーザーの方が遭遇したであろう2つのテストケースがどうしても `TLE（時間オーバー）` になる悲しみに遭遇しました。

```python
# -*- coding: utf-8 -*-
"""
解く前のメモ用
"""

from sys import setrecursionlimit
from itertools import permutations

setrecursionlimit(10**8)


def solve():
    act()


def act():
    N, K = map(int, input().split())
    S = input()
    ans = 0
    cands = set(permutations(S))
    for cand in cands:
        cand = list(cand)
        is_ok = True
        # 長さKの部分まで切り出した部分文字列を列挙
        for l in range(N-K+1):
            s = cand[l:l+K]
            if is_palindrome(s):
                is_ok = False
                break
        if is_ok is True:
            ans += 1
    print(ans)


def is_palindrome(s) -> bool:
    n = len(s)
    for i in range(n // 2):
        if s[i] != s[n - i - 1]:
            return False
    return True


solve()
```

### コンテスト後の提出と得ることができた知識

コンテスト終了直後に以下の解説を拝見して、まさに自分がTLEとしてハマったことを解説いただいていました。

https://youtu.be/nsa-yNV7Xbg?si=s0m-iqgdBr135o2B&t=61

おすすめの方法としてあげられていたのが、「AIにC++に変換してもらう」でしたが、「 **いや、きっと何か私が知らないことがあるだけでPythonでも解けるはずだろう** 」 と考えて、Pythonで正答を得ていた方のコードを見ているうちに `more-itertools` という外部ライブラリの存在を知りました。

実はこちらAtCoder側で利用可能な外部ライブラリとして明記いただいており、`numpy` や `pandas` などと同様に利用可能です。

https://img.atcoder.jp/file/language-update/language-list.html

ということで、実は `itertools の permutations` の部分を　`more_itertools distinct_permutations` に書き換えるだけで正答を得ることができました。

```python
# -*- coding: utf-8 -*-
"""
解く前のメモ用
"""

from sys import setrecursionlimit
from more_itertools import distinct_permutations

setrecursionlimit(10**8)


def solve():
    act()


def act():
    N, K = map(int, input().split())
    S = input()
    ans = 0
    cands = distinct_permutations(S)
    for cand in cands:
        cand = list(cand)
        is_ok = True
        # 長さKの部分まで切り出した部分文字列を列挙
        for l in range(N-K+1):
            s = cand[l:l+K]
            if is_palindrome(s):
                is_ok = False
                break
        if is_ok is True:
            ans += 1
    print(ans)


def is_palindrome(s) -> bool:
    n = len(s)
    for i in range(n // 2):
        if s[i] != s[n - i - 1]:
            return False
    return True


solve()

```

## まとめ

- ABC361に参加して、コンテスト中に2問だけ解くことができました
  - 私はC問題を絶対解きたい、D問題はできれば解きたいというスタンスで取り組んでいるので、非常に悔しかったです
- 一方で解法の方向性はよく、あとは「このライブラリを知っていれば解けた」というものでした
  - さらに今回悔しい思いをしたおかげで同じような問題に当たった時は冷静に対処法を考えることができると思います

私のような「リソースを競プロにあまり使えない人」こそ、「解けなかった一問だけ解説を見ながら解く」、「もっといい解法がないか解説を見てみる」ということが大事だと感じています。

今後も「楽しいからやる」というスタンスは保ちつつ、自分のペースで下がったり上がったりを繰り返しながら強くなっていければいいなと思います。

以上、bun913でした。ありがとうございました。
