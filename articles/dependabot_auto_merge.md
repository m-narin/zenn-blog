---
title: "DependabotのPRを指定ライブラリだけ安全に自動マージするGitHub Actions"
emoji: "🔀"
type: "tech"
topics:
  - GitHubActions
  - Dependabot
  - CI
  - CICD
  - DevOps
published: true
---

## はじめに

Dependabot を入れると依存更新 PR が毎日のように飛んできます。1 本ずつ中身を確認して merge していくのは地味に大変で、放っておくと PR が溜まって「結局まとめて雑に merge」になりがちです。

とはいえ、**全部を無条件に自動マージするのは怖い**です。メジャーアップデートで壊れたり、production の依存が予期せぬ挙動になったりするリスクがあります。

そこで本記事では、**「これは自動でマージしても安全」と判断できるものだけを、ホワイトリストで明示的に許可して自動マージする** GitHub Actions を作ります。安全側に倒すために、次の 3 つの条件をすべて満たしたときだけ自動マージします。

1. **patch 更新であること**（`x.y.Z` の Z だけが上がる更新）
2. **依存の種類で対象を分けること**（開発用 / production 用）
3. **ホワイトリストに載っているライブラリであること**（明示的に許可したものだけ）

「広く自動化」ではなく「**狭く確実に自動化**」する方針です。

## 安全装置の考え方

自動マージは便利な反面、一度間違えると壊れたものが main に入ってしまいます。なので「何を信用するか」を絞り込むのが肝心です。

- **patch 限定**: semver 的に後方互換が保たれる前提の更新だけを対象にする。major / minor は人間がレビューする
- **依存タイプで分離**: 開発用（linter / test / デバッグツール等）は壊れても影響が局所的なので緩めに、production 依存は厳しく。最初は **production のホワイトリストを空**にしておき、運用に慣れてから少しずつ追加する
- **ホワイトリスト**: 「壊れても影響範囲が分かっている」ものだけを名指しで許可する。`*`（全部許可）にはしない

この 3 段構えで、「自動マージが暴発して main が壊れる」事故をほぼ潰せます。

## ワークフロー全体

先に全体を貼ってから、ポイントを順に解説します。

```yaml
name: Dependabot Auto Merge

on: pull_request

permissions:
  contents: write
  pull-requests: write

env:
  # 開発用依存のホワイトリスト（linter/test/デバッグツールなど、壊れても影響が局所的なもの）
  DEV_WHITELIST: "rubocop,rubocop-rspec,factory_bot_rails,simplecov,byebug"
  # production 依存のホワイトリスト（安定してから少しずつ追加する。最初は空）
  PROD_WHITELIST: ""

jobs:
  auto-merge:
    if: >-
      github.event.pull_request.user.login == 'dependabot[bot]' &&
      github.event.pull_request.head.repo.full_name == github.repository
    runs-on: ubuntu-latest
    steps:
      - name: Fetch Dependabot metadata
        id: metadata
        uses: dependabot/fetch-metadata@v2
        with:
          github-token: "${{ secrets.GITHUB_TOKEN }}"

      - name: Check whitelist and auto-merge
        run: |
          UPDATE_TYPE="${{ steps.metadata.outputs.update-type }}"
          DEP_TYPE="${{ steps.metadata.outputs.dependency-type }}"
          DEP_NAMES="${{ steps.metadata.outputs.dependency-names }}"

          # (1) patch 以外はスキップ
          if [ "$UPDATE_TYPE" != "version-update:semver-patch" ]; then
            echo "Skipping: not a patch update ($UPDATE_TYPE)"
            exit 0
          fi

          # (2) 依存タイプごとに使うホワイトリストを切り替える
          WHITELIST=""
          if [ "$DEP_TYPE" = "direct:development" ]; then
            WHITELIST="$DEV_WHITELIST"
          elif [ "$DEP_TYPE" = "direct:production" ]; then
            WHITELIST="$PROD_WHITELIST"
          else
            echo "Skipping: dependency type $DEP_TYPE is not supported"
            exit 0
          fi

          # (3) ホワイトリストが空ならスキップ
          if [ -z "$WHITELIST" ]; then
            echo "Skipping: whitelist is empty for $DEP_TYPE"
            exit 0
          fi

          # (4) 更新対象がすべてホワイトリストに含まれるか確認
          IFS=',' read -ra DEPS <<< "$DEP_NAMES"
          for dep in "${DEPS[@]}"; do
            dep=$(echo "$dep" | xargs)
            if ! echo ",$WHITELIST," | grep -F -q ",$dep,"; then
              echo "Skipping: $dep is not in whitelist"
              exit 0
            fi
          done

          # (5) すべて条件を満たしたら approve + auto-merge を有効化
          echo "All dependencies are whitelisted. Approving and enabling auto-merge."
          gh pr review --approve "$PR_URL"
          gh pr merge --auto --merge "$PR_URL"
        env:
          PR_URL: ${{ github.event.pull_request.html_url }}
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

## ポイント 1: なぜ `pull_request_target` ではなく `pull_request` でいいのか

Dependabot がらみのワークフローでは、シークレットを使いたいために `pull_request_target` を使う場面があります（外部 API キーが必要なケースなど）。しかし**この自動マージワークフローは `on: pull_request` で十分**です。理由は、**使うトークンが `GITHUB_TOKEN` だけ**だからです。

ここで Dependabot 特有の注意点があります。

> Dependabot がトリガーしたワークフローでは、`GITHUB_TOKEN` はデフォルトで **read-only** になる

これは、悪意ある依存更新が勝手に書き込みできないようにするための仕様です。ただし、**ワークフロー側で `permissions:` を明示すれば、Dependabot イベントでも必要な書き込み権限を付与できます**。

```yaml
permissions:
  contents: write       # merge するため
  pull-requests: write  # approve / auto-merge を有効化するため
```

外部のシークレット（独自の API キー等）を使わず `GITHUB_TOKEN` だけで完結するので、PR の head コードを実行することもなく、`pull_request_target` のような取り回しの難しさやシークレット漏洩リスクを避けられます。「**できるだけ `pull_request` で済ませる**」のがセキュリティ的にも素直です。

## ポイント 2: 発火条件を Dependabot の同一リポPRに絞る

```yaml
if: >-
  github.event.pull_request.user.login == 'dependabot[bot]' &&
  github.event.pull_request.head.repo.full_name == github.repository
```

- **`user.login == 'dependabot[bot]'`**: PR の作成者が Dependabot のときだけ動かす
- **`head.repo.full_name == github.repository`**: PR のブランチが同一リポジトリにあるときだけ動かす

Dependabot の PR は同一リポジトリ内のブランチから作られるので後者は必ず通りますが、「Dependabot を騙る別経路の PR」を弾く防御として入れておきます。`GITHUB_TOKEN` が read-only に絞られる `pull_request` では `pull_request_target` ほどクリティカルではないものの、**入口を狭めておくのは常に正解**です。

## ポイント 3: `dependabot/fetch-metadata` で更新内容を取得する

自動マージの判定材料は、[`dependabot/fetch-metadata`](https://github.com/dependabot/fetch-metadata) が出力してくれます。

```yaml
- name: Fetch Dependabot metadata
  id: metadata
  uses: dependabot/fetch-metadata@v2
  with:
    github-token: "${{ secrets.GITHUB_TOKEN }}"
```

このアクションは PR のタイトルやメタデータを解析して、次のような output を返します。

- `update-type`: `version-update:semver-patch` / `...:semver-minor` / `...:semver-major`
- `dependency-type`: `direct:production` / `direct:development` / `indirect`
- `dependency-names`: 更新対象のライブラリ名（カンマ区切り）

これらを使って「patch か」「開発用か production 用か」「ホワイトリストに載っているか」を判定します。

## ポイント 4: ホワイトリスト判定のロジック

スクリプトは上から順に、条件を満たさなければ `exit 0`（＝何もせず正常終了）していくフィルタ構造になっています。

- **(1) patch 以外はスキップ**: `update-type` が `version-update:semver-patch` でなければ即終了。major / minor は人間に回す
- **(2) 依存タイプでホワイトリストを切り替え**: 開発用なら `DEV_WHITELIST`、production 用なら `PROD_WHITELIST` を使う。`indirect`（間接依存）など対象外のタイプはスキップ
- **(3) ホワイトリストが空ならスキップ**: production を最初は空にしておけば、production 依存は一切自動マージされない。安全に倒すための初期設定
- **(4) 全依存がホワイトリストに含まれるか確認**: 1 つでも載っていないものがあればスキップ。`,$WHITELIST,` のように**カンマで挟んで** `grep -F` するのがポイントで、`rubocop` が `rubocop-rspec` に部分一致してしまう事故を防げます
- **(5) 全条件クリアで approve + auto-merge**: `gh pr review --approve` で承認し、`gh pr merge --auto --merge` で auto-merge を有効化する

`--auto` を付けると 即マージではなく「必須チェックが通ったら自動でマージ」になります。CI を待ってからマージされるので、「patch だけど実はテストが落ちる」ような更新は merge されません。

## 動かすための前提設定

このワークフローは、リポジトリ側の設定が揃っていないと期待どおり動きません。

- **Auto-merge を有効化**: Settings → General → 「Allow auto-merge」を ON にする。これが無いと `gh pr merge --auto` が失敗します
- **GitHub Actions による承認を許可**: Settings → Actions → General → Workflow permissions で「Allow GitHub Actions to create and approve pull requests」を ON にする。これが無いと `gh pr review --approve` が弾かれます
- **ブランチ保護で必須チェックを設定**: main に required status checks を設定しておくと、`--auto` が「CI 通過後にマージ」を保証してくれます

特に最後のブランチ保護は重要で、これがあると「ラベルで CI を回す → 通ったら patch を自動マージ」という流れが綺麗につながります（CI 側の制御は[特定のラベルが付いた PR だけ CI を走らせる](https://zenn.dev/mandenaren/articles/ci_gate_by_pr_label)の記事を参照）。

## ホワイトリスト運用のコツ

- **production は空から始める**: 最初は開発用ツールだけ自動マージし、production 依存は人間がレビューする。運用に慣れて「これは patch なら自動で良い」と確信が持てたものだけ `PROD_WHITELIST` に足していく
- **入れるのは「壊れても影響が局所的」なもの**: linter / formatter / test / デバッグ系のツールは、壊れても CI で気づけるし production に影響しない。こういうものから始めるのが安全
- **patch 限定とセットで考える**: ホワイトリストに入れても対象は patch だけ。minor / major は必ず人間が見るので、ホワイトリストの“暴発”リスクは小さい

## 注意点

- **semver を守らないライブラリもある**: patch なのに破壊的変更が混じることが稀にあります。だからこそ「patch かつホワイトリスト」の二重条件にして、対象を絞り込んでおくのが安全です
- **`exit 0` でスキップする設計**: 条件を満たさないときに `exit 1`（失敗）にすると PR に赤いチェックが付いてしまいます。「対象外なら何もせず正常終了」にしておくと、自動マージしない PR がノイズになりません
- **同一リポ判定は残す**: `head.repo.full_name == github.repository` は外さないこと。入口を狭める防御は安いコストで効きます

## まとめ

- 自動マージは「広く」ではなく「**狭く確実に**」。patch 限定 + 依存タイプ分離 + ホワイトリストの 3 段構え
- `GITHUB_TOKEN` だけで完結するので `pull_request` + `permissions:` で書き込み権限を付けるのが素直（`pull_request_target` 不要）
- `dependabot/fetch-metadata` の output で判定し、`gh pr merge --auto` で「CI 通過後マージ」にする
- production は空から始めて、信用できるものだけ少しずつホワイトリストに足していく

「Dependabot PR が溜まってつらい」「でも全自動は怖い」というチームに、ちょうどよい落としどころだと思います。
