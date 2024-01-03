---
title: "【Julia】Agents.jlを利用したマルチエージェントシミュレーション①シェリングの分居モデル"
emoji: "💻"
type: "tech"
topics:
  - "Julia"
published: true
published_at: "2023-12-25 00:00"
---

# はじめに

Julia上でエージェントベースモデル(ABM)の構築を行うパッケージであるAgents.jlの例題を勉強、解説していこうというコーナーになります。研究の際にダイレクトに扱っていて、まだプログラミング経験の浅いときに公式ドキュメントの読解からスタートすることになったため、当初は非常に苦労した記憶がありました。なので、将来そのような人達に向けてお役に立てれば嬉しい次第です。今回のその初陣となります。初回なので、JuliaとAgents.jlそれぞれの解説から入っていこうと思います。

※こちらのシリーズではJuliaの基礎文法の解説は端折ります。
※Juliaの環境構築やJupyter Notebook上で利用する方法も端折ります。以下の記事が非常に参考になると思います。

▼Julia 高速チュートリアル

https://nbviewer.org/github/bicycle1885/Julia-Tutorial/blob/master/Julia%E9%AB%98%E9%80%9F%E3%83%81%E3%83%A5%E3%83%BC%E3%83%88%E3%83%AA%E3%82%A2%E3%83%AB.ipynb

▼Julia環境構築

https://qiita.com/hujuu/items/12a5d16354fcc629200e

環境
・Julia 1.6.4
・Jupyter Notebook 
・Agents 4.2.5

# そもそもJuliaとは

▼ホームページ

https://julialang.org/

`Julia was designed from the beginning for high performance. Julia programs compile to efficient native code for multiple platforms via LLVM.`

一言でいうと、シンプルな構文と早い実行速度を両立させた新時代の言語という理解になります。Pythonの後継となる、取って代わるといったことが期待されているみたいで、その意味でも計算の速さを全面にウリにしているような印象があります。実行速度に関しては、Python等と比較しても数倍数十倍速いというコメントが色々なところで聞かれています。2012年にMITの開発者たちによってOSSされ、2018年にバージョン1がリリースされたかなり若い言語です。そのため、欠点としてはPythonとかと比較してみても、ライブラリ等が少ない、ライブラリを利用するときに色々問題が生じる、日本語のドキュメントが少ないのでエラーの解決やしたいことを調べるのが大変、、、等といったものがあります。僕もこの点は現在進行形で苦労しているところです。ぴえん。


# Agents.jl

▼ホームページ

https://juliadynamics.github.io/Agents.jl/stable/

▼ABMの比較

https://juliadynamics.github.io/Agents.jl/stable/comparison/

▼ABMとは？

https://qiita.com/Keyskey/items/4cf57088841b3db1fb8b

Agent.jlはJuliaの特徴をそのまま引き継いだように、実行速度が速いABMのフレームワークです。上記記事では他のフレームワークの代表例である、NetLOGOやMASON、MESA等と比較されてますね。Agents.jlは新興のライブラリなので、日々Updateされています。メソッド等の書き方が微妙に変わったりするので、たまに困惑しちゃいます( ;∀;)
また、初めてABMに取り組むときにagents.jlからスタートすると作法が独特だったりするので困惑するかもしれないですが、とりあえず素直にそういうものかと受け止めてみましょう。

# シェリングの分居モデル

▼URL

https://juliadynamics.github.io/Agents.jl/stable/examples/schelling/

シェリングの分居モデルは、ABM界隈では古典的な教材としてよく載っているものです。隣人の選り好みして満足いくまで引っ越すのを繰り返すというもので、満足度の基準が緩い場合であっても、分居されてしまうという洞察が、コンピュータシミュレーションで明かされます。元ネタを辿るとアメリカにて黒人、白人の分居が起こる様子をコンピュータ上で再現しようとしたものがきっかけとなっているようです。

## エージェント、空間の定義

```julia:
using Agents

space = GridSpace((10, 10); periodic = false)

mutable struct SchellingAgent <: AbstractAgent
    id::Int             # The identifier number of the agent
    pos::NTuple{2, Int} # The x, y location of the agent on a 2D grid
    mood::Bool          # whether the agent is happy in its position. (true = happy)
    group::Int          # The group of the agent, determines mood as it interacts with neighbors
end
```

AbstractAgentというスーパークラスを継承して変更可能なagentの型(ShellingAgent)を生成します。エージェントの集合は、IDをキー、構造体をバリューに持つ辞書型となっています。今回の場合、idと2D空間の座標を表す整数タプルのpos、mood、group4つのフィールドを構造体中に持ちます。
イメージとしては以下のようになるでしょうか

```julia:
Dict(1=> {id:1, pos:(1,1), mood:true, group:1},,,)
```

空間は10 x 10のグリッド空間を用意しています。

## モデルの作成

```julia:
using Random # for reproducibility
function initialize(; numagents = 320, griddims = (20, 20), min_to_be_happy = 3, seed = 125)
    space = GridSpace(griddims, periodic = false)
    properties = Dict(:min_to_be_happy => min_to_be_happy)
    rng = Random.MersenneTwister(seed)
    model = ABM(
        SchellingAgent, space;
        properties, rng, scheduler = Schedulers.randomly
    )

    # populate the model with agents, adding equal amount of the two types of agents
    # at random positions in the model
    for n in 1:numagents
        agent = SchellingAgent(n, (1, 1), false, n < numagents / 2 ? 1 : 2)
        add_agent_single!(agent, model)
    end
    return model
end
```

モデルの生成は関数として用意することが通常用いられています。モデルの初期化関数です。まず空間パラメータやエージェント数などの引数を基にして、space、propertiesを定義します。model = ABMの部分でモデル作成メソッドを用いてモデルを生成します。rngはエージェントのランダムな初期値(pos)を固定するためのものです(シミュレーションごとにランダムな初期値が異なっていてはパラメータを変えた時の比較が意味をなさなくなる)。また`scheduler = Schedulers.randomly`は、各stepごとにstep関数が適用されるagentの順番をランダムにすることを表します(デフォルトはagentのIDの若い順)。


また、for文の中で、実際にエージェントを生成します。agent = の部分で各agentの属性に初期値を入れ、(三項演算子が用いられていますね)`add_agent_single!`メソッドで実際にモデル中にエージェントを生成します。このエージェントは、先ほども書いたように、model中にはIDをキー、構造体をバリューとする辞書型として、agentの数だけ辞書の要素と構造体(ShellingAgent)が生成されます`add_agent_single!`メソッド等はAPIとしてagents.jl内で用意されているものです。

## ステップ関数

```julia:
function agent_step!(agent, model)
    minhappy = model.min_to_be_happy
    count_neighbors_same_group = 0
    # For each neighbor, get group and compare to current agent's group
    # and increment count_neighbors_same_group as appropriately.
    # Here `nearby_agents` (with default arguments) will provide an iterator
    # over the nearby agents one grid point away, which are at most 8.
    for neighbor in nearby_agents(agent, model)
        if agent.group == neighbor.group
            count_neighbors_same_group += 1
        end
    end
    # After counting the neighbors, decide whether or not to move the agent.
    # If count_neighbors_same_group is at least the min_to_be_happy, set the
    # mood to true. Otherwise, move the agent to a random position.
    if count_neighbors_same_group ≥ minhappy
        agent.mood = true
    else
        move_agent_single!(agent, model)
    end
    return
end
```

エージェントの内部状態(属性)を更新する条件をステップ関数として記載します。引数として、各agent一つとmodelを受け取ります。`nearby_agents`は周辺にいるエージェントを取得するAPI、`move_agent_single!`はエージェントのposをランダムな場所に更新するAPIです。もし近所に同じグループのagentが一定以上いたらその`agent.mood`がtrueになり、そうでなかったら引っ越すという事象を記述しています。

## モデルの実体の生成

```julia:
model = initialize()
```

さきほど定義したinitialize関数を実行させることで、モデルとagentを実際に生成します。ここでtutorialにはないですが、各agentの構造を調べてみましょう。

```julia:
model[1]
```
辞書となっているキー番号を指定すると
`SchellingAgent(1, (14, 19), false, 1)`
例えばこのようにjuliaの構造体が帰ってきます。

```julia:
model[1].mood
```
`false`
ドットで該当の構造体にアクセスすることができます。

## 可視化

```julia:
using InteractiveDynamics
using CairoMakie # choosing a plotting backend

groupcolor(a) = a.group == 1 ? :blue : :orange
groupmarker(a) = a.group == 1 ? :circle : :rect
figure, _ = abm_plot(model; ac = groupcolor, am = groupmarker, as = 10)
figure # returning the figure displays it
```

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/614347/9645c2cb-0739-76d8-ca9b-ab4db6ff1dbf.png)


可視化のためのバックエンドを整えてくれるライブラリを用います。`abm_plot`を利用できるみたいです。色と形の関数を簡易的に定義して、初期画面のみ画像化しています。
※可視化ライブラリはまだまだ発展途上なので、このあたりはあんまり確立してない印象が少しあります。

## アニメーション

```julia:
model = initialize();

# tutorialとは異なりjuypter上で出力しやすいgif形式に変更
abm_video(
    "schelling.gif", model, agent_step!;
    ac = groupcolor, am = groupmarker, as = 10,
    framerate = 4, frames = 20,
    title = "Schelling's segregation model"
)

# gifを出力
display("image/gif", read("schelling.gif"))
```

`abm_video`を利用して、agentの内部状態を実際にステップさせてそれを可視化したアニメーションを作成できます。言い方を変えモデルを構築して実際にシミュレーションしたところ、分居が進むことがここから理解できると思います。

![schelling.gif](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/614347/26eb7279-395c-ee01-88b0-d0b7ef744b0b.gif)

## データ収集

```julia:
adata = [:pos, :mood, :group]

model = initialize()
data, _ = run!(model, agent_step!, 5; adata)
data[1:10, :] # print only a few rows
```

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/614347/5fdf3fae-5544-aade-fcff-a45c0d822a74.png)

`data`にシンボルの形でエージェントの属性を設定します。`run!`関数を使うと、Dataframeの形でagentをステップさせた分のagentの各属性のデータが記録されます。他にも色々なデータ収集の例が載っています。このDataframeを利用してパラメータを変えていき、結果の可視化をすることがよく行われたりします。ABMのシミュレーションとしては、例えば`min_to_happy`のパラメータを変えて結果を見てみる等ができます。DataframeといえばPythonでも用いられているので、互換性がありますね。

## おわりに

シェリングの分居モデルを通して、Agents.jlの概要を触れてみました。こちらの例題では、この後インタラクティブアプリケーションの構築や分散コンピューティングの話題が出てきます。