# 06. 経済的トレードオフ — コスト、ノイズ、サンプリング

## なぜこれを別ファイルにしたか

01 〜 05 で扱ってきたのは「コードの汚染」というトレードオフ。
ところが現実には、もう一つの重いトレードオフが並走している。

- 観測性ツールの料金（イベント数・転送量・保管料）
- ノイズの増加（量を増やすほど信号対雑音比が悪化する）
- ストレージとクエリ性能

「コードに踏み込んででも観測性を上げたい」と思う気持ちが、無制限の計装に流れると、
今度は「コードはクリーンだが料金が破綻」「ログはあるが探せない」という別の苦しみに変わる。

業界はここでも明確な議論を積み重ねている。

## 料金構造の現実

### OpenTelemetry の隠れたコスト

OpenTelemetry 自体は OSS で無料だが、データ転送・保管・バックエンドへの送信は有料。
"OpenTelemetry Costs Us $2,400/Month (And We're Only Tracing 1% of Requests)" という
記事タイトルが現実を端的に示している。

実際の構成要素:

- AWS のリージョン外データ転送: $0.09/GB
- 観測性 SaaS（Honeycomb, Datadog, New Relic 等）のイベント単価
- 内部の Kafka / ES / S3 のインフラコスト

1% のサンプリングですら月 $2,400 になる構成は珍しくない。
100% トレースしようと思うと、観測性のコストがアプリケーションのインフラコストを超える、
というのが今の相場感。

### 課金モデルの分類

主要なバックエンドの課金モデルは大きく 3 つに分かれる。

| 課金軸 | 例 | 影響 |
|---|---|---|
| ホスト/エージェント数 | Datadog APM | スケールアウト時に線形増 |
| イベント数 | Honeycomb | wide event の数が直接コスト |
| データ取り込み量 (GB) | Splunk, ELK 系 | フィールド数より総バイト数が効く |

wide events のように 1 リクエスト 1 イベントで多フィールドを乗せると、
イベント数課金のバックエンドではコスト効率が劇的に良い。逆に GB 課金だと
カーディナリティを上げると一気に高くなる。バックエンド選択と計装方針はカップリングしている。

## "Brute Force Data Collection" の罠

Honeycomb のブログ "Escaping the Cost/Visibility Tradeoff" がこの罠を明確に書いている。

> collecting all possible data around a service turns out to be very counterproductive at
> scale, given that brute force collection of data results in high costs and adds a lot
> of noise to your data, reducing signal to noise ratio

ポイントは二段になっている。

1. 全部取ろうとすると料金が破綻する
2. 仮に料金を払えても、ノイズで信号が埋もれて検索性が悪化する

つまり「金で殴る戦略」は、お金を払えても観測性として劣化する。
量より質、というのが業界の合流点。

## 戦略的サンプリング

ODD（04 章）のキーアイデアの一つがこれ。

### Head-based vs Tail-based サンプリング

| 方式 | 仕組み | 強み | 弱み |
|---|---|---|---|
| Head-based | リクエスト開始時にサンプリング判定 | シンプル、低オーバーヘッド | エラーや遅延を見落とす可能性 |
| Tail-based | リクエスト完了後に判定 | 興味あるリクエストを確実に捕捉 | バッファリングが必要、複雑 |

実用的には:

- 普通のリクエスト: head-based で 0.1% など低い率
- エラーリクエスト: 100% 保持
- 遅延の長いリクエスト: 100% 保持
- 特定の実験グループ: 100% 保持

の併用が定石。OpenTelemetry Collector の Tail Sampling Processor がこれを実現する。

### 重要なのは「捨て方」

サンプリング率を語るとき、エンジニアはついつい「率」だけ気にする。
だが本当に重要なのは「何を捨てて何を残すか」のルール設計。

「全リクエストを 10% でサンプリング」と「エラー 100% + 正常 1%」では、
取り込み量はほぼ同じでも、観測性は後者が圧倒的に高い。

## ノイズ削減の作法

Last9, Datadog, Spacelift などのベストプラクティス記事が共通して挙げているもの:

1. 構造化ログ（JSON）。grep ではなくクエリで絞り込めるように
2. ログレベルの厳密運用。DEBUG が本番で出ているとほぼ常にコスト源
3. 重複ログの削除。同じイベントを 3 箇所で出していないか
4. 一括処理ログ。ループ内で吐いているログは外で集約版を 1 行
5. 中央集約点（Collector）でのフィルタ・正規化・PII redaction

「環境ごとに verbosity を変える」も典型解。開発環境はうるさく、本番は絞る、test 環境は中間。

## 保管期間の階層化

これも料金と観測性の両立に効く戦略。

- ホットストレージ（数日〜2 週間）: 全ログ・全イベント、即時検索可能
- ウォーム（1 〜 3 ヶ月）: サンプリング済み、低速検索
- コールド（1 年以上）: S3 など低単価ストレージ、バッチ分析用

Stripe の canonical log lines が二系統出力（ログ基盤 + Kafka → S3）にしていたのも、
本質的にはこの階層化。検索性と長期分析を別ストアで担保する。

## 「事前ダッシュボード設計」の経済性

旧来のメトリクス文化では、ダッシュボードや SLI を事前に設計するのが王道だった。
しかしこれには次の経済問題がある:

- 知りたいことを「事前に」決めなければならない
- 設計に乗らない問いに答えるには、新しいメトリクスを足し、ビルド・デプロイを待つ
- メトリクスの cardinality 制限で、後から軸を増やせない

wide events / 高カーディナリティの哲学は、この経済問題を反転させる:

- 取り込み時点では「何が知りたいか」を決めなくていい
- 起きたら考える、考えて切り出す
- 軸を後から増減できる

事前設計の手間 / 緊急時の柔軟性 のトレードオフを、後者寄りに振った設計。
これは料金とは別軸の経済性（人的コストと時間）に効いている。

## 「観測性のコスト」を測るためのメタ計装

皮肉な話だが、観測性のコストそのものも観測対象になる。

- 1 リクエストあたりのイベント生成サイズ
- サービス別の取り込み量
- フィールド別の出現率（使われていないフィールドの削除候補）

Honeycomb は "Observability for Observability" としてこの概念を提供しているし、
Datadog や New Relic も類似の機能を持つ。「今月のログ料金を圧迫しているサービスはどれか」を
SQL ライクに問い合わせられる。

この種のメタ計装は、コスト最適化のフィードバックループを回すために必須。
入れていないと、ある日請求書を見て驚くことになる。

## トレードオフを整理すると

ここまでの整理を一枚に圧縮すると、以下のような構図になる。

| 戦略 | 料金 | ノイズ | 観測性 |
|---|---|---|---|
| 全部記録 + 全部保管 | 破綻 | 高 | 高（だが探せない） |
| 全部記録 + 戦略サンプリング | 抑制可能 | 中 | 高 |
| 高カーディナリティ wide events | イベント課金なら効率的 | 低 | 高 |
| 事前メトリクス設計のみ | 最安 | 低 | 限定的（unknown unknowns に弱い） |
| ハイブリッド（メトリクス + 戦略サンプル wide events） | 中 | 低 | 高 | 

現在のベストプラクティスは最後の「ハイブリッド」で、

- 定常的な健全性指標 → メトリクス（事前設計）
- 探索的な問い → wide events（戦略サンプリング）

を併用する。

## 我々のコードベースへの含意

ユーザーが感じる「観測性のメリットがコード汚染を上回る」という直感は正しい。
ただしそれを実装に落とすときに、

- 量で殴らない（戦略的サンプリングを最初から設計）
- 1 リクエスト 1 wide event の集約点を作る（イベント数課金との相性を意識）
- 普段見ないフィールドは定期的に棚卸し
- 観測性のコストそのものを観測する

を意識しないと、「コードはクリーンになったが請求書は爆発」となる。

レコメンドプラットフォームでは特に、

- ランキング推論の latency 分布
- ABテストグループ別の挙動

を細かく取りたい誘惑が強いが、これらは高カーディナリティ問題に直撃する。
wide events + tail-based sampling の組み合わせで、エラー・SLO 違反・実験中グループだけ
100%、それ以外は 1% といったルールを持つのが現実解。

## 出典

- [Escaping the Cost/Visibility Tradeoff in Observability Platforms — Honeycomb](https://www.honeycomb.io/blog/escaping-cost-visibility-tradeoff-observability-platforms)
- [OpenTelemetry Costs Us $2,400/Month — Medium](https://medium.com/codeelevation/opentelemetry-costs-us-2-400-month-and-were-only-tracing-1-of-requests-f741e0c4fc69)
- [How to Optimize Log Volume and Reduce Noise at Scale — Datadog](https://www.datadoghq.com/knowledge-center/log-optimization/)
- [Logging Best Practices to Reduce Noise and Improve Insights — Last9](https://last9.io/blog/logging-best-practices/)
- [Rethinking Observability Costs: How Structured Logging Can Save You Thousands — DEV Community](https://dev.to/anderson_leite/rethinking-observability-costs-how-structured-logging-can-save-you-thousands-54bh)
- [11 Key Observability Best Practices You Should Know in 2026 — Spacelift](https://spacelift.io/blog/observability-best-practices)
