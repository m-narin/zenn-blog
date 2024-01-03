---
title: "【Julia】Agents.jlを利用したマルチエージェントシミュレーション④ライフゲーム"
emoji: "💻"
type: "tech"
topics:
  - "Julia"
published: true
published_at: "2023-12-28 00:00"
---

# はじめに
今回はAgents.jlのコンウェイのライフゲームの例題を見ていきます。ライフゲームは、生命の誕生、進化、淘汰をコンピュータ上でモデル化してシミュレーションを試みたものです。生物や物理の種々の現象を、格子状のセルに離散化して簡単なルールに基づいて解き明かそうとするセルオートマトンの古典的な例題として親しまれています。

![game of life.gif](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/614347/81b09eca-32b9-4924-4a33-2e9c60703721.gif)


▼URL

https://juliadynamics.github.io/AgentsExampleZoo.jl/dev/examples/game_of_life_2D_CA/

▼ライフゲーム

https://ja.wikipedia.org/wiki/%E3%83%A9%E3%82%A4%E3%83%95%E3%82%B2%E3%83%BC%E3%83%A0

▼セルオートマトン

https://ja.wikipedia.org/wiki/%E3%82%BB%E3%83%AB%E3%83%BB%E3%82%AA%E3%83%BC%E3%83%88%E3%83%9E%E3%83%88%E3%83%B3

# ルールの設定

```julia:
using Agents, Random

rules = (2, 3, 3, 3) # (D, S, R, O)
```

rulesはそれぞれ、Death, Survival, Reprodution, Overproductionを表します。agentのlifeはそれぞれの数字に基づいて決定されます。

近隣にいるagentの数(生きているセルを表す)によって以下のルールが適用されます。

**生きているセル**
死: 生存セルが0,1か4以上のとき(過疎と過密で死滅する)
生存: 生存セルが2か3のとき

**死んでいるセル**
誕生: 生存セルがちょうど3のとき

# モデルの定義

```julia:
mutable struct Cell <: AbstractAgent
    id::Int
    pos::Dims{2}
    status::Bool
end
```

pos: Dim{2}はTuple{Int64, Int64}のこと
status: trueは生存、falseは死を表す

```julia:
function build_model(; rules::Tuple, dims = (100, 100), metric = :chebyshev, seed = 120)
    space = GridSpace(dims; metric)
    properties = Dict(:rules => rules)
    model = ABM(Cell, space; properties, rng = MersenneTwister(seed))
    idx = 1
    for x in 1:dims[1]
        for y in 1:dims[2]
            add_agent_pos!(Cell(idx, (x, y), false), model)
            idx += 1
        end
    end
    return model
end
```

build_model!でABMのを生成します。100x100のGrid空間を利用し、`metric = :chebyshev`によって、近隣に対角上のセルを含むように設定しています。propertiesにruleを設定しています。rngで乱数を固定しています。2重ループを使い、100x100の全てのセル上に死のagentを生成しています。

# ステップ関数

agentの内部状態を更新する際、各step数ごとに全agentを同期して(まとめて)更新するように記述していきます(通常はagent一つずつに更新が適用されていきます)。

```julia:
function ca_step!(model)
    new_status = fill(false, nagents(model))
    for agent in allagents(model)
        n = alive_neighbors(agent, model)
        if agent.status == true && (n ≤ model.rules[4] && n ≥ model.rules[1])
            new_status[agent.id] = true
        elseif agent.status == false && (n ≥ model.rules[3] && n ≤ model.rules[4])
            new_status[agent.id] = true
        end
    end

    for id in allids(model)
        model[id].status = new_status[id]
    end
end

function alive_neighbors(agent, model) # count alive neighboring cells
    c = 0
    for n in nearby_agents(agent, model)
        if n.status == true
            c += 1
        end
    end
    return c
end
```
`nearby_agents`メソッドを利用し、近隣のagentの生存数をカウントする関数(`alive_neighbors`)を下で定義しています

`ca_step!`は、まずnew_statusにmodel中のagentの数だけfalseを一次元配列として仮に用意しておきます。全てのagentに対し、近隣のagentの生存数によって、new_status配列のIndexをagent.idに相当するものとみなしてtrueに変更しています。次に、全agentのstatusをnew_statusに置き換えて更新します。

# モデルの作成

```julia:
model = build_model(rules = rules, dims = (50, 50))
```

ABMを生成します。

```julia:
for i in 1:nagents(model)
    if rand(model.rng) < 0.2
        model.agents[i].status = true
    end
end
```

20%分だけランダムなセルのstatusをtrueにします。生存セルの割合が20%のものを初期状態として生成することを意味しています。

# アニメーションとして実行

```julia:
using InteractiveDynamics
import CairoMakie

ac(x) = x.status == true ? :black : :white
am(x) = x.status == true ? '■' : '□'
abm_video(
    "game of life.gif",
    model,
    dummystep,
    ca_step!;
    title = "Game of Life",
    ac = :black,
    as = 12,
    am,
    framerate = 5,
    scatterkwargs = (strokewidth = 0,),
)
display("image/gif", read("game of life.gif"))
```

生きているagentを■、死んでいるagentを□とします。これは各セルの生死状態を表現しています。agents.jlでシミュレーションを実行する際、step関数には各agentの内部状態を個別に更新する関数と、大域的なmodelの状態を更新する関数の2種類を引数として受け取ることができます。今回の場合、`ca_step!`関数はmodel全体の更新を担当し、agentごとに個別の更新が必要ないためagentの更新関数の引数はdummystepという値を設置します。

▼ step関数の仕組み (参考)

https://juliadynamics.github.io/Agents.jl/stable/tutorial/#.-Evolving-the-model

# おわりに
ライフゲームをagents.jlで実装しました。イギリスの数学者であったコンウェイが1970年に考案したものを簡単に追随できたかと思います。また、agents.jlのstep関数の応用的な利用方法について知ることができる例題だったと思います。
