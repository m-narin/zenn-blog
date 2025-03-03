---
title: "SQLのIN句に大量データをコピペするときの小技"
emoji: "💽"
type: "tech"
topics:
  - SQL
published: true
---

## はじめに

データを調査する際、例えば SELECT の IN 句に適するフォーマットにするための小技について書きます。

## 目的の format

以下のように`("1","2","3",,,) `の形に持っていきます。
汎用性のため、文字列を扱います。

```sql
SELECT id, name
FROM users
WHERE IN ("1","2","3",,,)
```

## 元データ

一つの想定として CSV の列に調べたい ID 一覧が載っているものとします。

```csv
id, name,
1, "田中",
2, "佐藤",
3, "鈴木"
,,,
```

これがせいぜい数個であれば、手書きでも簡単に書けそうですが、2 桁、3 桁の数になってくると面倒なので一括で直していきたいですよね。
VSCode を使っている前提で簡単にそのようなデータを作れる小技があります。

## 方法

1. 適当なテキストファイルを作成する(Ctrl + n)
2. CSV をスプシ等にアップロードし、該当 id 列をコピペしてテキストファイルに貼り付ける

そうすると、以下のようなファイルができています。

![](https://storage.googleapis.com/zenn-user-upload/60540c534e8b-20250303.png)

3. 一番上の要素の右の部分にカーソルを持っていき、Opt + Shift を押して一番下の要素の右側をクリック

これにより、矩形選択されている状態になります。

![](https://storage.googleapis.com/zenn-user-upload/5ec1fe5b134b-20250303.png)

4. ダブルクオテーションを入力

これにより全ての行の左側にダブルクオテーションが入力されました。

![](https://storage.googleapis.com/zenn-user-upload/2d2aa9fc5709-20250303.png)

5. 一番上の要素の左側を選択したまま、2 行目の左側にカーソルを持っていきコピーする。

これにより改行コードを選択できます。

![](https://storage.googleapis.com/zenn-user-upload/4cfcca8c6617-20250303.png)

6. Crtl + f で検索窓を開き貼り付け。一括置換機能でクオテーションとカンマ`",`に置換する

実行前はこんな感じの画面になり、
![](https://storage.googleapis.com/zenn-user-upload/5b19196dddc5-20250303.png)

実行後は改行がなくなり、ダブルクオテーションとカンマを含む一行のデータになります。
![](https://storage.googleapis.com/zenn-user-upload/75fb4a649a27-20250303.png)

7. 末尾にダブルクオテーションを付与して完成

![](https://storage.googleapis.com/zenn-user-upload/6c3a374c1b74-20250303.png)

## まとめ

上記の操作により、SQL の IN 句にコピペして貼り付けたら即実行できるフォーマットの文字列を作り出すことができました。
何かの機会にご活用ください！
