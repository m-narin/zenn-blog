---
title: "【Julia】Agents.jlを利用したマルチエージェントシミュレーション③鳥の群れ"
emoji: "💻"
type: "tech"
topics:
  - "Julia"
published: true
published_at: "2023-12-27 00:00"
---

# はじめに
今回はAgents.jlの鳥の群れ(flocking)例題について見ていきます。各個体が3つのルールに基づいて周囲から影響を受けて行動をすることで、最終的には群れになっていく様子をコンピュータ上でモデル化していきます。こちらもABMの古典的な例題として親しまれています。

3つのルール
・衝突回避のための最小距離を維持する
・近隣の鳥の平均位置に向かって移動する
・近隣の鳥の平均方向に向かって移動する

▼URL

https://juliadynamics.github.io/Agents.jl/stable/examples/flock/

# モデルの定義

```julia:
using Agents, LinearAlgebra

mutable struct Bird <: AbstractAgent
    id::Int
    pos::NTuple{2,Float64}
    vel::NTuple{2,Float64}
    speed::Float64
    cohere_factor::Float64
    separation::Float64
    separate_factor::Float64
    match_factor::Float64
    visual_distance::Float64
end
```

連続空間を用いるため、鳥の属性としてposとvel、speedを用います。1stepごとに、velの方向にspeedの分だけposが更新されます。以下、移動に関わるその他の属性です。
・cohere_factor:近隣鳥の平均位置
・separation:衝突回避のための最小距離
・separate_factor:最小距離維持の動きの重み
・match_factor:近隣鳥の軌道に沿う動きの重み
・visual_distance:鳥の視界の範囲(半径)

```julia:
function initialize_model(;
    n_birds = 100,
    speed = 1.0,
    cohere_factor = 0.25,
    separation = 4.0,
    separate_factor = 0.25,
    match_factor = 0.01,
    visual_distance = 5.0,
    extent = (100, 100),
    spacing = visual_distance / 1.5,
)
    space2d = ContinuousSpace(extent, spacing)
    model = ABM(Bird, space2d, scheduler = Schedulers.randomly)
    for _ in 1:n_birds
        vel = Tuple(rand(model.rng, 2) * 2 .- 1)
        add_agent!(
            model,
            vel,
            speed,
            cohere_factor,
            separation,
            separate_factor,
            match_factor,
            visual_distance,
        )
    end
    return model
end
```

こちらの関数で初期化されたmodelを用意します。それぞれの鳥(agent)の属性の初期値を決め、vel(速度)のみランダムに定義してagentを100体生成します。また、100x100の連続空間を定義します。

# step関数

```julia:
function agent_step!(bird, model)
    # Obtain the ids of neighbors within the bird's visual distance
    neighbor_ids = nearby_ids(bird, model, bird.visual_distance)
    N = 0
    match = separate = cohere = (0.0, 0.0)
    # Calculate behaviour properties based on neighbors
    for id in neighbor_ids
        N += 1
        neighbor = model[id].pos
        heading = neighbor .- bird.pos

        # `cohere` computes the average position of neighboring birds
        cohere = cohere .+ heading
        if edistance(bird.pos, neighbor, model) < bird.separation
            # `separate` repels the bird away from neighboring birds
            separate = separate .- heading
        end
        # `match` computes the average trajectory of neighboring birds
        match = match .+ model[id].vel
    end
    N = max(N, 1)
    # Normalise results based on model input and neighbor count
    cohere = cohere ./ N .* bird.cohere_factor
    separate = separate ./ N .* bird.separate_factor
    match = match ./ N .* bird.match_factor
    # Compute velocity based on rules defined above
    bird.vel = (bird.vel .+ cohere .+ separate .+ match) ./ 2
    bird.vel = bird.vel ./ norm(bird.vel)
    # Move bird according to new velocity and speed
    move_agent!(bird, model, bird.speed)
end
```

`neighbor_ids`には、各birdの視界の半径の中に居る鳥を`nearby_ids`メソッドで取得しています。そして、for分の中で近隣鳥から受ける影響について計算しています。それぞれの影響は、`cohere`, `separate`, `match`3つに落とし込まれています。

・cohere:近隣鳥の平均位置へ向かう方向
・separate:最小距離以内にいる近隣鳥から離れる方向の平均
・match:近隣鳥の軌道(速度)の平均へ向かう方向

それぞれのの影響は、和の計算→正規化→重みづけ→velの更新という順に計算され、更新されたvelを用いて`move_agent!`メソッドによりagentの位置が移動します。

# 可視化

```julia:
using InteractiveDynamics
using CairoMakie

abm_video(
    "flocking.gif", model, agent_step!;
    framerate = 20, frames = 100,
    title = "Flocking"
)

# jupyter notebook上で見るためにgif形式
display("image/gif", read("flocking.gif"))
```

ライブラリを利用し、シミュレーションを実行して可視化を行います。

※tutorialには、plotの形状を任意に指定する方法が書かれていますが、可視化のためのバックエンドライブラリのupdateに伴う影響からかうまく動作できなかったので、デフォルトのものを使用しました。

![flocking.gif](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/614347/b76a4337-d57b-6492-6ae3-18527bbc5c65.gif)

agent同士が群れをなす様子が見て取れると思います。鳥の行動ルールとそこで指定のパラメータを用いてシミュレーションしました。しかし、群れを成した後、その位置に留まってしまうのでやや現実から乖離があるような気がします。そこで、パラメータにおいて、平均位置に留まる影響を小さくし、軌道に合わせる影響を大きくして試してみたいと思います。具体的には、
・cohere_factor: 0.25→0.1
・match: 0.01→0.1
・visual_distance: 5.0→10.0
にしてシミュレーションしてみます。

![flocking.gif](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/614347/cf2140fc-86ca-a765-e1b4-7a12f57f61cf.gif)

今度は位置に留まることなく、近隣鳥と同じ方向に動き続ける様子を再現することができました。こちらの方が、**なんとなく**より現実に沿っているような気がします。

# おわりに
鳥の群れができる様子を、ABMで実装しました。ルールやパラメータを色々と変えてみると、動きも変わっているのでとても興味深い内容だったと思います。ぜひ、オリジナルのルールを追加したり、パラメータを変えてみたりして遊んでみてください。
