# SQL Server STRING_AGG の 8000 バイト上限エラーと対処法

## 概要

SQL Server の `STRING_AGG` 関数で集計結果が 8000 バイトを超えると `ProgrammingError` が発生する。入力値のキャスト型を `VARCHAR` から `NVARCHAR` に変更することで解決できる。

## 詳細

### エラー内容

```
(pyodbc.ProgrammingError) ('42000', '[42000] [Microsoft][ODBC Driver 18 for SQL Server][SQL Server]
STRING_AGG 集計結果が 8000 バイトの上限を超えています。結果の切り捨てを防止するには LOB 型をお使いください。 (9829) (SQLFetch)')
```

### LOB型とは

LOB（Large Object）型は、SQL Server で大容量データを格納するためのデータ型の総称。通常のデータ型には格納サイズの上限があるが、LOB 型はそれを大幅に超えるデータを扱える。

| 型 | 分類 | 最大サイズ |
|----|------|-----------|
| `VARCHAR(MAX)` | 文字列 LOB | 約 2GB |
| `NVARCHAR(MAX)` | Unicode 文字列 LOB | 約 2GB |
| `VARBINARY(MAX)` | バイナリ LOB | 約 2GB |
| `TEXT` / `NTEXT` / `IMAGE` | レガシー LOB（非推奨） | 約 2GB |

通常の `VARCHAR(n)` / `NVARCHAR(n)` は最大 8000 バイト / 4000 文字までだが、`MAX` 指定にすることで LOB 型となり最大約 2GB まで格納可能になる。

`STRING_AGG` のエラーメッセージで言及される「LOB 型をお使いください」とは、入力を `NVARCHAR` にキャストして戻り値を `NVARCHAR(MAX)` に昇格させることを意味する。

### 原因

`STRING_AGG` の戻り値の型は、入力の型によって決まる：

| 入力型 | 戻り値型 | 上限 |
|--------|----------|------|
| `VARCHAR(n)` | `VARCHAR(8000)` | 8000 バイト |
| `NVARCHAR(n)` | `NVARCHAR(MAX)` | 2GB |

`VARCHAR` 型を入力に使うと、結果は最大 8000 バイトに制限される。大量のレコードを連結する場合にこの上限を超えてエラーになる。

### 解決方法

`CAST` の型を `VARCHAR` から `NVARCHAR` に変更する。

**修正前（エラーになる）:**

```sql
SELECT STRING_AGG(CAST(a.AreaID AS VARCHAR(20)), ',') AS AreaID_list
FROM SomeTable a
```

**修正後（正常動作）:**

```sql
SELECT STRING_AGG(CAST(a.AreaID AS NVARCHAR(20)), ',') AS AreaID_list
FROM SomeTable a
```

`NVARCHAR` 型を入力にすると、`STRING_AGG` の戻り値が自動的に `NVARCHAR(MAX)`（LOB型）に昇格し、2GB まで格納できるようになる。

## 参考

- [Microsoft Docs - STRING_AGG (Transact-SQL)](https://learn.microsoft.com/ja-jp/sql/t-sql/functions/string-agg-transact-sql)

---
Generated: 2026-02-18
