# FM-based Recall and KNN Vector Design Summary

## 概要
FMでユーザー特徴とアイテム特徴を同時に学習し、学習済み埋め込みをベクトル検索に再利用する設計について整理した。主な論点は、`k` と `K` の混同回避、2つのNotebookの設計差分、推論時にユーザー/アイテムベクトルを別管理して成立するかどうかである。

## 詳細

### 用語の固定
- `k`: FMの潜在因子次元（例: 16, 100, 200）
- `K`: KNN/ANNで返す近傍件数（Top-K）
- FMの最終出力: 通常は1スカラー（logit）
- 検索ベクトル次元: 実装により `k` または `k+1`

### FM定義に関する確認
- PyTorch実装のFMは、一次項・二次項・バイアスを含む標準形で妥当。
- ただし、その後のKNN入力に何を使うかは別設計であり、FM本体の最終logit次元と検索ベクトル次元は同一である必要はない。

### 2つのNotebookの共通点と差分
- 共通点:
  - 学習時は user特徴 + item特徴の組を使ってFMを学習。
  - 学習後は埋め込みを集約してユーザー/アイテムの検索ベクトルを作る。
- 差分:
  - `(Clone) ...FMモデル.ipynb` は主に `sum(V)` ベース（相性中心）。
  - `FM recall model deploy_script.ipynb` は先頭に補助スカラーを付与:
    - user: `[1, sum(V_user)]`
    - item: `[first+second, sum(V_item)]`
  - これにより内積が `b_item + u・v_item` になり、相性に加えてアイテム側のベーススコアを含められる。

### 次元の解釈
- `k=100` のとき、先頭1次元を付ける設計なら検索ベクトルは `101` 次元。
- `201` 次元で検索している場合は、実行時に `k=200` モデルを使っている可能性が高い。
- `K=1000` のような値は近傍件数であり、ベクトル次元とは無関係。

### 「学習はペア、推論は分離」で成立する理由
- FMスコアは user側要素、item側要素、user-itemクロス項に分解可能。
- 学習済み埋め込みから user_vec と item_vec を別々に構築し、内積検索に使う運用は成立する。
- 実務ではリコール段でANNを使い、必要に応じて後段ランカーで再スコアする運用が安定しやすい。

## コード例
```python
# 例: item側にベーススコアを追加した検索ベクトル
outputs = tf.concat((first_order + second_order, embedding_sum), axis=1)
# 内積検索時: [1, u] ・ [b_item, v_item] = b_item + u・v_item
```

## 参考
- FM定義と埋め込み集約の議論ログ（本対話）
- `ref/(Clone) エン転職_リコールのFMモデル.ipynb`
- `ref/FM recall model deploy_script.ipynb`

---
Generated: 2026-02-16
