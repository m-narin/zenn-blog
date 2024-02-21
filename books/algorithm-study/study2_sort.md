---
title: "ソートアルゴリズム"
---

# はじめに
本章ではソートアルゴリズムについて触れていきます。日付順に並べるといった処理はよく見かけますが、裏側は意外と奥が深いものです。
早速問題を見ていきましょう。

# 問題
次の数列を昇順に並べ替えてください。少なくとも2通り以上のアルゴリズムを考えてください。

```sh:input
5, 3, 2, 4, 1, 6, 9, 8, 7, 10
```


- テストケースは全て満たすこと
- 組み込みメソッドは使わないこと。つまり、分岐、繰り返し処理を自分で書くこと。
  - Array.findなどは使わない
- 解法を調べるためにchatgpt、インターネットは使用しないこと
  - 文法を調べることのみ可
- 計算量は問いません

## テストケース

```sh:input1
5, 3, 2, 4, 1, 6, 9, 8, 7, 10
```

```sh:output1
[1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
```

```sh:input2
482, 395, 749, 319, 584, 738, 293, 784, 492, 378
```

```sh:output2
[293, 319, 378, 395, 482, 492, 584, 738, 749, 784]
```

## コマンドラインからの入力を受け取る方法
```py:python
array_elements = list(map(int, input().split(',')))
```

## 解答
:::details クリックすると解答例が見られます

バブルソート
```py:python
array_elements = list(map(int, input().split(',')))

def bubble_sort(numbers):
    for i in range(len(numbers)):
        for j in range(len(numbers) - 1 - i):
            if numbers[j] > numbers[j + 1]:
                numbers[j], numbers[j + 1] = numbers[j + 1], numbers[j]
    return numbers

sorted_numbers = bubble_sort(array_elements)
print(sorted_numbers)
```
バブルソートは、隣接する要素を比較し、必要に応じて交換しながら大きな値を配列の一方の端へ「浮かび上がらせる」アルゴリズムです。
内側のループが一通り回ると一番右の数字が最大値であることが保証されます。
同様に残りの要素に関して、最大の要素が最後尾に固定されていくことを繰り返しながら並び替えていきます。

![バブルソート](https://storage.googleapis.com/zenn-user-upload/dcba9d77f6a8-20240221.png)

選択ソート
```py:python
numbers = list(map(int, input().split(',')))

def selection_sort(numbers):
    for i in range(len(numbers)):
        min_index = i
        for j in range(i+1, len(numbers)):
            if numbers[j] < numbers[min_index]:
                min_index = j
        numbers[i], numbers[min_index] = numbers[min_index], numbers[i]
    return numbers

sorted_numbers = selection_sort(numbers)

print(sorted_numbers)
```
未ソート部分から最小値（または最大値）を選び、それを未ソート部の最初の位置と交換していくアルゴリズムです。

![選択ソート](https://storage.googleapis.com/zenn-user-upload/55a470c75ad4-20240221.png)

:::

## 解説
解答例には2通りのソートアルゴリズムを載せましたが、他にも色々な方法があります。
挿入ソート、マージソート、クイックソートetc...

バブルソートと選択ソートは、基本的にO($n^2$)となるため計算量は大きいです。元データが巨大なものになってくるとパフォーマンスが非常に悪くなります。
しかし、マージソートは最悪でもO(n logn)で済むので計算量を抑えることができます。

それぞれのアルゴリズムの計算量は、元々の数列の状態に応じて変わってくる性質があります。
そのため、実際には条件に応じて様々なソートアルゴリズムを組み合わせたものが利用されています。

例えば、マージソートを挿入ソートを組み合わせた[ティムソート](https://ja.wikipedia.org/wiki/%E3%83%86%E3%82%A3%E3%83%A0%E3%82%BD%E3%83%BC%E3%83%88)はJava, Android, Swift, Rustで採用されているみたいです。

# おわりに
本章では、ソートアルゴリズムをテーマに学びました。

エンジニアリングをしていると、頻繁にソートする場面があります。
日付順に並べ替えたり、五十音順に並べ替えるといった操作をよくしますね。
その背景には、何らかのソートアルゴリズムが動いているはずです。

無味乾燥に.sortメソッドを使うのではなく、背景を想像しながら利用できるとまた楽しさが増すことでしょう。

# 参考文献

- http://www.rsch.tuis.ac.jp/~ohmi/software-basic/sort.html
- https://qiita.com/drken/items/44c60118ab3703f7727f
- https://ja.wikipedia.org/wiki/%E3%83%86%E3%82%A3%E3%83%A0%E3%82%BD%E3%83%BC%E3%83%88
