# Next.js を Cloudflare でどう動かすか

Next.js のアプリを、サーバー側の処理（SSR、Route Handlers、Server Actions）ごと Cloudflare で動かす方法を比べる。
対象は **vinext**、**OpenNext**、**Cloudflare Containers** の 3 つとする。

## 1. 解きたい課題と前提

Next.js は Vercel や Node.js のサーバーで動かす前提で作られている。
Cloudflare Workers は Node.js ではなく V8 の isolate で動くため、Next.js をそのままでは載せられない。
何らかの変換か作り直し、あるいは Node.js が動く別の実行環境が必要になる。

前提は次のとおりとする。

- SSR やサーバー側の処理を使う。静的エクスポート（`output: 'export'`）で済むアプリは対象外とする。静的なら Workers Static Assets にそのまま載せればよく、比べるまでもない。
- Workers の有料プラン（5 USD/月）を使う。Containers は有料プランでしか使えない。
- 料金と制限は執筆時点（2026 年 11 月）の Cloudflare のドキュメントの値を使う[^price]。

次のものは比較から外す。

- `@cloudflare/next-on-pages`: 非推奨で、新しく選ぶものではない。
- Next.js 公式の Adapter API を使う Cloudflare 向けアダプター: Adapter API は Next.js 16.2 で安定版になった。Cloudflare 向けのアダプターは OpenNext が Adapter API の上で開発しており、2026 年 3 月時点では年内に出る予定とされていた[^adapter]。OpenNext が Adapter API に移る話なので、ここでは OpenNext に含めて扱う。

## 2. 構成

### A. vinext

```
app/ のコード ─(vinext: Vite の上で Next.js の API を再実装)─▶ Worker
```

- [cloudflare/vinext](https://github.com/cloudflare/vinext)。Next.js の API（`app/`、`pages/`、`next.config.*`）を、Vite のプラグインとして作り直したものである。Cloudflare が開発している。
- Cloudflare のドキュメントは、Next.js を Workers で動かす既定の方法として vinext を推奨している[^vinext]。
- 既存のアプリには `vinext init` で追加できる。`next dev` と `next build` も残るので、並べて試せる。
- 作り直しなので、Next.js と動きが違う箇所や未対応の機能がある。執筆時点では、Cache Components（Next.js 16 の PPR を含む）の対応が途中である。事前に `npx vinext check` で互換性を確かめられる。
- Cloudflare は 2026 年 9 月にブログで 1.0 を発表し、本番で使えるとした[^vinext-1]。一方、Workers のドキュメントには執筆時点でもベータ版と書かれており、表記が揃っていない。
- Cloudflare Workers のほか、Netlify や AWS Lambda にもデプロイできるとされている[^vinext-1]。
- ISR と `unstable_cache` に対応する。`"use cache"` は一部の対応にとどまる。本番ではキャッシュの保存先を設定する。

### B. OpenNext

```
app/ のコード ─(next build)─▶ ビルド出力 ─(@opennextjs/cloudflare で変換)─▶ Worker
```

- [opennextjs/opennextjs-cloudflare](https://github.com/opennextjs/opennextjs-cloudflare)。本物の `next build` の出力を、Workers で動く形に変換するアダプターである。
- 本物の Next.js でビルドするので、互換性は vinext より高い。
- Cloudflare は、新しいアプリには vinext を勧め、OpenNext は既存のアプリを保守する経路と位置づけている[^opennext]。
- ISR などのキャッシュは、使う機能に応じて組み合わせる[^opennext-cache]。ページとデータの保存先は R2（KV は結果整合性のため非推奨）、時間ベースの再検証のキューは Durable Objects、`revalidateTag` / `revalidatePath` 用のタグのキャッシュは D1（大規模なら Durable Objects を分割したもの）を使う。SSR だけなら設定は要らない。
- Middleware で Node.js のランタイムを使う機能には、執筆時点で対応していない[^opennext]。
- Next.js の新しいバージョンが出ると、アダプターが追従するまで待つことがある。

### C. Cloudflare Containers で `next start`

```
リクエスト ─▶ Worker（転送するだけ）─▶ Durable Object ─▶ コンテナ（next start）
```

- Next.js を `output: 'standalone'` でビルドし、Node.js のコンテナとして Cloudflare Containers で動かす。中身はふつうの Node.js の Next.js なので、`next start` で動かすのと同じ挙動になる。
- コンテナは利用者から直接は見えず、前に置いた Worker が Durable Object 経由でリクエストを転送する[^containers]。
- インスタンスの大きさは `lite`（1/16 vCPU、256 MiB）から `standard-4`（4 vCPU、12 GiB）まで選べる。
- リクエストがない間はコンテナを休止させられる。休止させると、次のリクエストでコンテナの起動を待つことになる。
- Next.js のキャッシュ（ISR など）は、既定ではインスタンスのローカルディスクに置かれる。ローカルディスクは休止や再起動で消えるので、1 台でもキャッシュは保持されない。保持や共有をするには、`cacheHandler` を自分で書いて R2 などの外部に置く必要がある。
- コンテナの中から KV、R2、D1 などのバインディングを使うには、Worker 側に outbound handler（`WorkerEntrypoint`）を書き、コンテナからはそのホスト名へ HTTP でリクエストを送る[^containers-bindings]。

## 3. トレードオフ

### 機能の差

| 観点                                  | A: vinext                                | B: OpenNext                        | C: Containers                                      |
| ------------------------------------- | ---------------------------------------- | ---------------------------------- | -------------------------------------------------- |
| Next.js との互換性                    | 差分がある（Cache Components は途中）    | 高い                               | `next start` と同等                                |
| ビルド                                | Vite（速い）                             | `next build` + 変換                | `next build` + イメージのビルド                    |
| Node.js の API とネイティブモジュール | `nodejs_compat` の範囲。sharp などは不可 | 同左                               | 制限なし                                           |
| 実行場所                              | Workers の isolate                       | 同左                               | コンテナ（VM）                                     |
| 起動の待ち                            | ほぼない                                 | ほぼない                           | 休止から起こすと待ちが出る                         |
| メモリ                                | isolate あたり 128 MB                    | 同左                               | インスタンスの大きさ次第（最大 12 GiB）            |
| ISR とキャッシュの共有                | 保存先を設定する                         | R2、Durable Objects、D1 を設定する | 自前の `cacheHandler` が必要                       |
| KV、D1、R2 などのバインディング       | `cloudflare:workers` から直接使える      | 直接使える                         | HTTP で使える。outbound handler を書く             |
| Next.js の新機能                      | vinext が再実装するまで使えない          | アダプターの追従を待つことがある   | すぐ使える                                         |
| 成熟度                                | 1.0（ドキュメントにはベータ版の表記）    | 実績が多い                         | Containers 自体は新しめ。中の Next.js は枯れている |

Worker のサイズの上限は、2026 年 9 月に圧縮前で 64 MiB に緩和された[^size]。
以前は圧縮後 10 MB（有料プラン）の上限があり、OpenNext ではバンドルのサイズが問題になりやすかったが、圧縮後の上限は廃止されたので、以前ほど問題になりにくい。
ただし、Worker のグローバルスコープの初期化は 1 秒以内に終える必要があり、バンドルが大きいほどここに近づく。

### コスト

A と B は Workers の料金で、C はそれに Containers の料金が加わる。

| 項目                 | A / B（Workers）                                               | C（Containers）                                                                      |
| -------------------- | -------------------------------------------------------------- | ------------------------------------------------------------------------------------ |
| 固定費               | 有料プラン 5 USD/月                                            | 同左                                                                                 |
| 従量の単位           | リクエスト数と CPU 時間                                        | 稼働時間あたりのメモリ、ディスク（割り当て量）と vCPU（実際の使用量）                |
| 無料枠（月）         | 1,000 万リクエスト、CPU 3,000 万 ms                            | メモリ 25 GiB 時間、vCPU 375 分、ディスク 200 GB 時間                                |
| 超過分の単価         | 100 万リクエストあたり 0.30 USD、CPU 100 万 ms あたり 0.02 USD | メモリ 0.0000025 USD/GiB 秒、vCPU 0.00002 USD/vCPU 秒、ディスク 0.00000007 USD/GB 秒 |
| リクエストがないとき | 0                                                              | 休止させれば 0。前の Worker の分は A / B と同じくかかる                              |

C を 1 台常時動かした場合の月額（30 日、CPU の平均使用率 20% と仮定）は次のようになる。
egress、前段の Worker と Durable Object の費用は含まない。

| インスタンス | 割り当て              | メモリ    | vCPU     | ディスク | 合計（プラン料金を除く） |
| ------------ | --------------------- | --------- | -------- | -------- | ------------------------ |
| basic        | 1/4 vCPU、1 GiB、4 GB | 約 6 USD  | 約 2 USD | 約 1 USD | 約 9 USD                 |
| standard-1   | 1/2 vCPU、4 GiB、8 GB | 約 26 USD | 約 5 USD | 約 1 USD | 約 32 USD                |

メモリは使っていなくても割り当て量で課金されるので、常時動かすとメモリの費用が支配的になる。
A と B は、月数百万リクエスト程度で 1 リクエストあたりの CPU 時間が 10 ms 程度なら、プラン料金の 5 USD にほぼ収まる。B でキャッシュに R2、Durable Objects、D1 を使う場合は、その料金も加わる。
C を常時起動する場合、トラフィックが少ないほど、A / B との差は比率で大きくなる。

### 運用負荷

1. A: 設定が最も少ない。ただし「Next.js では動くが vinext では動かない」という差分の調査が発生しうる。
2. B: 使うキャッシュの機能によって、R2、Durable Objects、D1 の作成と設定が要る。Next.js の更新のたびに、アダプターの対応状況を確認する。
3. C: Dockerfile、イメージ、インスタンスの大きさと休止の設定、転送用の Worker と Durable Object を保守する。バインディングを使うなら outbound handler も保守する。ISR を複数のインスタンスで使うなら `cacheHandler` も持つ。代わりに Next.js 側でハマることはない。

### ロックイン

- A は Next.js の再実装に依存し、Next.js 本体から離れる度合いが最も大きい。Cloudflare 以外（Netlify、AWS Lambda）にも出せるが、Next.js 本体とのずれは残る。`next build` を並べて残しておけば、Next.js に戻る道もある。
- B はアプリが普通の Next.js のままで、Cloudflare 固有なのはアダプターの設定とキャッシュの保存先だけである。
- C は中身がただのコンテナなので、Cloud Run や ECS にそのまま移せる。ロックインは最も弱い。前に置く Worker だけが Cloudflare 固有になる。
- A と B でバインディング（`cloudflare:workers` の `env`）を直接使うほど、アプリのコードが Cloudflare に依存していく。

## 4. 選ぶ基準

A（vinext）が向くのは次のような場合である。

- 新しく作る、または既存のアプリで `npx vinext check` が通る。
- エッジで速く動かし、トラフィックが少ない時期の費用を最小にしたい。
- KV や D1 などのバインディングを、`env` から型付きの API でそのまま呼びたい。

B（OpenNext）が向くのは次のような場合である。

- 既存の Next.js のアプリで、vinext が対応していない機能を使っている。
- それでも Workers の上で動かすこと（休止から起こす待ちがないこと、バインディングを `env` から呼べること）は譲れない。

C（Containers）が向くのは次のような場合である。

- sharp などのネイティブモジュール、128 MB を超えるメモリ、Node.js 固有の API に依存している。
- Next.js の新機能をすぐ使いたい、または互換性の差分を調べる手間をかけたくない。
- 将来ほかのクラウドへ移す可能性がある。
- 常時起動の費用（basic で月 9 USD 程度から）と、休止から起こすときの待ちを許容できる。

迷ったら、`npx vinext check` を最初に流すのがよい。
通れば A、通らなければ B を試し、B でも動かない理由が Node.js への依存にあるなら C を選ぶ。

[^price]: [Workers の料金](https://developers.cloudflare.com/workers/platform/pricing/)、[Containers の料金](https://developers.cloudflare.com/containers/platform/pricing/)、[Workers の制限](https://developers.cloudflare.com/workers/platform/limits/) を参照。

[^adapter]: [Next.js Across Platforms: Adapters, OpenNext, and Our Commitments](https://nextjs.org/blog/nextjs-across-platforms) を参照。

[^vinext]: [Next.js · Cloudflare Workers docs](https://developers.cloudflare.com/workers/framework-guides/web-apps/nextjs/) を参照。

[^opennext-cache]: [Caching · OpenNext](https://opennext.js.org/cloudflare/caching) を参照。

[^opennext]: [OpenNext adapter · Cloudflare Workers docs](https://developers.cloudflare.com/workers/framework-guides/web-apps/opennext/) を参照。

[^vinext-1]: [Next.js applications, powered by Vite: introducing Vinext 1.0](https://blog.cloudflare.com/vinext-nextjs-on-vite/)（Cloudflare Blog、2026 年 9 月）を参照。

[^containers-bindings]: [Connect to Workers and Bindings](https://developers.cloudflare.com/containers/configuration/workers-connections/) を参照。

[^containers]: [Container Fundamentals](https://developers.cloudflare.com/containers/concepts/) と [Limits and Instance Types](https://developers.cloudflare.com/containers/platform/limits/) を参照。

[^size]: [Deploy larger Workers — up to 64 MiB for both free and paid plans](https://developers.cloudflare.com/changelog/post/2026-09-04-increased-worker-size-limit/) を参照。
