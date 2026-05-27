---
name: write-zenn-article
description: Zenn記事を新規作成し、push（GitHub連携経由でZennへ自動公開）までを一貫して実行する。task.md/memo.mdなどの素材を一般化して記事化し、npx zenn new:articleで雛形を作り、commit/pushする。
---

# Zenn記事作成スキル

`task.md` / `memo.md` などの素材を元に Zenn 記事を新規作成し、公開までを一気通貫で実行する。

## 引数

- 任意。引数なしで呼び出された場合は、リポジトリ直下の `task.md` を読んで指示に従う。
- 引数で素材ファイルやテーマが渡された場合はそれを優先する。

## 前提

- リポジトリは Zenn CLI ベースのブログ（`zenn-cli` が `package.json` に入っている）
- Zenn は GitHub 連携で同期する構成: master/main ブランチへの push 後、Zenn 側が自動で取り込んで公開する（GitHub Actions による相互コミットは発生しない）
- `README.md` の手順に従う:
  - 新規作成: `npx zenn new:article`（`--slug <slug>` でファイル名を指定可）
  - プレビュー: `npx zenn preview`
- 記事は `articles/` 配下に配置される（Qiita CLI の `public/` ではない）

## 手順

### 1. 素材の確認

- `task.md` を読み、指示内容（タイトル候補、参照する素材、配置先など）を把握する
- 素材として `memo.md` や `sample-project/` 配下が指定されていれば、それらも読む
- 既存記事のフォーマット（フロントマター、`topics` の粒度、文体）を `articles/` 配下の最近の記事 1〜2 本から確認する

### 2. 一般化が必要な箇所を洗い出す

社内 LT などの素材を記事化する場合、以下を一般化する:

- 社名・プロダクト名・チーム名（例: 固有プロダクト名 → 「あるプロジェクト」）
- 特徴的なネーミングを含む環境名・ブランチ名・チャネル名
- 具体的な実行回数・件数などの統計値（推測されうるもの）
- 内部 Slack チャネル名、内部 URL
- メンバー名・個人を特定しうる情報

技術的な内容（コード例、設定例、構文）は教育的価値があるので残す。

判断に迷うものは `AskUserQuestion` でユーザーに確認する。

### 3. タイトルとスラッグを決定

- タイトルは記事内容を端的に表すものを提案
- 既存記事のタイトル傾向に合わせる
- スラッグ（ファイルベース名）はスネークケースの英数字 12〜50 文字（例: `sql_in_technique`, `google_slides_progress_bar`）
- Zenn のスラッグ規約: 半角英小文字 (`a-z`)、数字 (`0-9`)、ハイフン (`-`)、アンダースコア (`_`) のみ、12〜50 文字
- 複数案がある場合は `AskUserQuestion` でユーザーに選んでもらう

### 4. Zenn CLI で雛形を作成

```bash
npx zenn new:article --slug <スラッグ>
```

- `articles/<スラッグ>.md` が生成される
- 生成された雛形のフロントマターは以下のような形:

```yaml
---
title: ""
emoji: "😀"
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: []
published: false
---
```

### 5. 記事本文の作成

- `Write` ツールで雛形を上書きする前に、必ず一度 `Read` する（Write の制約）
- フロントマターを更新:
  - `title`: 決定したタイトル（ダブルクオートで囲む）
  - `emoji`: 記事内容に合う絵文字を 1 つ（既存記事の選び方を参考に）
  - `type`: 技術記事なら `"tech"`、ポエム/アイデア記事なら `"idea"`
  - `topics`: 5 個程度。既存記事のトピック粒度に合わせる（例: `SQL`, `Python`, `ChatGPT`）
  - `published`: 公開するなら `true`
  - `published_at`: 公開予約したい場合のみ `"YYYY-MM-DD HH:MM"` 形式で指定（任意）
- 本文は素材を再構成して書く。素材の構造に引きずられすぎず、読者目線で読みやすい流れにする

### 6. ユーザー確認

- 記事を書いたら、`npx zenn preview` でローカルプレビューできることをユーザーに伝え、内容確認を促す
- 修正指示があれば反映し、commit/push の許可を得る

### 7. Commit & Push

`git add` は対象ファイルを明示的に指定する（`memo.md` `task.md` `sample-project/` などをうっかり含めないため）。

```bash
git add articles/<スラッグ>.md
git commit -m "<コミットメッセージ>"
git push
```

コミットメッセージは既存のスタイル（日本語、短め）に合わせる。**Co-Authored-By トレーラーは付けない**（ユーザー設定）。

### 8. 公開確認

Zenn は GitHub 連携によって push 後に自動で記事を取り込む（CI 経由で id/updated_at がリポジトリに書き戻されることはない）。push 完了をもって作業は完了。

- Qiita CLI のように `npx zenn pull` を待つ必要はない
- Zenn 側への反映は数十秒〜数分のラグがあるため、ユーザーが Zenn 上で確認する想定

### 9. 完了報告

- 作成したファイルパス（`articles/<スラッグ>.md`）
- 公開予定の状態（`published: true/false`、`published_at` の有無）
- Zenn 側で記事が反映されるまで少しラグがある旨

を報告して終了する。

## 既存記事の修正フロー

新規作成ではなく既存記事を修正する場合も、編集後は同じ「commit → push」のフローを実行する。タイトル変更でも本文修正でも同様。push すれば Zenn 側が自動的に更新を取り込む。

## アンチパターン

- `git add .` や `git add -A` で広く追加する（作業用ファイルが混入する）
- `Co-Authored-By` トレーラーを commit に付ける
- `npx zenn pull` のような存在しないコマンドを実行する（Zenn CLI には pull はない）
- `public/` 配下に記事を作る（Zenn は `articles/` を使う）
- フロントマターの `type` を tech/idea 以外にする
- 素材の固有名詞を残したまま公開する
- 既存記事のタイトル傾向や `topics` の粒度を無視する
