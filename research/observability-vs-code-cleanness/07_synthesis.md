# 07. シンセシス — 結局、我々はどう振る舞うべきか

## 出発点を再確認する

ユーザーの問題意識は次の三点だった。

- 障害発生時の迅速な対応
- 新規参画者のオンボーディングコスト軽減
- AI とのコンテキスト共有

そして「オブザーバビリティのメリットがコード汚染のデメリットを上回る」という判断。
この判断自体は、業界の現在地と一致している。

## ただし、論点はもう一段先にある

01 章で書いたとおり、業界の議論は

「コードを汚すか、汚さないか」

ではなく

「計装の複雑性をどのレイヤーに隔離するか」

に進化している。この問いの立て方をすれば、ユーザーの直感はそのまま活かしながら、
コードのクリーンネスも犠牲にしない設計が可能になる。

## 抽象モデル: 四層構成

02 〜 06 で出てきた要素を統合すると、次の四層構成が浮かび上がる。

```
[ 1. インフラ層計装 ]   ← OpenTelemetry 自動計装に任せる
       |
       v
[ 2. 横断情報伝播 ]    ← contextvars / structlog bind / context.Context
       |
       v
[ 3. 業務イベント計装 ] ← Domain Probe（業務語彙のインターフェース）
       |
       v
[ 4. リクエスト境界集約 ] ← Canonical Log Line / Wide Event を 1 件出力
```

各層の責任:

1. インフラ層: HTTP / DB / 外部 API の自動 trace、業務コードに侵入しない
2. 横断情報: request_id / user_id / experiment_group を関数引数を汚さず伝播
3. 業務イベント: ドメインクラスは Domain Probe にだけ依存、内部に logger を持たない
4. 終端集約: ミドルウェアが context を 1 行の wide event に変換、サンプリング判定もここで

この四層が揃うと、業務コードに残る計装関連の記述は基本的に Domain Probe の呼び出しだけ。
それ以外（ログ書式・メトリクス名・サンプリング・PII redaction・送信先）はすべて
別レイヤーの責任になる。

## トレードオフがどう解消されるか

ユーザーが感じていた「コード汚染」の正体は、

- ドメインコードに `logger.info("...", ...)` が散らばる
- メトリクス名 / ログ名の議論がレビューで起きる
- テストでロガーやメトリクスをモックしないと壊れる
- 計装変更のためにビジネスコードを再ビルド

四層構成は、これらを各層に分配することで解消する。

- Domain Probe が業務語彙の呼び出しを提供 → ドメインコードは業務的に読める
- 計装の集約は Probe / ミドルウェア → 命名規則は 1 箇所で管理
- Probe をモックすれば業務テストが書ける、Probe 自体は別にテスト
- 集約層を入れ替えれば、計装の出力先を切り替え可能

そして料金面（06 章）も、

- 量より質: 高カーディナリティの wide events を戦略サンプリングで
- 観測性のコストを観測する: 棚卸しで不要フィールド削除

で抑える、というのが揃った構図。

## 我々のレコメンドプラットフォームに当てはめる

ここはユーザー固有の文脈なので、推測も混じる。

### 既に強い場所

- structlog ベースの構造化ログが入っている（推測。input/recommend-api 系を見て確認したい）
- New Relic / CloudWatch などのバックエンドは整備されている
- OpenSearch / Valkey / RDS の踏み台アクセス手段がある（bastion-datastore-access スキルあり）

### 弱そうな場所、Domain Probe 化が効きそうな場所

レコメンド API の業務的意思決定が密集している部分。

- recall 戦略の選択ロジック: なぜこの戦略を選んだか、候補数はどう変化したか
- ranking のモデル切替（feature flag, A/B test）: どのモデルが選ばれたか、信頼度
- 後段フィルタ: フィルタ理由ごとの除外件数（unrecommendable, NG ワード, etc）
- フィーチャー fallback: どのフィーチャーが取得失敗したか、fallback の質

これらは「業務語彙で観測したい」典型例で、Domain Probe + wide event との相性が良い。

### Canonical Log Line を入れるなら

レコメンド API のエンドポイント（推論 API）に、1 リクエスト 1 wide event を導入する。
最低限のフィールドはこれくらい:

```
request_id, user_id, store_id, experiment_groups[],
recall_strategies_used[], candidates_after_recall,
ranking_model, ranking_latency_ms,
filtered_out_count, filter_reasons{},
final_recommendations_count, total_latency_ms,
status, error_type (if any)
```

これが揃うと、

- 「ある実験グループで latency が悪化していないか」が SQL ライクに引ける
- 「特定店舗で candidates が常に少ない」を発見できる
- 障害時に「このリクエストで起きたこと」を 1 行で再現できる
- AI に「このユーザーへのレコメンドで何が起きたか」を 1 行で渡せる

ユーザーが挙げた三つのメリット（障害対応、オンボーディング、AI コンテキスト）の
すべてに直接効く。

### バッチ・長時間処理は別作法

embedding 更新バッチや特徴量パイプラインは、wide events と相性が悪い。
これらは:

- 進捗イベントを定期発行（every N items processed）
- フェーズ別にメトリクス（recall index update, model serving update, etc.）
- 完了時にサマリイベント

の組み合わせが現実解。リアルタイム API とバッチは別作法、と最初から分けるのがいい。

## 段階導入の提案

一気にやらない。優先順位はこの順序が現実的。

1. 既存のホットスポット（最も計装が密で読みにくい関数）を 1 つ選んで Domain Probe 化
   - 改善前後でコードの読みやすさを比較できる、効果を実感しやすい
2. リクエスト境界に context オブジェクトを導入、ミドルウェアで wide event を 1 行出力
   - 既存のログとは別ストリームに出して、両立期間を作る
3. recall / ranking / filter の意思決定計装を Probe 化、context に積み上げ
   - ここで初めて canonical log の "豊かさ" が見える
4. 戦略サンプリング導入（エラー・遅延・実験グループは 100%、それ以外を絞る）
   - 料金が現実的になる
5. OpenTelemetry 自動計装で I/O 層を補強
   - 業務計装と分離されているので追加は楽
6. 観測性のコストを観測する（不要フィールドの棚卸し）

これは半年〜1 年がかりの作業になる。
全部一度に提案するとレビューがしんどいので、1 箇所ずつ PR で示すのが現実的。

## マインドセットとしての要点

- 計装は汚染ではない、コードの一部。ただし業務語彙で書く
- 「コードを汚すか観測するか」ではなく、「複雑性をどこに隔離するか」
- 量で殴らず、質と切り出しやすさで勝つ
- 観測性は AI 時代の前提インフラ、後付けではなく一級市民

最後の一点が、ユーザーが感じている「AI とのコンテキスト共有」の本質。
ソースコードよりテレメトリのほうが「今ここで起きていること」を高い解像度で AI に伝えられる。
これは Charity Majors の最近の主張とも完全に一致する。

## このワークスペースをどう活かすか

このメモは「読み物」として残すが、実際の実装に進めるなら次のいずれかが入り口になる。

- 既存のレコメンド API のホットスポット 1 関数を Domain Probe 化する PR を出す
- canonical log line のミドルウェアプロトタイプを別ブランチで試す
- 観測性に関する社内向け勉強会の資料（このワークスペースをベースに）

どれもユーザーの問題意識を活かしながら、コードのクリーンネスを犠牲にしない方向に進める。

## 出典（このシンセシスは 02 〜 06 章全体の合成）

- [Domain-Oriented Observability — Martin Fowler / Pete Hodgson](https://martinfowler.com/articles/domain-oriented-observability.html)
- [Canonical log lines — Stripe](https://stripe.com/blog/canonical-log-lines)
- [Structured Events Are the Basis of Observability — Honeycomb](https://www.honeycomb.io/blog/structured-events-basis-observability)
- [Observability Engineering, Ch. 11 — O'Reilly](https://www.oreilly.com/library/view/observability-engineering/9781492076438/ch11.html)
- [Escaping the Cost/Visibility Tradeoff — Honeycomb](https://www.honeycomb.io/blog/escaping-cost-visibility-tradeoff-observability-platforms)
