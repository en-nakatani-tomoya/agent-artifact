# python exception chaining with from

## 概要
Pythonの例外チェーン（from e）の仕組みと、from eあり・なし・from Noneの違いを整理

## 詳細
Pythonの例外には__cause__（明示的チェーン）と__context__（暗黙的チェーン）の2つの属性がある。\n\n### from e なし\nexceptブロック内でraiseすると__context__に元の例外が暗黙的に設定される。トレースバックには「During handling of the above exception, another exception occurred」と表示される。\n\n### from e あり\nraise NewException() from eと書くと__cause__に元の例外が明示的に設定される。トレースバックには「The above exception was the direct cause of the following exception」と表示される。元の例外を意図的にラップしている場合はこちらが適切。\n\n### from None\nraise NewException() from Noneと書くと元の例外トレースバックが表示されなくなる。内部実装の詳細を外部に漏らしたくない場合に使う。\n\n### 使い分け\n| パターン | 設定属性 | 意味 |\n|---|---|---|\n| from e なし | __context__（暗黙） | 処理中にたまたま別の例外が起きた |\n| from e あり | __cause__（明示） | この例外が原因で変換した |\n| from None | __cause__ = None | 元の例外を意図的に隠す |

## コード例
```
# from e あり（明示的チェーン）\ntry:\n    response = db.query()\nexcept DatabaseError as e:\n    raise InfrastructureException(str(e)) from e\n\n# from None（元の例外を隠す）\ntry:\n    value = config['key']\nexcept KeyError:\n    raise ValueError('Invalid config') from None
```

## 参考
- https://docs.python.org/3/reference/simple_stmts.html#the-raise-statement
- https://docs.python.org/3/library/exceptions.html#exception-chaining

---
Generated: 2026-02-27
