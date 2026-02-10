# First Class Collection（ファーストクラスコレクション）

## 概要

First Class Collection とは、プリミティブなコレクション（list, set 等）を1つだけフィールドに持つ専用のラッパークラスを作る設計パターン。DDD やオブジェクト指向設計において、コレクションにビジネスルールや振る舞いを持たせるために用いられる。

## 「First Class」の語源

プログラミング言語論における **First Class Citizen（第一級オブジェクト/第一級市民）** に由来する。

「第一級」とは、言語の中で他の要素と同等の権利を持つことを意味する：

- 変数に代入できる
- 引数として渡せる
- 戻り値として返せる
- 独自の振る舞い（メソッド）を持てる

生のコレクション（`list[WorkId]` など）は「ただのデータの入れ物」にすぎないが、専用クラスでラップすることで、ドメインの中で独立した意味を持つ「第一級」の存在に昇格させる、というのがこのパターンの本質。

## 生のコレクションとの比較

| 状態 | 扱い |
|---|---|
| `list[WorkId]`（生のリスト） | ただのデータ構造。ビジネスルールを持てない |
| `WorkIds`（ラップしたクラス） | 独自の振る舞いとルールを持つドメインオブジェクト |

## コード例

Python（Pydantic）での実装例：

```python
from pydantic import field_validator
from app.domain.value_objects.value_object import ValueObject
from app.domain.value_objects.work.work_id import WorkId


class WorkIds(ValueObject):
    work_id_list: list[WorkId]

    # ビジネスルール: 空リスト禁止
    @field_validator("work_id_list")
    @classmethod
    def work_ids_must_not_be_empty(cls, v: list[WorkId]) -> list[WorkId]:
        if len(v) == 0:
            raise ValueError("work_ids must not be empty")
        return v

    # 重複排除しながら追加
    def add(self, work_id: WorkId) -> "WorkIds":
        work_id_list = list(set(self.work_id_list + [work_id]))
        return WorkIds(work_id_list=work_id_list)

    def extend(self, work_ids: "WorkIds") -> "WorkIds":
        work_id_list: list[WorkId] = list(set(self.work_id_list + work_ids.work_id_list))
        return WorkIds(work_id_list=work_id_list)

    def convert_to_str_list(self) -> list[str]:
        return [work_id.value for work_id in self.work_id_list]

    def __iter__(self):
        yield from self.work_id_list
```

## メリット

1. **ビジネスルールのカプセル化**: バリデーション（空リスト禁止）や重複排除などのルールをクラス内に閉じ込められる
2. **型安全性の向上**: `list[WorkId]` より `WorkIds` の方が意図が明確で、型チェックも厳密になる
3. **利用側のコード簡潔化**: ガード節やバリデーションを利用側で書く必要がなくなる
4. **変更の局所化**: コレクションに関するロジック変更がクラス内で完結する

## 出典

ThoughtWorks 社の Jeff Bay が「Object Calisthenics（オブジェクト指向エクササイズ）」というルール集の中で提唱。「すべてのプリミティブなコレクションをラップせよ」というルールとして紹介された。

## 参考

- [ThoughtWorks Anthology - Object Calisthenics](https://www.cs.helsinki.fi/u/luMDluo/ohjelmointi/rules.html)
- [Martin Fowler - Collection Object](https://martinfowler.com/bliki/CollectionObject.html)

---
Generated: 2026-02-10
