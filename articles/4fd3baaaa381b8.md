---
title: "Pythonのproduct関数で全数列パターン列挙できる【AtCoder Beginner Contest 367 振り返り（茶）】"
emoji: "🔢"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["AtCoder", "Python"]
published: true
publication_name: "moneyforward"
---

こんにちは。ダイの大冒険ガチ勢の[bun913](https://x.com/bun7623514)と申します。

皆さんはAtCoderという競技プログラミングに気軽に参加できるサービスをご存知でしょうか？

https://atcoder.jp/

競プロと聞くと一見とっつきにくいですが、普段プログラミングができない方でも「**このくらいのデータ量なら素直にデータを全列挙すれば2秒以内にプログラムは終わるはず**」と浸れるので、とてもおすすめです。

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

## [A - Shout Everyday](https://atcoder.jp/contests/abc367/tasks/abc367_a)

### 考えたこと and 不正解だった実装

- え・・・これどうなるんだっけ？
- めんどうだな。寝た時間が夜、起床時間が朝とかだとAの時間に寝ているかどうかの判定がめんどくさいぞ
- さらにAの時間も0だと24時っぽいからそのへんも考慮するともっとめんどくさそう
- え？じゃあこんな感じで起床時間のほうが遅ければ+24して、さらにAの叫ぶ時間が0時なら24時と変換すればいいのでは？
- と考えて以下のコードを提出しましたが間違ってしまいました

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
    A, B, C = map(int, input().split())
    if C < B:
        C += 24
    if A == 0:
        A = 24
    if B <= A <= C:
        print("No")
        exit()
    print("Yes")


solve()

```

### 正解になった実装

- いや、条件分けを楽しようと思って逆に難しくしてしまっているな
- 素直に就寝時間が起床時間を超えているかの場合分けを入れよう
- と考えて今度は問題を素直に解くことで正解に辿り着きました
- サンプルを通って満足する謎実装はやめようと心に誓いました

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
    A, B, C = map(int, input().split())
    # 起きた時間が素直に寝始めた時間より大きい場合
    if B < C:
        if B <= A <= C:
            print("No")
        else:
            print("Yes")
    # 起きた時間が寝始めた時間より小さい場合
    else:
        # その場合B以上（A,B,Cは異なるので同じ数は考えなくても良いが）
        if C <= A <= B:
            print("Yes")
        else:
            print("No")


solve()
```

## [B - Cut .0](https://atcoder.jp/contests/abc367/tasks/abc367_b)

### 考えたこと and 正解になった実装

- 今日の問題は[このまえ](https://atcoder.jp/contests/abc366/tasks/abc366_b)と違って素直な問題だぞ
- あまり実数とか難しく考えずに文字列として受け取って、最後に0がある限り消し続ければいいだけかな
- その上で、最後に`.`がある場合はそれも消そう
- と、考えて提出したコードが以下のとおりです


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
    X = input()
    ans = [s for s in X]
    # 0が続いている場合は含めない
    while True:
        if ans[-1] == "0":
            ans.pop()
            continue
        break
    if ans[-1] == ".":
        ans.pop()
    print("".join(ans))


solve()
```

## [C - Enumerate Sequences](https://atcoder.jp/contests/abc367/tasks/abc367_c)

### 考えたこと and 正解になった実装

- 要するにたとえばNが3の場合は
  - `[1,1,1]`, `[1,1,2]` みたいに順番に数列を作ればよさそう
  - ということはNが最大で8、Rが1以上5以下なのでパターンとしては `5^8` くらいまでしかない
- 任意の数列`[1,2,3,4,5]` からN個選んだ全ての組み合わせを列挙するという時には、`itertools.product` が便利だったよね
  - 参考: [Pythonで複数のリストの直積（デカルト積）を生成するitertools.product | note.nkmk.me](https://note.nkmk.me/python-itertools-product/)
  - こちらの「同じリストを繰り返し使用: 引数repeat」というパターンを利用しています

```python
# -*- coding: utf-8 -*-
"""
解く前のメモ用
Nが8で、Rも1以上5しかないのですべての数を列挙してもO(10**6)くらい
"""

from sys import setrecursionlimit
from itertools import product

setrecursionlimit(10**8)


def solve():
    act()


def act():
    N, K = map(int, input().split())
    R = list(map(int, input().split()))
    # NOTE: コンテスト後追記(1,1,1),(1,1,2)のようにN個の要素の数列を作ってくれる(candに1つの数列が代入される)
    # 1以上5以下の要素をN個選んだ全ての組み合わせを列挙する
    for cand in product(range(1, 6), repeat=N):
        # R[i]以下でないといけない
        is_break = False
        for i in range(N):
            r = R[i]
            c = cand[i]
            if c > r:
                is_break = True
                break
        if is_break is True:
            continue
        if sum(cand) % K == 0:
            print(*cand)
solve()
```

## まとめ

- ABC367に参加して、コンテスト中に3問だけ解くことができました
  - 私はC問題は絶対解きたい、D問題は簡単なら解きたい。というスタンスで参加しているので最低目標は達成できました
  - 一方でA問題を一回間違えてしまったのが非常に悔しい結果となりました
- D問題を解けないのがリアルな実力という感じはありますが、後日解説をみながら解き直そうと思います

私のような「リソースを競プロにあまり使えない人」こそ、「解けなかった一問だけ解説を見ながら解く」、「もっといい解法がないか解説を見てみる」ということが重要だと感じています。

今後も「楽しいからやる」というスタンスは保ちつつ、自分のペースで下がったり上がったりを繰り返しながら強くなっていければいいなと思います。

以上、bun913でした。ありがとうございました。
