# 05. 言語レベルのテクニック — Decorator, Context, AOP

## このファイルで扱うもの

02 〜 04 で扱ったのは「パターン」や「文化」だった。
ここでは具体的に Python / TypeScript / Go で計装を書くときの言語機構を整理する。
ユーザーが我々のレコメンドプラットフォームで実際に手を動かすときに役立つ部分。

## 1. Decorator パターン（関数横断の計装）

### 何ができるか

関数本体に手を加えずに、前後の計装を差し込める。

```python
from functools import wraps

def instrumented(operation_name):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            with tracer.start_span(operation_name) as span:
                span.set_attribute("args_count", len(args))
                try:
                    result = func(*args, **kwargs)
                    span.set_attribute("status", "ok")
                    return result
                except Exception as e:
                    span.set_attribute("status", "error")
                    span.set_attribute("error.type", type(e).__name__)
                    raise
        return wrapper
    return decorator

@instrumented("ranking.score")
def score_candidates(user_id, candidates):
    return _do_scoring(user_id, candidates)
```

`score_candidates` の本体はクリーンに保たれ、計装はデコレータに移動する。

### 強い場面

- 関数の入り口・出口だけ観測すれば十分なケース
- レイテンシ・成功/失敗のような汎用メトリクス
- 既存コードに後から横断的にメトリクスを足したいとき

### 弱い場面（Hodgson が指摘した点）

「ドメインで観測したい粒度」と「関数の粒度」がずれる場合に破綻する。

例: `apply_discount` という 1 つの関数の中に、

- 適格性チェック失敗
- 適用成功（金額情報あり）
- 上限到達による打ち切り

という 3 つの異なる業務イベントがある。デコレータは「関数 1 回呼ばれた」しか観測できないので、
内部の業務イベントは別の手段（Domain Probe や context への積み上げ）で取らないといけない。

つまりデコレータは技術的な計装に向き、業務イベントの計装には向かない。
両者は併用するのが現実解。

## 2. ContextVars / Async Context（横断情報の伝播）

### 何ができるか

リクエスト ID やユーザー ID のような「全ログに付けたい横断情報」を、
関数の引数として明示的に渡さずに伝播させる。

Python 3.7+ の `contextvars`:

```python
import contextvars

request_context = contextvars.ContextVar("request_context")

# リクエスト入り口（ミドルウェア）
def middleware(request, call_next):
    request_context.set({
        "request_id": request.headers["x-request-id"],
        "user_id": request.user.id,
    })
    return call_next(request)

# 業務コードのどこからでも参照できる
def some_deep_function():
    ctx = request_context.get()
    logger.info("event", request_id=ctx["request_id"], ...)
```

これは async/await でも安全に伝播する設計になっている（thread-local より進化した仕組み）。

### Go の context.Context との対比

Go では `context.Context` を関数の第一引数として明示的に渡すのが慣習で、暗黙伝播はしない。
これはこれで一貫していて読みやすい。Python の contextvars と Go の context.Context は、
「横断情報を関数引数の汚染なく伝える」という同じ目的を、違う美学で解く。

### 03 章の wide events との接続

contextvars に「リクエスト中に蓄積したい全部のフィールド」を持たせ、ミドルウェア終端で
1 イベントに変換する、というのが現代の典型構成。

```python
observation = contextvars.ContextVar("observation")

# 入り口
observation.set({})

# 各層
observation.get()["recall_strategies"] = ["A", "B"]
observation.get()["candidates_count"] = 120

# 終端
log_canonical(observation.get())
```

Python だと structlog のバインディングや、OpenTelemetry の Baggage / Span Attributes も
似た役割を果たす。

## 3. structlog の Bound Logger

### 何ができるか

ロガーに値を bind しておくと、以降のログにそれが自動付与される。
これは Python だと structlog がきれいに提供している。

```python
import structlog

log = structlog.get_logger()

def handle_request(request):
    log = structlog.get_logger().bind(
        request_id=request.headers["x-request-id"],
        user_id=request.user.id,
    )
    process(log, request)

def process(log, request):
    log.info("processing.start")  # request_id, user_id が自動で乗る
    log = log.bind(stage="recall")
    do_recall(log)
```

### Domain Probe との関係

structlog の bound logger は「ロガーをドメイン用語にラップする」初手として使える。

```python
class RankingProbe:
    def __init__(self, base_log):
        self._log = base_log.bind(component="ranking")

    def scoring_started(self, model_name, candidates_count):
        self._log.info("ranking.scoring.start",
                       model=model_name, candidates_count=candidates_count)

    def scoring_completed(self, latency_ms, top_score):
        self._log.info("ranking.scoring.done",
                       latency_ms=latency_ms, top_score=top_score)
```

業務語彙のメソッド `scoring_started` / `scoring_completed` を提供しつつ、
内部実装は構造化ログ。02 章の Domain Probe を Python で素直に書くとこうなる。

## 4. AOP（Aspect-Oriented Programming）

### 概念

クロスカット関心事（ログ、メトリクス、トランザクション、認可など）を、
業務コードから分離して別場所に書く方法論。

Java の Spring AOP、Python の `aspectlib` などが代表。

```python
# 業務コードは無垢
class RankingService:
    def score(self, user_id, candidates):
        return self._compute(user_id, candidates)

# どこか別の場所で「`score` メソッドの呼び出しの前後にログを差し込む」と宣言する
@aspect.before("RankingService.score")
def log_score_start(context):
    logger.info("ranking.score.start", args=context.args)

@aspect.after("RankingService.score")
def log_score_end(context):
    logger.info("ranking.score.end", duration=context.duration)
```

### Python AOP の現実

Python では、AOP のフルセット（pointcut, advice, weaving）を真面目に使うことは稀。
代わりに以下で同等のことをする:

- デコレータ（コンパイル時 weaving に相当）
- メタクラス・descriptor（クラス全体に横断的な変更を与える）
- monkey patching（runtime weaving）

実用的には「デコレータでだいたい足りる」が答え。フルセットの AOP は学習コストと
追跡困難性が割に合わないことが多い。

### AOP / デコレータ系の限界（再掲）

Hodgson の指摘どおり、「関数の前後」という粒度は、業務イベントの粒度と一致しないことが多い。
だから AOP は「技術的な計装」には強いが「ドメイン計装」には Domain Probe に劣る。

## 5. OpenTelemetry の自動計装

### 何ができるか

ライブラリ側が OpenTelemetry の API に対応していれば、ユーザコードに手を入れずに
HTTP リクエスト・DB クエリ・外部 API 呼び出しなどが自動で span として記録される。

```python
# requirements の追加と環境変数の設定だけで、
# requests, sqlalchemy, fastapi などの計装が自動で動く
from opentelemetry.instrumentation.auto_instrumentation import sitecustomize
```

### コード汚染ゼロのインフラ計装

これは「コードを 1 行も足さずに観測性が上がる」最強の例で、特にレイテンシ分析・依存関係の可視化に強い。

ただし限界もあり、自動計装は「フレームワークが何をしているか」しか観測できない。
「業務上どういう意思決定をしたか」（02 章の Domain Probe が扱う領域）は引き続き手書きの計装が必要。

### 推奨ステップ

OpenTelemetry のドキュメントが推奨しているのは:

1. 自動計装（zero-code）から始めて即座に可視性を得る
2. 業務固有の観測点を、手動計装でドメイン語彙で追加する

この二段階で「技術計装は自動で、業務計装は Domain Probe で」という分業が成立する。

## 6. ミドルウェア層への集約（再掲）

03 章の canonical log lines が示したように、もっとも侵襲性の低い計装ポイントは
ミドルウェア層（あるいは Python なら ASGI middleware、Go なら handler chain）。

ここで:

- リクエスト入り口で context 初期化
- リクエスト終端で wide event 出力
- panic / exception の捕捉とエラー計装

を一括で扱うと、業務コードに「計装の終端責任」が漏れ出さない。

## まとめ表

| テクニック | 強い場面 | 弱い場面 |
|---|---|---|
| Decorator | 関数の入口/出口の技術計装 | 業務イベントが関数粒度と一致しない場合 |
| ContextVars | 横断情報の暗黙伝播 | 明示的なほうが好まれる文化のとき |
| Bound Logger | 構造化ログの第一歩 | 業務語彙の抽象化までは届かない |
| AOP | 純粋な技術クロスカット関心事 | ドメイン粒度の計装 |
| 自動計装 (OTel) | フレームワーク・I/O 層の可視化 | 業務上の意思決定の観測 |
| ミドルウェア集約 | リクエスト境界での集約 | バッチ・ストリーミング処理 |

## 我々のコードベースへの含意

レコメンドプラットフォームに当てはめると、

- インフラ層（HTTP、DB、Redis、OpenSearch）: OpenTelemetry 自動計装に任せる
- 横断情報（request_id, user_id, experiment_group）: contextvars / structlog の bind で伝播
- 業務イベント（recall 戦略選択、ランキングモデル、フィルタ理由）: Domain Probe + canonical log line
- リクエスト終端: ミドルウェアで wide event を 1 行出力

この四層構成が、現代的な観測性のスタンダードに近い。
個別の層を一気に揃える必要はなく、価値の高い場所から段階導入できる。

## 出典

- [Aspect Oriented Programming in Python using Decorators — Helper Code](https://helpercode.com/2010/12/10/aspect-oriented-programming-in-python-using-decorators/)
- [Efficient Logging Mechanisms with the Decorator Pattern — Moments Log](https://www.momentslog.com/development/design-pattern/efficient-logging-mechanisms-with-the-decorator-pattern)
- [OpenTelemetry Instrumentation — opentelemetry.io](https://opentelemetry.io/docs/concepts/instrumentation/)
- [OpenTelemetry Zero-Code Instrumentation — DEV Community](https://dev.to/uptrace/opentelemetry-zero-code-instrumentation-1e1o)
- [structlog documentation](https://www.structlog.org/)
