# Lambda で Web アプリケーションを公開するときの入口

Lambda で動く Web アプリケーションに、CloudFront の後ろで HTTP リクエストを渡す方法を比べる。
候補は **API Gateway 経由**、**Lambda 関数 URL（Function URL）経由**、**Lambda Web Functions** の三つである。

前の二つは従来の Lambda 関数（イベントを受け取るハンドラーとして書くもの。Web Functions の登場に合わせて AWS は Event Functions と呼ぶようになった）の前に何を置くかの違いで、三つ目は関数の形そのものが変わる。
Web Functions は 2026 年 10 月に発表された新しい形態で、ポートで待ち受ける普通の HTTP サーバーをそのまま Lambda で動かし、HTTPS エンドポイントも Lambda 自身が持つ[^webfn]。

本ファイルは [README](../README.md) の「クラウドごとの構成」の代わりに、AWS 内の三つの構成を並べる。
Google Cloud と Cloudflare の対応する構成は、別ファイルで扱う。

## 1. 解きたい課題と前提

Lambda を HTTP で呼び出すには、HTTPS のエンドポイントをどこかに用意する必要がある。
その役を API Gateway に持たせるか、Lambda 自身が持つ関数 URL に持たせるか、エンドポイントを最初から持つ Web Functions を使うかで、使える機能と費用が変わる。

前提は次のとおりとする。

- 利用者からのリクエストは、どちらの構成でも CloudFront で受ける。独自ドメイン、TLS 証明書、WAF、静的ファイルのキャッシュは CloudFront 側に置く。
- Lambda 上のアプリケーションは、Express や FastAPI のような Web フレームワークで書かれた一つの関数（いわゆる Lambdalith）でもよいし、エンドポイントごとに分かれた複数の関数でもよい。
- WebSocket のような双方向通信は対象外とする（関数 URL と Web Functions では実現できず、API Gateway の WebSocket API を使うしかないため。Web Functions は `Upgrade` ヘッダーを取り除いてからアプリに渡す）。

## 2. 構成

### A. CloudFront + API Gateway + Lambda

```
Client ─▶ CloudFront ─▶ API Gateway (Regional) ─▶ Lambda
```

- API Gateway は Regional エンドポイントで作る。Edge-optimized を選ぶと AWS 管理の CloudFront が内部に入り、自前の CloudFront と二重になるためである。
- API Gateway には REST API と HTTP API の二種類がある。REST API は機能が多く高価で、HTTP API は機能を絞った代わりに安い。以下では両者を区別して書く。
- CloudFront から API Gateway へのアクセスを CloudFront 経由に限定する仕組み（OAC）は API Gateway には用意されていない。CloudFront で秘密のカスタムヘッダーを付与し、REST API なら AWS WAF で、HTTP API なら Lambda オーソライザーかアプリケーションで、そのヘッダーを検証するのが一般的である。

### B. CloudFront + 関数 URL + Lambda

```
Client ─▶ CloudFront ─(OAC / SigV4)─▶ Lambda Function URL ─▶ Lambda
```

- 関数 URL の認証タイプを `AWS_IAM` にし、CloudFront の **オリジンアクセスコントロール（OAC）** で SigV4 署名を付けさせる。これで CloudFront 以外からの呼び出しを拒否できる。
- Lambda のリソースベースポリシーで、CloudFront のサービスプリンシパルに `lambda:InvokeFunctionUrl` と `lambda:InvokeFunction` の両方を許可する必要がある[^oac]。
- 関数 URL は一つの関数に一つである。パスごとに関数を分けたい場合は、CloudFront のビヘイビア（パスパターンごとのオリジン振り分け）で複数の関数 URL に振り分ける。

### C. CloudFront + Lambda Web Functions

```
Client ─▶ CloudFront ─(OAC / SigV4)─▶ Web Function endpoint ─▶ Web Function（HTTP サーバー）
```

- アプリケーションは Express や Hono のような HTTP サーバーとして書き、環境変数 `AWS_LAMBDA_HTTP_ENDPOINT` のホストとポートで待ち受ける。ハンドラーへの書き換えや Lambda Web Adapter は要らない。
- 一つの実行環境が複数のリクエストを同時に処理する（既定 64、最大 128）。A と B では一つの実行環境が一度に一つのリクエストしか処理しない。
- エンドポイントの認証タイプを `IamAuth` にし、CloudFront の OAC を付ける。署名の動作は `always-amz-auth`（常に署名し、閲覧者の `Authorization` ヘッダーは残す）しか選べない[^oac-web]。
- Web Function のリソースベースポリシーで、CloudFront のサービスプリンシパルに `lambda:InvokeWebFunctionEndpoint` を許可する。
- CloudFront のオリジンリクエストポリシーは `AllViewerExceptHostHeader` を使う。`Host` ヘッダーがディストリビューションのドメインのままだと、エンドポイントは 404 を返す。
- コードと設定は変更できないリビジョンとして保存され、エンドポイントは一つか二つのリビジョンに重み付きで振り分ける。カナリアリリースが追加の仕組みなしにできる。
- 発表時点の制約として、ランタイムは Node.js 24（Arm64）だけで、実行環境の大きさは 2 vCPU / 2 GB に固定されている。使えるリージョンは東京を含む 5 つである。

## 3. トレードオフ

### 機能の差

| 観点 | A: API Gateway (REST) | A: API Gateway (HTTP) | B: 関数 URL | C: Web Functions |
| --- | --- | --- | --- | --- |
| ルーティング | パスとメソッド単位で複数の関数へ | パスとメソッド単位で複数の関数へ | 1 URL = 1 関数（振り分けは CloudFront かアプリ内） | 1 エンドポイント = 1 Web Function（振り分けは CloudFront かアプリ内） |
| 認証と認可 | IAM / Cognito / Lambda オーソライザー | IAM / JWT / Lambda オーソライザー | アプリ内で実装（`AWS_IAM` は OAC に使うため） | アプリ内で実装（`IamAuth` は OAC に使う。閲覧者の `Authorization` ヘッダーはアプリまで届く） |
| リクエスト検証 | スキーマによる検証あり | なし | なし（アプリ内） | なし（アプリ内） |
| スロットリング | ステージとメソッド単位、使用量プランと API キー | ステージとルート単位 | 予約同時実行数のみ（CloudFront 側の WAF レートベースルールで補う） | エンドポイント単位のレート制限（100〜10,000 rps）と実行環境数の上限 |
| AWS WAF の直接適用 | 可 | 不可 | 不可（CloudFront に適用） | 不可（CloudFront に適用） |
| キャッシュ | API Gateway キャッシュあり | なし | なし（CloudFront でキャッシュ） | なし（CloudFront でキャッシュ） |
| 統合タイムアウト | 既定 29 秒、Regional / private はクォータ申請で延長可、ストリーミング時は最大 15 分 | 最大 30 秒 | Lambda のタイムアウト（最大 15 分） | リクエストタイムアウト（既定 30 秒、3〜900 秒で設定） |
| リクエストサイズ | 10 MB（Lambda 側の 6 MB 制限が先に効く） | 10 MB（同上） | 6 MB | ヘッダーは 128 KiB。ボディの上限はドキュメントに記載がない |
| レスポンスストリーミング | 可（2025 年 11 月から） | 不可 | 可（最大 200 MB） | 可（SSE、chunked 転送。リクエスト側のストリーミングも可） |
| 変換とモック | マッピングテンプレート、モック統合あり | パラメータマッピングのみ | なし | なし |
| リリース管理 | ステージとカナリア設定 | ステージ | 関数のエイリアスと重み付け | リビジョンの重み付け（エンドポイントあたり二つまで） |
| マルチリージョン | 自前で各リージョンに構築 | 自前で各リージョンに構築 | 自前で各リージョンに構築 | エンドポイントの種類を選ぶだけ（レイテンシーベースの振り分けも Lambda が行う） |

関数 URL と Web Functions の列が空いているように見えるのは、その機能を API Gateway が担っていた部分を、アプリケーションか CloudFront に移す構成だからである。
認証、入力検証、ルーティングは Web フレームワークがもともと持っていることが多い。
アプリケーションが Lambdalith であれば、API Gateway の機能の多くは使われずに終わる。

C は B と同じく API Gateway の機能を持たないが、B にない機能がエンドポイントに入っている。
レート制限、カナリアリリース、マルチリージョン配置である。
一方で、ランタイムと実行環境の大きさを選べないという、A と B にはない制約がある。

### 性能

- A は CloudFront と Lambda のあいだに API Gateway のホップが一つ増える。その分のレイテンシが毎リクエストに加わる。
- コールドスタートの影響は A と B で変わらない。C は一つの実行環境で複数のリクエストを処理するので、アクセスが急に増えても新しい実行環境を作る回数が少なく、コールドスタートに当たるリクエストが減る。代わりに、同じ実行環境のリクエストどうしが CPU とメモリを取り合う。CPU を多く使う処理では、一つの実行環境が受ける同時リクエスト数を下げる必要がある。
- 長時間処理の扱いは、どの構成も CloudFront の **オリジン応答タイムアウト** に縛られる。この値は既定 30 秒で、設定で 120 秒まで、クォータ申請で 180 秒まで延ばせる。B の関数 URL は Lambda の 15 分まで、C は設定したリクエストタイムアウトまで待てるが、CloudFront を通す限り、オリジンから応答が届かない時間がこのタイムアウトを超えると 504 になる。ストリーミングで少しずつ返せば、この制限は回避しやすい。
- HTTP API は 30 秒が上限で延長できない。生成 AI の応答のように時間のかかる処理には、REST API のストリーミングか B か C を選ぶことになる。
- C のリクエストタイムアウトは、ストリーミングでもストリーム全体の長さに効く。時間切れになると、ステータス 200 のまま途中で切れたボディが返るので、クライアント側で切断を検知できるようにしておく。

### コスト

CloudFront と Lambda の料金は両者で共通なので、差は API Gateway のリクエスト料金だけである。
us-east-1 の執筆時点の単価で、月 1 億リクエストの場合の追加費用は次のとおり[^price]。

| 構成 | 単価（100 万リクエストあたり、最初の段階） | 月 1 億リクエストの追加費用 |
| --- | --- | --- |
| A: REST API | 3.50 USD | 約 350 USD |
| A: HTTP API | 1.00 USD | 約 100 USD |
| B: 関数 URL | 追加料金なし | 0 USD |

REST API のキャッシュを有効にすると、キャッシュサイズに応じた時間課金がさらに加わる。
リクエスト数が少ないうちは差は小さいが、リクエスト数に比例して開いていく。

C は API Gateway の料金がかからない点では B と同じだが、Lambda 自体の課金のしかたが違うので、上の表の延長では比べられない[^price-web]。

- A と B の Lambda は、リクエスト数と実行時間（メモリ量 × 経過時間）で課金される。データベースや外部 API の応答を待っているあいだも課金が続く。
- C は、リクエスト数、CPU を実際に使った時間、実行環境が存在するあいだのメモリで課金される。I/O 待ちのあいだの CPU 時間は課金されず、一つの実行環境の費用を同時に処理しているリクエストで分け合う。

そのため、C が安くなるのは、データベースや生成 AI の応答を待つ時間が長い I/O 中心の処理である。
CPU を使い続ける処理では、待ち時間がないので課金の差は小さい。
メモリは実行環境が存在するあいだ課金されるので、リクエストがまばらで実行環境が残り続ける使い方では、この分が割高になりうる。

### 運用負荷

- A は API Gateway のリソース（ステージ、デプロイ、ルート、オーソライザー）を IaC で管理する必要がある。ルートを追加するたびに API Gateway 側の定義も変える構成だと、アプリケーションと API 定義の二か所を同期させ続けることになる。
- A では API Gateway のアクセスログとメトリクス（4xx / 5xx、レイテンシ、統合レイテンシ）が手に入る。B では Lambda の関数 URL 用メトリクス（`UrlRequestCount`、`Url4xxCount`、`Url5xxCount` など）と CloudFront のログで代替する。
- B で OAC を使う場合、`PUT` と `POST` のリクエストでは、クライアントがボディの SHA256 を計算して `x-amz-content-sha256` ヘッダーに入れる必要がある。Lambda は署名なしのペイロードを受け付けないためである。ブラウザのフォーム送信や、ヘッダーを制御できない外部サービスからの Webhook では、この制約がそのまま障害になる。回避するには、次の三つのどれかを選ぶ。
  - クライアント側でハッシュを付ける実装を入れる。
  - OAC を外し、オリジンリクエストで動く Lambda@Edge にボディを含めて SigV4 署名させる[^edge-sign]。クライアントは何も付けなくてよいが、Lambda@Edge に渡るボディは 1 MB で切り詰められるので、それより大きいボディは署名が合わずに失敗する。書き込みのたびに Lambda@Edge の料金と実行時間が加わり、us-east-1 に置く Lambda@Edge をアプリケーションとは別に管理することになる。また、署名が `Authorization` ヘッダーを上書きするので、アプリケーションはこのヘッダーで認証できない。
  - OAC を外して秘密ヘッダーによる検証に切り替える（その場合、関数 URL は認証タイプ `NONE` で公開されることになる）。
- B ではレート制限を API Gateway に任せられないので、CloudFront の WAF レートベースルールと予約同時実行数で守ることになる。予約同時実行数は関数全体の上限であり、特定のクライアントだけを絞ることはできない。
- C の OAC では、`PUT`、`POST`、`PATCH` のリクエストを CloudFront がボディを署名に含めずに署名する。B のような `x-amz-content-sha256` の制約がなく、ブラウザのフォーム送信や外部サービスの Webhook も CloudFront 経由で受けられる。
- C でも、アプリが閲覧者の `Authorization` ヘッダーを使う場合は、そのヘッダーを CloudFront のキャッシュポリシーに含める必要がある。含めないと、`GET` と `HEAD` では CloudFront がヘッダーを取り除いてオリジンに送る。
- C のエンドポイントに OAC を付けたビヘイビアでは、Lambda@Edge のオリジンリクエストとオリジンレスポンスのトリガーを使えない。CloudFront 側にロジックを置いている構成からは、そのまま移せない。
- C のレート制限はエンドポイント全体の上限であり、B の予約同時実行数と同じく、特定のクライアントだけを絞ることはできない。クライアント単位の制限は、引き続き CloudFront の WAF で行う。
- C では、`AWS/Lambda/Web/Usage` 名前空間にエンドポイント単位のリクエスト数やスロットリング数のメトリクスが出る。レスポンスヘッダーの `lambda-web-request-id` でアプリのログと突き合わせられる。
- C はまだ発表直後で、IaC ツールや監視ツールの対応が揃っていない可能性がある。採用前に、使っているツールが Web Functions のリソースを扱えるかを確かめる。

### ロックイン

- A は API Gateway の定義（オーソライザー、マッピングテンプレート、使用量プラン）にロジックを置くほど、AWS 以外への移行が難しくなる。
- B はルーティングと認証がアプリケーションのコードにあるため、Lambda Web Adapter などを使っていれば、同じアプリケーションをコンテナにして ECS や Cloud Run に移しやすい。
- C はアプリケーションが普通の HTTP サーバーなので、三つのなかで最も移しやすい。Lambda 固有なのは待ち受けるホストとポートを環境変数から読む部分だけである。ただし、マルチリージョン配置やリビジョンの重み付けに頼っている場合、移行先で同じ仕組みを作り直すことになる。
- どの構成も Lambda と CloudFront には依存する。ロックインの差は、API Gateway や Web Functions のエンドポイントの機能にどれだけ依存するかで決まる。

## 4. 選ぶ基準

B（関数 URL）が向くのは次のような場合である。

- アプリケーションが Web フレームワークで書かれた Lambdalith で、ルーティングと認証をアプリ内で持っている。
- リクエスト数が多く、API Gateway のリクエスト料金が無視できない。
- 30 秒を超える処理や、ストリーミングで大きなレスポンスを返す処理がある。
- 書き込み系のリクエストが、`x-amz-content-sha256` を付けられるクライアント（自前のフロントエンドなど）から来る。

A（API Gateway）が向くのは次のような場合である。

- 外部に公開する API で、API キー、使用量プラン、クライアント単位のスロットリングが必要（REST API）。
- エンドポイントごとに関数を分け、認可やリクエスト検証をアプリケーションの外で統一したい。
- ブラウザのフォームや外部サービスの Webhook のように、署名ヘッダーを付けられないクライアントからの `POST` を受ける。
- API Gateway のアクセスログやメトリクスを監視の基盤にしている。

C（Web Functions）が向くのは次のような場合である。

- アプリケーションが Node.js の Web フレームワークで書かれていて、2 vCPU / 2 GB の実行環境に収まる。
- データベースや生成 AI の応答を待つ時間が長く、待ち時間に課金されたくない。
- アクセスの急増があり、コールドスタートに当たるリクエストを減らしたい。
- 書き込み系のリクエストを、署名ヘッダーを付けられないクライアントから受けつつ、オリジンは CloudFront からしか呼べないようにしたい。
- 複数のリージョンに配置したいが、リージョンごとの構築とレイテンシーベースの振り分けを自前で作りたくない。

C を避けるのは、Node.js 以外のランタイムや 2 GB を超えるメモリが要る場合、使うリージョンが対応していない場合、CloudFront のオリジン側で Lambda@Edge を使っている場合である。

A を選ぶ場合、REST API と HTTP API の選択は、使用量プラン、WAF の直接適用、リクエスト検証、ストリーミング、29 秒超のタイムアウトのどれかが要るかで決まる。
どれも要らなければ HTTP API のほうが安い。

### B と C のどちらにするか

B と C は、どちらも API Gateway を置かずにアプリケーションがルーティングと認証を持つ構成である。
そのため、API Gateway を外すと決めたあとに、B と C のどちらにするかを決めることになる。
C が B より有利になるのは、B の制約にぶつかる場面である。

- **書き込み系のリクエストを、署名ヘッダーを付けられないクライアントから受けるとき。** B の OAC では、`PUT` と `POST` のたびにクライアントが `x-amz-content-sha256` を付ける必要がある。ブラウザのフォーム送信や外部サービスの Webhook では付けられない。B でも、OAC の代わりに Lambda@Edge で署名すれば CloudFront 側で署名できるが、ボディが 1 MB までになり、Lambda@Edge という部品と、リクエストごとの費用と実行時間が加わる。C では CloudFront の OAC がボディを署名に含めずに署名するので、追加の部品なしにこの制約がなくなる。
- **アプリケーションが `Authorization` ヘッダーのトークン（JWT など）で認証しているとき。** B の OAC では `Authorization` ヘッダーが CloudFront の署名に使われるので、閲覧者のトークンはアプリケーションまで届かない。トークンを Cookie や別のヘッダーに移す必要がある。C の OAC は署名を別のヘッダーに入れ、閲覧者の `Authorization` ヘッダーをそのまま渡す。この署名の動作は Web Functions のオリジンでしか使えない[^oac-web]。
- **I/O 待ちの長いリクエストが多いとき。** たとえば、1 リクエストで生成 AI の応答を 10 秒待ち、そのあいだ CPU をほとんど使わない処理を考える。B では一つの実行環境がそのリクエスト専用になり、10 秒 × メモリ量がそのまま課金される。同時に 100 リクエスト来れば、実行環境も 100 必要になる。C では待ち時間の CPU は課金されず、一つの実行環境が複数のリクエストを同時に処理するので、少ない実行環境で済む。
- **アクセスが急に増えるとき。** B は同時リクエスト数と同じ数の実行環境を必要とするので、急増するとコールドスタートが重なり、アカウントの同時実行数の上限（既定 1,000）にも早く達する。C は動いている実行環境を使い回すので、新しい実行環境を作る回数が少ない。
- **カナリアリリースやマルチリージョン配置を手軽にしたいとき。** B でも関数のエイリアスの重み付けでカナリアリリースはできるが、マルチリージョン配置は各リージョンでの構築と Route 53 での振り分けを自前で作ることになる。C はエンドポイントの設定だけで済む。

逆に、次の場合は B のままにする。

- Node.js 以外のランタイムで書いている。
- 2 GB を超えるメモリや、CPU を使い続ける処理が要る。CPU を使い続ける処理では、C の課金の利点もほとんどない。
- CloudFront のオリジン側で Lambda@Edge を使っている。
- 使っている IaC ツールや監視ツールが、まだ Web Functions に対応していない。

まとめると、Node.js で書かれ、I/O 待ちが長く、ブラウザや Webhook から書き込みを受ける Web アプリケーションなら C が有利である。
それ以外では、B から C に移す理由は小さい。

[^webfn]: [Announcing AWS Lambda Web Functions](https://aws.amazon.com/blogs/aws/announcing-aws-lambda-web-functions-for-fast-consistent-web-app-performance-anywhere/) と [Lambda クォータ](https://docs.aws.amazon.com/lambda/latest/dg/gettingstarted-limits.html) の Lambda Web Functions の節を参照。本ファイルの C の記述は 2026 年 10 月時点の情報である。
[^oac]: CloudFront の OAC と関数 URL の設定手順は [Restrict access to an AWS Lambda function URL origin](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/private-content-restricting-access-to-lambda.html) を参照。
[^oac-web]: Web Functions の OAC は関数 URL と設定が異なる。手順は [Restrict access to an AWS Lambda Web Function origin](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/private-content-restricting-access-to-lambda-web.html) を参照。
[^edge-sign]: 構成例は [Protecting an AWS Lambda function URL with Amazon CloudFront and Lambda@Edge](https://aws.amazon.com/blogs/compute/protecting-an-aws-lambda-function-url-with-amazon-cloudfront-and-lambdaedge/) を参照。ボディの切り詰めは [Restrictions on Lambda@Edge](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/lambda-at-edge-function-restrictions.html) を参照。
[^price]: 単価はリージョンと利用量の段階によって変わる。最新の値は [Amazon API Gateway の料金](https://aws.amazon.com/api-gateway/pricing/) を参照。
[^price-web]: 単価は [AWS Lambda の料金](https://aws.amazon.com/lambda/pricing/) の Web Functions のタブを参照。
