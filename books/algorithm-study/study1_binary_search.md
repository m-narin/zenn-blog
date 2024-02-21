---
title: "二分探索"
---

# はじめに
初回は探索アルゴリズムの基礎について触れていきます。早速問題を見ていきましょう。

# 問題
一つの数字とソートされた数列が以下のように与えられます。

```sh:input
7
1,2,3,4,5,6,8,9,10
```

1行目の数字が2行目の数列に含まれているならYes と、含まれていないなら No と出力するプログラムを書いてください。なお、以下の条件に従ってください。

- テストケースは全て満たすこと
- 組み込みメソッドは使わないこと。つまり、分岐、繰り返し処理を自分で書くこと。
  - Array.findなどは使わない
- 解法を調べるためにchatgpt、インターネットは使用しないこと
  - 文法を調べることのみ可
- 計算量は小さい方が望ましいです

## テストケース

```sh:input1
3
1,2,3,4,5
```

```sh:output1
Yes
```

```sh:input2
7
0,2,3,5,6,8,9,10
```

```sh:output2
No
```

## コマンドラインからの入力を受け取る方法
```py:python
number_to_find = int(input())
array_elements = list(map(int, input().split(',')))
```

## 解答
:::details クリックすると解答例が見られます

全探索(線形探索)
```py:python
number_to_find = int(input())
array_elements = list(map(int, input().split(',')))

def linear_search(array, number):
    for element in array:
        if element == number:
            return "Yes"
    return "No"
    

print(linear_search(array_elements, number_to_find))
```
配列の頭から順に探索していきます。

二分探索
```py:python
number_to_find = int(input())
array_elements = list(map(int, input().split(',')))

def binary_search(sorted_array, number):
    left, right = 0, len(sorted_array) - 1
    
    while left <= right:
        mid = (left + right) // 2

        if sorted_array[mid] == number:
            return "Yes"
        elif sorted_array[mid] < number:
            left = mid + 1
        else:
            right = mid - 1
                    
    return "No"
    
print(binary_search(array_elements, number_to_find))
```
「ソートされた数列」をヒントに、半分ずつ探索対象を絞っていくことで計算量を小さくできます。
このような探索手法のことを二分探索と呼びます。

:::

## 解説
この問題からは、二分探索で計算量を抑えられるという学びがあります。

![二分探索](https://storage.googleapis.com/zenn-user-upload/330e1d476f9e-20240219.png)

そもそも計算量とは何でしょうか？
計算量は、入力されるデータ量によって、実行時間がどのような割合で変化するのかを表す指標です。
実質計算時間のようなものと理解して差し支えありません。
これは、O記法(オーダー記法)で表します。

![O記法](https://storage.googleapis.com/zenn-user-upload/1f0b5e3b6e5c-20240219.png)

if文のような比較処理は、コンピュータが計算する一つの単位とみなすことができます。

配列の頭からしらみ潰しで全ての要素と比較していくような方法を全探索を言います。
これは、配列の要素数が増えるほど比例して計算時間が増えていく(=比較処理が増えていく)ため、オーダー記法でO(n)と書きます。
数学で言うと、1次関数のことですね。

一方で二分探索の場合、配列の要素が増えても、半分ずつ探索対象が減っていくため比較回数を抑えることができます。

![二分探索の計算量](https://storage.googleapis.com/zenn-user-upload/b6eb5dbdb710-20240219.png)

計算量の数字を見ると、どうやら規則性がありそうです。

式で表すと、データ量=2^(計算量)が成り立つため、計算量=log(データ量)となります。
このことから、二分探索はO(log)と表現されます。
グラフを見ると、データ量が大きくなるほど、計算量の増加は緩やかなのでより高いパフォーマンスを出すことができるわけですね。

反対に、二重ループの場合のような処理の場合、計算量=データ量^2になるのでO(n^2)と表現します。
これは、データ量の増加に従って爆発的に計算量も増加する=指数関数的増加と表現することがあります。

二分探索は基礎的なアルゴリズムですが、DBのインデックスはこれの代表的な応用例です。
エンジニアリングをしていると、検索パフォーマンスを上げるためにDBのカラムにIndexを貼りますよね。

では、Indexを貼るをなぜパフォーマンスが良くなるのでしょうか？
Indexを貼ると、裏側にB-treeというものが作られます。これはデータを大小関係に沿って木構造の形に並べたものです。

「データが順番に並んでいる」ということは、以下図のように二分探索の要領で該当のデータに辿り着くことができるんですね。
なので、しらみ潰しに頭から検索するよりも計算量が抑えられ、早く検索できるようになるというわけです✨

![B-tree](https://storage.googleapis.com/zenn-user-upload/7c470d487d5c-20240219.png)

# おわりに
勉強会初回の内容として、二分探索をテーマに学びました。

「データが順番に並んでいる」と、半分ずつ探索を絞っていけるため比較回数を抑えることができ、計算量を小さくできます。
このようにアルゴリズムの工夫によって、パフォーマンスを向上することができます。

基礎的なアルゴリズムですが、世の中で様々な応用例があり、代表的なものとしてDBのインデックスを挙げました。
この記事を読んだ方は、DBのインデックスを貼るとなぜ検索が早くなるのか？を理解できているはずです。
一つ教養を得ましたね。

このような学びを重ねることで、普段のエンジニアリングの背景が分かってきます。
そうすると、納得感が増して、エンジニアリングが無味乾燥なものから鮮やかな景色が見えてくる感覚があることでしょう。


# 参考文献

- https://breezegroup.co.jp/202006/algorithm-search/
- https://www.techscore.com/blog/2016/08/08/%E9%96%8B%E7%99%BA%E6%96%B0%E5%8D%92%E3%81%AB%E6%8D%A7%E3%81%90%E3%80%81%E5%9F%BA%E6%9C%AC%E3%81%AE%E3%82%A2%E3%83%AB%E3%82%B4%E3%83%AA%E3%82%BA%E3%83%A0%E3%81%A8%E8%A8%88%E7%AE%97%E9%87%8F/
- https://gihyo.jp/admin/serial/01/rdbms-autumn-sky/0004
