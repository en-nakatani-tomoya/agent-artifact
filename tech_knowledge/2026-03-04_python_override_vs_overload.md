# python override vs overload

## 概要
Pythonにおけるオーバーライドとオーバーロードの違い。親クラスのメソッドを再定義するのはオーバーライド。

## 詳細
### オーバーライド（override）
親クラスから継承したメソッドを子クラスで同じシグネチャで再定義すること。
例：objectが持つ__hash__を子クラスで再定義する場合はオーバーライド。

```python
class WorkId(ValueObject):
    value: str

    def __hash__(self) -> int:  # objectの__hash__をオーバーライド
        return self.value.__hash__()
```

### オーバーロード（overload）
同じ名前のメソッドを異なるシグネチャ（引数の型や数）で多重定義すること。
Java や C++ では言語機能として存在するが、Pythonには言語レベルのオーバーロード機構はない。

```python
# typing.overloadは型ヒント用であり、実行時には機能しない
from typing import overload

@overload
def process(x: int) -> int: ...
@overload
def process(x: str) -> str: ...
```

### まとめ
- 親クラスのメソッドを再定義 → オーバーライド
- 同名メソッドを異なるシグネチャで定義 → オーバーロード（Pythonでは言語レベルで非サポート）

---
Generated: 2026-03-04
