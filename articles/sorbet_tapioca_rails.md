---
title: "SorbetとTapiocaでRailsに型を導入する"
emoji: "🍧"
type: "tech"
topics:
  - Rails
  - Ruby
  - Sorbet
  - Tapioca
  - RubyLSP
published: true
---

## はじめに

Rails で大きめのアプリを書いていると、

- このメソッドが返すのは結局何の型なのか分からない
- `params[:something]` や変数がどんな構造なのか、毎回出力して確認する
- リファクタしたら、ランタイム環境で `NoMethodError: undefined method 'foo' for nil:NilClass`が発生する

といった「動かしてみないと型が分からない」ストレスが積み上がってきます。

そこで本記事では、Rails アプリに [**Sorbet**](https://sorbet.org/) と [**Tapioca**](https://github.com/Shopify/tapioca) を導入して **静的型付け** で書く方法を紹介します。コントローラ（`articles#index`）を題材に、型あり/型なしの違いとエディタ（ruby-lsp）でどんな体験になるかを見ていきます。

## Sorbet と Tapioca とは

**[Sorbet](https://sorbet.org/)** は Stripe が開発した Ruby 向けの静的型チェッカです。Ruby 自体に型を後付けするための仕組みで、

- メソッドのシグネチャ（引数と戻り値の型）を `sig { ... }` ブロックで書く
- ファイル冒頭の `# typed: true` などで「このファイルでは型チェックを有効にする」と宣言する
- 静的チェック（`srb tc`）と、本番でも効くランタイムチェック（`sorbet-runtime`）の両方を提供する

という構成になっています。

**[Tapioca](https://github.com/Shopify/tapioca)** は Shopify が開発した、Sorbet のための **RBI（Ruby Interface）ファイル生成ツール**です。Sorbet は Ruby の「動的に生えるメソッド」（Rails のスコープ、`has_many`、Devise の `authenticate_user!` 等）をそのままでは認識できません。Tapioca はこうした「フレームワークや gem が生やす動的メソッド」のシグネチャを **自動で RBI として書き出してくれる** ので、Rails のような魔法多めのフレームワークでも Sorbet がちゃんと動くようになります。

ざっくりまとめると:

- **Sorbet**: 型チェッカ本体（自分で書く `sig` を読み取って検査）
- **Tapioca**: Rails / gem 側の動的メソッドを RBI として書き出す補助ツール

の二人三脚です。

なお、**Tapioca は Sorbet 公式（[sorbet.org](https://sorbet.org)）が RBI 生成手段として推奨しているツール**で、現在の Sorbet × Rails ではデファクトスタンダードになっています。以前は `sorbet-rails` という別 gem を使う構成もありましたが、新規プロジェクトで Sorbet を Rails に入れるなら Tapioca を選んでおけば間違いありません。

## 型を入れると何が嬉しいか

### 1. リファクタが「壊さない」と分かったうえでできる

戻り値の型を `T::Struct` で固めておくと、フィールド名を変えたり追加したりしたときに、参照側のミスマッチが**実行前に**全部洗い出せます。「ここを直したら他のどこに飛び火するか」を grep じゃなく型チェッカに教えてもらえます。

### 2. メソッドの契約が「書いてある」

`sig { params(id: Integer).returns(Article) }` のように、メソッドの入り口と出口がコードに書いてある状態になります。**ドキュメントが本体と一緒にコミットされる**感覚に近く、しかも嘘がついていればチェッカが落ちます。

### 3. エディタ補完が劇的に効く

[**ruby-lsp**](https://github.com/Shopify/ruby-lsp) と組み合わせると、VSCode などのエディタ上で「このメソッドが返すオブジェクトのプロパティ補完」「型違いをその場で警告」が効くようになります。`a.` まで打ったら `id`, `title`, `published_at` が候補に出てくる、というやつです。後半で実例を見ます。

### 4. `nil` チェックを「強制」できる

`T.nilable(String)` と書いたフィールドは、参照側で nil チェックなしには使えません。「ぬるぽが本番に出てから気づく」を減らせます。

## セットアップの全体像（概要）

詳細な手順は公式に譲りますが、Rails への導入は概ねこの流れです。

```ruby
# Gemfile
group :development do
  gem "sorbet"
  gem "tapioca", require: false
end

gem "sorbet-runtime"
gem "sorbet-coerce" # 後述の TypeCoerce を使うために追加
```

```bash
# 初期化（sorbet/config と sorbet/rbi/ が作られる）
bundle exec tapioca init

# gem の RBI を生成
bundle exec tapioca gems

# Rails の動的メソッド（DSL）の RBI を生成
bundle exec tapioca dsl

# 型チェック実行
bundle exec srb tc
```

ポイントは **`tapioca gems` と `tapioca dsl` を生成し直す運用**になることです。

- 新しい gem を追加したら `tapioca gems`
- Rails モデルにカラムを足したら `tapioca dsl`

を忘れると型情報が古くなるので、CI でチェックする運用にしておくのが定番です（後述）。

## 実例: `articles#index` を型ありで書く

ここからが本題です。よくある「記事一覧 API」を題材に、型なしと型ありを並べてみます。

### 型なし版（よくある書き方）

```ruby
class ArticlesController < ApplicationController
  def index
    articles = fetch_articles(limit: params[:limit] || 10)

    render json: {
      articles: articles.map { |a|
        {
          id: a.id,
          title: a.title,
          published_at: a.published_at,
        }
      },
      total: articles.size,
    }
  end

  private

  def fetch_articles(limit:)
    Article.order(created_at: :desc).limit(limit)
  end
end
```

これでも動きますが、

- `limit` は文字列？整数？（実は `params` から来るので文字列。`.limit("10")` でも一応動くが暗黙の型変換に依存している）
- レスポンスの `articles[].id` は何の型？
- `published_at` は nil になりうる？

がコードを読んでも即答できません。

### 型あり版

同じ処理を Sorbet + `T::Struct` で書き直すとこうなります。

```ruby
# typed: strict

class ArticlesController < ApplicationController
  extend T::Sig

  # 入力パラメータの型
  class IndexParams < T::Struct
    const :limit, Integer, default: 10
  end

  # レスポンスの 1 件分の型
  class ArticleResponse < T::Struct
    const :id, Integer
    const :title, String
    const :published_at, T.nilable(ActiveSupport::TimeWithZone)
  end

  # レスポンス全体の型
  class IndexResponse < T::Struct
    const :articles, T::Array[ArticleResponse]
    const :total, Integer
  end

  sig { void }
  def index
    input = TypeCoerce[IndexParams].new.from(
      params.permit(:limit).to_h,
    )

    articles = fetch_articles(limit: input.limit)

    response = IndexResponse.new(
      articles: articles.map { |a|
        ArticleResponse.new(
          id: a.id,
          title: a.title,
          published_at: a.published_at,
        )
      },
      total: articles.size,
    )

    render(json: response.as_json)
  end

  private

  sig { params(limit: Integer).returns(T::Array[Article]) }
  def fetch_articles(limit:)
    Article.order(created_at: :desc).limit(limit).to_a
  end
end
```

何が変わったか:

- **`# typed: strict`**: このファイルは strict モードで型チェックする宣言。すべてのメソッドに `sig` が必要になる代わりに、最も厳しくチェックされる
- **`extend T::Sig`**: `sig { ... }` を使えるようにする mixin
- **`IndexParams` / `ArticleResponse` / `IndexResponse`**: 入出力を `T::Struct` で定義。フィールドごとに型と必須/任意（`T.nilable(...)`）を明示
- **`TypeCoerce[IndexParams].new.from(...)`**: `params`（中身は基本 String）を `IndexParams` の型に**変換**してから使う。文字列の `"1"` を `Integer` 1 にしてくれる。なお `TypeCoerce` は Sorbet 本体ではなく [`sorbet-coerce`](https://github.com/chanzuckerberg/sorbet-coerce) gem が提供しているヘルパー
- **`sig { void }`**: `index` アクションは戻り値を使わない（`render` で済む）ので `void` を宣言
- **`fetch_articles(limit:)` の sig**: クエリ部分を private メソッドに切り出し、`sig { params(limit: Integer).returns(T::Array[Article]) }` で「`Integer` を受け取って `Article` の配列を返す」という契約を明示。`# typed: strict` ではすべての public/private メソッドに `sig` が必要になるので、こうした内部メソッドの戻り値型も自然と書く流れになる

レスポンスの形が `IndexResponse` という型として明示されているので、「このエンドポイントの返り値は何か？」がコードを読めば一発で分かります。

### TypeCoerce はストロングパラメータの役割も兼ねる

ここで地味に効いてくるのが **`TypeCoerce[IndexParams].new.from(...)` の入り口** で、これは **ストロングパラメータ + 型バリデーション** を一手に引き受けてくれます。

Rails 本来のストロングパラメータ（`params.permit(:limit)`）は、

- **キーの絞り込み**は行う（`limit` 以外のキーは弾く）
- ただし **値の型までは検証しない**（`params[:limit]` が `"10"` でも `"abc"` でも、`permit` 自体は素通り）

という挙動です。`"abc"` が来ていることに気づくのは、後続コードで `.to_i` した結果 `0` になってバグった、というタイミングだったりします。

TypeCoerce + T::Struct を被せると、

- **構造の絞り込み**: `IndexParams` で宣言したフィールド（ここでは `limit`）以外は最終的な `input` に乗らない
- **型の検証と型変換**: `"10"` のような文字列は `10` の `Integer` に自動変換され、`"abc"` のように **Integer に変換不能なものはここで例外として弾かれる**
- **`nil` 安全と必須チェック**: `T.nilable(...)` を付けていないフィールドが欠けていれば、ここで弾かれる

つまりこの 1 行で、**「許可フィールドの絞り込み（≒ 従来の strong parameters）」と「型ベースの入力バリデーション（≒ 独自バリデーター）」が同時に走る**形になっています。コントローラ本体に届く `input` は「フィールドが揃っていて、型もきれい」が保証された状態なので、後続のロジックを書くときに `nil?` チェックや `to_i` を散らす必要がなくなります。

## Sorbet は静的＋ランタイムの両方で型を検査する

Sorbet の型チェックは **コードを書いている時** だけでなく、**アプリが動いている時** にも効きます。「型はビルド時だけで実行時には消える」 TypeScript のような仕組みとは、ここがけっこう違うポイントです。

### 2 種類のチェックが同時に走っている

`sig` や `T::Struct` を 1 つ書くと、次の 2 つのチェックが**同時に**有効になります。

- **静的チェック (`srb tc`)**: コードを書いた段階で型違反を検出する（コードは実行されない）。VSCode に出る赤波線もこれ
- **ランタイムチェック (`sorbet-runtime`)**: 実際にメソッドが呼ばれた瞬間や `T::Struct.new` が走った瞬間に、宣言した型と渡ってきた値が合っているかを検査する。違反していれば `TypeError` を投げる

たとえば次のコード:

```ruby
class IndexParams < T::Struct
  const :limit, Integer
end

IndexParams.new(limit: "abc")
```

これは **静的にも**（`String` を `Integer` のフィールドに渡してる、と `srb tc` が怒る）**ランタイムでも**（実行された瞬間に `TypeError`）、両方で検出されます。

### なぜランタイムでも検査するのか

特に効くのは **外部から来るデータ** に対してです。`params` / JSON ボディ / 環境変数 / 外部 API のレスポンスは、コードを読んでも実行時にしか中身が分かりません。**静的チェックは中身まで握れないので、ランタイムで「来た値が宣言と合っているか」を見るしかない**わけです。

前述の `TypeCoerce` がやっているのもまさにこれで、

- `params` から来た文字列 `"10"` を、`IndexParams` の `Integer` フィールドに入る `10` に**変換する**（こうした「宣言した型に合わせて値を直す」処理のことを **コアシオン (coercion)** と呼びます。要するに「型に合うように値を変換する」こと）
- `"abc"` のように変換できない値が来たら、その場で raise

という処理をしてくれます。

### 運用上の注意

- **デフォルトで全 sig がランタイムでも検査される**: 通常は気にならないオーバーヘッドですが、ホットパス（毎リクエスト数万回など）で気になる場合は、個別 sig に `.checked(:never)` を付けてランタイム検査だけ切ることもできます
- **本番でも raise する**: 型崩れに即座に気づける反面、想定外の例外が深部から飛んでくる可能性も。API なら境界（コントローラ等）で `rescue_from` して 4xx/5xx に変換する設計が安全。`TypeCoerce` の例外もここで拾うのが定番

要するに、**コードのバグは静的に、入力データの不整合はランタイムに**、と二段構えで型崩れを潰せる、というのが Sorbet の特徴です。

## ruby-lsp 連携でエディタ体験を上げる

[**ruby-lsp**](https://github.com/Shopify/ruby-lsp)（および Rails 向けの [`ruby-lsp-rails`](https://github.com/Shopify/ruby-lsp-rails)）は、Sorbet の型情報を読んでエディタに補完・診断を提供してくれます。VSCode なら `Shopify.ruby-extensions-pack` 拡張を入れるだけで使えます。

ruby-lsp / Sorbet 拡張を有効化すると、VSCode のステータスバーに **「Indexing files...」** のような表示が出て、プロジェクトのソースや RBI を解析していることが確認できます。解析が終わると、以下で紹介する型違反の警告や補完がエディタ上で効くようになります。

### 例 1: フィールドの型を間違えると即座に警告される

たとえば `ArticleResponse.new(...)` の `id` フィールドは `Integer` ですが、ここに誤って `String` を渡そうとしてみます。

```ruby
ArticleResponse.new(
  id: a.title,            # ← String を Integer のフィールドに渡そうとしている
  title: a.title,
  published_at: a.published_at,
)
```

エディタ上では `id: a.title` の行に **赤波線**が出て、「`Integer` が期待される位置に `String` が来ている」という診断が表示されます。

![](/images/sorbet_tapioca_rails/image1.png)

### 例 2: 補完が型を見て効く

`articles.map { |a| ... }` の `a` は `Article` の AR インスタンス。Tapioca が `dsl` で生成した RBI のおかげで、`a.` のように打つと **`id` / `title` / `published_at` / `created_at` / ...** といったカラムが候補に出てきます。

![](/images/sorbet_tapioca_rails/image2.png)

### 例 3: `nil` を意識させてくれる

`published_at` は `T.nilable(Time)` で「nil の可能性あり」と宣言しているので、ここを直接 `.strftime(...)` しようとすると **「nil の可能性があるオブジェクトに対するメソッド呼び出し」** として診断されます。

```ruby
# ❌ published_at が nil の可能性があるため警告
ArticleResponse.new(
  ...,
  published_at: a.published_at.strftime("%Y-%m-%d"),
)
```

![](/images/sorbet_tapioca_rails/image3.png)

`a.published_at&.strftime("%Y-%m-%d")` か、明示的に `T.must(...)` で潰すかを書くと警告は消えます。

## Tapioca が裏で何をしているか

上記のコード補完で「`a.title` の戻り値が `String` になる」ことや「`a.published_at` が `T.nilable(...)` で nil の可能性がある」ことを ruby-lsp が知っているのは、Tapioca が `sorbet/rbi/dsl/article.rbi` のような RBI を**事前に書き出している**からです。生成された RBI はこんな雰囲気になります。

```ruby
# typed: true
# DO NOT EDIT MANUALLY
# This is an autogenerated file for dynamic methods in `Article`.

class Article
  include GeneratedAttributeMethods
  include EnumMethodsModule

  module GeneratedAttributeMethods
    sig { returns(Integer) }
    def id; end

    sig { returns(String) }
    def title; end

    sig { returns(T.nilable(ActiveSupport::TimeWithZone)) }
    def published_at; end
    # ...
  end

  module EnumMethodsModule
    sig { returns(T::Boolean) }
    def draft?; end

    sig { returns(T::Boolean) }
    def published?; end
    # ...
  end
end
```

Rails のマイグレーションでカラムを追加したら、`bin/tapioca dsl` を再生成すればここに新しいカラムが反映されます。これを忘れると **「型情報が古くて補完されない」「実態とズレた型で警告される」** といった現象が起きるので、CI でチェックするのが定番です。

### `bin/tapioca dsl` の再生成が必要になる主なケース

「Rails 側で動的にメソッドが生える」操作をしたときは、原則 `bin/tapioca dsl` を回し直す必要があります。よく出てくる例を挙げておきます。

- **マイグレーションでカラムを追加した**: `t.string :slug` を追加 → `slug` / `slug=` / `slug?` のゲッター/セッターが Tapioca で生成される。再生成しないと「`a.slug` が `T.untyped`」になり補完も警告も効かない
- **モデルに `enum` を定義した**: たとえば `enum :status, { draft: 0, published: 1 }` を足すと、Tapioca が `draft?` / `draft!` / `published?` / `published!` といった述語メソッドや、`Article.draft` / `Article.published` のようなスコープを RBI に書き出してくれる。再生成を忘れると `a.draft?` が「メソッドがない」扱いで赤波線が出る
- **`scope` を追加した**: `scope :recent, -> { order(created_at: :desc) }` のようなスコープも Tapioca が `Article.recent` の sig を生成する対象。新しいスコープを足したら再生成しないと呼び出し側で補完されない
- **`has_many` / `belongs_to` などのアソシエーションを追加した**: `comments` / `comments=` / `build_comment` 等のアソシエーション系メソッドが Tapioca で生成される
- **モデルに自分で書いたメソッドを足した**: これは Tapioca ではなく Sorbet が**直接ソースを読む**ので再生成は不要。ただし **`sig { ... }` を書かないと型情報がつかない** ので、引数や戻り値の型を効かせたいなら手で sig を足す。`# typed: strict` のファイルでは sig 必須

ざっくり言えば、「**Rails / gem の DSL で動的に生やしたものは Tapioca、自分で書いたメソッドは Sorbet 直読み**」という棲み分けです。前者は再生成を忘れがちなので、CI に組み込んで自動検出するのが定石になっています。

## CI で「型情報の更新忘れ」を防ぐ

実運用では、`tapioca dsl` の生成結果が `git` 上で最新かどうかを CI で検査するのが定番です。考え方はシンプルで、

```bash
bin/tapioca dsl
git add --intent-to-add .
git diff --exit-code
```

を CI で回し、「diff が出る = 開発者が再生成を忘れている」をエラーにする、というやり方です。これがあれば、Gemfile やマイグレーションを書き換えたときに RBI 更新を忘れる事故を防げます。

## 段階的に導入するコツ

最後に、現実のコードベースに後から型を入れていくときのコツです。

- **ファイル単位で `# typed:` レベルを上げる**: Sorbet は `false` → `true` → `strict` → `strong` の段階があります。最初は全ファイル `# typed: false` から始め、書き直したファイルから順に `true` → `strict` に上げていけば、巨大な PR にせず徐々に型を広げられます
- **新規ファイルは最初から `strict`**: 既存コードは緩く、これから書くファイルだけ厳しく、というポリシーにすると無理なく増えます
- **`T.untyped` を恐れない**: 詰まったらいったん `T.untyped` で逃がして、後で型を絞っていく、で OK。100% 厳密を最初から目指さない

## まとめ

- **Sorbet** は Ruby の静的型チェッカ、**Tapioca** は Rails や gem の動的メソッドを RBI として書き出すツール。Rails で Sorbet を使うなら Tapioca はほぼ必須
- 入出力を `T::Struct` で固めると、コントローラのレスポンス型が「コードに書いてある」状態になる
- `ruby-lsp` と組み合わせると、エディタ上で「型違反の警告」「型を見た補完」「nil 安全」が手に入る
- Rails 側のスキーマ変更時は `bin/tapioca dsl` の再生成を忘れずに（CI で検査するのが定番）
- 段階的導入が前提。新規ファイルは `strict`、既存は `false` から徐々に、で十分実用になる

「Ruby はとりあえず動かしてから直す言語」という雰囲気から一段抜けて、**書いている時点で間違いに気づける環境**を Rails でも作れます。型ありの安心感を Ruby でも、というのは思っていたよりずっと現実的です。
