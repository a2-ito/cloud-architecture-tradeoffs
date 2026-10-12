# Lambda で Web アプリケーションを公開するときの入口

Lambda で動く Web アプリケーションに、CloudFront の後ろで HTTP リクエストを渡す方法を比べる。
候補は **API Gateway 経由** と **Lambda 関数 URL（Function URL）経由** の二つである。

本ファイルは [README](../README.md) の「クラウドごとの構成」の代わりに、AWS 内の二つの構成を並べる。
Google Cloud と Cloudflare の対応する構成は、別ファイルで扱う。

## 1. 解きたい課題と前提

Lambda を HTTP で呼び出すには、HTTPS のエンドポイントをどこかに用意する必要がある。
その役を API Gateway に持たせるか、Lambda 自身が持つ関数 URL に持たせるかで、使える機能と費用が変わる。

前提は次のとおりとする。

- 利用者からのリクエストは、どちらの構成でも CloudFront で受ける。独自ドメイン、TLS 証明書、WAF、静的ファイルのキャッシュは CloudFront 側に置く。
- Lambda 上のアプリケーションは、Express や FastAPI のような Web フレームワークで書かれた一つの関数（いわゆる Lambdalith）でもよいし、エンドポイントごとに分かれた複数の関数でもよい。
- WebSocket のような双方向通信は対象外とする（関数 URL では実現できず、API Gateway の WebSocket API を使うしかないため）。

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

## 3. トレードオフ

### 機能の差

| 観点 | A: API Gateway (REST) | A: API Gateway (HTTP) | B: 関数 URL |
| --- | --- | --- | --- |
| ルーティング | パスとメソッド単位で複数の関数へ | パスとメソッド単位で複数の関数へ | 1 URL = 1 関数（振り分けは CloudFront かアプリ内） |
| 認証と認可 | IAM / Cognito / Lambda オーソライザー | IAM / JWT / Lambda オーソライザー | アプリ内で実装（`AWS_IAM` は OAC に使うため） |
| リクエスト検証 | スキーマによる検証あり | なし | なし（アプリ内） |
| スロットリング | ステージとメソッド単位、使用量プランと API キー | ステージとルート単位 | 予約同時実行数のみ（CloudFront 側の WAF レートベースルールで補う） |
| AWS WAF の直接適用 | 可 | 不可 | 不可（CloudFront に適用） |
| キャッシュ | API Gateway キャッシュあり | なし | なし（CloudFront でキャッシュ） |
| 統合タイムアウト | 既定 29 秒、Regional / private はクォータ申請で延長可、ストリーミング時は最大 15 分 | 最大 30 秒 | Lambda のタイムアウト（最大 15 分） |
| リクエストサイズ | 10 MB（Lambda 側の 6 MB 制限が先に効く） | 10 MB（同上） | 6 MB |
| レスポンスストリーミング | 可（2025 年 11 月から） | 不可 | 可（最大 200 MB） |
| 変換とモック | マッピングテンプレート、モック統合あり | パラメータマッピングのみ | なし |

関数 URL の列が空いているように見えるのは、その機能を API Gateway が担っていた部分を、アプリケーションか CloudFront に移す構成だからである。
認証、入力検証、ルーティングは Web フレームワークがもともと持っていることが多い。
アプリケーションが Lambdalith であれば、API Gateway の機能の多くは使われずに終わる。

### 性能

- A は CloudFront と Lambda のあいだに API Gateway のホップが一つ増える。その分のレイテンシが毎リクエストに加わる。
- コールドスタートの影響は両者で変わらない。
- 長時間処理の扱いは、どちらも CloudFront の **オリジン応答タイムアウト** に縛られる。この値は既定 30 秒で、設定で 120 秒まで、クォータ申請で 180 秒まで延ばせる。B の関数 URL は Lambda の 15 分まで待てるが、CloudFront を通す限り、オリジンから応答が届かない時間がこのタイムアウトを超えると 504 になる。ストリーミングで少しずつ返せば、この制限は回避しやすい。
- HTTP API は 30 秒が上限で延長できない。生成 AI の応答のように時間のかかる処理には、REST API のストリーミングか B を選ぶことになる。

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

### 運用負荷

- A は API Gateway のリソース（ステージ、デプロイ、ルート、オーソライザー）を IaC で管理する必要がある。ルートを追加するたびに API Gateway 側の定義も変える構成だと、アプリケーションと API 定義の二か所を同期させ続けることになる。
- A では API Gateway のアクセスログとメトリクス（4xx / 5xx、レイテンシ、統合レイテンシ）が手に入る。B では Lambda の関数 URL 用メトリクス（`UrlRequestCount`、`Url4xxCount`、`Url5xxCount` など）と CloudFront のログで代替する。
- B で OAC を使う場合、`PUT` と `POST` のリクエストでは、クライアントがボディの SHA256 を計算して `x-amz-content-sha256` ヘッダーに入れる必要がある。Lambda は署名なしのペイロードを受け付けないためである。ブラウザのフォーム送信や、ヘッダーを制御できない外部サービスからの Webhook では、この制約がそのまま障害になる。回避するには、クライアント側でハッシュを付ける実装を入れるか、OAC を外して秘密ヘッダーによる検証に切り替える（その場合、関数 URL は認証タイプ `NONE` で公開されることになる）。
- B ではレート制限を API Gateway に任せられないので、CloudFront の WAF レートベースルールと予約同時実行数で守ることになる。予約同時実行数は関数全体の上限であり、特定のクライアントだけを絞ることはできない。

### ロックイン

- A は API Gateway の定義（オーソライザー、マッピングテンプレート、使用量プラン）にロジックを置くほど、AWS 以外への移行が難しくなる。
- B はルーティングと認証がアプリケーションのコードにあるため、Lambda Web Adapter などを使っていれば、同じアプリケーションをコンテナにして ECS や Cloud Run に移しやすい。
- どちらも Lambda と CloudFront には依存する。ロックインの差は、API Gateway 固有の設定にどれだけ依存するかで決まる。

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

A を選ぶ場合、REST API と HTTP API の選択は、使用量プラン、WAF の直接適用、リクエスト検証、ストリーミング、29 秒超のタイムアウトのどれかが要るかで決まる。
どれも要らなければ HTTP API のほうが安い。

[^oac]: CloudFront の OAC と関数 URL の設定手順は [Restrict access to an AWS Lambda function URL origin](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/private-content-restricting-access-to-lambda.html) を参照。
[^price]: 単価はリージョンと利用量の段階によって変わる。最新の値は [Amazon API Gateway の料金](https://aws.amazon.com/api-gateway/pricing/) を参照。
