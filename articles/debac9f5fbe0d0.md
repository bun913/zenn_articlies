---
title: "3次元の累積和を学べるコンテストでした【AtCoder Beginner Contest 366 振り返り（茶）】"
emoji: "🚌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["AtCoder", "Python"]
published: true
publication_name: "moneyforward"
---

こんにちは。ダイの大冒険ガチ勢の[bun913](https://x.com/bun76235104)と申します。

皆さんはAtCoderという競技プログラミングに気軽に参加できるサービスをご存知でしょうか？

https://atcoder.jp/

競プロと聞くと一見とっつきにくいですが、普段プログラミングができない方でも「**あ〜。この課題に必要なデータ構造は配列ではなく集合型、もしくは辞書型**」と浸れるので、とてもおすすめです。

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

## [A - Election 2](https://atcoder.jp/contests/abc366/tasks/abc366_a)

### 考えたこと and 正解になった実装

- 過半数ってどうやって出すんだっけ？
- あでも今回はNが奇数っていう条件があるから、単純に2で割って1を足せば良いよね
  - だって5人の場合 `5//2 + 1 = 3` だし、7人の場合 `7//2 + 1 = 4` だし
- ということで以下のようにコードを書いて提出しました

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
    N, T, A = map(int, input().split())
    kahan = N // 2 + 1
    if T >= kahan or A >= kahan:
        print("Yes")
        exit()
    print("No")


solve()

```

## [B - Vertical Writing](https://atcoder.jp/contests/abc366/tasks/abc366_b)

### 考えたこと and 正解になった実装

- 問題の意味がさっぱりわからない。何言ってるんだ。難しいよ。
  - 最近のB問題がわからないよ。さっぱりだよ。助けて
- とりあえず入力例と出力例を見てみよう
  - a,b,c,d,e,f の文字が入力前と出力後でどう変わるか見てみよう
  - あれ、これ縦書きに直してるっぽいな。長さが合わない時は `*` で埋めているみたいな感じか
- じゃあとりあえず出力するべき長さははっきりしているから、まずは `*` で埋めてあとは縦書きに埋めれるところを埋めていけばいいかな


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
    N = int(input())
    S = [input() for _ in range(N)]
    # sの各要素の長さの最大値
    M = max(len(s) for s in S)
    # Tを初期化する
    T = [["*"] * N for _ in range(M)]
    # T[i]を出力する
    for i in range(N):
        s = S[i]
        l = len(s)
        # 言われた通りにT[j][N-i-1]に順にs[j]を詰める
        for j in range(l):
            T[j][N-i-1] = s[j]
    for t in T:
        ts = "".join(t)
        # 末尾の*は消す
        ts = ts.rstrip("*")
        print(ts)


solve()

```

## [C - Balls and Bag Query](https://atcoder.jp/contests/abc366/tasks/abc366_c)

### 考えたこと and 不正解になった実装

- ボールを袋の中に入れていくと
  - 1のクエリの時はボールを1つ入れる
  - 2のクエリの時はそのボールを1つ取り出す
  - 3のクエリの時はボールの種類数を出力する
- え？これただ集合型でボールの種類を管理すればいいだけじゃん（間違い）
  - と短絡的に考えて最初は以下のコードを提出してしまいました

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
    Q = int(input())
    S = set()
    for i in range(Q):
        query = list(input().split())
        if query[0] == "1":
            S.add(query[1])
        elif query[0] == "2":
            # NOTE: コンテスト後追記。この実装だと2個ボールが入ってても、全部取り出しちゃう実装になっている
            if query[1] in S:
                S.remove(query[1])
        else:
            print(len(S))


solve()
```

### 正解になった実装

- 「あっ！ボールって1個しかないんじゃなくて同じ数が書いてあるボールが複数あるじゃん！ **今の実装は2のクエリの時に1個だろうが2個だろうが、関係なく袋から全部除去しちゃってるじゃん！** 」
- じゃあ「今何個のボールを持っているかっていうのを管理しないと」
- で2のクエリが来た時にボールが1個しかない場合は、「そのボールが袋に入っている」という状態ごと削除するようにすればいいか
- そうすれば辞書型でボールの数を管理すればいいね
  - かつ `defaultdict` を使えば、「そのボールがもう袋に入っているか」ということを気にせずに `+=1` すればいいだけだ
  - 参考: [AtCoder向けpythonチートシート（その２）｜Pythonで競プロを始める人向け｜Aru's テクログ（Aruaru0）](https://tech.aru-zakki.com/atcoder-python-cheatsheet2/)

```python
# -*- coding: utf-8 -*-
"""
解く前のメモ用
"""

from sys import setrecursionlimit
from collections import defaultdict

setrecursionlimit(10**8)


def solve():
    act()


def act():
    Q = int(input())
    # 1のボール: 2個, 2のボール: 3個, 3のボール: 1個, 4のボール: 1個 みたいに管理する
    memo = defaultdict(int)
    for i in range(Q):
        query = list(map(int, input().split()))
        if query[0] == 1:
            # defaultdictを使っているので、存在しないキーにアクセスしてもエラーにならない
            memo[query[1]] += 1
            continue
        if query[0] == 2:
            if memo[query[1]] == 1:
                # 0のまま袋に残っているふうにしたくない
                # よって管理しているという事実ごと削除
                del memo[query[1]]
                continue
            if memo[query[1]] > 1:
                memo[query[1]] -= 1
        if query[0] == 3:
            # 種類数を出力
            print(len(memo))
            continue


solve()
```

## [D - Cuboid Sum Query](https://atcoder.jp/contests/abc366/tasks/abc366_d)

### 考えたこと 提出前の整理

- 3次元で数列を管理するみたいな感じですか？
- まずこの数式を冷静に整理しようぜ
    - 私はあまり数学が得意ではないので、よくわからない時は「まず全探索で素直に実装したらどうなるか」と考えることが多いです

$$
\sum_{x=Lx_i}^{Rx_i} \sum_{y=Ly_i}^{Ry_i} \sum_{z=Lz_i}^{Rz_i} A_{x,y,z}
$$

- これを計算量とか気にせずに、全探索で実装するとこうなるってことだよね

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
    N = int(input())
    A = []
    for _ in range(N):
        a = [[] for _ in range(N)]
        for j in range(N):
            a[j] = list(map(int, input().split()))
        A.append(a)
    # まずは素直に言われた通りに全探索する
    Q = int(input())
    for _ in range(Q):
        lx, rx, ly, ry, lz, rz = map(int, input().split())
        ans = 0
        for x in range(lx, rx+1):
            for y in range(ly, ry+1):
                for z in range(lz, rz+1):
                    ans += A[x-1][y-1][z-1]
        print(ans)


solve()
```

### 考えたこと 正解にいたるまでの実装

- ただ、当然これだと `O(N**3 * Q)` となってしまい、 `O(2*10**11)` とかになってしまうから間に合わないよね
  - `O(10**8)` くらいじゃないと当然難しいはず
- ん？なんかこれって結局「**特定の範囲の和**」を求めるっていう問題なのでは？
  - ようするに特定の範囲の直方体を考える感じ？
- であれば、「累積和とかいもす法」などの考え方が使えるのでは？
- 2次元の累積和であれば、以前似たような典型問題を解いたことがあるからそれを参考にしてみるか
- というようなアプローチで考え始めて、実装しました

```python
# -*- coding: utf-8 -*-
"""
解く前のメモ用
多分いもす法や3次元の累積和を使えば特定の範囲の表の特定の範囲の和を求めることができる
https://qiita.com/kooji/items/4aff0c4b4898f7e64e08
"""

from sys import setrecursionlimit

setrecursionlimit(10**8)


def solve():
    act()


def act():
    N = int(input())
    # 1-indexedで3次元配列を初期化
    # 二次元の累積和と同じで、0行目、0列目、0階層目は0で初期化した方が都合がよい
    # A = [[[0 for _ in range(N+1)] for _ in range(N+1)] for _ in range(N+1)]
    A = [[[0] * (N+1) for _ in range(N+1)] for _ in range(N+1)]
    # Aを取得する
    for i in range(1, N+1):
        for j in range(1, N+1):
            row = list(map(int, input().split()))
            for k in range(1, N+1):
                A[i][j][k] = row[k-1]

    # 3次元の累積和の箱を作る
    sums = [[[0 for _ in range(N+1)] for _ in range(N+1)] for _ in range(N+1)]
    for i in range(1, N+1):
        for j in range(1, N+1):
            for k in range(1, N+1):
                # 現在の要素を追加
                s = A[i][j][k]
                # 1次元の累積和
                s += sums[i-1][j][k]
                s += sums[i][j-1][k]
                s += sums[i][j][k-1]
                # 2次元の累積和（平面）を除去する
                s -= sums[i-1][j-1][k]
                s -= sums[i-1][j][k-1]
                s -= sums[i][j-1][k-1]
                # 頂点の重複を追加する
                s += sums[i-1][j-1][k-1]
                sums[i][j][k] = s
    # クエリ処理
    Q = int(input())
    for _ in range(Q):
        lx, rx, ly, ry, lz, rz = map(int, input().split())
        # 直方体の和
        total = sums[rx][ry][rz]
        # 直方体をまずは引く （2次元と同じでわかりやすい）
        # 包除原理と同じように考えるとわかりやすい
        total -= (sums[lx-1][ry][rz] + sums[rx][ly-1][rz] + sums[rx][ry][lz-1])
        # 二重に引かれた分を足す
        total += (sums[lx-1][ly-1][rz] + sums[lx-1]
                  [ry][lz-1] + sums[rx][ly-1][lz-1])
        # 三重に足された分を引く
        total -= sums[lx-1][ly-1][lz-1]
        print(total)


solve()
```

## まとめ

- ABC366に参加して、コンテスト中に4問だけ解くことができました
  - ただ正直D問題を解けたのは、まぐれだと思っています
  - ルールに違反するようなことは一切していませんが、AIに「3次元の累積和の考え方」を教えてもらいました
    - ※ この問題を解くために特定のデータを与えることはしていません
    - 「2次元の累積和はこんな感じのコードで解けるよね（コードを書いて例示）。3次元の場合はどういうふうに考えればいい？」というような使い方をしました。
- C問題を一回間違ってしまったことは非常に後悔しています

私のような「リソースを競プロにあまり使えない人」こそ、「解けなかった一問だけ解説を見ながら解く」、「もっといい解法がないか解説を見てみる」ということが重要だと感じています。

今後も「楽しいからやる」というスタンスは保ちつつ、自分のペースで下がったり上がったりを繰り返しながら強くなっていければいいなと思います。

以上、bun913でした。ありがとうございました。
