---
title: "チュートリアル"
---

# はじめに
まずは、形式に慣れるためのチュートリアルから入ります。

プログラミングの問題演習では手軽なオンライン実行環境の[Paiza.io](https://paiza.io/ja/projects/new)を利用します。

# Paiza.ioの使い方
1. [Paiza.io](https://paiza.io/ja/projects/new)にアクセス
2. 以下画像のように下記を入力する
  a. プログラミング言語の選択
  b. ソースコード記入
  c. 「入力」にテストケースのinputを記入
3. 実行して「出力」を見て、outputが合っているか確認する。

![Paiza.ioの使い方](https://storage.googleapis.com/zenn-user-upload/d2b59c27fc6d-20240219.png)

# 問題
一つの数字Aと1から始まる連続する数列が以下のように与えられます。

```sh:input
7
1,2,3,4,5,6,7,8,9,10,11,12,13,14,15
```

数列の順番に、Aまで、3の倍数だった場合は「Fizz」を、5の倍数だった場合は「Buzz」を、15の倍数だった場合は「Fizz Buzz」と出力、いずれも当てはまらなかったら数字をそのまま出力するプログラムを書いてください。なお、以下の条件に従ってください。

- テストケースは全て満たすこと
- 組み込みメソッドは使わないこと。つまり、分岐、繰り返し処理を自分で書くこと。
  - Array.findなどは使わない
- 解法を調べるためにchatgpt、インターネットは使用しないこと
  - 文法を調べることのみ可
- 計算量は問いません

## テストケース

```sh:input1
8
1,2,3,4,5,6,7,8,9,10
```

```sh:output1
1
2
Fizz
4
Buzz
Fizz
7
8
```

```sh:input2
15
1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17
```

```sh:output2
1
2
Fizz
4
Buzz
Fizz
7
8
Fizz
Buzz
11
Fizz
13
14
Fizz Buzz
```

## コマンドラインからの入力を受け取る方法
この本ではPythonをサポートしています。アルゴリズムの理解が本質なので、制御構文が書ければなんでも大丈夫です。
```py:python
last_number = int(input())
array_elements = list(map(int, input().split(',')))
```

上記をPaiza.ioのソースコードに記入し、その下に処理を追記します。
「入力」にテストケースのinputを記入し、実行することで「出力」がoutputに合っているかを確認します。

## 解答
:::details クリックすると解答例が見られます

```py:python
last_number = int(input())
array_elements = list(map(int, input().split(',')))

def fizz_buzz(last_number,numbers):
    for i in range(last_number):
        number = array_elements[i]
        if number % 15 == 0:
            print("Fizz Buzz")
        elif number % 3 == 0:
            print("Fizz")
        elif number % 5 == 0:
            print("Buzz")
        else:
            print(number)
            
fizz_buzz(last_number, array_elements)
```
古典的なFizzBuzz問題ですね。ループで指定の回数回していき、倍数を割り切れるかどうかの条件式で判定していきます。

:::

# おわりに
チュートリアルは以上となります。次の章から本格的にアルゴリズムの世界に入っていきます！
楽しみにしていてください！