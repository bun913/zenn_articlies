---
title: "freee プログラミングコンテスト2023（ABC310）に参加したので冒頭3問を振り返ります"
emoji: "💻"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["AtCoder","競技プログラミング", "Python"]
published: true
---

こんにちは。

ダイの大冒険という漫画のガチ勢の[bun913](https://twitter.com/bun76235104)といいます。

皆さんはAtCoderという競技プログラミングを楽しめるサービスをご存知でしょうか？

https://atcoder.jp/?lang=ja

私は以前よりちょくちょく趣味でコンテストに参加しているのですが、先日の2023年7月15日（土）に開催された[freee プログラミングコンテスト2023（AtCoder Beginner Contest 310） - AtCoder](https://atcoder.jp/contests/abc310)にも参加していました！

私は現在C問題までをほぼ確実に解ききるマンを目指しており、今回はC問題までの3問を解答することがきました。

## 解いた問題と正解に至るまで

### [A - Order Something Else](https://atcoder.jp/contests/abc310/tasks/abc310_a)

- AtCoderのドリンクが定価であるP円で飲める
- ただしQ円で飲めるクーポンがあるが、料理の中から一品と一緒に買う必要がある

という条件なので、以下のように2つのパターンから最小値を出力する形式で回答しました。

```python
N, P, Q = map(int, input().split())
D = list(map(int, input().split()))
min_ryori = min(D)
cand1 = P
cand2 = Q + min_ryori

print(min(cand1, cand2))
```

### [B - Strictly Superior](https://atcoder.jp/contests/abc310/tasks/abc310_b)

今回よく条件を読まず、3回も誤った回答を出してしまった問題です。今回は特にD問題を解答できている人が少なかったので、こういうところでミスをしてしまうのが残念なところです。

問題自体は長く感じますが、以下の条件をしっかり満たしているのか判定してやれば良い問題なので、素直に実装すれば良い問題ですね。

- P[i] >= P[j]である
    - 以上であることに注意
- j番目の製品はi番目の製品の機能を全てもつ
- P[i] > P[j] `or` j番目の製品がi番目の製品にない機能を持つ

```python
N, M = map(int, input().split())
P = []
C = []
functions = []

for _ in range(N):
    p, c, *f = map(int, input().split())
    P.append(p)
    C.append(c)
    functions.append(set(f))

for i in range(N):
    for j in range(N):
        if i == j:
            continue
        # 1つ目の条件
        if P[i] >= P[j]:
            # 2つ目の条件（機能を全て持っているか）
            j_funcs = functions[j]
            i_funcs = functions[i]
            intersect = j_funcs.intersection(i_funcs)
            if len(intersect) == len(i_funcs):
                # 3つ目の条件（jがiにない機能を持っているか）
                if P[i] > P[j] or len(j_funcs) > len(intersect):
                    print("Yes")
                    exit()
print("No")
```

機能の比較には集合を扱う`set`を利用しました。

積集合を取ったり和集合を取ったりと、非常に便利です。

### [C - Reversible](https://atcoder.jp/contests/abc310/tasks/abc310_c)

- N個の文字列が与えられる
- N個の文字列の種類数を答える
    - ただし、特定の文字列を反転させたものは同じ種類としてカウントする

というように理解しながら、素直に実装したのが以下のコードとなります。

```python
N = int(input())
S = set()
for _ in range(N):
    s = input()
    l = list(s)
    rl = list(reversed(l))
    r = "".join(rl)
    if s not in S and r not in S:
        S.add(s)
print(len(S))
```

**Pythonって文字列操作（反転）って遅いんだっけ**と不安になり、何となく文字列をlistにしていますが、今回のケースだったらあまり意味はなさそうだなと思いました。

原案者の方の実装はとってもシンプルですね。

https://atcoder.jp/contests/abc310/editorial/6792

## 最後に

C問題までを解くことができましたが、D問題がさっぱり分かりませんでした。（解説を聞いて、考え方と考察方法を理解していきます）

次回以降も引き続きゆるく続けていきたいと思います！

最後まで読んでいただきありがとうございました！
