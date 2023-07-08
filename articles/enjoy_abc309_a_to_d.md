---
title: "デンソークリエイトプログラミングコンテスト2023（AtCoder Beginner Contest 309）に参加したので冒頭4問を振り返ります"
emoji: "💻"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["AtCoder","競技プログラミング", "Python"]
published: true
---

こんにちは。

ダイの大冒険という漫画のガチ勢の[bun913](https://twitter.com/bun76235104)といいます。

皆さんはAtCoderという競技プログラミングを楽しめるサービスをご存知でしょうか？

https://atcoder.jp/?lang=ja

私は以前よりちょくちょく趣味でコンテストに参加しているのですが、先日の2023年7月8日（土）に開催された[デンソークリエイトプログラミングコンテスト2023（AtCoder Beginner Contest 309） - AtCoder](https://atcoder.jp/contests/abc309)にも参加していました！

私は現在C問題までをほぼ確実に解ききるマンを目指しているのですが、今回はA〜Dの4問を回答することができました！（嬉しい）

## 解いた問題と正解に至るまで

### [A - Nine](https://atcoder.jp/contests/abc309/tasks/abc309_a)

`A < B` であることが保証されている整数A,B（1以上9以下）が与えられます。

Aが書かれたマスとBが書かれたマスが左右に隣接しているか判定する。という問題です。

正直これは、特に良い方法が思いつきませんでした。

ただBがAより大きいことはわかっているので、AがBの右隣にあるかという判定をするということで、以下のようなコードで正解となりました。

```python
A, B = map(int, input().split())

if B - A == 1:
    # Bが右になりうるにはAが↓のどれかじゃないといけないので
    cand = set([1, 2, 4, 5, 7, 8])
    if A in cand:
        print("Yes")
        exit()
print("No")

```

### [B - Rotate](https://atcoder.jp/contests/abc309/tasks/abc309_b)

問題を見て、言いたいことはわかったのですが、どうしてもスパッと答えを出す方法がわからず苦悩しました。

実はこの問題は飛ばしてC、D問題へと進み、B問題を正解で提出したのはコンテスト終了の20秒前でした。（とてもドキドキしました）

以下みたいなことを悩んで悩んで、結局if文を書きまくる実装となりました。

- え・・・なんか良い感じに余りとかとれば解けるのかな？
- 一番上の行の時はいけるところまで右にずらす
- 一番右の列の時はいけるところまで下にすらず
- みたいなことしかわからないぜ！

```python
from copy import deepcopy

N = int(input())
A = [list(input()) for _ in range(N)]
B = deepcopy(A)

for i in range(N):
    for j in range(N):
        # 一番外側の辺にならないますはスキップ
        if i == 0 or j == 0 or i == N-1 or j == N-1:
            # 一番上の行は右にずらす
            if i == 0 and j != N-1:
                B[i][j+1] = A[i][j]
            # 一番右の列は下にずらす
            if j == N-1 and i != N-1:
                B[i+1][j] = A[i][j]
            # 一番下の行は左にずらす
            if i == N-1 and j != 0:
                B[i][j-1] = A[i][j]
            # 一番左の列は上にずらす
            if j == 0 and i != 0:
                B[i-1][j] = A[i][j]
for b in B:
    print(''.join(b))
```

あとで解説をみたら、似たようなことをやっていた様な気がするので、最初から愚直にやればよかったなと思いました。

https://atcoder.jp/contests/abc309/editorial/6744

B問題とかで解き方わからないと慌てるんですよねぇ。

### [C - Medicine](https://atcoder.jp/contests/abc309/tasks/abc309_c)

N種類の薬を渡されて、それぞれの薬には飲むべき数と飲み続けるべき日数が定義されているという問題となります。

最初は戸惑ったのですが、以下の様に思考していくことで解きほぐれていきました。

- うーん？aもbも10**9ととても大きいので、こっちでループするのはできないな
- ていうか最初は全部の薬を飲まないといけないんだよね
- つまり bの合計値が最初に飲むべき薬の錠数か
- で、決まった日が立てばどんどん飲むべき薬を減らして良いんだよね
- てことは (飲むべき日数,飲むべき錠数)というタプルでデータを配列に入れていき、ソートしてどんどん薬を減らしていけばよいのでは？
- で、飲むべき薬がK錠以下になったらその日が答えになるんだな

という思考で実装したコードが以下の通りです。

```python
N, K = map(int, input().split())
drugs = []
days = []
day_drugs = []
for _ in range(N):
    day, drug = map(int, input().split())
    days.append(day)
    drugs.append(drug)
    day_drugs.append((day, drug))

sum_drugs = sum(drugs)
# すでにK錠以下の場合
if sum_drugs <= K:
    print(1)
    exit()

day_drugs.sort()
ans = 0
for day, drug in day_drugs:
    sum_drugs -= drug
    ans = day + 1
    if sum_drugs <= K:
        break
print(ans)
```

最初はaとbを逆に読み違えるなどでミスをしてしまったので、途中から変数名を自分にとってわかりやすく変えました。

変数名は大事だなぁ（なおD問題）

### [D - Add One Edge](https://atcoder.jp/contests/abc309/tasks/abc309_d)

無向グラフの問題とわかって、しかも出力例1がとてもわかりやすい図があったので助かりました。

- 単純に頂点1がある側の隣接成分に含まれている頂点で、1から最も遠い頂点と
- 頂点N1+N2がある側の隣接成分に含まれている頂点で、N1+N2の頂点から最も遠い頂点
- を紐づける様にすればよいのでは？

という考えに至りました。

ということで、愚直だけど幅優先探索を3回すれば素直に答えが求められそうということで実装しました。

```python
from collections import deque

# まずは条件通りグラフを作る
N1, N2, M = map(int, input().split())
G = [[] for _ in range(N1 + N2 + 1)]
for _ in range(M):
    u, v = map(int, input().split())
    G[u].append(v)
    G[v].append(u)

# 頂点1と隣接していて、頂点1から最も遠い頂点を求める
q = deque([1])
dis1 = [-1] * (N1 + N2 + 1)
dis1[1] = 0
while q:
    cur = q.popleft()
    for nex in G[cur]:
        # 訪問済みならスキップ
        if dis1[nex] != -1:
            continue
        dis1[nex] = dis1[cur] + 1
        q.append(nex)
most_far_index1 = dis1.index(max(dis1))

# 頂点N1+N2と隣接していて、頂点N1+N2から最も遠い頂点を求める
q = deque([N1 + N2])
dis2 = [-1] * (N1 + N2 + 1)
dis2[N1 + N2] = 0
while q:
    cur = q.popleft()
    for nex in G[cur]:
        # 訪問済みならスキップ
        if dis2[nex] != -1:
            continue
        dis2[nex] = dis2[cur] + 1
        q.append(nex)
most_far_index2 = dis2.index(max(dis2))

# 求めた頂点同士を繋げる
G[most_far_index1].append(most_far_index2)
G[most_far_index2].append(most_far_index1)

# であとは頂点1から頂点N1+N2までの最短経路を求める
q = deque([1])
dis3 = [-1] * (N1 + N2 + 1)
dis3[1] = 0
while q:
    cur = q.popleft()
    for nex in G[cur]:
        # 訪問済みならスキップ
        if dis3[nex] != -1:
            continue
        dis3[nex] = dis3[cur] + 1
        q.append(nex)
print(dis3[-1])
```

競技プログラミングだから急いでるんですけど、変数名とか実装は後から見ると厳しいですね。目に毒です。

なお、社用ブログとなるのですが、この数日前におふざけで幅優先探索を実装していたので、とてもタイムリーな気持ちで問題を解くことができました。

https://dev.classmethod.jp/articles/bfs_for_meeting_president_20ann/

## 最後に

D問題までを公式（Unratedになったコンテストを除く）でとれたのは初めてだったので、とても嬉しかったです。

終わってみれば、D問題が典型的なグラフ問題というのもあり、典型的なアルゴリズムを理解して実装を重ねていくことの重要性が身に染みた気がします。

引き続きゆるく続けていきたいと思います！

最後まで読んでいただきありがとうございました！
