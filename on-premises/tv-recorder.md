# テレビ録画サーバーを何で組むか

自宅の Linux サーバーにチューナーをつなぎ、地上波や BS の番組を録画するソフトウェアを比べる。
あわせて、録画した番組を **Google TV** と **Amazon Fire TV（Fire TV Stick）** で見る方法を比べる。

本ファイルは自宅に置く機器の構成なので、`on-premises/` に置く。

## 1. 解きたい課題と前提

市販のレコーダーではなく自宅のサーバーで録画すると、次のことができる。

- キーワードやジャンルで自動予約し、容量の許す限り録りためる。
- 録画をエンコードして容量を減らし、NAS に保管する。
- 家の中のどのテレビやスマートフォンからでも、同じ録画を見る。

前提は次のとおりとする。

- サーバーは Linux（x86_64 か arm64）で、各ソフトウェアは Docker で動かす。
- チューナーは Linux で使えるもの（PX-Q3U4 などの PX 系、DVB ドライバで動くものなど）を使い、B-CAS 相当の復号ができる環境は用意済みとする。
- 視聴する端末は、Google TV（Google TV Streamer や Google TV 搭載テレビ）と Fire TV とする。
- バージョンと開発状況は執筆時点（2026 年 11 月）の GitHub の状態を使う。

日本のテレビ録画は、チューナーを扱う層と、予約や録画を管理する層に分かれる。
比較の対象は主に後者だが、前者の選び方が後者の選択肢を決めるので、両方を扱う。

```
チューナー ─▶ チューナーサーバー ─▶ 録画管理 ─▶ 録画ファイル（TS / MP4）─▶ 視聴クライアント
             （Mirakurun / mirakc）  （EPGStation / EDCB）                    （Web / TV アプリ）
```

## 2. 構成

### チューナーサーバー

| ソフトウェア                                       | 概要                                                                                                                                                       |
| -------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [Mirakurun](https://github.com/Chinachu/Mirakurun) | Node.js 製の定番。チューナーの共有、番組表（EPG）の取得、ストリームの配信を受け持つ。4.x 系が現行（執筆時点で 4.3.0）                                      |
| [mirakc](https://github.com/mirakc/mirakc)         | Rust 製で、Mirakurun と互換の API を持つ。メモリ消費が少なくシングルボードコンピュータでも動く。単体での録画と、常時録りためるタイムシフト録画の機能も持つ |

EPGStation と KonomiTV はどちらにもつながるので、ここでの選択は後から変えやすい。
mirakc の README も、Mirakurun を置き換えるだけで既存の EPGStation が動くとしている。

### A. EPGStation

```
Mirakurun / mirakc ─▶ EPGStation（Web UI、予約、録画、エンコード）─▶ 録画ファイル + DB
```

- [l3tnun/EPGStation](https://github.com/l3tnun/EPGStation)。Mirakurun / mirakc の上で動く録画管理ソフトで、Linux + Docker の構成の定番として広く使われている。
- Web UI で番組表、キーワードでの自動予約（ルール）、録画一覧、ライブ視聴、録画の再生を扱う。ARIB の字幕（日本の放送の字幕方式）も Web 上で表示できる。
- 録画後に FFmpeg などのコマンドを呼んでエンコードする仕組みを持つ。エンコードの設定は自分で書く。
- 安定版は v2 系である。v3 系はベータ版（執筆時点で v3.0.0-beta4）で、Node.js 24 以上が必要になり、Docker イメージは amd64 と arm64 のみになる。
- リポジトリの既定ブランチと Docker の `latest` タグは、すでに v3 のベータ版を指している。安定版で組むなら v2 を明示して選ぶ。
- API が公開されていて、TV 向けや iOS 向けの非公式クライアントがある。シリーズ管理を強化した [フォーク版](https://github.com/stuayu/EPGStation) もある。このフォークは、同じ作者のフォーク版 Mirakurun と組むことを前提にしている。

### B. EDCB（Linux 版）

```
Mirakurun（BonDriver 経由）─▶ EDCB（EpgTimerSrv + EMWUI）─▶ 録画ファイル
```

- [xtne6f/EDCB](https://github.com/xtne6f/EDCB)。もとは Windows の録画ソフトで、予約の細かさ（録画の前後のマージン、チューナーの割り当てなど）で長く使われてきた。
- 2024 年 6 月のリリース（240622）から Linux でネイティブに動く。それ以前は Wine で動かす [EDCB-Wine](https://github.com/tsukumijima/EDCB-Wine) が使われていた。
- Web UI は同梱の EMWUI（Lua 製）で、EPGStation と比べると見た目は素朴である。
- Docker イメージは公式にはなく、コミュニティのもの（[legnoh/edcb](https://hub.docker.com/r/legnoh/edcb) など）を使うか自分で作る。
- 設定ファイルの体系が Windows 由来で、Linux での情報は EPGStation より少ない。

### C. KonomiTV

```
EDCB ──────────┐
               ├─▶ KonomiTV（視聴 UI、録画予約は EDCB 経由のみ）─▶ ブラウザ / PWA / TV アプリ
Mirakurun ─────┘
```

- [tsukumijima/KonomiTV](https://github.com/tsukumijima/KonomiTV)。録画管理ソフトというより、ライブ視聴と録画再生のためのメディアサーバーである。ニコニコ実況のコメントや SNS の実況を重ねて見られる。
- バックエンドに EDCB か Mirakurun / mirakc を選ぶ。**録画予約と番組検索は EDCB バックエンドでのみ使える**。Mirakurun バックエンドや EPGStation とは録画の機能で連携しない[^konomitv-rec]。
- 録画の再生は、指定したフォルダの録画ファイルを読んでライブラリを作る方式である。TS のまま保存しておくと、番組情報や放送時のコメントを表示できる。MP4 にするとこれらが失われる。
- 再生時のトランスコードには、Intel QSV、NVIDIA NVENC、AMD VCE、Rockchip のハードウェアエンコーダーを使う。ソフトウェアエンコードは推奨されていない。
- Linux の x86_64 では、公式の Dockerfile で手元でビルドして動かせる。ARM では Docker でのインストールはサポートされておらず、専用のインストーラーを使う。
- 執筆時点の最新は v0.14.1 で、作者自身がまだ動作を保証できないベータ版としている。更新で破壊的な変更が入ることがある。

### D. Chinachu

- [Chinachu/Chinachu](https://github.com/Chinachu/Chinachu)。Mirakurun と同じ作者による録画管理ソフトで、かつては定番だった。
- 最後のコミットは 2022 年 1 月で、開発は事実上止まっている。新しく組むなら選ばない。

### 補助: エンコードと CM カット

- [rigaya/Amatsukaze](https://github.com/rigaya/Amatsukaze)。CM カットとエンコードを自動で行うツールの改造版で、Windows と Linux の x86_64 に対応している（arm64 は対象外）。Linux では GUI がなく、Web UI を使う。
- 録画後の自動連携の手順は EDCB 向けのみ README にある。EPGStation からは、録画後のコマンドでタスクを追加する形を自分で組む。
- 録画の映像だけを H.264 / H.265 に置き換えて TS のまま残す [tsreplace](https://github.com/rigaya/tsreplace) にも対応している。容量を減らしつつ番組情報を残したいときに使う。

### 組み合わせの例

| 構成                                          | 録画管理   | 視聴 UI    | 向いている使い方                       |
| --------------------------------------------- | ---------- | ---------- | -------------------------------------- |
| 1. Mirakurun / mirakc + EPGStation            | EPGStation | EPGStation | 録りためた番組を見る。定番で情報が多い |
| 2. Mirakurun + EDCB + KonomiTV                | EDCB       | KonomiTV   | ライブ視聴と実況コメントも楽しみたい   |
| 3. Mirakurun / mirakc + EPGStation + KonomiTV | EPGStation | 両方       | 録画は 1、ライブ視聴は KonomiTV で     |

構成 3 では、KonomiTV から録画予約はできず、KonomiTV は EPGStation の録画フォルダを読むだけになる。

## 3. 録画を Google TV / Fire TV で見る

### 端末側の前提

ここでの選択肢の多くは、ストアにない Android アプリ（APK）を手動で入れる **サイドロード** を前提にする。
日本の放送の録画を扱うアプリは、公開後に第三者の指摘でストアから削除された例があるためである[^store]。

- **Google TV**: サイドロードできる。ストアにある「Downloader」アプリで APK を取ってきて入れるのが手軽である。
- **Fire TV（Fire OS）**: Fire TV Stick 4K Max などの Fire OS（Android ベース）の機種はサイドロードできる。Fire OS のアプリは 32 bit（armeabi-v7a）で動くので、64 bit 専用の APK は入らない[^fireos-abi]。
- **Fire TV（Vega OS）**: Fire TV Stick 4K Select（2025 年）以降、Fire TV Stick 4K（第 3 世代）や Fire TV Stick HD（第 2 世代）なども Linux ベースの Vega OS で出ており、今後も増える見込みである。**Vega OS は Android アプリを動かせず、サイドロードもできない**[^vega]。使えるのは Amazon のストアにあるアプリだけになる。

Amazon は製品ページで OS を明示していないので、Fire TV を新しく買うなら、購入前に脚注の一覧などで機種ごとの OS を確認する。

### P1. EPGStation 専用の TV アプリ（EPGStation の録画を見る）

- [daig0rian/epcltvapp](https://github.com/daig0rian/epcltvapp)。Android TV / Fire TV 向けの EPGStation クライアントで、リモコンの上下左右だけで操作できる Android TV 標準の UI を使っている。
- EPGStation v2 / v3 の録画一覧、検索（リモコンのマイクでの音声検索を含む）、再生に対応する。放送中の番組のライブ視聴もできる。予約の操作はできない。内蔵プレーヤーは ARIB の字幕を表示できる。VLC などの外部プレーヤーにも渡せる。
- 作者のテスト環境に Google TV Streamer と Fire TV Stick 第 3 世代が含まれる。執筆時点の最新は v1.42（2026 年 11 月）で、開発は続いている。
- ストアからは 2022 年に削除されたため、サイドロードで入れる。入れた後はアプリ内から更新できる。

### P2. TV 向けの統合クライアント（Komorebi）

- [BeerEgg2001/Komorebi](https://github.com/BeerEgg2001/Komorebi)。KonomiTV、EDCB、EPGStation をバックエンドにできる Android TV / Fire TV 向けのクライアントで、ライブ視聴、録画の再生、予約を TV の UI で扱う。
- EPGStation は前述のフォーク版でのみ動作確認されている。本家の EPGStation で使えるかは未確認とされている。フォーク版はフォーク版 Mirakurun との組み合わせが前提なので、構成 1 から使うとチューナーサーバーごと替えることになる。
- ARIB の字幕はアプリ側で描画する。
- EPGStation のログイン認証（`auth.enabled`）には対応していない。認証を有効にすると、再生が途中で止まり予約の操作も失敗する。
- EDCB をバックエンドにする場合は、EDCB 側に連携用のスクリプトを置く必要がある。
- 要求は Android 7.1.2 以上か Fire OS 6 以上で、サイドロードで入れる。

### P3. Jellyfin / Plex の公式アプリ

```
録画（MP4 にエンコード、字幕は .vtt などに変換）─▶ Jellyfin / Plex のライブラリ ─▶ ストアの公式アプリ
```

- 録画フォルダを Jellyfin や Plex のライブラリとして登録し、各社の公式アプリで見る。
- Google TV と Fire OS ではストアから入れられるので、サイドロードが不要である。家族が使う端末に向く。Vega OS の機種向けに提供されているかはアプリごとに異なる。
- Plex は Vega OS 向けの公式アプリがあるが、直接再生できるのは H.264 + AAC で 7 Mbps までで、それ以外はサーバー側でトランスコードになる[^plex-vega]。Jellyfin は執筆時点で Vega OS 向けの公式クライアントが確認できない[^jellyfin-vega]。
- **ARIB の字幕はそのままでは表示できない**。字幕が必要なら、[tsreadex](https://github.com/xtne6f/tsreadex) と [b24tovtt](https://github.com/xtne6f/b24tovtt) などで WebVTT に変換して、動画と同じ名前で横に置く。
- 番組情報（チャンネル、放送日時、番組の説明）は取り込まれない。ファイル名の付け方を工夫して補う。
- MPEG-2 のままの TS は、端末によってはサーバー側でのトランスコードになり、サーバーの負荷が上がる。録画後に H.264 / H.265 の MP4 にエンコードしておくのが前提になる。

### P4. Kodi / VLC でファイル共有を開く

- 録画フォルダを SMB や NFS で共有し、Kodi や VLC で開く。サーバー側に追加のソフトウェアがほぼ要らない。
- Kodi 向けの EPGStation アドオン（[pvr.epgstation](https://github.com/Harekaze/pvr.epgstation)、[plugin.video.epgstation](https://github.com/l3tnun/plugin.video.epgstation)）は、最後のコミットが 2020 年と 2021 年で、現行の Kodi で動く保証はない。
- 番組情報は出ず、ファイルの一覧から選ぶことになる。リモコンでの操作性は P1、P2 より劣る。
- Kodi は Google TV ではストアから入れられるが、Fire TV ではサイドロードが必要である。VLC は Google TV と Fire OS のどちらもストアから入れられる。
- VLC は Vega OS 向けにも 2026 年 7 月にベータ版（Early Access）が出ており、SMB の共有を再生できる[^vlc-vega]。

### P5. TV のブラウザで Web UI を開く

- Fire TV の Silk ブラウザなどで、EPGStation や KonomiTV の Web UI をそのまま開く。
- Web UI はマウスやタッチ向けに作られているので、リモコンでの操作は厳しい。Google TV には標準のブラウザがない。KonomiTV は Chrome を強く推奨しており、TV のブラウザは想定していない。
- 試しに見る、ほかの手段がないときの代替、という位置づけになる。

## 4. トレードオフ

### 録画管理の比較

| 観点                 | A: EPGStation                | B: EDCB（Linux）                 | C: KonomiTV                                     |
| -------------------- | ---------------------------- | -------------------------------- | ----------------------------------------------- |
| 役割                 | 録画管理 + 視聴 UI           | 録画管理                         | 視聴 UI（録画予約は EDCB 経由）                 |
| 単独で録画できるか   | できる                       | できる                           | できない                                        |
| 自動予約             | キーワード、ジャンルのルール | キーワード、ジャンルなど細かい   | EDCB の機能を使う                               |
| Web UI               | 実用的                       | 素朴                             | 最も作り込まれている                            |
| ライブ視聴           | できる                       | 限定的                           | 得意（実況コメントつき）                        |
| Docker イメージ      | 公式あり                     | コミュニティのみ                 | 公式の Dockerfile（手元でビルド、ARM は非対応） |
| 安定度               | v2 は安定、v3 はベータ       | 長い実績、Linux 版は比較的新しい | ベータ版で破壊的な変更がある                    |
| Linux での情報の多さ | 多い                         | 少ない                           | 中程度                                          |

### TV での視聴方法の比較

| 観点               | P1: epcltvapp | P2: Komorebi                             | P3: Jellyfin / Plex        | P4: Kodi / VLC                    | P5: ブラウザ     |
| ------------------ | ------------- | ---------------------------------------- | -------------------------- | --------------------------------- | ---------------- |
| 対応する録画管理   | EPGStation    | KonomiTV、EDCB、EPGStation（フォーク版） | 問わない（ファイルを読む） | 問わない                          | 各 Web UI        |
| 番組情報           | 出る          | 出る                                     | 出ない                     | 出ない                            | 出る             |
| ARIB の字幕        | 出る          | 出る（アプリ側で描画）                   | 変換が必要                 | 出ない前提                        | 出る             |
| ライブ視聴         | できる        | できる                                   | できない                   | できない                          | できる           |
| 予約の操作         | できない      | できる                                   | できない                   | できない                          | できる           |
| Google TV          | サイドロード  | サイドロード                             | ストア                     | ストア                            | 標準ブラウザなし |
| Fire TV（Fire OS） | サイドロード  | サイドロード                             | ストア                     | Kodi はサイドロード、VLC はストア | Silk             |
| Fire TV（Vega OS） | 使えない      | 使えない                                 | Plex のみ（制約あり）      | VLC（ベータ）のみ                 | Silk             |
| リモコン操作       | 良い          | 良い                                     | 良い                       | 普通                              | 悪い             |

### 運用負荷

- 構成 1（EPGStation）は、公式の Docker 構成（[docker-mirakurun-epgstation](https://github.com/l3tnun/docker-mirakurun-epgstation)）がそのまま使え、最も軽い。ただし既定ブランチは v3（ベータ版）なので、安定版で組むなら v2 のブランチを指定する。
- 構成 2（EDCB + KonomiTV）は、コンテナが増え、EDCB の Docker 化と設定を自分で持つことになる。KonomiTV はベータ版で、更新に破壊的な変更が入ることがある。
- P3 は Jellyfin / Plex というサーバーが 1 つ増えるうえ、エンコードと字幕の変換を録画後処理に組み込む必要がある。代わりに端末側の手間（サイドロードとアプリの更新）がなくなる。
- P1、P2 はサイドロードしたアプリを各端末で更新する手間がある。どちらもアプリ内の更新機能を持つので、手間は小さい。

### コスト

ソフトウェアはどれも無償で使える。
費用の差は、トランスコードとストレージに出る。

- **トランスコードのハードウェア**: KonomiTV のライブ視聴と、P3 で端末が直接再生できない形式の録画は、サーバー側でトランスコードする。Intel の QSV が使える CPU か GPU を用意すると負荷が下がる。KonomiTV はソフトウェアエンコードを推奨していない。
- **Plex のサブスクリプション**: Plex でハードウェアトランスコードを使うには、有料の Plex Pass が必要である。Jellyfin は無償で使える。
- **ストレージ**: MPEG-2 の TS のままだと、地上波の HD 番組で 1 時間あたりおおむね 7〜8 GB になる。H.264 / H.265 にエンコードすると数分の 1 まで減るので、録りためる量が多いほど、エンコード用の計算資源とディスクの費用の釣り合いで決めることになる。

### ロックイン

- 録画ファイルは TS か MP4 なので、録画管理ソフトを替えてもファイルは残る。ただし予約のルールや録画の一覧は各ソフトのデータベースにあり、移行の手段はない。
- **TS のまま残すか、MP4 にするか** が、後から最も変えにくい選択である。TS は番組情報と字幕を持つが容量が大きく、P3 や P4 では扱いにくい。MP4 は容量が小さくどこでも再生できるが、番組情報と字幕が落ちる。tsreplace で映像だけを置き換えた TS は、その中間になる。
- P1、P2 は個人が開発する非公式のアプリで、ストアにも載っていない。開発が止まったときの代わりが少ない。P3 は録画管理ソフトに依存しないので、最も替えが利く。

## 5. 選ぶ基準

構成 1（Mirakurun / mirakc + EPGStation）が向くのは次のような場合である。

- 録りためた番組を見るのが主な用途で、ライブ視聴はおまけでよい。
- Docker で手早く組み、情報の多い構成で運用したい。
- TV では P1（epcltvapp）で見る。Google TV と Fire OS の Fire TV なら、これが最も手軽である。

構成 2（Mirakurun + EDCB + KonomiTV）が向くのは次のような場合である。

- ライブ視聴や実況コメントをよく使い、視聴 UI の作り込みを重視する。
- 予約の細かい制御が欲しい、または Windows で EDCB に慣れている。
- ベータ版の更新に付き合い、構成を自分で保守できる。
- TV では P2（Komorebi）で見る。

チューナーサーバーは、x86_64 のサーバーなら Mirakurun、シングルボードコンピュータで省メモリにしたい、あるいはタイムシフト録画を使いたいなら mirakc を選ぶ。

P3（Jellyfin / Plex）を視聴に足すのが向くのは次のような場合である。

- 家族が使う端末にサイドロードしたくない、または Vega OS の Fire TV を使っている。
- 録画を MP4 にエンコードして長く保管する運用で、番組情報が出なくても困らない。
- 録画以外の動画も同じアプリで見たい。

Fire TV を新しく買う場合、P1 と P2 を使うなら Fire OS の機種（Fire TV Stick 4K Max など）を選ぶ。
Vega OS の機種では、録画を見る手段が P3 の Plex（直接再生の制約あり）、P4 の VLC（ベータ版）、P5 に限られる。
録画を見るのが主な目的なら、サイドロードできる Google TV のほうが選択肢は広い。

[^konomitv-rec]: KonomiTV v0.14.0 のリリースノートに「録画予約機能・番組検索機能は EDCB バックエンドを前提に設計されています。Mirakurun バックエンドや EPGStation には対応していません」とある。

[^store]: epcltvapp は 2021 年に Google Play と Amazon アプリストアで公開されたが、2022 年に第三者からのコンテンツポリシー違反の指摘を受けて削除され、Google Play での公開も停止した（同リポジトリの README による）。

[^vega]: [These are the Fire TVs that don't support sideloading or Downloader due to Vega OS](https://www.aftvnews.com/these-are-the-fire-tvs-that-dont-support-sideloading-or-downloader-due-to-vega-os/)（AFTVnews）を参照。対象の機種は今後も増えるので、購入時に確認する。

[^fireos-abi]: [Fire TV のデバイス仕様](https://developer.amazon.com/docs/device-specs/device-specifications-fire-tv-streaming-media-player.html)（Amazon）で、Fire OS の各機種のアプリケーション ABI は 32 bit とされている。

[^plex-vega]: [Plex on Vega OS devices](https://forums.plex.tv/t/plex-on-vega-os-devices-amazon-fire-tv-stick-4k-select/932354)（Plex Forum、2025 年 10 月）を参照。

[^vlc-vega]: [VLC for Vega OS](https://jbkempf.com/blog/2026/vlc-vegaos/)（VLC の開発者のブログ、2026 年 7 月）を参照。

[^jellyfin-vega]: [Fire TV Stick 4K Select — Jellyfin not supported, sideload blocked](https://forum.jellyfin.org/t-fire-tv-stick-4k-select-%E2%80%94-jellyfin-not-supported-sideload-blocked?pid=67436)（Jellyfin Forum）を参照。コミュニティの回答で、Vega OS は Android ベースではないため専用のクライアントが必要とされている。
