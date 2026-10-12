# CloudWatch Logs のログを長期間保存するときの置き場所

CloudWatch Logs に集めたアプリケーションログを、年単位で保存する方法を比べる。
候補は **CloudWatch Logs に長期間置いたままにする** 方法と、**CloudWatch Logs には短期間だけ置き、S3 にアーカイブする** 方法の二つである。

本ファイルは [README](../README.md) の「クラウドごとの構成」の代わりに、AWS 内の二つの構成を並べる。
Google Cloud と Cloudflare の対応する構成は、別ファイルで扱う。

## 1. 解きたい課題と前提

監査や障害調査のために、ログを 1 年から 7 年ほど保存したいことがある。
ただし、日々検索するのは直近の数週間分であり、古いログはほとんど読まれない。
古いログを検索のしやすい CloudWatch Logs に置き続けるか、安い S3 に移すかで、費用と使い勝手が変わる。

前提は次のとおりとする。

- ログはすでに CloudWatch Logs に送られている（Lambda、ECS、CloudWatch エージェントなど）。CloudWatch Logs を経由せず、最初から S3 に直接出す構成は対象外とする。
- 直近 1 か月分は、どちらの構成でも CloudWatch Logs に置き、Logs Insights で検索する。
- 単価は us-east-1 の執筆時点の値を使う[^price]。
- 圧縮率は 5:1 とする。CloudWatch Logs の保存料金は圧縮後のサイズにかかり、AWS の料金例もこの比率を使っているためである。圧縮が効かないログの場合は、[5 章](#5-損益分岐点)の最後で扱う。

## 2. 構成

### A. CloudWatch Logs に長期間置く

```
App ─▶ CloudWatch Logs（保持期間 = 必要な保存期間）
```

- ロググループの保持期間を、必要な保存期間（例: 5 年）に設定するだけで済む。
- 古いログも Logs Insights でそのまま検索できる。
- CloudWatch Logs の **S3 Tables 連携** を有効にすると、同じログを Apache Iceberg 形式のテーブルとして Athena や Redshift から SQL で検索できる。追加料金はない[^s3tables]。

S3 Tables 連携は S3 にデータを置くが、B のようなアーカイブには使えない。
S3 テーブルのデータはロググループの保持期間に従って削除され、保存料金も CloudWatch Logs の料金のままだからである。
A の検索手段を増やす機能として扱う。

### B. CloudWatch Logs に短期間置き、S3 にアーカイブする

```
App ─▶ CloudWatch Logs（保持期間 = 1 か月）
              │
              ├─ B1: export タスク（日次バッチ）─────────▶ S3 ─(ライフサイクル)─▶ Glacier IR
              └─ B2: サブスクリプションフィルター ─▶ Firehose ─▶ S3 ─(ライフサイクル)─▶ Glacier IR
```

- CloudWatch Logs の保持期間は 1 か月にし、S3 側に必要な保存期間分を置く。
- S3 では、ライフサイクルルールで **S3 Glacier Instant Retrieval（Glacier IR）** に移す。Glacier IR はミリ秒で取り出せるので、Athena で直接検索できる。取り出しに数時間かかってよいなら、より安い Glacier Deep Archive も選べる。中間に Glacier Flexible Retrieval もあるが、Glacier IR より 1 割ほど安いだけで、取り出しに数分から 12 時間かかる。
- S3 への出し方は二つある。

| 観点 | B1: export タスク | B2: Firehose |
| --- | --- | --- |
| 仕組み | `CreateExportTask` を定期実行して、期間を指定して書き出す | サブスクリプションフィルターでログを Firehose に流し続ける |
| 遅延 | 書き出せるようになるまで最大 12 時間 | 数分（Firehose のバッファ時間） |
| 費用 | 書き出し自体の料金はない（S3 の PUT 料金のみ） | Firehose の取り込み料金 0.029 USD/GB |
| 起動の仕組み | EventBridge Scheduler と Lambda などを自前で用意する | 設定すれば常時動く |
| IA ログクラス | 使える | 使えない（IA はサブスクリプションフィルターに非対応） |
| AWS の推奨 | 継続的なアーカイブには推奨されていない[^export] | 推奨されている |

B1 は安いが、定期実行の仕組みと失敗時の再実行を自前で持つ必要がある。
1 回の export タスクは 24 時間でタイムアウトするので、ログの量が多いロググループでは期間を細かく分けて実行することになる。
B2 は費用がかかる代わりに、仕組みを AWS に任せられる。

#### B1 の定期実行の組み方

export タスクは、実行中のものが **アカウントとリージョンごとに 1 つまで** という制約がある。
このクォータは引き上げられない[^quota]。
複数のロググループを書き出すには、一つの完了を待ってから次を開始する必要がある。

ロググループが数個なら、EventBridge Scheduler から Lambda を日次で起動し、前日分の期間を指定して `CreateExportTask` を呼ぶ構成で足りる。
ロググループが多い場合は、完了待ちが Lambda の最大実行時間（15 分）を超えうるので、Step Functions で順番に実行する。

```
EventBridge Scheduler（日次）─▶ Step Functions
                                  ├─ Map（ロググループごと、同時実行数 1）
                                  │    ├─ CreateExportTask
                                  │    └─ DescribeExportTasks で完了するまで Wait とループ
                                  └─ 失敗時に通知
```

Step Functions は AWS の API を直接呼べるので、Lambda を書かずに組める。
再実行と失敗の通知も Step Functions にまとめられる。

## 3. トレードオフ

### 機能の差

| 観点 | A: CloudWatch Logs のみ | B: CloudWatch Logs + S3 |
| --- | --- | --- |
| 古いログの検索 | Logs Insights でそのまま検索 | Athena で検索（テーブル定義が必要） |
| 検索の料金 | Logs Insights: 0.005 USD/GB スキャン | Athena: 5 USD/TB スキャン、Glacier IR からの取り出し料金 0.03 USD/GB |
| 検索までの手間 | なし | Athena のテーブルとパーティションを用意する |
| 改ざん防止 | 保持期間の設定と IAM で守る | S3 オブジェクトロックで WORM（書き込み後変更不可）にできる |
| 他のツールからの利用 | CloudWatch Logs の API 経由 | gzip のファイルなので、任意のツールで読める |
| メトリクスフィルター、Live Tail | 使える | 直近 1 か月分のみ（CloudWatch Logs 側） |

改ざん防止の要件がある場合、差は大きい。
CloudWatch Logs には WORM の仕組みがなく、権限を持つ人はロググループを削除できる。
S3 オブジェクトロックのコンプライアンスモードを使えば、root ユーザーでも保存期間中は削除できない。

### コスト

取り込み料金（Standard クラスで 0.50 USD/GB）はどちらの構成でもかかる。
差が出るのは、保存料金と、S3 へ出す仕組みの費用だけである。

| 項目 | 単価 |
| --- | --- |
| CloudWatch Logs の取り込み（Standard / IA） | 0.50 / 0.25 USD/GB（一度だけ） |
| CloudWatch Logs の保存 | 0.03 USD/GB・月（圧縮後） |
| S3 Standard | 0.023 USD/GB・月 |
| S3 Glacier IR | 0.004 USD/GB・月（最低 90 日） |
| S3 Glacier Flexible Retrieval | 0.0036 USD/GB・月（最低 90 日、取り出しに数分から 12 時間） |
| S3 Glacier Deep Archive | 0.00099 USD/GB・月（最低 180 日、取り出しに数時間） |
| Firehose の取り込み | 0.029 USD/GB（5 KB 単位で切り上げ） |

保存料金だけを比べると、CloudWatch Logs は Glacier IR の 7.5 倍高い。
ただし、取り込み料金がそれよりずっと大きいため、合計額で見ると差は小さくなる。
期間ごとの具体的な差は [5 章](#5-損益分岐点)で計算する。

古いログを頻繁に検索する場合は、検索の料金も効いてくる。
Glacier IR からの取り出しは 0.03 USD/GB かかり、これは CloudWatch Logs の 1 か月分の保存料金と同じである。
古いログを月に何度も全期間検索するなら、Glacier IR ではなく S3 Standard に置くほうがよい。

### 運用負荷

- A は保持期間を設定するだけで、追加の仕組みはない。
- B1 は、export タスクを定期実行する仕組み、失敗したときの再実行、書き出し漏れの検知を自前で持つ。ロググループが増えるたびに対象に加える必要もある。
- B2 は、サブスクリプションフィルターと Firehose を IaC で管理する。ロググループが増えたら、アカウント単位のサブスクリプションフィルターで漏れを防げる。
- B では、Athena で検索するためのテーブル定義とパーティションの管理が加わる。B2 で Firehose の展開（decompression）を有効にすると、ログが 1 行 1 イベントで S3 に書かれ、Athena で扱いやすくなる（0.00325 USD/GB が加わる）。

### ロックイン

- A は、ログが CloudWatch Logs の中にしかない。別の基盤に移るときは、まず export する必要がある。
- B は、ログが gzip のファイルとして S3 にあるので、Athena 以外の分析基盤や、他のクラウドにも持ち出しやすい。

## 4. 選ぶ基準

A（CloudWatch Logs のみ）が向くのは次のような場合である。

- 保存期間が 1 年以内で、圧縮がよく効くログである。
- 古いログも Logs Insights で気軽に検索したい。
- アーカイブの仕組みを持つ運用の手間を避けたい。

B（CloudWatch Logs + S3）が向くのは次のような場合である。

- 保存期間が年単位である。
- 改ざん防止（WORM）が必要である。
- 圧縮が効きにくいログで、保存料金の比重が大きい。
- ログを他の分析基盤でも使いたい。

B を選ぶ場合、B1 と B2 の選択は次で決まる。

- IA ログクラスを使っているなら B1 しか選べない。
- 書き出しの遅延を数分に抑えたい、または定期実行の仕組みを持ちたくないなら B2 にする。
- 費用を最小にしたく、日次バッチの運用を受け入れられるなら B1 にする。

## 5. 損益分岐点

### 1 GB あたりの式

取り込んだログ 1 GB（圧縮前）を T か月保存するときの費用を、Standard クラスで計算する。
圧縮率 5:1 なので、保存されるサイズは 0.2 GB である。

- A: `0.50 + 0.03 × 0.2 × T` = `0.50 + 0.006T`
- B: `0.50 + 0.03 × 0.2 × 1 + 0.004 × 0.2 × T + p` = `0.506 + 0.0008T + p`

`p` は S3 へ出す費用である。
B1 ではほぼ 0 である。
B2 では Firehose の料金で、Firehose が圧縮後のサイズに課金するなら約 0.0058 USD、圧縮前のサイズに課金するなら 0.029 USD になる[^firehose]。
B では、直近 1 か月分を S3 にも置くので、保存料金を少し多めに見積もっている。

A と B の費用が等しくなる T は、`(0.006 + p) / 0.0052` か月である。

| S3 への出し方 | p | 保存料金で元が取れる期間 |
| --- | --- | --- |
| B1: export タスク | 約 0 | 約 1.2 か月 |
| B2: Firehose（圧縮後に課金） | 約 0.0058 USD | 約 2.3 か月 |
| B2: Firehose（圧縮前に課金） | 0.029 USD | 約 6.7 か月 |

保存期間が 1 か月を超えた時点から、B のほうが安くなっていく。
ただし、これは「どちらが安いか」の分岐点であり、「差が気になるほど大きいか」は別の問題である。

### 合計額で見た差

月に 100 GB（圧縮前）のログを取り込む場合の、保存期間ごとの月額である。
保存期間分のログがたまりきった後の定常状態の値で、取り込み料金 50 USD を含む。

B2 の料金は、Firehose が圧縮後と圧縮前のどちらのサイズに課金するかで変わる[^firehose]。
どちらになるかを確認できていないため、両方の値を並べる。
実際の B2 の月額は、この 2 列のあいだに入る。

| 保存期間 | A | B1 | B2（圧縮後に課金） | B2（圧縮前に課金） | A に対する B1 の削減率 |
| --- | --- | --- | --- | --- | --- |
| 1 年 | 57.2 USD | 51.6 USD | 52.1 USD | 54.5 USD | 10 % |
| 3 年 | 71.6 USD | 53.5 USD | 54.1 USD | 56.4 USD | 25 % |
| 5 年 | 86.0 USD | 55.4 USD | 56.0 USD | 58.3 USD | 36 % |
| 7 年 | 100.4 USD | 57.3 USD | 57.9 USD | 60.2 USD | 43 % |

B2 が圧縮後に課金される場合、B1 との差は月 0.6 USD しかない。
その場合は、定期実行の仕組みを持たずに済む B2 を選ぶ理由が強くなる。

保存期間が 1 年なら、差は合計額の 5〜10 % にとどまる。
この程度の差であれば、アーカイブの仕組みを持たずに済む A を選ぶ理由は十分にある。
保存期間が 3 年を超えると差は 25 % 以上に開き、ログの量が多いほど金額の差も大きくなる。

### 前提が変わると分岐点も動く

圧縮が効かないログ（すでに圧縮された形式や、ランダムな値が多いログ）では、保存料金の比重が大きくなる。
圧縮率 1:1 の場合、1 年保存でも B1 は A より約 33 % 安い。
「1 年なら差は小さい」と言えるのは、圧縮がよく効くログに限られる。

IA ログクラスでは取り込み料金が半分になり、保存料金は変わらない。
そのため、合計額に占める保存料金の比重が上がり、1 年保存でも B1 は A より約 18 % 安くなる。

[^price]: 単価はリージョンと利用量の段階によって変わる。最新の値は [Amazon CloudWatch の料金](https://aws.amazon.com/cloudwatch/pricing/)、[Amazon S3 の料金](https://aws.amazon.com/s3/pricing/)、[Amazon Data Firehose の料金](https://aws.amazon.com/firehose/pricing/) を参照。
[^export]: [Exporting log data to Amazon S3](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/S3Export.html) に「継続的なアーカイブのために定期的に S3 へ export することは推奨しない。その用途にはサブスクリプションを使うことを推奨する」とある。
[^s3tables]: [Access logs with S3 Tables Integration](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/s3-tables-integration.html) を参照。連携を作成した後に届いたログだけが対象で、既存のログはさかのぼって入らない。
[^quota]: [CloudWatch Logs quotas](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/cloudwatch_limits_cwl.html) の「Active export task」を参照。
[^firehose]: サブスクリプションフィルターは、ログを gzip で圧縮した状態で Firehose に送る。Firehose は受け取ったレコードのサイズに 5 KB 単位で課金するため、実際の料金は両者のあいだに収まると考えられる。
