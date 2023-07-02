---
title: "AtCoder Beginner Contest 308に参加したので冒頭3問を振り返ります"
emoji: "💻"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["AtCoder","競技プログラミング", "Python"]
published: true
---

こんにちは。

ダイの大冒険という漫画のガチ勢の[bun913](https://twitter.com/bun76235104)といいます。

皆さんはAtCoderという競技プログラミングを楽しめるサービスをご存知でしょうか？

https://atcoder.jp/?lang=ja

私は以前よりちょくちょく趣味でコンテストに参加しているのですが、先日の2023年7月1日（土）に開催された[CodeQUEEN 2023 予選 (AtCoder Beginner Contest 308)](https://atcoder.jp/contests/abc308)にも参加していました！

今回はコンテスト中にA問題、B問題しかコンテスト中に解くことができなかったのですが、コンテスト後にC問題へのさまざまなアプローチに感心したので、振り返ってみようと思います。

## 解いた問題と正解に至るまで

### [A - New Scheme](https://atcoder.jp/contests/abc308/tasks/abc308_a)

数列Sが以下の条件を満たすか考えるというお話ですね。

- S1 <= S2 <= S3 ... < S8で
- 数列Sの要素すべてが100以上、675以下であること
- 数列Sの要素すべてが25の倍数であること

一つ目の条件を考えるのが手間で、`あらかじめソートした配列を用意して、Sの要素と順番が変わっていればNG`と考えました。

ということで、実装は以下のようにして提出しました。

```python
S = list(map(int, input().split()))
# あらかじめソートした配列
so = sorted(S)

for i, s in enumerate(S):
    # 1個目の条件
    if s != so[i]:
        print("No")
        exit()
    # 2個目の条件
    if s < 100 or s > 676:
        print("No")
        exit()
    # 3個目の条件
    if s % 25 != 0:
        print("No")
        exit()
print("Yes")
```

### [B - Default Price](https://atcoder.jp/contests/abc308/tasks/abc308_b)

スッと問題を見た時に瞬間「ん？」となったけど、以下みたいに単純に考えれば良さそうですね。

- Cの配列には高橋くんが食べた皿の色が書いてある
- 皿の色で料金が決まっているので、お皿の色ごとの料金が分かれば良さそうですね
- D1の色がP1の料金に、D2の色がP2の料金に・・と対応している
- Dにない色は全てP0が料金になっている

ということで、以下のように実装して見ました。
なお、本番は急いでいたので無駄実装が入っています。

```python
from collections import defaultdict

setrecursionlimit(10**7)
N, M = map(int, input().split())
# i皿目がi番目に食べた色
C = list(map(str, input().split()))
# Diの皿の料理は1皿Piとなる
D = list(map(str, input().split()))
P = list(map(int, input().split()))
# まずCに出てくる色とDに出てくる色を見て価格の対応表を作る
pc = defaultdict(int) #defaultdictの機能を使えていないので普通にdictでよかったです
for i, d in enumerate(D):
    pc[d] = P[i+1]
ans = 0
for c in C:
    # Dに色がなかった場合はP[0]を使う
    if c not in pc:
        ans += P[0]
        continue
    ans += pc[c]
print(ans)
```

### [C - Standings](https://atcoder.jp/contests/abc308/tasks/abc308_c)

今回コンテスト中に解けずに、泣きを見た問題となります。

- N人の人がいて、各自Ai回コイントスに成功、Bi回コイントスに失敗
- 各自のコイントスの成功率が高い順番に並べる
- またコイントスの成功率が同じ場合には、人の番号が若い順番に並べる

という問題ですね。問題に記載のあるとおり、成功率は `A[i] / A[i] + B[i]`で求めることができますが、何も考えずにこのまま計算すると小数によ誤差でうまく動作しないことは明らかでした。

そこでなんとか整数で計算しないとなーと思い、以下誤回答をしてしまいました。

### 誤回答1 最小公倍数で整数問題に帰結してまえ解法

- 発想は以下のような感じでした
    - 結局今回は各人の試行回数（A+B）が異なるから率を考える時に複雑なのでは？
    - てことはN人それぞれの試行回数で、最小公倍数を考えておけば、以下のように計算すれば小数を取り扱わなくていいのでは！？
        - 各人の成功率のスコア: `（最小公倍数 // A[i]+B[i]） * A[i]`

と考えて以下のように提出しました。

```python
from math import gcd
from functools import reduce


def lcm(a, b):
    return a * b // gcd(a, b)


N = int(input())
A = []
B = []
S = []
for _ in range(N):
    a, b = map(int, input().split())
    A.append(a)
    B.append(b)
    S.append(a+b)
lc = reduce(lcm, S)
 
scores = []
for i in range(N):
    # ここで成功率のスコアを算出している
    scores.append((lc//S[i]*A[i], i+1))
# scores[i]の2番目の要素が同じなら、1番目の要素が小さい順に並べる
sort_list = sorted(scores, reverse=True, key=lambda x: (x[0], -x[1]))
ans_list = []
for i, score in enumerate(sort_list):
    ans_list.append(score[1])
print(*ans_list, sep=' ')
```

しかし、A[i]とB[i]が大きい数のケースがあるため↑のように計算するとTLEとなってしまいます。

### 誤回答2 Fractionの有理数で扱ったれ解法

えぇい！小数の誤差があるなら、有理数でやったれ！と思って提出したコードが以下のとおりです。

```python
# -*- coding: utf-8 -*-
"""
これだけは使いたくなかったFractionを使ってみる
"""
from fractions import Fraction
 
N = int(input())
fractions = []
for _ in range(N):
    a, b = map(int, input().split())
    # 小数ではなく有理数で扱う
    success_rate = Fraction(a, a + b)
    fractions.append(success_rate)
 
people = list(range(1, N + 1))
people.sort(key=lambda i: (-fractions[i-1], i))
 
print(*people, sep=" ")
```

以下の記事にとてもわかりやすく書いてありますが、`Fraction`はとても計算速度が遅いようで同様にTLEになりました。

https://qiita.com/papi_tokei/items/37a4e31949ba8efb6897

### 誤回答3 えぇい！だったらDecimalで解いてくれるわ！回答（おしかった）

上記記事を見て、「じゃあDecimalならいけるかも？」と思って提出したコードが以下のとおりです。

```python
# -*- coding: utf-8 -*-
"""
これも使いたくなかったがDecimalで試してみる
"""
from decimal import Decimal
 
N = int(input())
decimals = []
for _ in range(N):
    a, b = map(int, input().split())
    success_rate = Decimal(a) / Decimal(a + b)  # Decimalを使用して成功率を計算
    decimals.append(success_rate)
 
people = list(range(1, N + 1))
people.sort(key=lambda i: (-decimals[i-1], i))
 
print(*people, sep=' ')
```

このコードは手癖でpypyで提出したのですが、Python3で提出しておけば普通に正解コードとなりました。（コンテスト後に知りました）

pypyでは浮動小数点が絡む計算では逆に遅くなることがあるそうです。（初めて知りました）

具体的にどのような処理が遅いのかもっと調べてみたいですね。

### コンテスト後に提出してみたOKだった解法

Pythonでは多倍長整数が使えるので、桁数をめっちゃ大きくした整数でやってみようと思って提出したコードが以下の通りです。

```python
# -*- coding: utf-8 -*-
"""
方向を変えてめっちゃでかい数を利用して整数演算をしてみる
"""
N = int(input())
success_rates = []
for _ in range(N):
    a, b = map(int, input().split())
    # めっちゃでかい数で整数演算を盛り込んでみる
    success_rate = a * (10**20) // (a + b)
    success_rates.append(success_rate)

people = list(range(1, N + 1))
people.sort(key=lambda i: (-success_rates[i-1], i))

print(*people, sep=' ')
```

このようにしてみることで、精度があがり正解となりました。

なお、公式の解説でさまざまな方法が書かれていて、ここをみるだけでとても面白いです。

https://atcoder.jp/contests/abc308/editorial/6722

## 最後に

現在のところ私は「C問題までをほぼ確実に解くマン」を目指しているのですが、C問題に気を取られた挙句、最後まで答えを出せず少し悔しかったです。

ただ、それ以上に以下のような点が学びになったので、参加してよかった！と思えたコンテストでした。

- Decimalはpypyだと逆に遅い
- Pythonだと多倍長整数が扱えるので、とっても大きい10の倍数をかけることで手軽に精度をあげることが可能
- 公式解法のソートがとっても素敵
    - cmp_to_keyでソートのキーを渡す方法も知らなかった
    - [解説 - CodeQUEEN 2023 予選 (AtCoder Beginner Contest 308)](https://atcoder.jp/contests/abc308/editorial/6722)

引き続きゆるく続けていきたいと思います！

最後まで読んでいただきありがとうございました！
