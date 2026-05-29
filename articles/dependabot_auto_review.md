---
title: "DependabotのPRをClaude Codeに自動レビューさせるGitHub Actions"
emoji: "🤖"
type: "tech"
topics:
  - GitHubActions
  - Dependabot
  - ClaudeCode
  - CICD
  - AI
published: true
---

## はじめに

Dependabot を入れていると、依存パッケージの更新 PR が毎日のように飛んできます。ありがたい反面、1 本 1 本まじめにレビューしようとすると、

- そもそも何のライブラリで、自分たちのコードのどこで使っているのか
- CHANGELOG / リリースノートに破壊的変更はないか
- メジャー / マイナー / パッチのどれで、どれくらいリスクがあるのか

を毎回手で調べることになり、地味に消耗します。結果、「とりあえずまとめて merge」か「放置して溜まる」のどちらかになりがちです。

そこで本記事では、**Dependabot が PR を作ったら、Claude Code に一次レビューを自動で書かせて PR にコメントさせる** GitHub Actions を作ります。[`anthropics/claude-code-action`](https://github.com/anthropics/claude-code-action) を使い、「ライブラリ概要・更新内容・破壊的変更・使用箇所・リスク評価・推奨アクション」を決まったフォーマットで投稿させる、という構成です。

人間のレビューを置き換えるものではなく、**「最終判断の前の調べ物」を肩代わりさせる**のが狙いです。

## 完成形の挙動

1. Dependabot が依存更新 PR を open する
2. ワークフローが発火し、Claude Code が PR の diff・本文・コードベースを調べる
3. 構造化されたレビュー結果が PR コメントとして自動投稿される

実際に投稿されるコメントのイメージはこんな感じです。

```markdown
### 📦 ライブラリ概要
foo は HTTP クライアントライブラリで、本プロジェクトでは API 呼び出し全般に利用しています。

### 📝 更新内容
v1.2.0 → v1.3.0。リトライ処理のデフォルト挙動が変更されています。

### ⚠️ 破壊的変更
なし（デフォルト値の変更のみ、明示的に指定していれば影響なし）

### 📍 使用箇所
- src/api/client.ts
- src/api/retry.ts

### 🔍 リスク評価
Low（パッチ相当の変更で、利用箇所も限定的）

### ✅ 推奨アクション
マージ推奨。リトライ間隔をデフォルト依存にしている場合のみ挙動を確認。
```

## ワークフロー全体

先に全体を貼ってから、ポイントを順に解説します。

```yaml
name: Dependabot PR Review

on:
  pull_request_target:
    types: [opened]

permissions: {}

jobs:
  review:
    if: >-
      github.actor == 'dependabot[bot]' &&
      github.event.pull_request.head.repo.full_name == github.repository
    permissions:
      contents: read
      pull-requests: write
      issues: write
    runs-on: ubuntu-latest
    timeout-minutes: 15
    steps:
      - uses: actions/checkout@v5
        with:
          ref: ${{ github.event.pull_request.head.sha }}

      - uses: anthropics/claude-code-action@v1
        if: env.ANTHROPIC_API_KEY != ''
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        with:
          anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
          github_token: ${{ secrets.GITHUB_TOKEN }}
          allowed_bots: "dependabot[bot]"
          prompt: |
            あなたはDependabotが作成した依存関係更新PRのレビュアーです。
            以下の手順でレビューし、日本語で結果をPRコメントとして投稿してください。

            ## レビュー手順

            1. `gh pr diff ${{ github.event.pull_request.number }}` で変更内容を確認
            2. `gh pr view ${{ github.event.pull_request.number }}` でPR本文のリリースノートを確認
            3. PR本文の情報が不足している場合（「truncated」と表示されている等）、WebFetchでライブラリのGitHubリリースページやCHANGELOGを確認
            4. コードベース内での使用箇所をGrep/Globで特定

            ## レビュー観点

            1. **ライブラリ概要**: 何のライブラリで、このプロジェクトでどう使われているか
            2. **更新内容**: バージョン差分とCHANGELOG/リリースノートの要約
            3. **破壊的変更**: Breaking Changesの有無と影響範囲
            4. **使用箇所**: コードベース内での使用箇所の特定
            5. **リスク評価**: High/Medium/Lowで評価
            6. **推奨アクション**: マージ可否の推奨と、手動確認が必要な場合はその内容

            ## 出力

            以下のフォーマットでレビュー結果を作成し、`gh pr comment ${{ github.event.pull_request.number }} --body-file <一時ファイル>` で投稿してください。

            ### 📦 ライブラリ概要
            （ライブラリの説明と用途）

            ### 📝 更新内容
            （バージョン差分とCHANGELOGの要約）

            ### ⚠️ 破壊的変更
            （ありの場合は詳細、なしの場合は「なし」）

            ### 📍 使用箇所
            （ファイルパスのリスト）

            ### 🔍 リスク評価
            （High/Medium/Low と理由）

            ### ✅ 推奨アクション
            （マージ推奨/手動確認必要 と具体的な確認内容）
          claude_args: |
            --allowedTools "Bash(gh pr comment:*),Bash(gh pr diff:*),Bash(gh pr view:*),Read,Grep,Glob,WebFetch"
            --max-turns 30
```

## ポイント 1: なぜ `pull_request` ではなく `pull_request_target` なのか

ここがこのワークフローで一番大事なところです。

普通の CI なら `on: pull_request` を使いますが、**Dependabot が作った PR で `pull_request` を使うと、`GITHUB_TOKEN` は read-only になり、シークレットも基本的に渡ってきません**。これは「フォークや bot からの PR が、勝手にシークレットを盗んだり書き込んだりできないようにする」ための GitHub のセキュリティ仕様です。

しかし今回は、

- `ANTHROPIC_API_KEY`（Claude を呼ぶための API キー）をシークレットから読みたい
- レビュー結果を PR に**コメントする＝書き込み権限**が欲しい

ので、read-only では困ります。

そこで `pull_request_target` を使います。これは **PR のコードではなく、ベースリポジトリ（マージ先ブランチ）のコンテキストで動く**トリガーで、**シークレットへのアクセスと read/write トークンが使えます**。

```yaml
on:
  pull_request_target:
    types: [opened]
```

`types: [opened]` に絞っているのは、PR が作られた最初の 1 回だけレビューすれば十分だからです（commit のたびにレビューを投げると API コストがかさみます）。

## ポイント 2: `pull_request_target` の危険性と、それを潰すガード

`pull_request_target` は強力なぶん危険です。**シークレットと書き込み権限を持った状態で動く**ため、もし PR の head コード（＝外部から持ち込まれたコード）をうかつに実行すると、シークレットを抜き取られる典型的な脆弱性につながります。

これを潰しているのが、ジョブ冒頭の `if` 条件とトップレベルの `permissions: {}` です。

```yaml
permissions: {}    # まずデフォルトで全部の権限を剥奪

jobs:
  review:
    if: >-
      github.actor == 'dependabot[bot]' &&
      github.event.pull_request.head.repo.full_name == github.repository
    permissions:     # 必要な権限だけジョブに付け直す
      contents: read
      pull-requests: write
      issues: write
```

それぞれの意味は次の通りです。

- **`permissions: {}`（トップレベル）**: ワークフロー全体のデフォルト権限を空にする。そのうえでジョブ単位で必要最小限だけ付け直す。「デフォルト全許可」を避ける定石です
- **`github.actor == 'dependabot[bot]'`**: 実行者が Dependabot のときだけ動かす。人間や他の bot の PR では発火させない
- **`github.event.pull_request.head.repo.full_name == github.repository`**: **PR のブランチが同じリポジトリにあるときだけ動かす**。Dependabot の PR は同一リポジトリ内のブランチから作られるのでここを通過しますが、フォークから「自分は Dependabot だ」と詐称してくる PR はここで弾けます。`pull_request_target` でシークレットを扱うときの最重要ガードです
- **ジョブの `permissions`**: コメント投稿に必要な `pull-requests: write` / `issues: write` と、コードを読むための `contents: read` だけを許可

さらに、後述の `claude_args` で **Claude が使えるツールを読み取り系＋PR コメントだけに制限**しているので、「更新された依存パッケージそのものを実行する」ような操作は行いません。

## ポイント 3: head を明示的に checkout する

`pull_request_target` はベースブランチのコンテキストで動くため、`actions/checkout` をそのまま使うと**マージ先のコード**がチェックアウトされ、PR の変更内容が手元に来ません。レビュー対象である PR の差分を見るために、head の SHA を明示的に指定します。

```yaml
- uses: actions/checkout@v5
  with:
    ref: ${{ github.event.pull_request.head.sha }}
```

ここで「外部コードを持ってくる」ことになるので、ポイント 2 の同一リポジトリ判定がいっそう効いてきます。Dependabot の同一リポジトリ PR に限定しているからこそ、安心して head を取得できます。

## ポイント 4: `claude-code-action` の設定

```yaml
- uses: anthropics/claude-code-action@v1
  if: env.ANTHROPIC_API_KEY != ''
  env:
    ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
  with:
    anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
    github_token: ${{ secrets.GITHUB_TOKEN }}
    allowed_bots: "dependabot[bot]"
    prompt: |
      （レビュー指示）
    claude_args: |
      --allowedTools "Bash(gh pr comment:*),Bash(gh pr diff:*),Bash(gh pr view:*),Read,Grep,Glob,WebFetch"
      --max-turns 30
```

それぞれの勘所です。

- **`if: env.ANTHROPIC_API_KEY != ''`**: API キーが未設定の環境では action をスキップする。キーを登録していない fork や、設定前のリポジトリでワークフローが落ちないための安全弁
- **`allowed_bots: "dependabot[bot]"`**: `claude-code-action` はデフォルトで bot が作ったイベントに反応しないことがあるため、Dependabot の PR に対して動けるよう明示的に許可する
- **`claude_args` の `--allowedTools`**: Claude が使えるツールをホワイトリストで絞る。ここでは「`gh pr diff` / `gh pr view`（PR を読む）」「`gh pr comment`（結果を投稿する）」「`Read` / `Grep` / `Glob`（コードベース調査）」「`WebFetch`（CHANGELOG 等の取得）」だけを許可。**任意のシェルコマンドは実行させない**のがセキュリティ上重要
- **`--max-turns 30`**: 暴走・コスト超過を防ぐためのターン数上限

## ポイント 5: レビュー内容を決めるプロンプト

レビューの質はプロンプト次第です。上のワークフローの `prompt` のように、**「レビュー手順」「レビュー観点」「出力フォーマット」を分けて**書くと、毎回ブレない構造化レビューになります。

- **手順**: `gh pr diff` / `gh pr view` で差分とリリースノートを確認 → 不足していれば WebFetch で CHANGELOG を取得 → Grep/Glob で使用箇所を特定、という調べ方の順序を明示する
- **観点**: ライブラリ概要・更新内容・破壊的変更・使用箇所・リスク評価・推奨アクションの 6 点を必ず埋めさせる
- **出力**: セクション見出し付きの Markdown フォーマットを固定する

最後に投稿方法のコツが 1 つあります。レビュー本文には改行・バッククォート・特殊文字が多く含まれるので、`gh pr comment --body "..."` で本文を直接渡すとシェルのエスケープで壊れがちです。**一時ファイルに書き出してから `--body-file` で投稿**させると安全です。

## Dependabot 側の設定（おまけ）

このワークフローは「Dependabot が PR を作る」ことが前提なので、`.github/dependabot.yml` 側の設定も軽く触れておきます。最小構成はこれだけです。

```yaml
version: 2
updates:
  - package-ecosystem: npm
    directory: "/"
    schedule:
      interval: daily
    open-pull-requests-limit: 10
    labels:
      - "dependencies"
```

`labels:` で付けたラベルは、別の仕組み（[特定のラベルが付いた PR だけ CI を走らせる](https://zenn.dev/mandenaren/articles/ci_gate_by_pr_label)ようなゲート）と組み合わせると、「Dependabot PR には自動でラベルを付けて、CI もレビューも自動で回す」という連携が作れます。

## 注意点

- **コストに上限を設ける**: `types: [opened]` で発火回数を絞り、`--max-turns` でターン数を制限しておく。Dependabot の PR は数が出るので、commit ごとに発火させると API 課金が積み上がります
- **`pull_request_target` のガードは削らない**: 同一リポジトリ判定（`head.repo.full_name == github.repository`）と `actor` 判定は、シークレット保護の生命線です。「とりあえず外す」は厳禁
- **AI レビューは一次情報**: あくまで「調べ物の自動化」であり、最終的な merge 判断は人間が行う前提で運用するのが安全です
- **`allowedTools` は最小に**: 任意のシェル実行を許可すると、`pull_request_target` の文脈では事故りやすくなります。読み取り＋コメントだけに絞るのがおすすめです

## まとめ

- Dependabot PR の一次レビューは `claude-code-action` で自動化できる
- シークレットと書き込みが要るので `pull_request_target` を使う。ただし **`permissions: {}` + actor 判定 + 同一リポジトリ判定**でガードする
- head SHA を明示 checkout してレビュー対象の差分を取得する
- `--allowedTools` と `--max-turns` で権限とコストを絞る
- プロンプトで「手順・観点・出力フォーマット」を構造化すると、毎回安定したレビューになる

「Dependabot PR が溜まって放置気味」「更新内容を毎回調べるのがだるい」というチームには、一次レビューの自動化はかなり効きます。
