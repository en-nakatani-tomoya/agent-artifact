# 02. Domain-Oriented Observability — 計装をドメイン用語で隔離する

## どこに書かれているか

Pete Hodgson が martinfowler.com 上で書いた長文記事 "Domain-Oriented Observability"。
Martin Fowler のサイトに掲載されているということは、業界の正典に近い扱いを受けている。
原文は実コード（割引適用のショッピングカート）を題材に進む。

## 何を問題視しているか

冒頭の問題提起がそのまま我々の問題意識と重なる。

> The observability added to systems tends to be rather low level and technical in nature,
> and too often it seems to require littering our codebase with crufty, verbose calls to
> various logging, instrumentation, and analytics frameworks.

意訳: 計装は本来テクニカルで低レベルなものだが、それを直接ドメインコードに書くと、
ロガー・メトリクス・アナリティクスの呼び出しがコードベースを汚す。

そしてカートに割引を適用する関数の例で「業務ロジックはわずか 25%、残りは計装」と
具体的に示す。これが Domain Probe パターンの出発点。

## Domain Probe パターンの本体

### 構造

```
[ ShoppingCart ]   ← ドメインロジック（業務用語のみ）
       |
       v
[ DiscountInstrumentation ]   ← Domain Probe（業務用語のインターフェース）
       |
       v
[ Logger ] [ Metrics ] [ Analytics ]   ← テクニカルな計装システム
```

`ShoppingCart` は `Logger` や `Metrics` を直接知らない。代わりに
`DiscountInstrumentation` という名前の Domain Probe を 1 つ依存として持つ。
Probe のメソッドは業務用語で命名される。`probe.discount_applied(amount=...)` のように。

### Before / After

Before（直接計装）:

```python
def apply_discount(self, customer):
    self.logger.info("discount.apply.start", cart_id=self.id)
    self.metrics.increment("discount.apply.attempt")
    if not customer.is_eligible_for_discount():
        self.logger.info("discount.apply.ineligible")
        self.metrics.increment("discount.apply.ineligible")
        return
    discount = self._calculate_discount(customer)
    self.logger.info("discount.apply.applied", amount=discount.amount)
    self.metrics.histogram("discount.amount", discount.amount)
    self.analytics.track("discount_applied", customer_id=customer.id)
    self.apply(discount)
```

After（Domain Probe 経由）:

```python
def apply_discount(self, customer):
    self.probe.discount_attempted(self.id, customer.id)
    if not customer.is_eligible_for_discount():
        self.probe.discount_ineligible(customer.id)
        return
    discount = self._calculate_discount(customer)
    self.probe.discount_applied(self.id, discount.amount)
    self.apply(discount)
```

`DiscountInstrumentation` 側に、3 つのテクニカルシステムへの実際の呼び出しが集約される:

```python
class DiscountInstrumentation:
    def __init__(self, logger, metrics, analytics):
        self.logger = logger
        self.metrics = metrics
        self.analytics = analytics

    def discount_attempted(self, cart_id, customer_id):
        self.logger.info("discount.apply.start", cart_id=cart_id, customer_id=customer_id)
        self.metrics.increment("discount.apply.attempt")

    def discount_ineligible(self, customer_id):
        self.logger.info("discount.apply.ineligible", customer_id=customer_id)
        self.metrics.increment("discount.apply.ineligible")

    def discount_applied(self, cart_id, amount):
        self.logger.info("discount.apply.applied", cart_id=cart_id, amount=amount)
        self.metrics.histogram("discount.amount", amount)
        self.analytics.track("discount_applied", cart_id=cart_id, amount=amount)
```

### この分離が何を生むか

1. ドメインコードの読みやすさ復活: 業務の意味が線形に読める
2. 計装の一元管理: ログ名・メトリクス名の命名規則を 1 ファイルで揃えられる
3. テストの分割可能性が劇的に上がる
   - ドメインテスト: 「`probe.discount_applied` が呼ばれたか」だけ検証すればよい（mock の Probe を渡せる）
   - 計装テスト: Probe クラス自体に対し、「ログ出力の形式が正しいか」を別途テストする
4. 計装システムの差し替えが、Probe クラス内に閉じ込められる

## ObservationContext によるメタデータ注入

リクエスト ID やテナント ID のような「全ログに付けたい横断情報」を、
ドメインコードが直接知らずに済むようにする工夫として `ObservationContext` を導入する。

部分適用（partial application）の発想で、Probe の生成時にこの context を注入しておけば、
ドメインから渡された業務情報と、context 由来のリクエスト情報が、Probe の中で合流して
ログに乗る。ドメインコードは「今のリクエスト ID は何か」を知らないままでよい。

これは Python なら `contextvars`、Go なら `context.Context` の典型的な使い方と相性がいい。

## なぜ AOP / デコレータでは不十分なのか

記事は AOP / 横断的関心事として計装を扱う方式を検討した上で、推奨しない理由を述べている。

> The granularity at which we observe domain behavior often doesn't match the granularity
> of the code.

ドメインで観測したい粒度（「割引が適用された」「顧客がレート上限に達した」）と、
コード上の関数呼び出し粒度はずれている。1 つの業務イベントは複数関数にまたがる場合もあれば、
1 つの関数の中に複数の業務イベントが含まれる場合もある。AOP で「関数の前後にログを差し込む」
タイプの解決策は、コード粒度に張り付くので、ドメインの語彙でログを残すのが難しくなる。

これは重要な指摘で、後の `05_techniques-aop-decorator-context.md` で再訪する。

## イベントベースの代替

Domain Probe の発展形として、ドメインが「割引が適用された」イベントを発火（announce）し、
複数のリスナーがそれぞれログ・メトリクス・アナリティクスを処理する形も挙げられている。

メリット: 計装システムの追加が、ドメインコードを一切変えずに済む
デメリット: 制御フローが追いにくくなる、テストでイベントバスのモックが必要になる

Hodgson は基本形（直接 Probe 呼び出し）を推奨し、複雑度が必要になってからイベント方式に
進化させる、というスタンス。

## 適用すべき場面・避けるべき場面

### 推奨される場面

- 高価値なビジネスドメインの計装: 課金、決済、レコメンド、ABテストなど
- 業務用語で観測したい場面（プロダクトメトリクスとして残したい場面）

### 避けるべき場面

- 既存コードへの一括導入。ホットスポット（最も計装が密になっている関数）から段階的に
- 低レベルのプラミング層（DB ドライバの計装など）。ここは技術用語でよいので、Probe を挟む価値が薄い

> Domain classes hold no direct reference to instrumentation systems

これが採用判定の指標として記事に書かれている。ドメインクラスが `Logger` を import している、
あるいはコンストラクタで受け取っている時点で、Probe 化の候補になる。

## 我々のコードベースへの含意

レコメンドプラットフォームでいうと、

- ランキング後処理（フィルタリング、リランキング、ビジネスルール適用）
- レコール戦略の切り替えロジック
- フィーチャー取得の fallback 判定

このあたりは「業務語彙で観測したい」典型的な場所。今は `logger.info("...", ...)` で
都度書いていると思うが、`RankingProbe` のような Domain Probe に集約すると、
ログ名の命名揺れも一掃できる。

ただし、ホットパスのループ内など、計装の粒度がコードの粒度と一致している場所では
そのまま logger を呼んだ方が読みやすい。原則は「ドメインの語彙で書きたい場所」だけ Probe 化。

## 出典

- [Domain-Oriented Observability — Martin Fowler / Pete Hodgson](https://martinfowler.com/articles/domain-oriented-observability.html)
