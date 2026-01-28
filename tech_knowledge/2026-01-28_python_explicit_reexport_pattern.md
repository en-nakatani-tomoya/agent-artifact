# Python の明示的 re-export パターン

## 概要

`from .module import X as X` という記法は、型チェッカーに対して「これは意図的な re-export である」ことを明示するPythonのベストプラクティスです。

## 詳細

### 型チェッカーの挙動

型チェッカー（mypy, pyright）は、インポートの書き方によって公開APIかどうかを判断します：

| 記法 | 解釈 |
|------|------|
| `from X import Y` | プライベートインポート（内部使用のみ） |
| `from X import Y as Y` | 明示的な re-export（公開API） |

### なぜ必要か

- mypy の `--strict` モードや `implicit_reexport = False` 設定では、`as X` がないと他のモジュールからインポートした際にエラーになる
- 型チェッカーは「このモジュールが本当に公開したい型」と「内部実装で使っているだけの型」を区別できる

### TYPE_CHECKING ブロックでの使用例

```python
from typing import TYPE_CHECKING

if TYPE_CHECKING:
    # 明示的 re-export: 公開APIとして認識される
    from .transformer import Transformer as Transformer
    
    # プライベートインポート: 内部使用のみ
    from .internal import _Helper
```

### ベストプラクティス

`__all__` と `as X` の両方を使うことで、ランタイムと型チェック両方で正しく公開APIとして認識されます：

```python
from typing import TYPE_CHECKING

__all__ = ["Transformer", "process"]

if TYPE_CHECKING:
    from .transformer import Transformer as Transformer

from .processor import process as process
```

## 参考

- [PEP 484 - Type Hints](https://peps.python.org/pep-0484/)
- [mypy - Stub files](https://mypy.readthedocs.io/en/stable/stubs.html)
- [pyright - Type Stubs](https://github.com/microsoft/pyright/blob/main/docs/type-stubs.md)

---
Generated: 2026-01-28
