# GitHub Actions のジョブをどこで動かすか

GitHub Actions のジョブを実行するランナーの置き場所を比べる。
GitHub が用意する **GitHub-hosted runner** と、自分のクラウドアカウントで動かす **セルフホストランナー** の各方式を並べる。

本ファイルは AWS、Google Cloud、GitHub にまたがるため、`cross-cloud/` に置く。

## 1. 解きたい課題と前提

GitHub-hosted runner は設定なしで使えるが、次のような理由でほかの置き場所を検討することがある。

- 実行分数が増えて、GitHub-hosted runner の料金が無視できなくなった。
- VPC 内のデータベースやプライベートなエンドポイントに、ジョブから直接アクセスしたい。
- AWS や Google Cloud へのデプロイに、クラウド側の IAM ロールやサービスアカウントをそのまま使いたい。

前提は次のとおりとする。

- 対象は Linux のジョブとする。macOS と Windows は扱わない。
- 対象リポジトリはプライベートとする。GitHub は、セルフホストランナーをパブリックリポジトリで使わないよう推奨している（フォークからのプルリクエストで任意のコードが実行されうるため）。
- 料金は執筆時点の us-east-1（AWS）と Tier 1 リージョン（Google Cloud）の単価を使う[^price]。

## 2. 構成

### A. GitHub-hosted runner

```
GitHub ─▶ GitHub が用意する VM（ジョブごとに新規作成）
```

- `runs-on: ubuntu-latest` と書くだけで使える。ジョブごとに新しい VM が用意され、終われば破棄される。
- Docker、主要な言語ランタイム、各種 CLI がはじめから入っている。
- VPC 内のリソースに届かせるには、larger runner（有料の大きいランナー）と Azure の仮想ネットワーク（Azure private networking）を組み合わせる必要がある。AWS や Google Cloud の VPC には直接つながらない。
- 固定 IP は larger runner でのみ使える。Azure private networking とは同時に使えない[^static-ip]。

### B. AWS CodeBuild-hosted runner

```
GitHub ─(workflow_job Webhook)─▶ CodeBuild ─▶ ジョブごとにビルド環境を起動
                                             ├─ B-EC2: EC2 コンピュート
                                             └─ B-Lambda: Lambda コンピュート
```

- CodeBuild がジョブの開始通知（`workflow_job` の Webhook）を受け取り、ジョブ 1 件ごとにビルド環境を起動して、使い捨てのランナーを登録する。ジョブが終わればビルド環境ごと破棄される。
- ワークフロー側は `runs-on: codebuild-<プロジェクト名>-${{ github.run_id }}-${{ github.run_attempt }}` のように書く。ラベルでイメージやインスタンスサイズを上書きできる。
- オートスケーラーやランナーの登録処理を自前で持つ必要がない。セルフホストの中では運用が最も軽い。
- コンピュートは EC2 と Lambda から選べる。両者は別物と考えたほうがよい。

| 観点 | B-EC2 | B-Lambda |
| --- | --- | --- |
| Docker | 使える（特権モードを有効にする） | 使えない |
| root 権限 | あり | なし（`yum` などは使えない） |
| 書き込める場所 | 制限なし | `/tmp` のみ |
| ジョブの最大時間 | ビルドのタイムアウト（最大 36 時間） | 15 分 |
| VPC 接続 | 可 | 不可 |
| 起動 | 数十秒〜（リザーブドキャパシティなら短縮できる） | 速い |
| 課金単位 | 分単位で切り上げ（起動待ちの時間も含む） | 秒単位で切り上げ |

B-Lambda はテストやリント、静的サイトのビルドのように、Docker を使わず 15 分以内に終わるジョブに限れば非常に安い[^lambda]。

### C. Cloud Run worker pools

```
GitHub ─(workflow_job)─▶ オートスケーラー（CREMA、Cloud Run サービス）
                              │ 台数を増減
                              ▼
                         Cloud Run worker pool（ランナーのコンテナ、ジョブごとに使い捨て）
```

- Google Cloud が公式のチュートリアルで示している構成である[^cloudrun]。worker pool は、リクエストを受けずに常駐して処理を続けるタイプの Cloud Run である。
- コンテナはジョブ 1 件ごとに使い捨てのランナーとして登録される。オートスケーラー（CREMA）が待ちジョブ数に応じて worker pool の台数を増減し、ジョブがなければゼロ台まで減らす。
- オートスケーラーは別の Cloud Run サービスとして常時動かす。その分の固定費がかかる。
- Cloud Run では特権コンテナを動かせないため、ジョブの中で Docker デーモンを使う `docker build` や `docker run` は使えない。イメージのビルドは Cloud Build に任せるか、デーモンを使わないビルドツールを使う。
- Direct VPC egress で VPC 内のリソースに届く。Google Cloud へのデプロイは、ランナーのサービスアカウントの権限で行える。

### D. EC2 で自前運用

```
GitHub ─(workflow_job Webhook)─▶ API Gateway + Lambda ─▶ SQS ─▶ Lambda ─▶ EC2（スポット、ジョブごとに使い捨て）
```

- [terraform-aws-github-runner](https://github.com/github-aws-runners/terraform-aws-github-runner) のような OSS か、[RunsOn](https://runs-on.com/) のような製品を使い、自分の AWS アカウントの EC2 でランナーを動かす。
- 既定でスポットインスタンスを使い、ジョブがなければゼロ台まで減らす。インスタンスタイプを自由に選べるので、GPU や大容量メモリのジョブも扱える。
- 普通の VM なので、Docker も root 権限も制約なく使える。
- スケーリングの仕組み（Webhook の受け口、キュー、起動と停止の Lambda）は OSS が用意するが、そのバージョンアップと障害対応は自分で持つ。RunsOn は年額のライセンス料（商用で 300 EUR/年から）がかかる代わりに、構築と保守が軽い。

### E. Kubernetes 上の Actions Runner Controller（ARC）

```
GitHub ◀─(ロングポーリング)─ ARC のリスナー ─▶ ランナーの Pod（ジョブごとに使い捨て）
                                              （EKS / GKE などのクラスタ上）
```

- GitHub 公式の Kubernetes オペレーターである。ランナースケールセットを作ると、待ちジョブ数に応じてランナーの Pod が増減する。
- ジョブの中で Docker を使うには、Docker-in-Docker モードで特権 Pod を動かすか、ジョブのコンテナを別の Pod として起動する Kubernetes モードを使う。
- すでに Kubernetes クラスタを運用しているチームなら、既存の監視やノードの自動増減（Karpenter など）にそのまま乗せられる。クラスタがなければ、ランナーのためだけに運用するのは重い。

### F. 外部のランナーサービス

- Blacksmith、Depot、Namespace、Ubicloud など、GitHub-hosted runner の代わりに使える有料のランナーサービスがある。ワークフローの `runs-on` を書き換えるだけで移行できるものが多い。
- 多くは GitHub-hosted より安い単価（2 vCPU で 0.004 USD/分前後）か、速い CPU とキャッシュをうたう。
- ジョブとソースコードが第三者の基盤で動くため、社内のセキュリティ審査が必要になる。VPC 内のリソースに届かない点は A と同じである。

## 3. トレードオフ

### 機能の差

| 観点 | A: GitHub-hosted | B-EC2: CodeBuild | B-Lambda: CodeBuild | C: Cloud Run | D: EC2 自前 | E: ARC |
| --- | --- | --- | --- | --- | --- | --- |
| Docker | 可 | 可 | 不可 | 不可 | 可 | 可（特権 Pod など） |
| ジョブの最大時間 | 6 時間 | 36 時間 | 15 分 | 5 日（GitHub の上限） | 5 日 | 5 日 |
| VPC 内への到達 | Azure VNet のみ | 可 | 不可 | 可 | 可 | 可 |
| クラウドの権限 | OIDC で取得 | IAM ロール | IAM ロール | サービスアカウント | IAM ロール | IRSA / Workload Identity |
| ジョブごとの使い捨て | 常に | 常に | 常に | 構成次第（チュートリアルは使い捨て） | 設定次第（既定は無効） | 既定で有効 |
| ツールの初期状態 | 豊富 | CodeBuild のイメージ | 限定的 | 自前のイメージ | 自前の AMI | 自前のイメージ |
| スケーリングの仕組み | 不要 | 不要 | 不要 | オートスケーラーを自前で運用 | OSS か製品を運用 | ARC を運用 |

ジョブごとの使い捨ては、セキュリティ上の差になる。
ランナーを使い回すと、前のジョブが残したファイルやプロセスが次のジョブに影響しうる。
セルフホストでは、使い捨ての設定を必ず有効にする。

ツールの初期状態も見落としやすい差である。
GitHub-hosted のイメージには多くのツールが入っているため、セルフホストに移すとワークフローが動かないことがある。
その場合は `setup-node` などのアクションで入れるか、自前のイメージに入れておく。

### コスト

2 vCPU クラスの Linux ジョブの 1 分あたりの単価を比べる。

| 構成 | 1 分あたりの単価 | 備考 |
| --- | --- | --- |
| A: GitHub-hosted（x64） | 0.006 USD | Arm は 0.005 USD。プランに含まれる無料分数あり |
| B-EC2: CodeBuild general1.small（x64、3 GB） | 0.005 USD | Arm（arm1.small）は 0.0034 USD |
| B-Lambda: CodeBuild Lambda（Arm、1 GB） | 0.0006 USD | 0.00001 USD/秒。メモリ 1 GB |
| C: Cloud Run worker pool（2 vCPU、4 GiB） | 約 0.0016 USD | オートスケーラーの固定費が別にかかる |
| D: EC2 m7i.large（2 vCPU、8 GB）オンデマンド | 約 0.0017 USD | スポットなら 3 分の 1 程度まで下がる |
| E: ARC | D と同程度 | クラスタの固定費（EKS なら 73 USD/月）が別にかかる |
| F: 外部サービス | 0.004 USD 前後 | サービスによる |

単価だけを見ると、セルフホストは GitHub-hosted の 3 分の 1 から 10 分の 1 になる。
ただし、セルフホストには次の費用が加わる。

- **固定費**: C のオートスケーラー、D の NAT ゲートウェイ（プライベートサブネットに置く場合、約 33 USD/月から）、E のクラスタ、RunsOn のライセンス料。
- **起動のオーバーヘッド**: インスタンスやコンテナの起動とランナーの登録にかかる時間も課金される。短いジョブが多いほど、この比率が上がる。B-EC2 は分単位の切り上げなので、30 秒のジョブでも 1 分分かかる。
- **運用の人件費**: ランナーのイメージの更新、スケーリングの仕組みの保守、障害対応。金額にしにくいが、たいていは最も大きい。

もう一つの不確定要素として、GitHub はセルフホストランナーの利用に 0.002 USD/分の課金を始める計画を 2025 年 12 月に発表し、反発を受けて延期している[^selfhosted-fee]。
CodeBuild や Cloud Run のランナーも GitHub から見ればセルフホストランナーなので、導入されれば B から E のすべてに加わる。
この場合、B-EC2（x64）の実質単価は 0.007 USD/分となり、GitHub-hosted を上回る。

### 運用負荷

運用が軽い順に並べると次のようになる。

1. A: 何もしなくてよい。
2. F: `runs-on` を書き換え、サービスと契約するだけ。
3. B: CodeBuild のプロジェクトと Webhook を IaC で管理するだけで、スケーリングは AWS が持つ。
4. D（RunsOn）: CloudFormation のスタックを入れて更新する。
5. C、D（OSS）: オートスケーラーやスケーリング用の Lambda を、自前のインフラとして保守する。
6. E: Kubernetes クラスタと ARC の両方を保守する。クラスタを既に運用していれば負荷は下がる。

どのセルフホストでも、ランナーのバージョンを追い続ける必要がある。
GitHub は古いバージョンのランナーからの接続を一定期間後に拒否するため、更新を止めるとジョブが動かなくなる。
B では CodeBuild がランナーを用意するため、この作業も AWS に任せられる。

### ロックイン

- ワークフローの記述は GitHub Actions のままなので、どの構成でも `runs-on` を書き換えれば戻せる。
- B は `runs-on` のラベルに CodeBuild 固有の書式を使う。C、D、E はラベルを自由に決められる。
- ジョブの中でクラウド固有の権限（IAM ロールやサービスアカウント）に頼るほど、そのクラウドから動かしにくくなる。OIDC で権限を取る書き方にしておけば、A を含むどの構成にも戻しやすい。

## 4. 選ぶ基準

A（GitHub-hosted）が向くのは次のような場合である。

- 実行分数がプランの無料分数に収まるか、超過分の料金を許容できる。
- VPC 内のリソースにアクセスする必要がない。
- ランナーの運用に人を割きたくない。

B（CodeBuild）が向くのは次のような場合である。

- AWS を主に使っていて、VPC 内のリソースや IAM ロールをジョブから直接使いたい。
- セルフホストにしたいが、スケーリングの仕組みは持ちたくない。
- Docker を使わない短いジョブが多いなら、B-Lambda で大幅に安くできる。

C（Cloud Run）が向くのは次のような場合である。

- Google Cloud を主に使っていて、VPC 内のリソースやサービスアカウントをジョブから直接使いたい。
- ジョブの中で Docker デーモンを使わない。

D（EC2 自前）が向くのは次のような場合である。

- 実行分数が多く、スポットインスタンスで単価を最小にしたい。
- GPU や大容量メモリなど、特殊なインスタンスが必要なジョブがある。
- スケーリングの仕組みを保守できる体制がある（RunsOn ならこの負荷を減らせる）。

E（ARC）が向くのは次のような場合である。

- Kubernetes クラスタを既に運用していて、ランナーもその上に乗せたい。

F（外部サービス）が向くのは次のような場合である。

- ジョブを速くしたい、または安くしたいが、ランナーを運用したくない。
- VPC 内のリソースにアクセスする必要がなく、第三者の基盤で動かすことを社内で許容できる。

## 5. 損益分岐点

GitHub-hosted からセルフホストに移すと得になるのは、単価の差で固定費を回収できるときである。
月の実行分数を `M`、セルフホストの固定費を `F`（USD/月）、1 分あたりの単価を `c` とすると、分岐点は次の式になる。

```
M > F / (0.006 - c)
```

起動のオーバーヘッドは、`c` を実際のジョブ時間に対する比率で割り増して扱う。
運用の人件費は `F` に含めるべきだが、ここでは含めない。

| 構成 | 固定費 F の例 | 単価 c | 分岐点 M |
| --- | --- | --- | --- |
| B-EC2（x64） | 0 | 0.005 USD | 常に B が安い（差は 1 分あたり 0.001 USD） |
| B-EC2（Arm） | 0 | 0.0034 USD | 常に B が安い |
| C | オートスケーラー 30 USD/月と仮定 | 0.0016 USD | 約 6,800 分/月 |
| D（スポット、NAT あり） | NAT 33 USD/月 | 0.0006 USD | 約 6,100 分/月 |
| D（スポット、NAT と RunsOn） | 33 USD + 約 27 USD/月 | 0.0006 USD | 約 11,000 分/月 |
| E（EKS、スポット） | 73 USD/月 | 0.0006 USD | 約 13,500 分/月 |

月数千分から 1 万分を超えるあたりで、C、D、E が A より安くなる。
ただし、運用の人件費を月数時間分でも `F` に入れると、分岐点は数万分のオーダーまで上がる。
金額だけで決めるなら B が最も確実であり、C から E は、VPC への到達や特殊なインスタンスといった機能上の理由があるときに選ぶのが無難である。

セルフホストへの 0.002 USD/分の課金が導入された場合、分岐点は次のように動く。

- B-EC2（x64）は単価が 0.007 USD となり、GitHub-hosted より高くなる。Arm（0.0054 USD）は、わずかに安いままである。
- C、D、E は、`0.006 - c` が 0.002 USD 縮むため、分岐点が 1.6 倍から 1.8 倍ほど後ろにずれる（C は約 12,500 分/月、E は約 21,500 分/月）。

[^price]: 単価は変わることがある。最新の値は [GitHub Actions runner pricing](https://docs.github.com/en/billing/reference/actions-runner-pricing)、[AWS CodeBuild の料金](https://aws.amazon.com/codebuild/pricing/)、[Cloud Run の料金](https://cloud.google.com/run/pricing)、[Amazon EC2 の料金](https://aws.amazon.com/ec2/pricing/on-demand/) を参照。
[^static-ip]: [About Azure private networking for GitHub-hosted runners in your organization](https://docs.github.com/en/organizations/managing-organization-settings/about-azure-private-networking-for-github-hosted-runners-in-your-organization) を参照。
[^lambda]: B-Lambda の制約は [Run builds on AWS Lambda compute](https://docs.aws.amazon.com/codebuild/latest/userguide/lambda.html) を参照。
[^cloudrun]: [Host GitHub runners with Cloud Run worker pools](https://docs.cloud.google.com/run/docs/tutorials/github-runner) を参照。
[^selfhosted-fee]: 2026 年 3 月 1 日から課金する予定だったが、GitHub は「アプローチを見直すため延期する」としている。新しい開始日は執筆時点で発表されていない。
