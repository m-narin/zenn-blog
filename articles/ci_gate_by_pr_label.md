---
title: "特定のラベルが付いたPRだけCIを走らせてGitHub Actionsのコストを削減する"
emoji: "🚦"
type: "tech"
topics:
  - GitHubActions
  - CI
  - CICD
  - DevOps
published: true
---

## はじめに

GitHub Actions のコストは、開発が活発になるほど無視できない金額になっていきます。特に、

- 大量の `git push` のたびに重い test/build/e2e が走る
- WIP 中のドラフト PR でも CI が回り続ける
- typo 修正のような軽微な変更でも全 CI が走る

このあたりが効いてくると、月のコストが従量課金でどんどん積み上がります。

そこで、**「特定のラベルが付いた PR のときだけ CI を走らせる」という仕組み**を考えます。たとえば `Activate CI` というラベルを付けたときだけ、本格的なテスト群が動く、という設計です。

この記事では、その仕組みを GitHub Actions の reusable workflow として実装するパターンを紹介します。

## 実現したい挙動

1. PR に `Activate CI`（名前は何でもよい）ラベルが付いていれば、CI を走らせる
2. ラベルが付いていなければ、重いジョブは一切走らない
3. あとからラベルを付けたら、その時点で CI が走り出す
4. ラベル付け替えのたびに CI が二重に走らないようにする

ポイントは **3 と 4** です。「ラベル付与で発火できる」ようにしつつ、「すでに通っているのに付け替えで再実行する」を防ぎたい。

## ステップ 1: CI 本体のトリガーに `labeled` を加える

まずは CI 本体のワークフロー側で、ラベル付与イベントでもワークフローが発火するようにします。

```yaml
name: Backend CI

on:
  pull_request:
    types: [opened, synchronize, reopened, labeled]

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
```

GitHub Actions の `pull_request` トリガーはデフォルトで `opened` / `synchronize` / `reopened` の 3 種のイベントで発火しますが、これに **`labeled` を明示的に追加する**ことで、後からラベルを付けた瞬間にもワークフローが起動するようになります。

これがないと「ラベル付け忘れで PR を作成 → あとからラベルを付ける」というフローのときに CI が動きません。

## ステップ 2: ラベル判定を reusable workflow に切り出す

ラベル判定そのものは backend / frontend / e2e など複数の CI で使い回したいので、reusable workflow にしておくとスッキリします。

ファイルは `.github/workflows/reusable-ci-label-check.yaml` として作ります。

```yaml
name: Reusable CI Label Check

on:
  workflow_call:
    inputs:
      required_jobs:
        description: "再実行スキップ判定に使うジョブ名（'|' 区切り）"
        type: string
        required: true
    outputs:
      should_skip:
        description: "CIをスキップしてよいか"
        value: ${{ jobs.check.outputs.should_skip }}

jobs:
  check:
    runs-on: ubuntu-latest
    permissions:
      checks: read
      pull-requests: read
    outputs:
      should_skip: ${{ steps.skip-check.outputs.should_skip }}
    steps:
      - name: Skip if all required jobs already succeeded on this SHA
        id: skip-check
        uses: actions/github-script@v8
        with:
          script: |
            // (1) デフォルトはスキップしない
            core.setOutput('should_skip', 'false');

            // (2) ラベル付与イベント以外（普通のpush等）は通常通り走らせるので即return
            if (context.payload.action !== 'labeled') return;

            // (3) このPRのHEAD SHAに紐づくcheck runsを取得
            const { data: checkRuns } = await github.rest.checks.listForRef({
              owner: context.repo.owner,
              repo: context.repo.repo,
              ref: context.payload.pull_request.head.sha,
              status: 'completed',
              filter: 'latest',
              per_page: 100,
            });

            // (4) 呼び出し側から渡された「必要なジョブ」のリスト
            const requiredJobs = '${{ inputs.required_jobs }}'.split('|').map(s => s.trim());

            // (5) 各ジョブが既に success/skipped で完了しているかを判定
            const jobsResults = requiredJobs.map(job => {
              const matching = checkRuns.check_runs.filter(run => run.name === job);
              if (matching.length === 0) return false;
              const latest = matching[0];
              return latest.conclusion === 'success' || latest.conclusion === 'skipped';
            });

            // (6) 1つでも未完了/失敗があれば走らせる
            if (jobsResults.length === 0) return;

            // (7) 必要なジョブが全部既に通っているなら、再実行スキップ
            if (jobsResults.every(Boolean)) {
              core.setOutput('should_skip', 'true');
            }

      - name: Check required label
        if: steps.skip-check.outputs.should_skip != 'true'
        uses: mheap/github-action-required-labels@v5
        with:
          mode: minimum
          count: 1
          labels: "Activate CI"
```

このワークフローのポイントを順に見ていきます。

### `permissions` で必要な権限だけ付与する

`github-script` の中で `checks.listForRef` を呼んで、PR の HEAD SHA に紐づく check runs（過去ジョブの実行結果）を取得しています。

ジョブに `permissions:` ブロックを書いて、必要最小限の権限を付けます。

```yaml
permissions:
  checks: read
  pull-requests: read
```

- `checks: read`: `checks.listForRef` のため
- `pull-requests: read`: `mheap/github-action-required-labels` が PR のラベルを読むため

`actions/github-script` は `github-token` を渡さなければ自動的に `secrets.GITHUB_TOKEN` を使うので、これだけで動きます。組織のポリシーで `GITHUB_TOKEN` の権限が絞られている場合のみ、別途 PAT を用意して `github-token` に渡してください。

### `skip-check` ステップの中身

skip-check ステップは「ラベルを付け替えただけのときに、既に緑だった CI を**もう一度走らせない**」ためのガードです。スクリプトは順番に次の判定をしています。

- **(1)** デフォルトでは `should_skip=false`（= CI を走らせる）
- **(2)** イベントの種類が `labeled` 以外（通常の push 等）なら、ここで処理終了。普通通り CI を走らせます
- **(3)** PR の HEAD SHA に対する、過去の check runs（=過去のジョブ実行結果）を取得
- **(4)** 呼び出し側から `required_jobs` で渡された、判定対象のジョブ名を `|` 区切りで配列化
- **(5)** 各ジョブについて、最新の実行結果が `success` または `skipped` かを確認
- **(6)** 過去の実行履歴が 1 件もない（= 初めてのラベル付与）なら、普通に CI を走らせる
- **(7)** 必要なジョブが**全部既に通っている**なら `should_skip=true` を出力 → 後続ジョブはまるごとスキップ

これによって、たとえばレビュー中にラベルを外して付け直す、別のラベルを付け直す、といった操作で「すでに緑の CI がもう一度フル実行される」というお金の無駄遣いを防げます。

### `Check required label` ステップ

skip-check で `should_skip != 'true'` だった場合（= 普通の push、または初めてのラベル付与）に、`mheap/github-action-required-labels@v5` で **`Activate CI` ラベルが付いているか**を判定します。

付いていなければここでジョブが失敗します。「失敗する」というのが重要で、後続の重いジョブはこのジョブを `needs` で待つようにしておくので、ここで止まれば連鎖的に動かなくなる、という設計です。

なお `mode: minimum` `count: 1` にしているので「`Activate CI` が **最低 1 つ**付いていれば OK」になります。他のラベル（`bug`, `enhancement` 等）が同時に付いていても問題ありません。

## ステップ 3: 呼び出し側でゲートする

最後に、本物の CI ワークフロー（backend / frontend など）から、この reusable workflow を呼び出します。

```yaml
jobs:
  ci-label-check:
    uses: ./.github/workflows/reusable-ci-label-check.yaml
    with:
      required_jobs: "ci-label-check / check|test|lint"

  test:
    needs: ci-label-check
    if: needs.ci-label-check.outputs.should_skip != 'true'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5
      # 重いテスト処理...

  lint:
    needs: ci-label-check
    if: needs.ci-label-check.outputs.should_skip != 'true'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5
      # Lint / 型チェック等...
```

### なぜ `ci-label-check` がゲートになるのか？

GitHub Actions の `needs:` キーワードは「依存関係」を表すと同時に「**待ち合わせと前提条件**」も意味します。あるジョブ A が `needs: B` を持っているとき、B が失敗 / skip されると **A は実行されません**（明示的に `if: always()` などを付けない限り）。

つまり今回の設計では:

- `ci-label-check` が失敗（ラベル無し）→ 後続の `test` / `lint` は **GitHub Actions のランナーがそもそも起動しない**。これがコスト削減の本丸
- `ci-label-check` が成功して `should_skip=true` → ランナーは一瞬起動するが、`if` の評価で即終了し、重い処理は実行されない
- `ci-label-check` が成功して `should_skip=false` → 通常通り全部走る

「ラベル判定ジョブを共通の入口に置く」ことで、後続のどんなに重いジョブも、たった 1 行（`needs:` と `if:`）追加するだけでまとめてゲートできる、というのが綺麗なところです。

### `required_jobs` には何を入れる？

reusable workflow に渡す `required_jobs` には、**「これらが全部既に通っていれば再実行を省略してよい」というジョブ名のリスト**を `|` 区切りで指定します。

- 自分自身（ラベル判定ジョブ）: `ci-label-check / check`
- 本来走らせたい重いジョブ: `test`, `lint`, ...

これらが**過去の実行で全部 `success`/`skipped` になっている**ときだけ「もう走らせなくていいよね」と判断する仕組みです。

ジョブ名は GitHub Actions の UI で表示される check run 名と一致させる必要があります（reusable workflow 経由のジョブは `<呼び出し側のジョブ名> / <reusable側のジョブ名>` という形式になる点に注意）。

## draft PR ではどうなる？

「draft のうちは CI を回したくない」というのはよくあるニーズですが、ここは少し注意が必要です。

GitHub Actions の `pull_request` トリガーは **draft / ready を区別しません**。`opened` も `synchronize` も `labeled` も、draft 状態の PR で普通に発火します。なので素朴に組むと「draft でも CI が走る」状態になります。

ところが今回のラベル方式だと、**実質的にラベルで二段階のゲートがかかる**ので、ここがうまく効きます。

- **draft + ラベル無し** → ラベル判定で落ちるので、重いジョブは走らない（コスト 0）
- **draft + ラベル有り** → 走る

つまり「draft のうちは CI を回したくない」なら、**ラベルを付けなければいいだけ**で自然に達成できます。これはラベル方式の地味に嬉しい副作用です。

それでも「ラベルが付いていても draft の間は絶対に走らせたくない」と明示したい場合は、ゲートに draft 判定を足します。

```yaml
jobs:
  ci-label-check:
    if: github.event.pull_request.draft == false
    uses: ./.github/workflows/reusable-ci-label-check.yaml
    with:
      required_jobs: "ci-label-check / check|test|lint"
```

逆に「draft → Ready に変えた瞬間に走らせたい」なら、トリガーに `ready_for_review` を足す手もあります。

```yaml
on:
  pull_request:
    types: [opened, synchronize, reopened, labeled, ready_for_review]
```

## 注意点

- `mheap/github-action-required-labels` の `mode` は `exactly` にすると「ちょうど 1 つ付いている」状態を要求するので、他のラベルが混ざっていると失敗してしまいます。多くのチームでは他ラベル（`bug` / `enhancement` 等）と併用したいはずなので、上の例のように `mode: minimum` `count: 1` にして「最低 1 つ付いていれば OK」にしておくのが扱いやすいです
- `checks.listForRef` の `per_page: 100` を超える数の check runs が同じ SHA にぶら下がっている場合、再実行スキップ判定が正しく動かない可能性があります。CI ジョブ数が多いリポではページネーション対応を検討してください

## まとめ

- `pull_request` の `types: [..., labeled]` でラベル付与時にも発火させる
- reusable workflow にラベル判定を切り出して再利用する
- 「同 SHA で既に通っていれば skip」のガードを入れて二重実行を防ぐ
- 後続ジョブは `needs` + `if: should_skip != 'true'` でゲートする

シンプルですが、CI コストにじわじわ効いてくる施策です。GitHub Actions の請求書を見て「ちょっと多いな…」と思ったときに、検討してみる価値があります。
