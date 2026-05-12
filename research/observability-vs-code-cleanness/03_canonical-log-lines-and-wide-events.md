# 03. Canonical Log Lines と Wide Events — 集約点で 1 イベント

## このパターンの一行要約

リクエスト処理中はあちこちでログを吐かず、context にデータを積み続けて、
ミドルウェアの終端で 1 リクエスト = 1 行 = 1 イベントに集約して出す。

Stripe はこれを Canonical Log Lines と呼び、Honeycomb は Wide Events と呼んだ。
両者は独立に発展したが、結論はほぼ同じ場所に到達している。

## Stripe の Canonical Log Lines

### 動機

Stripe は元々ログをたくさん吐いていた。各層（認証、レート制限、DB アクセス、ビジネスロジック）が
それぞれログを出すと、1 リクエストに対して数十〜数百行のログが残る。これを後から「あるユーザーの
レート上限超過リクエストを集計したい」と思っても、複数行を相関させる必要があり、検索コストが高い。

メトリクスは事前定義したものしか取れないし、ログは検索性が悪い。Stripe の解決策は両者の中間を
取ること。

> over producing data to the extent that it's possible to procure just about anything

「後で何でも取り出せるくらい広く取る」が canonical log line の哲学。

### 形

リクエスト 1 件につき、こういう 1 行が出る:

```
canonical-log-line alloc_count=9123 auth_type=api_key database_queries=34 \
  duration=0.009 http_method=POST http_path=/v1/charges http_status=200 \
  rate_allowed=true user_id=usr_123 ...
```

key=value 形式（あるいは JSON）で、数十〜数百のフィールドが付く。
1 リクエストに対して 1 行だけが「正規 (canonical) ログ」として扱われ、
他のデバッグログとは別ストリームになる。

### 実装の仕組み

各処理層は context オブジェクトに情報を「装飾」していく:

```ruby
# 認証層
env["canonical"][:user_id] = user.id
env["canonical"][:auth_type] = "api_key"

# レート制限層
env["canonical"][:rate_allowed] = allowed
env["canonical"][:rate_remaining] = remaining

# DB 層
env["canonical"][:database_queries] = query_count
```

ミドルウェアの終端で、env を一気に 1 行に変換して出力:

```ruby
class CanonicalLineLogger
  def call(env)
    start_time = Time.now
    status, headers, body = @app.call(env)
    env["canonical"][:duration] = Time.now - start_time
    env["canonical"][:http_status] = status
    logger.info("canonical-log-line " + env["canonical"].to_kv)
    [status, headers, body]
  ensure
    # ロギング失敗がリクエストを壊さないように
  end
end
```

### コード汚染という観点での秀逸さ

各処理層は「ログを出す」のではなく「文脈に値を足す」だけ。

- `logger.info("rate limit checked", ...)` ではなく
- `context["rate_allowed"] = allowed`

文法的に「値を足す」のは代入文 1 行で、業務ロジックの読み心地をほぼ汚さない。
出力の責任はミドルウェアに完全に閉じ込められる。

これは Domain Probe（02 章）と直交していて、両方を併用できる。
Probe で業務イベントを表現しつつ、Probe の内部で context に追記する、という構成が自然。

### Stripe のさらなる工夫

- canonical line を Protocol Buffer で型定義し、形式を強制
- 出力先を二系統に分ける: ログ基盤（検索用）と Kafka → S3（長期分析用）
- 長期保管のサイズが現実的になる（行数が予測可能なので）

## Honeycomb の Wide Events

### 哲学

Honeycomb の主張は Stripe より過激で、「ログという概念をやめて、構造化イベントに統一しろ」。

> Replace log lines with arbitrarily wide structured events that describe the request
> and its context, one event per request per service.

各サービス・各ホップで 1 イベントを発行する。成熟したサービスでは 1 イベントに
300〜400 フィールドが付く、と Charity Majors 自身が書いている。

### Logs vs Structured Events

charity.wtf の "Logs vs Structured Events" 記事が、この立場を最もはっきり打ち出している。

- ログ: 人間が読むためのテキスト。複数行に情報が分散する
- 構造化イベント: 機械が処理するための JSON。1 件で完結する文脈を持つ

「ログは死んだ、イベントが残る」というのが Honeycomb の立場で、これに伴い
Honeycomb の課金モデルもイベント数ベースになっている（行数ベースではない）。

### High-Cardinality の重視

Wide events のもう一つの主張は「フィールドの中身に高カーディナリティ
（ユーザー ID、リクエスト ID、SKU など、値の種類が膨大なもの）を恐れず入れろ」。

メトリクス基盤は高カーディナリティに弱い（タグの組み合わせ爆発で課金が破綻する）が、
イベント基盤は本質的に行ベースなので耐性が高い。

結果として、「どのユーザーの、どの商品の、どの実験グループで」といった軸で
後から自由にスライスできる。事前にダッシュボードを設計する必要がない。

これが Charity Majors の言う "Unknown unknowns に対処できる" 状態。

## 二つのアプローチが同じ場所に到達した意味

Stripe と Honeycomb は別々の動機（Stripe は検索効率、Honeycomb はカーディナリティ）から
出発したが、結論は「1 リクエスト 1 イベント、context に積み上げて終端で出す」で一致している。

これはおそらく、分散システムでの観測性の問題に対する局所最適解ではなく、
かなり安定した構造的な解になっている。

## コード上の含意

このパターンを取ると、関数本体に書かれる計装コードは大幅に減る。

```python
def apply_discount(cart, customer):
    if not customer.is_eligible_for_discount():
        observation.add(discount_eligible=False)
        return cart
    discount = calculate_discount(cart, customer)
    observation.add(discount_amount=discount.amount, discount_type=discount.type)
    cart.apply(discount)
    return cart
```

`observation.add(...)` は事実上「dict に値を足す」だけなので、
- ログレベルの議論が不要
- メトリクス名の議論が不要
- アナリティクスへの送信判断が不要

すべては終端ミドルウェアの責任。ドメインコードは「観測したい値を伝える」ことに集中する。

## 注意点

このパターンが万能ではない場合がある。

- 長時間バッチ処理: 「終わってから 1 イベント」だと、途中の進行が見えない
- ストリーミング処理: そもそも「リクエスト」の境界が曖昧
- エラーで途中終了するケース: ensure / finally で確実に出すミドルウェア設計が必須

特に 1 番目は重要で、長時間処理では従来通り途中ログを残すか、進捗イベントを別に発行する
必要がある。ユーザーのレコメンドプラットフォームでいうと、リアルタイム推論 API は wide events
向き、フィーチャー更新の長時間バッチは別の作法が必要、と分けて考えるべき。

## 我々のコードベースへの含意

レコメンド API のリクエスト境界（HTTP リクエスト 1 件）は wide events と相性が良い。

- リクエスト到着時に context 初期化
- 各処理層（recall, ranking, post-filter）が context にメタデータ追記
  - recall: `recall_strategies=[A, B, C], candidates_count=120`
  - ranking: `ranking_model=deepfm_v3, scoring_latency_ms=42`
  - post-filter: `filtered_out_count=18, filter_reasons={...}`
- ミドルウェア終端で 1 行の canonical log として出力

これを実装しておくと、後から「ある実験グループで scoring_latency が悪化していないか」
「特定の店舗 ID で candidates_count が常に少ない原因は何か」といった調査が、
ダッシュボードの事前設計なしにできる。これは AI に渡すコンテキストとしても優秀で、
「このリクエストで何が起きたか」を 1 行のキーバリューで提示できる。

## 出典

- [Fast and flexible observability with canonical log lines — Stripe](https://stripe.com/blog/canonical-log-lines)
- [Logs vs Structured Events — charity.wtf](https://charity.wtf/2019/02/05/logs-vs-structured-events/)
- [Structured Events Are the Basis of Observability — Honeycomb](https://www.honeycomb.io/blog/structured-events-basis-observability)
- [Observable systems with wide events — Honeybadger](https://www.honeybadger.io/blog/observable-systems-wide-events/)
- [Wide logging: Stripe's canonical log line pattern — alcazarsec](https://blog.alcazarsec.com/tech/posts/wide-logging)
