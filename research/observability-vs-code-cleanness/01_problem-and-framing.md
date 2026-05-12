# 01. 問題提起と枠組み

## そもそも何が問題なのか

実プロジェクトのコードを読んでいると、こういう光景に頻繁に出会う。

```python
def apply_discount(cart, customer):
    logger.info("apply_discount.start", cart_id=cart.id, customer_id=customer.id)
    metrics.increment("discount.apply.attempt")

    if not customer.is_eligible_for_discount():
        logger.info("apply_discount.ineligible", customer_id=customer.id)
        metrics.increment("discount.apply.ineligible")
        return cart

    discount = calculate_discount(cart, customer)
    logger.info("apply_discount.calculated", amount=discount.amount)
    metrics.histogram("discount.amount", discount.amount)

    cart.apply(discount)
    logger.info("apply_discount.applied", cart_id=cart.id, amount=discount.amount)
    metrics.increment("discount.apply.success")
    analytics.track("discount_applied", {...})

    return cart
```

「割引を適用する」という業務ロジックの本体は数行のはずなのに、ログ・メトリクス・アナリティクスの
呼び出しが大半を占めている。Pete Hodgson は martinfowler.com の "Domain-Oriented Observability"
で、この状態を「ビジネスロジックがコードの 25% しか占めない」と表現した。これは比喩ではなく、
実際に多くのプロジェクトで観測される割合だ。

問題は単に行数が多いことではない。

- 業務ロジックを読みたい人が、計装の海から本筋を救い出さないといけない
- レビューで「ここのログ、レベル何が適切？」「メトリクス名は揃ってる？」といった、業務とは関係ない議論が割合を占める
- リファクタリング時に計装も一緒に動かさないといけない
- テストで `logger` や `metrics` をモックしないと壊れる、あるいは検証したい挙動と関係ない呼び出しが混ざる

つまり「計装の責任」と「業務の責任」が同じ関数に同居している状態が、保守性のコストを生んでいる。

## ユーザーの問題意識との対応

ユーザーは「コード汚染よりもオブザーバビリティのメリットが上回る」と判断している。
このトレードオフの認識は正しい。ただし、業界の議論はその先に進んでいる。

論点は次の二極化ではない。

- 計装を入れない（クリーン、しかし観測できない）
- 計装を入れる（汚れる、しかし観測できる）

業界が辿り着いた問いは、

- 計装の複雑性を、どのレイヤーに隔離するか
- ビジネスロジックの記述粒度を、計装の存在によって歪めないようにする手段は何か

である。この問いの立て方をすると、Domain Probe・Canonical Log Lines・Wide Events・ODD などの
パターンが、それぞれ「複雑性を別の場所に押し込むためのレシピ」として一望できる。

## 三つの主要な方向

調査の結果、ほぼ独立に発展した三つの方向性が見つかった。それぞれ別ファイルで扱う。

### A. パターンで隔離する: Domain Probe

ドメインクラスが直接 `logger` や `metrics` を持たないようにし、業務用語で書かれた中間オブジェクト
（Domain Probe）に投げる。技術詳細はそこに閉じ込める。Martin Fowler / Pete Hodgson が代表。
→ `02_domain-oriented-observability.md`

### B. 集約点で一気に出す: Canonical Log Lines / Wide Events

リクエスト処理中は context にデータを積むだけ、ミドルウェアの終端で 1 リクエスト 1 イベントを
吐き出す。Stripe (Canonical Log Lines) と Honeycomb (Wide Events) が独立に同じ場所に到達した。
→ `03_canonical-log-lines-and-wide-events.md`

### C. 計装を一級市民とする: Observability-Driven Development (ODD)

「計装はコードの汚染ではなく、コードの一部だ」と再定義する立場。Charity Majors が代表。
TDD のように「どう観測するか」を先に考える "shift left" の発想。
→ `04_observability-driven-development.md`

A と B はパターン的なアプローチ、C は文化的・方法論的アプローチで、現実には組み合わせて使う。

## なぜ今この議論が重要なのか

ユーザーが挙げた三つの理由

- 障害対応
- オンボーディング
- AI とのコンテキスト共有

の最後の項目が、業界の議論の最前線で急速に重みを増している。Charity Majors は最近、
「AI が生成したコードを本番で理解するには、良い observability が前提条件になる」と
明言している。コードを書くスピードが上がるほど、それが本番で何をしているかを問い直す
帯域が必要になる。観測性は AI 時代の前提インフラに格上げされつつある、というのが現在地。

## 出典

- [Domain-Oriented Observability — Martin Fowler / Pete Hodgson](https://martinfowler.com/articles/domain-oriented-observability.html)
- [How observability is redefining the roles of developers — Stack Overflow Blog](https://stackoverflow.blog/2022/07/18/how-observability-is-redefining-the-roles-of-developers/)
- [Observability: the present and future, with Charity Majors — Pragmatic Engineer](https://newsletter.pragmaticengineer.com/p/observability-the-present-and-future)
