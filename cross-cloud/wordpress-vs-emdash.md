# CMS のサイトをどこでどう動かすか（WordPress on EC2 / Lightsail と EmDash）

ブログやコーポレートサイトのような CMS のサイトを、どの CMS とどの基盤で動かすかを比べる。
軸は、AWS で動かす **WordPress**（EC2 か Lightsail）と、Cloudflare が公開した **EmDash** である。
あわせて、マネージド WordPress、ヘッドレス CMS などの代替も並べる。

EmDash は、Astro の上に TypeScript で書かれたオープンソース（MIT ライセンス）の CMS である。
Cloudflare は WordPress の「精神的な後継（spiritual successor）」と位置づけている[^emdash-blog]。
2026 年 4 月に v0.1.0 のプレビューとして公開され、2026 年 9 月 28 日に 1.0 が出た[^emdash-1.0]。
執筆時点の最新は 1.3.1（2026 年 10 月 18 日）である。

本ファイルは AWS と Cloudflare にまたがるため、`cross-cloud/` に置く。

## 1. 解きたい課題と前提

WordPress を EC2 で動かしていると、次のような負担が積み上がる。

- OS、PHP、MySQL、WordPress 本体、プラグインの更新を自分で追い続ける必要がある。
- WordPress の脆弱性の多くはプラグインに由来する（Cloudflare は 96% としている[^emdash-blog]）。プラグインは WordPress と同じプロセスで動き、DB やファイルに制限なく触れるため、一つの脆弱性がサイト全体に及ぶ。
- アクセスが急増したときのスケーリングとキャッシュを、自分で設計する必要がある。

前提は次のとおりとする。

- 対象は、記事やページを編集して公開する一般的な CMS のサイトとする。EC サイトや会員制サイトのような重い用途は、補足として触れるにとどめる。
- 月間のページビューは数万から数十万程度の、小規模から中規模のサイトとする。
- 料金は執筆時点の us-east-1（AWS）の単価と、Cloudflare の公開料金を使う[^price]。

## 2. 構成

### A. WordPress on EC2

```
Client ─▶ (CloudFront) ─▶ EC2（Apache/Nginx + PHP + WordPress）─▶ MySQL（同居 or RDS）
                                   └─ wp-content/uploads（EBS or S3 + プラグイン）
```

- 最小構成は、EC2 1 台に Web サーバー、PHP、MySQL をすべて載せる形である。
- 冗長化する場合は、EC2 を Auto Scaling グループにし、DB を RDS、アップロードファイルを EFS か S3 に分け、前段に ALB と CloudFront を置く。
- テーマ、プラグイン、ページビルダー、WooCommerce など、WordPress のエコシステム（プラグインは 6 万件以上）をそのまま使える。
- OS からアプリケーションまで、パッチ当てとバックアップはすべて自分で持つ。

### B. WordPress on Lightsail

```
Client ─▶ (Lightsail CDN) ─▶ Lightsail インスタンス（WordPress ブループリント）─▶ MySQL（同居 or Lightsail マネージド DB）
                                         └─ wp-content/uploads（ディスク or Lightsail バケット）
```

- Lightsail は、VM、ディスク、固定 IP、転送量をまとめて月額固定で売る AWS のサービスである。WordPress が入った状態のインスタンスを、ブループリント（テンプレート）から数クリックで作れる。
- 中身は普通の VM なので、WordPress のエコシステムは A と同じように使える。
- 冗長化する場合は、Lightsail のロードバランサー、マネージドデータベース（高可用性プラン）、バケット、CDN ディストリビューションを組み合わせる[^lightsail-ha]。
- 自動スナップショットを有効にすると、日次のスナップショットを 7 世代保持する。
- OS とアプリケーションの更新は、A と同じく自分で持つ。Lightsail が楽にするのは構築と料金の見通しであって、運用そのものではない。
- Auto Scaling はない。スケールアップはスナップショットから大きいプランのインスタンスを作り直し、スケールアウトはロードバランサーの後ろに手で台数を足す。
- 選べるプランは 2 vCPU のバースト可能なインスタンスが中心で、CPU を使い続けると性能が落ちる。重い用途には向かない。
- AWS の他のサービス（RDS や VPC 内のリソース）とは、VPC ピアリングでつなぐ必要がある。

Bitnami がパッケージした WordPress のブループリントは非推奨になった[^bitnami]。
2026 年 5 月 19 日で更新が止まり、2026 年 11 月 19 日以降は新しく作れなくなる。
これから作るなら Lightsail がパッケージした WordPress のブループリントを使う。
Bitnami 版で動かしている既存のインスタンスはそのまま動き続けるが、移行の手順が用意されている。

### C. EmDash on Cloudflare Workers

```
Client ─▶ Cloudflare Workers（Astro + EmDash）─▶ D1（DB）
                    │                        └─▶ R2（メディア）
                    └─▶ Dynamic Workers（プラグインごとの isolate）
```

- EmDash は Cloudflare のランタイム（workerd）の V8 isolate で動く。isolate はリクエストに応じて起動し、アクセスがなければゼロまで縮む。
- DB は D1、メディアは R2 に置く。
- プラグインは一つずつ別の isolate（Dynamic Workers）で動く。プラグインはマニフェストで宣言した権限（`read:content`、`email:send` など）しか使えず、外部への通信も宣言したホストに限られる。管理者は、インストールする前に宣言された権限を確認できる。
- プラグインのサンドボックスを使うには Workers の有料プランが必要である。無料プランでもサイトは動くが、サンドボックス型のプラグインは使えない[^review]。
- MCP サーバーと CLI が組み込まれており、AI エージェントから記事の作成やメディアのアップロードを行える。
- 1.0 と同時に、AT Protocol 上に分散型のプラグインレジストリが公開された[^emdash-1.0]。

### D. EmDash を Node.js でセルフホスト

```
Client ─▶ Node.js 22.16+（Astro + EmDash）─▶ SQLite / libSQL / Postgres
                    │                     └─▶ ローカルディスク or S3 互換ストレージ
                    └─▶ workerd（子プロセス、プラグインのサンドボックス）
```

- EmDash は Cloudflare 以外でのセルフホストも正式にサポートしている。EC2、Lightsail、VPS、Docker（`node:22` イメージ）などで動く[^review]。
- DB は SQLite、libSQL、Postgres から、メディアの置き場所はローカルディスクか S3 互換ストレージ（AWS S3、R2、MinIO など）から選べる。
- プラグインのサンドボックスは、workerd を子プロセスとして起動して実現する。workerd が 60 秒以内に 5 回続けてクラッシュすると再起動が止まり、サーバーを再起動するまでサンドボックス型のプラグインが動かなくなる。
- 予約公開のスケジューラは Node.js のプロセスが動いている間しか動かない。
- C と比べると、スケーリングと DB の運用を自分で持つことになる。

## 3. トレードオフ

### 機能の差

A と B はどちらも WordPress なので、機能の差は CMS（WordPress と EmDash）の差になる。

| 観点                   | A / B: WordPress                     | C: EmDash on Workers                          | D: EmDash on Node.js       |
| ---------------------- | ------------------------------------ | --------------------------------------------- | -------------------------- |
| 言語・フレームワーク   | PHP                                  | TypeScript（Astro）                           | TypeScript（Astro）        |
| プラグインの数         | 6 万件以上                           | 少ない（コミュニティ製が出始めた段階）        | C と同じ                   |
| プラグインの隔離       | なし（同じプロセスで動く）           | isolate ごとに隔離し、権限を宣言させる        | workerd の子プロセスで隔離 |
| テーマ                 | PHP のテーマ。管理画面から差し替える | Astro のプロジェクトとして作る。DB は触れない | C と同じ                   |
| 非エンジニアの編集     | ブロックエディタ、ページビルダーあり | 管理画面はまだ若い。ビジュアルな編集は限定的  | C と同じ                   |
| マルチサイト・多言語   | あり（本体やプラグイン）             | 執筆時点では標準機能なし                      | C と同じ                   |
| EC                     | WooCommerce                          | コミュニティ製（DashCommerce など）           | C と同じ                   |
| AI エージェントの操作  | REST API や MCP のプラグインで対応   | MCP サーバーと CLI を内蔵                     | C と同じ                   |
| フロントエンドの自由度 | テーマ、またはヘッドレスで任意       | Astro のみ                                    | Astro のみ                 |

プラグインの隔離が、EmDash を選ぶ最大の理由になる。
WordPress では、プラグインを入れることは、そのプラグインに DB とファイルのすべてを渡すことと同じである。
EmDash では、プラグインが何にアクセスするかをインストール前に確認でき、宣言していない操作はできない。

一方で、テーマとプラグインの互換性はない。
EmDash には WordPress からの移行手段（WXR ファイルのインポート、Exporter プラグイン）があり、記事、メディア、カスタム投稿タイプは移せる。
ただし、テーマは Astro で作り直し、プラグインに頼っていた機能は代わりを探すか自作する必要がある。

基盤としての A と B の差は次のとおりである。

| 観点                   | A: EC2                                     | B: Lightsail                                       |
| ---------------------- | ------------------------------------------ | -------------------------------------------------- |
| 構築                   | AMI、VPC、セキュリティグループなどを組む   | ブループリントから数クリック                       |
| 料金                   | 従量課金。転送量や IPv4 などが別にかかる   | 月額固定。固定 IP と転送量（プランごとの枠）を含む |
| インスタンスの選択肢   | ほぼすべてのインスタンスタイプ             | 2 vCPU のバースト可能なプランが中心                |
| スケーリング           | Auto Scaling                               | 手動（スナップショットから作り直す、台数を足す）   |
| バックアップ           | AWS Backup や EBS スナップショットを自分で | 自動スナップショット（日次、7 世代）               |
| AWS の他サービスと連携 | VPC 内でそのままつながる                   | VPC ピアリングが必要                               |
| IaC                    | CloudFormation、Terraform で一通り扱える   | 扱えるが、対応するリソースは EC2 より少ない        |

### コスト

小規模なサイト（月間数万 PV）を想定した月額の目安を比べる。

| 構成                                  | 月額の目安  | 内訳                                                                       |
| ------------------------------------- | ----------- | -------------------------------------------------------------------------- |
| A: EC2 1 台（DB 同居）                | 約 18 USD   | t4g.small 約 12 USD + EBS gp3 20 GB 約 2 USD + パブリック IPv4 約 4 USD    |
| A: 冗長構成（ALB + 2 台 + RDS + EFS） | 70 USD 以上 | ALB 約 16 USD、RDS db.t4g.micro 約 12 USD など。台数とサイズで大きく変わる |
| B: Lightsail 1 台（DB 同居、2 GB）    | 12 USD      | 固定 IP と転送量 3 TB を含む。1 GB プランなら 7 USD                        |
| B: 冗長構成（LB + 2 台 + DB + CDN）   | 約 76 USD   | LB 18 USD + 2 GB × 2 台 24 USD + マネージド DB（高可用性）30 USD + CDN 等  |
| C: EmDash on Workers（無料プラン）    | 0 USD       | サンドボックス型のプラグインは使えない                                     |
| C: EmDash on Workers（有料プラン）    | 5 USD から  | 1,000 万リクエスト/月を含む。D1 と R2 は小規模なら含まれる枠に収まる       |
| D: EmDash on Node.js（VPS や EC2）    | 5〜18 USD   | 実行する VM の料金。DB を RDS などに分けるとその分が加わる                 |

1 台で動かすなら、B は A より安く、料金も読みやすい。
A では転送量とパブリック IPv4 が別にかかるが、B ではプランに含まれる。
冗長構成にすると、A と B の差はほとんどなくなる。
その場合は、Auto Scaling や VPC 内の連携が使える A のほうが得るものが多い。

C は、アクセスがなければほぼ費用がかからない。
アクセスが増えても、Workers の従量課金（100 万リクエストあたり 0.30 USD）とキャッシュで吸収しやすい。
A と B は、アクセスの多寡にかかわらず VM の料金がかかり、アクセスが増えれば台数かサイズを上げる必要がある。

ただし、A と B の最も大きな費用は、たいてい運用の人件費である。
WordPress 本体とプラグインの更新、脆弱性への対応、バックアップの確認は、月額の表には現れない。

### 運用負荷

運用が軽い順に並べると次のようになる。

1. C: サーバーの運用はない。D1 と R2 のバックアップ、EmDash の更新、テーマのデプロイを持つ。
2. D: VM、Node.js、DB、workerd の子プロセスを自分で持つ。プラグインの攻撃面は A、B より小さい。
3. B: 構築とバックアップは A より楽だが、OS、PHP、MySQL、WordPress 本体、プラグインの更新は自分で持つ。
4. A: B の負担に加えて、ネットワーク、バックアップ、スケーリングの仕組みも自分で組む。

B の運用で気をつける点は次のとおりである。

- ブループリントは作成時点の状態を入れるだけで、その後の更新はしてくれない。Bitnami 版のように、ブループリント自体の更新が止まることもある[^bitnami]。
- バースト可能なインスタンスは、CPU クレジットを使い切ると性能が大きく落ちる。プラグインが重い、またはアクセスが続くサイトでは、監視しておかないと急に遅くなる。

EmDash は新しいため、別の種類の負担がある。

- 予約公開は v0.19.0 まで動いていなかった[^review]。1.0 以降もこの種の不具合が見つかる可能性は WordPress より高い。
- サンドボックスには固定の上限（CPU 50 ms、メモリ 128 MB、実行時間 30 秒）がある。重い処理をするプラグインは作れない。
- 困ったときに参照できる情報や、頼める制作会社は、WordPress と比べて圧倒的に少ない。

### ロックイン

- A と B は、WordPress が動く基盤ならどこへでも移せる。ロックインはほぼない。B は、スナップショットを EC2 にエクスポートして A に移ることもできる。
- C は、D1、R2、Dynamic Workers という Cloudflare 固有のサービスに乗る。EmDash は MIT ライセンスで D の形にも移せるが、プラグインの隔離やゼロまで縮むスケーリングといった利点を最も得られるのは Cloudflare 上である[^lockin]。
- D は、Node.js が動く基盤ならどこへでも移せる。ただし、プラグインの隔離は Cloudflare 由来の workerd に依存する。
- EmDash に共通して、フロントエンドは Astro に固定される。

## 4. 代替の選択肢

A から D のほかに、次の選択肢がある。

| 種別                       | 候補                                     | 向くケース                                                                |
| -------------------------- | ---------------------------------------- | ------------------------------------------------------------------------- |
| マネージド WordPress       | WP Engine、Kinsta、WordPress.com         | WordPress のエコシステムは使い続け、インフラと更新は外に出したい          |
| ヘッドレス WordPress       | WordPress（管理画面）+ Astro / Next.js   | 編集体験は WordPress のまま、表示側を静的配信にして速さと安全性を上げたい |
| ブログ・メディア特化       | Ghost（セルフホスト、Ghost(Pro)）        | 有料会員やニュースレターが主目的                                          |
| コードで定義するヘッドレス | Payload、Strapi、Directus                | TypeScript のチームが自前で持ちたいが、フロントエンドは自由に選びたい     |
| SaaS のヘッドレス CMS      | Sanity、Contentful、microCMS             | DB の運用も持ちたくない。microCMS は日本語のサポートが厚い                |
| ノーコード                 | Webflow、Framer、STUDIO                  | マーケティング担当者だけでサイトを作り、更新したい                        |
| Git ベース                 | Astro + Markdown（+ Decap CMS、TinaCMS） | 書き手が開発者で、記事を Git で管理したい                                 |

マネージド WordPress は、B と比べて月額は高いが、WordPress 本体の更新、バックアップ、キャッシュ、WAF までを事業者が持つ。
B で残る運用（OS とアプリケーションの更新）まで手放したい場合の選択肢になる。

EmDash に最も近いのは Payload である。
どちらも TypeScript で書かれたオープンソースの CMS で、DB に記事を持ち、自前で運用できる。
違いは、Payload はフロントエンドを選ばない（Next.js と組み合わせることが多い）のに対し、EmDash は Astro に固定され、その代わりにプラグインの隔離とエッジでの実行を標準で持つ点である。

## 5. 選ぶ基準

A（WordPress on EC2）が向くのは次のような場合である。

- 既存のサイトが多くのプラグインやページビルダーに依存していて、作り直す余力がない。
- VPC 内の既存システムと密に連携する、Auto Scaling が必要、特定のインスタンスタイプが必要など、EC2 に置く理由がある。
- 冗長構成が前提で、B と費用の差がない。

B（WordPress on Lightsail）が向くのは次のような場合である。

- WordPress のエコシステムを使い続けたいが、EC2 のネットワークやバックアップを組むほどの規模ではない。
- 1 台で動く小規模なサイトで、料金を固定にしたい。
- OS と WordPress の更新を自分で持つことは許容できる。

A を選ぶ理由が「WordPress を使い続けたい」だけなら、B かマネージド WordPress のほうが、費用と運用負荷の両方で有利なことが多い。

C（EmDash on Workers）が向くのは次のような場合である。

- 新しく作るサイトで、開発者が Astro と TypeScript を扱える。
- プラグインに起因するセキュリティの不安を構造的に減らしたい。
- アクセスの少ない時間帯の費用をゼロに近づけたい、または急なアクセス増に備えたい。
- Cloudflare に寄せることを許容できる。

D（EmDash on Node.js）が向くのは次のような場合である。

- EmDash を使いたいが、Cloudflare 以外の基盤（社内の AWS アカウントなど）に置く必要がある。
- VM と DB を運用できる体制がある。

WordPress から EmDash への移行を検討するなら、まず使っているプラグインを棚卸しする。
プラグインへの依存が少なく、テーマを作り直す覚悟があるなら、移行の候補になる。
依存が多いなら、ヘッドレス WordPress か、マネージド WordPress への移行のほうが現実的である。

[^emdash-blog]: [Introducing EmDash](https://blog.cloudflare.com/emdash-wordpress/) を参照。

[^emdash-1.0]: [Cloudflare releases EmDash 1.0 CMS with decentralized plugin registry](https://www.cmswire.com/digital-experience/cloudflare-releases-emdash-1-0-cms-with-decentralized-plugin-registry/) を参照。バージョンは [emdash-cms/emdash のリリース](https://github.com/emdash-cms/emdash/releases) で確認した。

[^price]: 単価は変わることがある。最新の値は [Amazon EC2 の料金](https://aws.amazon.com/ec2/pricing/on-demand/)、[Amazon Lightsail の料金](https://aws.amazon.com/lightsail/pricing/)、[Cloudflare Workers の料金](https://developers.cloudflare.com/workers/platform/pricing/) を参照。

[^lightsail-ha]: 構成の例は [Deploying a highly available WordPress site on Amazon Lightsail](https://aws.amazon.com/blogs/compute/deploying-a-highly-available-wordpress-site-on-amazon-lightsail-part-4-increasing-performance-and-scalability-with-a-lightsail-load-balancer/) を参照。

[^bitnami]: [WordPress by Bitnami](https://docs.aws.amazon.com/lightsail/latest/userguide/amazon-lightsail-wordpress-bitnami.html) と [Migrating from Bitnami blueprints to Lightsail blueprints](https://docs.aws.amazon.com/lightsail/latest/userguide/migrate-from-bitnami-to-lightsail-blueprints.html) を参照。

[^review]: [EmDash CMS Review (2026)](https://www.bitdoze.com/emdash-cms-review/) を参照。第三者のレビューであり、執筆時点の公式ドキュメントで確認できていない記述を含む。

[^lockin]: Cloudflare への依存を指摘する論評として、[EmDash 1.0 challenges WordPress freedom with Cloudflare lock-in](https://www.searchenginejournal.com/emdash-1-0-challenges-wordpress-freedom-with-cloudflare-lock-in/591378/) がある。
