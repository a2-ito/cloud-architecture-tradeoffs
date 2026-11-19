# Lambda を内部 API として呼び出すときの入口

社内のほかのシステムからだけ呼び出す API（内部 API）を Lambda で作るとき、その入口をどうするかを比べる。
候補は **PrivateLink + API Gateway（プライベート REST API）**、**Lambda Web Functions**、**Lambda 関数 URL（Function URL）**、**内部 ALB（Application Load Balancer）** の四つである。

利用者向けに CloudFront の後ろで公開する場合の比較は [Lambda で Web アプリケーションを公開するときの入口](lambda-web-app-entrypoint.md) にある。
本ファイルは、CloudFront を置かず、呼び出し元がサーバー側のシステムに限られる場合を扱う。
Web Functions の仕組みと料金体系の詳細もそちらに書いたので、本ファイルでは内部 API として使うときに効く差に絞る。

本ファイルは [README](../README.md) の「クラウドごとの構成」の代わりに、AWS 内の四つの構成を並べる。

## 1. 解きたい課題と前提

内部 API に求められることは、主に次の二つである。

- 決められた呼び出し元以外からは呼べないこと。
- API のエンドポイントがインターネットから到達できないこと（VPC のプライベートアドレスでしか待ち受けないこと）。

一つ目は認証と認可（誰が呼べるか）の問題で、二つ目はネットワーク（どこから届くか）の問題である。
四つの構成の最大の違いは、二つ目をネットワークの仕組みで満たせるかどうかにある。
問題にするのは呼び出し元からの通信経路ではなく、エンドポイントそのものがインターネットから届くかどうかである。

前提は次のとおりとする。

- 呼び出し元は、同じ AWS アカウントか別アカウントの VPC 内で動くサーバー（ECS のタスク、EC2、VPC に接続した Lambda など）を主に想定する。VPC につながったネットワーク（VPC ピアリング、Transit Gateway、Direct Connect / VPN 経由のオンプレミス）からの呼び出しも含める。VPC の外で動く呼び出し元（VPC に接続していない Lambda、社外の SaaS など）がある場合は、その違いも書く。
- 呼び出し元は AWS の認証情報（IAM ロール）を持ち、リクエストに SigV4 署名を付けられる。
- WebSocket のような双方向通信は対象外とする。

## 2. 構成

### A. PrivateLink + API Gateway（プライベート REST API）+ Lambda

```
VPC 内の呼び出し元 ─▶ インターフェイス VPC エンドポイント（execute-api）─▶ API Gateway（Private REST API）─▶ Lambda
```

- API Gateway のエンドポイントタイプを **Private** にした REST API を作る。プライベート API は REST API でしか作れず、HTTP API では作れない。
- 呼び出し元の VPC に `com.amazonaws.<region>.execute-api` のインターフェイス VPC エンドポイントを作る。プライベート API はこの VPC エンドポイントを通してしか呼べず、インターネットからは到達できない。
- プライベート API にはリソースポリシーが必須で、ないとデプロイに失敗する。`aws:SourceVpce` または `aws:SourceVpc` を条件にし、呼び出してよい VPC エンドポイントや VPC を明示的に許可する[^private-api]。VPC エンドポイント側にもエンドポイントポリシーを付けられる。
- 誰が呼べるかは、ネットワークの制限とは別に、IAM 認証、Lambda オーソライザー、API キーなどで絞れる。リソースポリシーでネットワーク側の制限がかかっているので、認証を付けない運用もありうる。
- 別アカウントの VPC からも、そのアカウントに VPC エンドポイントを作り、API のリソースポリシーで許可すれば呼べる。
- オンプレミスからも、Direct Connect か VPN で VPC につなげば VPC エンドポイント経由で呼べる。オンプレミスで名前解決するには、Route 53 Resolver のインバウンドエンドポイントを置くか、VPC エンドポイント固有の DNS 名を使う。

### B. Lambda Web Functions

```
呼び出し元 ─(SigV4)─▶ Web Function エンドポイント（公開）─▶ Web Function（HTTP サーバー）
```

- エンドポイントの認証タイプを `IamAuth` にし、呼び出し元の IAM ロールと Web Function のリソースベースポリシーで `lambda:InvokeWebFunctionEndpoint` を許可する[^web-auth]。
- エンドポイントは Lambda が管理する公開の HTTPS エンドポイントだけで、VPC 内のプライベートアドレスでは待ち受けない。AWS のドキュメントも、`IamAuth` は呼び出せる主体を制御するが、エンドポイントをネットワーク層でプライベートにはしないと明記している[^web-net]。Web Function エンドポイントのドメイン名には、PrivateLink のインターフェイスエンドポイントも用意されていない。
- Web Functions の「VPC 接続」は、関数から VPC 内の RDS などへ出ていくための送信側の仕組み（egress network connector）である。これを付けても入口はプライベートにならない。また、egress network connector は `HomeRegion` エンドポイントでしか使えない。
- VPC 内の呼び出し元から呼ぶには、NAT ゲートウェイかインターネットゲートウェイを通って公開エンドポイントに出る経路が要る。

### C. Lambda 関数 URL + Lambda

```
呼び出し元 ─(SigV4)─▶ Lambda 関数 URL（公開）─▶ Lambda
```

- 関数 URL の認証タイプを `AWS_IAM` にし、呼び出し元の IAM ロールと関数のリソースベースポリシーで `lambda:InvokeFunctionUrl` と `lambda:InvokeFunction` を許可する。
- 関数 URL も `https://<url-id>.lambda-url.<region>.on.aws` 形式の公開エンドポイントだけで、プライベートにする設定はない。
- Lambda のインターフェイス VPC エンドポイント（`com.amazonaws.<region>.lambda`）で VPC 内から非公開で呼べるのは、`Invoke` などの Lambda API である[^lambda-vpce]。同ページが対象に挙げているのは Lambda API だけで、関数 URL のドメインは含まれていない。そのため、関数 URL を呼ぶ通信は B と同じく公開エンドポイントへ出る。

### D. 内部 ALB + Lambda

```
VPC 内の呼び出し元 ─▶ 内部 ALB（スキーム internal）─(リスナールール)─▶ ターゲットグループ ─▶ Lambda
```

- スキームを `internal` にした ALB を作り、Lambda 関数をターゲットグループに登録する。ALB はプライベート IP アドレスでしか待ち受けず、インターネットからは到達できない。
- 呼び出し元の制限は、ALB のセキュリティグループで送信元（セキュリティグループや CIDR）を絞って行う。IAM による認証の仕組みはない。
- 呼び出し元の認証が要る場合は、HTTPS リスナーで次のどれかを使う。ALB の OIDC / Cognito 認証はブラウザの利用者向けで、サーバー間の呼び出しには向かない。
  - **JWT の検証**: リスナールールで、ALB 自体に JWT を検証させる。サーバー間（マシン間）の通信向けの機能で、`iss` と `exp` の検証は必須、ほかのクレームも 10 個まで検証できる[^alb-jwt]。
  - **相互 TLS（mTLS）**: trust store でクライアント証明書を検証する verify モードと、証明書をヘッダーでターゲットに渡す passthrough モードがある[^alb-mtls]。
  - **アプリケーションでの認証**: Lambda のコードでトークンなどを検証する。
- リスナールールでパス、ホスト、ヘッダーごとに別のターゲットグループに振り分けられる。ターゲットグループ一つに登録できる Lambda 関数は一つである。
- ALB は Lambda を直接呼び出し、関数が応答するかタイムアウトするまで待つ。リクエストとレスポンスはそれぞれ 1 MB まで、WebSocket もレスポンスストリーミングも使えない。Lambda 関数とターゲットグループは同じアカウント、同じリージョンに置く[^alb-lambda]。
- HTTPS で受けるには、独自ドメインと ACM の証明書を用意し、Route 53 のプライベートホストゾーンで ALB に名前を向ける。VPC 内に閉じた API なので、HTTP リスナーで済ませる運用もある。ただしその場合、通信は暗号化されず、JWT の検証も mTLS も使えない。
- 別アカウントの VPC やオンプレミスからは、VPC ピアリング、Transit Gateway、Direct Connect / VPN でつなげば呼べる。ネットワークをつなげない別アカウントに見せるには、ALB を NLB のターゲットにして VPC エンドポイントサービス（PrivateLink）として公開する。
- AWS WAF を ALB に直接付けられる。
- ヘルスチェックは既定で無効である。有効にすると、ヘルスチェックのたびに関数が呼び出されて課金される。
- 関数は、エイリアスの ARN で登録することが推奨されている。

## 3. トレードオフ

### 機能の差

| 観点                       | A: PrivateLink + API Gateway                                                                                                                                               | B: Web Functions                                                                                                                                 | C: 関数 URL                                       | D: 内部 ALB                                                                                                       |
| -------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------- |
| ネットワークで閉じられるか | 閉じられる（VPC エンドポイント経由でしか届かない）                                                                                                                         | 閉じられない（公開エンドポイントのみ）                                                                                                           | 閉じられない（公開エンドポイントのみ）            | 閉じられる（プライベート IP でしか待ち受けない）                                                                  |
| 呼び出し元の制限           | リソースポリシーの `aws:SourceVpce` / `aws:SourceVpc`、IAM、オーソライザー                                                                                                 | `IamAuth`（SigV4）とリソースベースポリシー                                                                                                       | `AWS_IAM`（SigV4）とリソースベースポリシー        | セキュリティグループ（送信元の SG や CIDR）、ALB の JWT 検証、mTLS、アプリ内の認証。IAM 認証はない                |
| VPC 内からの通信経路       | VPC エンドポイントを通り、AWS のネットワーク内で完結する                                                                                                                   | NAT ゲートウェイかインターネットゲートウェイを通って公開エンドポイントへ（VPC 接続の有無にかかわらず、エンドポイントはインターネットからも届く） | B と同じ                                          | VPC 内で ALB に直接届く                                                                                           |
| VPC の外の呼び出し元       | VPC につながったネットワーク（ピアリング、Transit Gateway、Direct Connect / VPN 経由のオンプレミス）からは呼べる。VPC に接続していない Lambda や社外の SaaS からは届かない | 呼べる                                                                                                                                           | 呼べる                                            | A と同じ。ネットワークをつなげない別アカウントには、NLB を前に置いて PrivateLink で公開する（NLB の費用が加わる） |
| 呼び出し側の実装           | 普通の HTTP クライアント（IAM 認証を付けるなら SigV4 署名。プライベート DNS を無効にした場合は `Host` か `x-apigw-api-id` ヘッダーも要る）                                 | SigV4 署名が必須                                                                                                                                 | SigV4 署名が必須                                  | 普通の HTTP クライアント（mTLS を使うならクライアント証明書）                                                     |
| ルーティング               | パスとメソッド単位で複数の関数へ                                                                                                                                           | 1 エンドポイント = 1 Web Function（振り分けはアプリ内）                                                                                          | 1 URL = 1 関数（振り分けはアプリ内）              | リスナールールでパス、ホスト、ヘッダーごとに複数の関数へ                                                          |
| リクエスト検証・変換       | スキーマによる検証、マッピングテンプレート                                                                                                                                 | なし（アプリ内）                                                                                                                                 | なし（アプリ内）                                  | なし（アプリ内）                                                                                                  |
| スロットリング             | ステージとメソッド単位、使用量プランと API キー                                                                                                                            | エンドポイント単位のレート制限と実行環境数の上限                                                                                                 | 予約同時実行数のみ                                | なし（予約同時実行数と、WAF のレートベースルールで補う）                                                          |
| タイムアウト               | 既定 29 秒（クォータ申請で延長可）                                                                                                                                         | リクエストタイムアウト（3〜900 秒で設定）                                                                                                        | Lambda のタイムアウト（最大 15 分）               | Lambda のタイムアウト（最大 15 分）。ALB は関数の応答かタイムアウトまで待つ                                       |
| リクエストサイズ           | 10 MB（Lambda 側の 6 MB 制限が先に効く）                                                                                                                                   | ヘッダーは 128 KiB。ボディの上限はドキュメントに記載がない                                                                                       | 6 MB                                              | リクエスト、レスポンスとも 1 MB                                                                                   |
| レスポンスストリーミング   | 可（プロキシ統合）                                                                                                                                                         | 可                                                                                                                                               | 可（`RESPONSE_STREAM`）                           | 不可                                                                                                              |
| ランタイムと実行環境の制約 | なし                                                                                                                                                                       | Node.js 24（Arm64）のみ、2 vCPU / 2 GB 固定、対応リージョンが限られる                                                                            | なし                                              | なし                                                                                                              |
| 一つの実行環境での同時処理 | しない（1 実行環境 = 1 リクエスト）                                                                                                                                        | する（既定 64、最大 128）                                                                                                                        | しない                                            | しない                                                                                                            |
| アクセスログとメトリクス   | API Gateway のアクセスログと実行ログ、4xx / 5xx とレイテンシのメトリクス                                                                                                   | エンドポイント単位のリクエスト数やスロットリング数のメトリクス                                                                                   | 関数 URL 用のメトリクス（`UrlRequestCount` など） | ALB のアクセスログ、ターゲットごとのリクエスト数やエラー数のメトリクス                                            |

### Lambda を VPC に接続しても入口は閉じない

Lambda 関数を VPC に接続すれば、関数 URL も VPC の中に入るように見えるが、そうはならない。
VPC 接続で作られるネットワークインターフェイスは、関数から VPC 内の RDS などへ出ていくための送信用である。
関数は VPC のプライベート IP アドレスで待ち受けず、関数 URL への通信は VPC 接続の有無にかかわらず公開エンドポイントに届く。
B の egress network connector も同じで、入口には影響しない[^web-net]。
入口をネットワークで閉じたいなら、A か D の構成か、後述の `Invoke` API を VPC エンドポイント経由で呼ぶ方法を選ぶ。

B と C の違いは、[公開する場合の比較](lambda-web-app-entrypoint.md) と同じく、実行環境と課金のしかたにある。
内部 API としての立ち位置（公開エンドポイントを IAM で守る）は、B と C で変わらない。

### ネットワークで閉じることの意味

B と C も、`IamAuth` / `AWS_IAM` にしておけば、署名のないリクエストや許可されていない主体からのリクエストは 403 で拒否される。
認可の強さだけを見れば、A、B、C に大きな差はない。
D は IAM 認証を持たないので、ネットワークで閉じる代わりに、呼び出し元の認証はセキュリティグループ、mTLS、アプリケーションに任せることになる。

差が出るのは、次のような場面である。

- **セキュリティ要件や監査で「内部 API はインターネットから到達できないこと」が求められる。** これを満たせるのは A と D である。B と C の URL はインターネットから到達できるので、この要件は満たせない。IAM で拒否されることを根拠に説明することになり、通るかどうかは監査する側の判断による。
- **認証情報が漏れた。** A と D では、漏れた認証情報を使っても VPC の外からは届かない。ただし D をセキュリティグループだけで守っている場合、許可した送信元のホストが乗っ取られると、認証なしで呼ばれてしまう。B と C では、漏れた認証情報があればどこからでも呼べる。C では関数のリソースベースポリシーの条件（`aws:SourceIp` など）で補えるが、NAT ゲートウェイの IP アドレスを管理し続けることになる。B のリソースベースポリシーで同じ条件が使えるかは、執筆時点で確認できていない。
- **呼び出し元が VPC を持たない。** A と D は VPC と、VPC につながったネットワークからしか呼べないので、VPC に接続していない Lambda や社外の SaaS から呼ぶ API には使えない。B と C はそのまま呼べる。

### 性能

- A は呼び出し元と Lambda のあいだに VPC エンドポイントと API Gateway のホップが入る。B と C はエンドポイントが Lambda 自身にある。
- B と C は VPC 内から呼ぶと NAT ゲートウェイを通る。NAT ゲートウェイの帯域とホップが加わるが、一般的な内部 API で問題になることは少ない。
- D は呼び出し元と Lambda のあいだに ALB のホップが一つ入る。A のような、VPC エンドポイントと API Gateway の二つのホップはない。
- コールドスタートの扱いは A、C、D で同じである。B は一つの実行環境が複数のリクエストを同時に処理するので、アクセスが急に増えても新しい実行環境を作る回数が少ない。

### コスト

Lambda 自体の料金は A、C、D で共通である。
差は、A の VPC エンドポイントと API Gateway の料金、D の ALB の料金、B と C で VPC から外に出るための NAT ゲートウェイの料金にある。

東京リージョンの執筆時点の単価は次のとおりである[^price]。

| 項目                                             | 単価                                                          | 1 か月（730 時間）の固定費の目安 |
| ------------------------------------------------ | ------------------------------------------------------------- | -------------------------------- |
| インターフェイス VPC エンドポイント（A）         | 0.014 USD / 時間 / AZ、データ処理 0.01 USD / GB（最初の段階） | 3 AZ で約 30.7 USD               |
| API Gateway REST API（A）                        | 4.25 USD / 100 万リクエスト（最初の段階）                     | なし（従量のみ）                 |
| ALB（D）[^alb-price]                             | 0.0243 USD / 時間、LCU 0.008 USD / 時間                       | 約 17.7 USD（LCU の分を除く）    |
| NAT ゲートウェイ（B と C を VPC 内から呼ぶ場合） | 0.062 USD / 時間 / AZ、データ処理 0.062 USD / GB              | 3 AZ で約 135.8 USD              |

- A は、リクエストがなくても VPC エンドポイントの固定費がかかる。VPC エンドポイント一つで同じ VPC から複数のプライベート API を呼べるので、API が増えても固定費は増えない。
- A は API Gateway のリクエスト料金がかかる。月 1 億リクエストなら約 425 USD で、リクエスト数が多いほど B、C、D との差が開く。
- B と C は、呼び出し元の VPC にすでに NAT ゲートウェイがあれば、追加の固定費はない。NAT ゲートウェイがない VPC から呼ぶためにわざわざ置くなら、固定費は A の VPC エンドポイントや D の ALB より高くつく。
- B と C で NAT ゲートウェイを通る通信には、データ処理料金がかかる。GB あたりの単価は A の VPC エンドポイントの約 6 倍なので、転送量が多い API ほど差が開く。
- D は、ALB の時間料金に加えて、LCU（ロードバランサーの処理量の単位）の料金がかかる。LCU は 1 時間の平均の新規接続数、アクティブ接続数、処理したデータ量、ルール評価数のうち最も多いもので決まる。Lambda をターゲットにした場合の処理データ量は、1 LCU あたり 0.4 GB / 時間である[^alb-price]。リクエスト数ではなくデータ量で課金されるので、小さなリクエストが多い内部 API では A より大幅に安い。
- D の試算の例として、月 1 億リクエスト（平均約 38 件 / 秒）で、ヘッダーを含むリクエストとレスポンスの合計が 1 件 2 KB、接続を使い回す場合を考える。処理データ量は約 0.27 GB / 時間で約 0.68 LCU（月約 4 USD）になり、時間料金と合わせて月 22 USD 前後である。A の API Gateway のリクエスト料金だけで約 425 USD かかるのと比べると、差は大きい。
- この試算は前提で変わる。接続を使い回さないと新規接続数で LCU が決まり（約 1.5 LCU、月約 9 USD の増加）、ルールが 10 を超えるとルール評価数も LCU に加わる。ピークのある時間帯は、その時間の平均で LCU が増える。
- D で mTLS を使う場合は、trust store に 0.0054 USD / 時間（月約 3.9 USD）がかかる。また、1 LCU あたりのアクティブ接続数が 3,000 / 分から 1,500 / 分に減り、ALB がヘッダーに入れる証明書の情報も処理データ量に数えられる。
- D の ALB は、一つで複数の関数（リスナールールで振り分け）を受けられる。API が増えても ALB の時間料金は増えない。
- B と C の違い（B は実行環境が残っているあいだもメモリに課金され、C は無料枠がある）は、[公開する場合の比較](lambda-web-app-entrypoint.md) のコストの節を参照。

### 運用負荷

- A は、プライベート DNS を有効にした `execute-api` の VPC エンドポイントがある VPC から、**ほかの公開 API Gateway をデフォルトのドメイン名（`*.execute-api.<region>.amazonaws.com`）で呼べなくなる**[^private-api]。公開 API にカスタムドメイン名を付けるか、プライベート DNS を無効にしてプライベートホストゾーンを自分で管理する必要がある。同じ VPC で外部の API Gateway を呼んでいる場合は、導入前に影響を確認する。
- A は、VPC エンドポイント、セキュリティグループ、API のリソースポリシーの三つを揃えないと呼べない。呼べないときに確認する箇所が多い。
- A でプライベート API に独自ドメインを付けるには、プライベートカスタムドメイン名の設定が別に要る。
- D は、ALB、リスナー、ターゲットグループ、セキュリティグループ、（HTTPS にするなら）証明書とプライベートホストゾーンを揃える。部品は多いが、どれも EC2 や ECS の前に ALB を置く場合と同じで、経験のあるチームには扱いやすい。
- D は、Lambda が受け取るイベントの形が ALB 用である。API Gateway や関数 URL 用に書かれたアプリケーションをそのまま載せると、イベントの形の違い（マルチバリューヘッダーなど）で動かないことがある。Lambda Web Adapter や、ALB に対応したフレームワークのアダプターを使う。
- D は、レスポンスが 1 MB を超えると、関数が正常に終わっても ALB が 502 を返す。大きなレスポンスを返す API は、S3 の署名付き URL を返すなどの設計にする。
- B と C は、呼び出し元のコードに SigV4 署名の処理が要る。AWS SDK の署名機能や、署名を付ける HTTP クライアントのライブラリを使う。呼び出し元の言語やフレームワークによっては、この部分を自前で書くことになる。
- B と C は、関数側のリソースベースポリシーで呼び出し元の IAM ロールを許可するだけで済み、ネットワークの設定は要らない。
- B は Web Functions 固有の制約（Lambda Extensions が使えない、ロググループが無期限保持で作られるなど）がある。詳細は [公開する場合の比較](lambda-web-app-entrypoint.md) の運用負荷の節を参照。

### ロックイン

- A は API Gateway のリソースポリシー、オーソライザー、マッピングテンプレートに依存した分だけ移しにくい。ネットワークで閉じる仕組み自体は、ほかのクラウドの内部ロードバランサーやプライベートエンドポイントに置き換えられる。
- B と C は IAM の SigV4 署名に依存する。ほかのクラウドに移すと、呼び出し元の認証の仕組みを作り直すことになる。
- D は ALB のイベントの形に依存するが、ALB 自体は ECS や EC2 の前にも置けるので、関数をコンテナに移すときに入口を作り直さなくて済む。
- アプリケーションのコード自体は、普通の HTTP サーバーとして書く B が最もコンテナに移しやすい。D も Lambda Web Adapter を使って普通の HTTP サーバーとして書いていれば同じく移しやすく、入口の ALB もそのまま使える。

## 4. 選ぶ基準

A か D（ネットワークで閉じる構成）が向くのは次のような場合である。

- 内部 API がインターネットから到達できないことを、セキュリティ要件や監査で求められている。
- 呼び出し元がすべて VPC か、それにつながったネットワークにある。
- 認証情報が漏れたときの被害を、ネットワークでも抑えたい。

そのうえで、A（PrivateLink + API Gateway）が向くのは次のような場合である。

- ネットワークで閉じたうえで、IAM による認可も組み合わせたい（D には IAM 認証がない）。
- 使用量プラン、リクエスト検証など、API Gateway の機能を使う。
- リソースポリシーで、呼び出してよい VPC エンドポイントや VPC を API ごとに限定したい。

D（内部 ALB）が向くのは次のような場合である。

- リクエスト数が多く、API Gateway のリクエスト料金を払いたくない。
- 呼び出し元の認証は、セキュリティグループ、ALB の JWT 検証、mTLS、アプリケーションの認証で賄える。IAM による認可は要らない。
- 同じ VPC に、ECS や EC2 の前に置いた内部 ALB がすでにあり、そこに Lambda のターゲットを足せる。
- リクエストとレスポンスが 1 MB に収まり、レスポンスストリーミングも要らない。

ネットワークで閉じることと IAM による認可を両立する選択肢には、ほかに VPC Lattice もある。本ファイルでは扱わない。

B（Web Functions）が向くのは次のような場合である。

- IAM による認可で十分で、ネットワークで閉じることは求められていない。
- アプリケーションが Node.js の Web フレームワークで書かれていて、2 vCPU / 2 GB の実行環境に収まる。
- 生成 AI の呼び出しの中継のように、1 リクエストの待ち時間が長く、同時に処理しているリクエストが多い。

C（関数 URL）が向くのは次のような場合である。

- IAM による認可で十分で、ネットワークで閉じることは求められていない。
- 呼び出し元に VPC を持たないもの（VPC に接続していない Lambda、CI のジョブ、社外の SaaS など）が含まれる。
- リクエスト数が少なく、Lambda の無料枠に収まる程度で、固定費をかけたくない。
- B では使えない Node.js 以外のランタイムを使う。

B と C を選ぶ場合も、呼び出し元の VPC に NAT ゲートウェイがないなら、その費用を含めて A や D と比べる。

### HTTP の形にこだわらない場合

呼び出し元が AWS SDK を使えて、HTTP の API である必要がないなら、Lambda のインターフェイス VPC エンドポイント（`com.amazonaws.<region>.lambda`）を通して `Invoke` API で関数を直接呼ぶ方法もある[^lambda-vpce]。

```
VPC 内の呼び出し元 ─▶ インターフェイス VPC エンドポイント（lambda）─▶ Lambda（Invoke API）
```

- A と同じく、通信は VPC エンドポイントを通り、Lambda API をインターネットに公開された経路で呼ぶ必要がない。
- API Gateway のリクエスト料金がかからず、固定費は VPC エンドポイントの分だけである。
- 呼び出し元は HTTP のパスやメソッドではなく、関数名とイベントの JSON で呼ぶ。呼び出し元が Lambda の呼び出し方に依存するので、あとから HTTP の API に変えると呼び出し元の修正が要る。
- ペイロードは同期呼び出しで 6 MB までである。

社内のバッチやワーカーから特定の処理を呼ぶだけなら、A より安く、D と比べても API Gateway や ALB の部品が要らない分だけ単純である。
複数のチームやシステムが呼ぶ API として契約（パスやスキーマ）を見せたい場合は、A か D のほうが扱いやすい。

[^private-api]: [Private REST APIs in API Gateway](https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-private-apis.html) を参照。リソースポリシーが必須である点は [Best practices for designing Amazon API Gateway private APIs and private integration](https://docs.aws.amazon.com/whitepapers/latest/best-practices-api-gateway-private-apis-integration/rest-api.html) を参照。プライベート DNS を有効にすると公開 API のデフォルトのエンドポイントを呼べなくなる点も、同ページのベストプラクティスに記載がある。

[^web-auth]: [Lambda Web Function endpoint access control](https://docs.aws.amazon.com/lambda/latest/dg/web-functions-endpoint-access-control.html) を参照。認証タイプは `IamAuth` と `ApplicationManaged` の二つである。

[^web-net]: [Configuring networking for AWS Lambda Web Functions](https://docs.aws.amazon.com/lambda/latest/dg/web-functions-networking.html) の Inbound connectivity の節を参照。本ファイルの B の記述は 2026 年 11 月時点の情報である。

[^lambda-vpce]: [Connecting inbound interface VPC endpoints for Lambda](https://docs.aws.amazon.com/lambda/latest/dg/configuration-vpc-endpoints.html) を参照。

[^price]: 単価はリージョンと利用量の段階によって変わる。最新の値は [AWS PrivateLink の料金](https://aws.amazon.com/privatelink/pricing/)、[Amazon API Gateway の料金](https://aws.amazon.com/api-gateway/pricing/)、[Amazon VPC の料金](https://aws.amazon.com/vpc/pricing/)（NAT ゲートウェイ）を参照。

[^alb-lambda]: [Use Lambda functions as targets of an Application Load Balancer](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/lambda-functions.html) の Limits を参照。

[^alb-price]: [Elastic Load Balancing の料金](https://aws.amazon.com/elasticloadbalancing/pricing/) の LCU の説明を参照。

[^alb-jwt]: [Verify JWTs using an Application Load Balancer](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/listener-verify-jwt.html) を参照。

[^alb-mtls]: [Mutual authentication with TLS in Application Load Balancer](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/mutual-authentication.html) を参照。
