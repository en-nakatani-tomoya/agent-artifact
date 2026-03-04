# python __hash__ implementation with tuple packing

## 概要
Pythonで__hash__を実装する際、属性をタプルにまとめてhash()を呼ぶ方法が公式ドキュメントで推奨されている。

## 詳細
Python公式ドキュメント（Data Model - object.__hash__）に以下の記述がある：

「it is advised to mix together the hash values of the components of the object that also play a part in comparison of objects by packing them into a tuple and hashing the tuple.」

つまり、__eq__で比較に使う属性と同じ属性をタプルにまとめてhash()を呼ぶのが推奨パターン。

### 推奨実装例

```python
def __hash__(self):
    return hash((self.name, self.nick, self.color))
```

### 属性が1つだけの場合

属性が1つだけの場合は、その属性の__hash__()を直接呼んでも実質同じ。

```python
def __hash__(self) -> int:
    return self.value.__hash__()
```

### 注意点
- __hash__を実装する場合は__eq__も実装すべき
- hashが同じでも__eq__がTrueでなければdict/setで同一とは扱われない（ハッシュ衝突として処理される）
- ハッシュに使う属性は不変（immutable）であるべき

## 参考
- https://docs.python.org/3/reference/datamodel.html#object.__hash__
- https://docs.python.org/3/glossary.html#term-hashable

---
Generated: 2026-03-04
