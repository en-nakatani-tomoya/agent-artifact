# python exception chaining with from

## 概要
Pythonの例外チェーン（from e）の仕組みと、from eあり・なし・from Noneの違いを整理

## 詳細
Pythonの例外には__cause__（明示的チェーン）と__context__（暗黙的チェーン）の2つの属性がある。

### from e なし
exceptブロック内でraiseすると__context__に元の例外が暗黙的に設定される。トレースバックには「During handling of the above exception, another exception occurred」と表示される。

### from e あり
raise NewException() from eと書くと__cause__に元の例外が明示的に設定される。トレースバックには「The above exception was the direct cause of the following exception」と表示される。元の例外を意図的にラップしている場合はこちらが適切。

### from None
raise NewException() from Noneと書くと元の例外トレースバックが表示されなくなる。内部実装の詳細を外部に漏らしたくない場合に使う。

### 使い分け
| パターン | 設定属性 | 意味 |
|---|---|---|
| from e なし | __context__（暗黙） | 処理中にたまたま別の例外が起きた |
| from e あり | __cause__（明示） | この例外が原因で変換した |
| from None | __cause__ = None | 元の例外を意図的に隠す |

## コード例
```python
# from e あり（明示的チェーン）
try:
    response = db.query()
except DatabaseError as e:
    raise InfrastructureException(str(e)) from e

# from None（元の例外を隠す）
try:
    value = config['key']
except KeyError:
    raise ValueError('Invalid config') from None
```

## 参考
- https://docs.python.org/3/reference/simple_stmts.html#the-raise-statement
- https://docs.python.org/3/library/exceptions.html#exception-chaining

---
Generated: 2026-02-27
