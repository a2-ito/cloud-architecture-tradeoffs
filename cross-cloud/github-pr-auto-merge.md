# GitHub の PR をどう自動マージするか

GitHub のプルリクエストを、条件を満たしたら人の操作なしにマージする構成を比べる。
GitHub ネイティブの **auto-merge** と **merge queue**、GitHub Actions から直接マージする方法、Renovate や外部アプリに任せる方法を並べる。
使える機能はリポジトリの公開範囲（パブリックかプライベートか）と GitHub のプランで変わるため、その組み合わせごとに整理する。

本ファイルは特定のクラウドに属さない GitHub の構成のため、`cross-cloud/` に置く。

## 1. 解きたい課題と前提

PR の自動マージを検討するのは、次のような場合である。

- Dependabot や Renovate の依存更新 PR が多く、CI が通ったものを手でマージする作業が負担になっている。
- レビューの承認から CI の完了まで待ち、最後にマージボタンを押すだけの作業をなくしたい。
- マージの頻度が高く、マージ直前に main を取り込み直して CI を回し直す作業が詰まっている。

前提は次のとおりとする。

- CI は GitHub Actions で動かす。
- 対象は github.com（GitHub Enterprise Server は扱わない）。
- プランは、個人アカウント向けの GitHub Free と GitHub Pro、Organization 向けの GitHub Free、GitHub Team、GitHub Enterprise Cloud を扱う。
- 料金は執筆時点の値を使う[^pricing]。

## 2. プランと公開範囲で使える機能

自動マージの構成は、次の 3 つの機能が使えるかでほぼ決まる。

- **ブランチ保護 / ルールセット**: 「承認が N 件」「必須のステータスチェックが成功」といったマージの条件を main に課す。
- **auto-merge**: 条件を満たした時点で GitHub が PR をマージする。条件を課していないと有効にできない。
- **merge queue**: マージ待ちの PR を順に並べ、main の最新に積み重ねた状態で CI を回してからマージする。

GitHub のドキュメントに書かれた提供範囲[^gated]をまとめると、次のようになる。

| 機能 | Free（パブリック） | Free（プライベート） | Pro / Team（プライベート） | Enterprise Cloud（プライベート） |
| --- | --- | --- | --- | --- |
| ブランチ保護 / ルールセット | 可 | **不可** | 可 | 可 |
| auto-merge | 可 | **不可** | 可 | 可 |
| merge queue | Organization 所有なら可 | 不可 | **不可** | Organization 所有なら可 |
| Organization 単位のルールセット | 不可 | 不可 | 不可 | 可 |

この表から、読み落としやすい点が 3 つある。

- **Free のプライベートリポジトリでは、ネイティブの自動マージは一切使えない。** マージの条件を課せないため、auto-merge のボタンも出ない。
- **merge queue は Organization 所有のリポジトリに限られる。** 個人アカウントのリポジトリでは、パブリックでも Pro でも使えない。
- **プライベートで merge queue を使うには Enterprise Cloud が要る。** Team では auto-merge までしか使えない。

auto-merge の操作には、リポジトリの設定で **Allow auto-merge** を有効にしておく必要もある。
既定では無効である。

## 3. 構成

### A. ネイティブの auto-merge

```
PR ─▶ auto-merge を有効化（人 or ワークフロー）
        │
        ▼
     ブランチ保護 / ルールセットの条件（承認、必須チェック）を満たす
        │
        ▼
     GitHub がマージ
```

- PR の画面の **Enable auto-merge** か、`gh pr merge --auto` で有効にする。以降は、条件を満たした時点で GitHub がマージする。
- 条件の判定は GitHub が行うため、CI の完了を待つ処理を自分で書く必要がない。
- 依存更新 PR に限って自動で有効にするには、PR の作成をきっかけに動くワークフローで `gh pr merge --auto` を実行する。GitHub は Dependabot 向けにこの例を載せている[^dependabot]。

```yaml
on: pull_request

permissions:
  contents: write
  pull-requests: write

jobs:
  auto-merge:
    if: github.event.pull_request.user.login == 'dependabot[bot]'
    runs-on: ubuntu-latest
    steps:
      - id: metadata
        uses: dependabot/fetch-metadata@v2
      - if: steps.metadata.outputs.update-type == 'version-update:semver-patch'
        run: gh pr merge --auto --squash "$PR_URL"
        env:
          PR_URL: ${{ github.event.pull_request.html_url }}
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

- 承認を必須にしている場合は、承認も自動化しないとマージされない。ワークフローから `gh pr review --approve` を実行するには、リポジトリか Organization の設定で **Allow GitHub Actions to create and approve pull requests** を有効にする。PR の作成者は自分の PR を承認できないので、承認するトークンは PR を作ったものと別にする。
- write 権限のない人が PR に push すると、auto-merge は自動で無効になる。

### B. merge queue

```
PR ─▶ 条件を満たす ─▶ キューに追加（Merge when ready）
                         │
                         ▼
                      gh-readonly-queue/main/... ブランチ
                      （main の最新 + 前に並ぶ PR + この PR）
                         │ merge_group イベントで CI
                         ▼
                      成功したら main にマージ
```

- A は「PR を作った時点の main」に対する CI の結果でマージする。そのあいだに main が進むと、PR 単体では通っていても、マージ後の main で壊れることがある。merge queue は、前に並ぶ PR を積み重ねた状態で CI を回し直すため、これが起きない。
- ブランチ保護の「マージ前にブランチを最新にする（Require branches to be up to date）」を使うと同じ問題は防げるが、PR ごとに手で main を取り込み、CI を待ち直すことになる。merge queue はこの作業をなくす。
- CI のワークフローに `merge_group` イベントを追加しないと、必須チェックが報告されず、キューの PR が永遠にマージされない。
- 組み込みの `GITHUB_TOKEN` では PR をキューに追加できない[^dependabot]。ワークフローから追加するには、GitHub App のトークンか個人アクセストークンを使う。

### C. Actions のワークフローから直接マージする

```
PR ─▶ ワークフロー ─▶ 自分で CI の成功を待つ（gh pr checks --watch）
                      │
                      ▼
                   gh pr merge（--auto なし）
```

- ブランチ保護も auto-merge も使えない Free のプライベートリポジトリで、自動マージを実現する唯一のネイティブな方法である。
- マージの条件を GitHub が強制しないため、条件の判定はワークフローの中に自分で書く。CI の完了を待つ処理、待っているあいだに新しい commit が push された場合の扱い（`gh pr merge --match-head-commit <SHA>` で、確認した commit 以外ならマージしない）などを考える必要がある。
- 条件は「ワークフローが確認したかどうか」だけであり、人が手でマージボタンを押すことも止められない。保護の仕組みとしては A より弱い。

### D. Renovate の automerge

- Renovate は設定ファイルの `automerge: true` で自動マージする。既定（`platformAutomerge: true`）では、GitHub ネイティブの auto-merge を有効にするだけで、実際のマージは A と同じ仕組みで行われる。そのため、Allow auto-merge の設定が要る[^renovate]。
- `platformAutomerge: false` にすると、Renovate 自身が CI の成功を確認してマージする。ブランチ保護がない Free のプライベートリポジトリでも動く。中身は C を Renovate が代行する形である。
- `automergeType: "branch"` にすると、PR を作らずにブランチで CI を回し、通れば main に直接 push する。PR の通知が減る代わりに、main への直接 push を許す必要がある。
- 既定では、ステータスチェックが 1 つも成功していないとマージしない。テストのないリポジトリでは `ignoreTests: true` が必要になる。

### E. 外部アプリ

- Mergify などの GitHub App は、ラベルや条件の記述で自動マージし、独自のマージキューも持つ。
- ネイティブの merge queue が使えない Team のプライベートリポジトリや、個人アカウントのリポジトリで、キューの仕組みがほしいときの選択肢になる。
- ソースコードへの書き込み権限を第三者の App に渡すため、社内のセキュリティ審査が必要になる。料金はプライベートリポジトリでは有料のことが多い。

## 4. 公開範囲とプランごとの組み合わせ

前節の構成を、どのリポジトリで使えるかに当てはめると次のようになる。

| リポジトリ | A: auto-merge | B: merge queue | C: Actions で直接 | D: Renovate | E: 外部アプリ |
| --- | --- | --- | --- | --- | --- |
| 個人・Free・パブリック | 可 | 不可 | 可 | 可 | 可 |
| 個人・Free・プライベート | 不可 | 不可 | 可 | `platformAutomerge: false` で可 | 可 |
| 個人・Pro・プライベート | 可 | 不可 | 可 | 可 | 可 |
| Org・Free・パブリック | 可 | 可 | 可 | 可 | 可 |
| Org・Free・プライベート | 不可 | 不可 | 可 | `platformAutomerge: false` で可 | 可 |
| Org・Team・プライベート | 可 | 不可 | 可 | 可 | 可 |
| Org・Enterprise Cloud・プライベート | 可 | 可 | 可 | 可 | 可 |

パブリックリポジトリには、プライベートにはない注意点がある。

- フォークから来た PR で `pull_request` イベントのワークフローが動くとき、`GITHUB_TOKEN` は読み取り専用になり、シークレットも渡されない。そのため、フォークからの PR に対してワークフローから auto-merge を有効にすることはできない。
- これを回避するために `pull_request_target` を使い、フォークのコードを書き込み権限付きで動かすのは危険である。フォークからの PR の自動マージは、外部の人が書いたコードを main に入れることと同じなので、そもそも避ける。対象は Dependabot や Renovate など、リポジトリ内のブランチから作られる PR に絞る。

## 5. トレードオフ

### 安全性

| 観点 | A | B | C | D（platformAutomerge: false） | E |
| --- | --- | --- | --- | --- | --- |
| マージ条件を GitHub が強制する | する | する | しない | しない | 構成次第 |
| 手動のマージを止められる | 止められる | 止められる | 止められない | 止められない | 構成次第 |
| マージ後の main で壊れる可能性 | あり | なし | あり | あり | キューを使えばなし |

A と B は、条件を満たさない PR を誰もマージできない。
C と D（platformAutomerge: false）は、ワークフローや Renovate が「確認してからマージする」だけで、確認を通らないマージを止める仕組みはない。

### トークンの扱い

`GITHUB_TOKEN` が起こしたイベントでは、新しいワークフローの実行が起動しない（`workflow_dispatch` と `repository_dispatch` を除く）。
`GITHUB_TOKEN` でマージした結果の main への push でも、`on: push` のデプロイなどが起動しない。
`GITHUB_TOKEN` で auto-merge を有効にした場合も、同じく後続のワークフローが動かないという報告がある。
マージ後にワークフローを動かしたいなら、マージや auto-merge の有効化に GitHub App のトークンを使う。

### コスト

| 構成 | 追加の費用 |
| --- | --- |
| A | プライベートなら Pro（個人、4 USD/月）か Team（4 USD/ユーザー/月） |
| B | プライベートなら Enterprise Cloud（21 USD/ユーザー/月）。キューで CI をもう一度回すため、Actions の実行分数も増える |
| C、D | なし（Actions の実行分数のみ） |
| E | サービスの料金（プライベートでは有料のことが多い） |

プライベートで B を使うために Team から Enterprise Cloud に上げると、ユーザーあたりの費用は約 5 倍になる。
merge queue のためだけに上げるのは割に合わないことが多く、E の外部アプリのほうが安く済む場合がある。

### 運用負荷

1. A: ブランチ保護と Allow auto-merge を設定し、必要なら有効化のワークフローを置くだけ。
2. D: Renovate の設定ファイルに書くだけ。
3. B: A に加えて、CI を `merge_group` に対応させ、キューのサイズやタイムアウトを調整する。
4. E: App の設定と契約、権限の審査。
5. C: 条件の判定とエッジケースの処理を自分で書いて保守する。

### ロックイン

- A と B は GitHub の機能であり、GitHub から動かない限りロックインは問題にならない。
- E は App 固有の設定ファイル（`.mergify.yml` など）に依存する。外すときは A か B に書き直す。

## 6. 選ぶ基準

A（ネイティブの auto-merge）が向くのは次のような場合である。

- パブリックリポジトリか、Pro / Team 以上のプライベートリポジトリである。
- 依存更新 PR の自動マージや、承認後のマージ待ちをなくしたい。
- マージの頻度がそれほど高くなく、main が進んで壊れる事故がまれである。

B（merge queue）が向くのは次のような場合である。

- Organization 所有のパブリックリポジトリか、Enterprise Cloud のプライベートリポジトリである。
- 1 日に多数の PR がマージされ、PR 単体の CI が通ってもマージ後の main で壊れることがある。

C（Actions で直接マージ）が向くのは次のような場合である。

- Free のプライベートリポジトリで、有料プランに上げずに自動マージしたい。
- マージの条件を GitHub が強制しないことを許容できる（個人や少人数のリポジトリなど）。

D（Renovate）が向くのは次のような場合である。

- 自動マージしたいのが依存更新 PR だけである。
- Free のプライベートリポジトリでは、C を自分で書くより `platformAutomerge: false` のほうが手間が少ない。

E（外部アプリ）が向くのは次のような場合である。

- Team のプライベートリポジトリや個人アカウントのリポジトリで、マージキューがほしい。
- 第三者の App に書き込み権限を渡すことを社内で許容できる。

[^pricing]: 料金は変わることがある。最新の値は [GitHub の料金](https://github.com/pricing) を参照。
[^gated]: [Managing auto-merge for pull requests in your repository](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/configuring-pull-request-merges/managing-auto-merge-for-pull-requests-in-your-repository)、[Managing a merge queue](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/configuring-pull-request-merges/managing-a-merge-queue)、[About protected branches](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches)、[About rulesets](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/about-rulesets) の提供範囲の記載を参照。
[^dependabot]: [Automating Dependabot with GitHub Actions](https://docs.github.com/en/code-security/dependabot/working-with-dependabot/automating-dependabot-with-github-actions) を参照。
[^renovate]: [Renovate の Automerge configuration and troubleshooting](https://docs.renovatebot.com/key-concepts/automerge/) を参照。
