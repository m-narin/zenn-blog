---
title: "【Julia】Electron.jlを使ってカウンターアプリを作る"
emoji: "💻"
type: "tech"
topics:
  - "Julia"
published: true
published_at: "2023-12-30 00:00"
---

# はじめに
この記事ではElectron.jlで以下のようなGUIのカウンターアプリを作ります。これを通してElectron.jlの基本的な使い方を習得できるかなと思います。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/614347/5c0dbcc8-091e-4b08-2979-0ac850f40ffa.png)

筆者自身大学の研究プロジェクトでJuliaを用いたことがあり、可視化用のアプリケーションを作成する必要がありました。そこで調べたところElectron.jlが使えそうとなりましたが、

* [Electron.jlの公式Github](https://github.com/davidanthoff/Electron.jl)の解説があまりにも簡素
* 解説記事が少ない上に、独特の記法がいくつかある

このような事情で慣れるまでに時間がかかった記憶があります。そのため、Electron.jlの使い方を理解できるチュートリアルとしてのサンプル兼解説記事を書こうと思いました。ただしこの記事ではHTML, CSS, JavaScript, Juliaの基本は理解されている前提で書いています。

特徴
* 入力値のカウントアップ
* カウンターのリセット
* カウンターの累積画像を作成し、更新

# Electron.jlとは?
Electron.jlはElectronをJuliaで使えるようにしたものです。ElectronはJavaScript、HTML、CSSを使ったデスクトップアプリケーションのことです。Juliaは数値計算が得意で、実行速度が速いという特徴があります。従来のPythonに置き換わる手段として、研究分野での需要が高まりつつある言語です。つまり、Electron.jlはこのようなメリットのあるJuliaをバックエンドに据えたデスクトップアプリを作ることができます。

# コード
この記事で作成するカウンターアプリのソースコードは以下のGithubからダウンロードできます。

[electronjl_sample_counter](https://github.com/mando0520/electronjl_sample_counter)

READMEを見ながら必要なライブラリを揃えれば、この記事のElectron.jlのカウンターアプリを起動することができます。

この記事では各ファイルのソースコードの意味について触れていきます。

```html:main.html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Counter</title>
  <link rel="stylesheet" href="main.css">
</head>

<script type="text/javascript" src="main.js"></script>

<body>
  <div class="counter_container">
    <div class="title" >Counter</div>
      <div class="counter_input">
        <input id="input_number" type="text" value="1" size="5">
        <input type="button" value="+" onClick="plus();">
        <input type="button" value="reset" onClick="reset();">
      </div>
    <div id="count_number">0</div>
    <fieldset class="data_image_field">
      <legend>AccumulationImage</legend>
      <img id="accumulation_image" src="../output_images/default_image.png" width="400" height="300" alt="accumulation_image">
    </fieldset>
  </div>
</body>

</html>
```

HTMLでアプリのデザインの骨格を作成します。入力、ボタン、画像等の要素を入れています。buttonではJavaScriptの関数を呼んでいます。

```css:main.css
.counter_container{
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 20px;
}
.counter_input{
  display: flex;
  gap: 20px;
}
.title{
  font-weight: 600;
  font-size: 26px;
}
#count_number{
  font-size: 26px;
}
```

CSSでHTMLのデザインを整えます。

```js:main.js
function plus(){
  let input_number = Number(document.getElementById("input_number").value)
  let jsonString = makeJsonString("plus", input_number)
  let jsonData = JSON.parse(jsonString)
  sendMessageToJulia(jsonData)
}

function reset(){
  let jsonString = makeJsonString("reset")
  let jsonData = JSON.parse(jsonString)
  sendMessageToJulia(jsonData)
}

function makeJsonString(order, input_number=0){
  let jsonString = `{"order":"${order}","input_number":${input_number}}`;
  return jsonString
}

function setCountNumber(count_number){
  let countNumberElement = document.getElementById("count_number");
  countNumberElement.textContent = count_number
}

function setImage(file_path){
  let imageElement = document.getElementById("accumulation_image");
  imageElement.src = file_path;
}
```
JSで色々な関数を定義します。`sendMessageToJulia`はElectron.jlで定義されている、Juliaに情報を送信するための専用の関数です。送信する情報は互換性と拡張性の観点からJson形式にします。JS側でJsonの文字列を生成し、Json形式に直してJuliaに送信します。
また、Julia側で利用するためのJSの関数も定義します。Julia側で更新されたカウントの数字そのものと画像を、HTMLの要素を書き換えることで出力します。

```julia:main.jl
using Electron

include("useImage.jl")

app = Application()

main_html_uri = string("file:///", replace(joinpath(@__DIR__, "main.html"), '\\' => '/'))

win = Window(app, URI(main_html_uri))

ElectronAPI.setBounds(win, Dict("width"=>600, "height"=>600))

cmd = open("main.js") do file
  read(file, String)
end

run(win, cmd)
ch = msgchannel(win)
request = Dict()
dataArray = [0]

while true
  try 
    global request = take!(ch)
    println("request: $(request)")
  catch 
    println("channle closed")
    break
  end

  if request["order"] == "plus"
    global dataArray
    push!(dataArray, dataArray[end]+request["input_number"])
    run(win,"setCountNumber(\"$(dataArray[end])\");")

  elseif request["order"] == "reset"
    dataArray = [0]
    run(win,"setCountNumber(0);")
  end

  file_name = make_accumulation_image(dataArray)
  run(win,"setImage(\"../output_images/accumulation_image/$(file_name)\");")

  println("regenerate window")

end
```

まず、ElectronアプリのWindowを立ち上げるための処理を書きます。そしてJSからの情報を受け取るためのChannelを作成します。待機中のChannelにJSから情報が届くと`take!`で中身を取得します。

`while true`はWindowの存在に対応しています。Channelから情報を取り出す度に以降の処理が走ります。

今回の場合、Json形式のデータは以下のようにJulia側で辞書型として受け取ることになります。

```julia:
request: Dict{String, Any}("order" => "plus", "input_number" => 1)
```

このデータは命令と数字情報から成ります。命令に応じてif文で処理を分岐します。グローバル変数の`dataArray`には入力された数値が累積値としてどんどん入っていきます。次に`run`関数でJSの関数を呼び出します。`run`関数の第二変数は、Juliaの変数を埋め込んだJS関数の実行文字列です。

また、`make_accumulation_image`で画像を作成します。これは`useImage.jl`で定義されたJuliaの関数です。

```julia:useImage.jl
using Plots, Images, Dates

function make_accumulation_image(dataArray)
	rm("../output_images/accumulation_image/",recursive=true)
	mkdir("../output_images/accumulation_image")
	p = plot(dataArray, st=:bar, title="Accumulation", label="data")
	file_name = string(Dates.format(now(), "yyyymmddHHMMSS"),"_accumulation.png")
	save("../output_images/accumulation_image/$(file_name)", p)
	return file_name
end
```
3つのライブラリを用い、画像を作成します。更新される画像はファイル形式を取ります。デスクトップアプリには簡単にファイルの削除や作成ができる利点があります。作成される画像ファイルは`output_images/accumulation_image`フォルダーに置かれます。


この関数は、まず以前の画像ファイルを削除します。そして`Plot`で`dataArray`の累積画像を作成します。新しく作成された画像は上記のフォルダーに保存されます。ファイル名は一意に特定できるように時間情報を入れます。そして`file_name`を返します。

返された`file_name`は`main.jl`の`run`関数でJSの関数呼び出しの引数に利用します。このようにして、更新された画像をHTMLに出力することができます。

# おわりに
基本的なカウンター機能に加え、画像をアプリケーション上で出力する機能を作成しました。これを応用することでJavaScriptとJuliaの通信を自由に設定することができます。そのためJuliaで処理をさせて可視化する色々なGUIパターンを作ることができます。この記事が皆さんの研究等のお役に立てれば幸いです。
