# Factorization Machine (FM) の仕組み

## 概要

Factorization Machine (FM) は、高次元スパースデータにおける特徴間の交互作用を効率的にモデル化する機械学習手法です。特にレコメンデーションシステムやCTR予測で広く使用されています。

## 詳細

### 基本式

FMの予測式は以下の通りです：

```
ŷ = w₀ + Σᵢ wᵢxᵢ + Σᵢ Σⱼ₍ⱼ>ᵢ₎ <vᵢ, vⱼ> xᵢxⱼ
```

各項の意味：

| 項 | 説明 |
|---|------|
| `w₀` | バイアス項（グローバルな傾向） |
| `Σᵢ wᵢxᵢ` | 線形項（各特徴の個別の影響） |
| `Σᵢ Σⱼ₍ⱼ>ᵢ₎ <vᵢ, vⱼ> xᵢxⱼ` | 交互作用項（特徴間の組み合わせ効果） |

### 交互作用項の計算量削減トリック

愚直に計算すると O(n²k) の計算量がかかりますが、以下の数学的トリックで O(nk) に削減できます：

```
Σᵢ Σⱼ₍ⱼ>ᵢ₎ <vᵢ, vⱼ> xᵢxⱼ = 0.5 × Σₖ [(Σᵢ vᵢₖxᵢ)² - Σᵢ (vᵢₖxᵢ)²]
```

つまり：
- `sum_square = (Σvᵢ)²` 
- `square_sum = Σ(vᵢ²)`
- `fm_part = 0.5 × (sum_square - square_sum)`

## コード例

### PyTorch実装

```python
import torch
import torch.nn as nn

class FactorizationMachine(nn.Module):
    def __init__(self, num_features: int, embed_dim: int):
        super().__init__()
        # 線形項: 各特徴に1次元の重み wᵢ を割り当て
        self.linear = nn.Embedding(num_features, 1)
        
        # 交互作用項: 各特徴に embed_dim 次元のベクトル vᵢ を割り当て
        self.embedding = nn.Embedding(num_features, embed_dim)
        
        # グローバルバイアス w₀
        self.bias = nn.Parameter(torch.zeros(1))
        
    def forward(self, x: torch.Tensor) -> torch.Tensor:
        """
        Args:
            x: (batch_size, num_fields) 特徴インデックス
        Returns:
            (batch_size,) 予測スコア
        """
        # 線形項: Σ wᵢxᵢ
        linear_part = self.linear(x).sum(dim=1).squeeze()
        
        # 交互作用項の計算（O(nk) トリック）
        embed = self.embedding(x)  # (batch, fields, embed_dim)
        
        sum_square = embed.sum(dim=1).pow(2)  # (Σvᵢ)²
        square_sum = embed.pow(2).sum(dim=1)   # Σ(vᵢ²)
        
        fm_part = 0.5 * (sum_square - square_sum).sum(dim=1)
        
        return self.bias + linear_part + fm_part
```

### 重み値の取得

```python
# 特定の特徴の線形重み wᵢ を取得
feature_idx = 42
linear_weight = fm.linear.weight[feature_idx].item()

# 特定の特徴の埋め込みベクトル vᵢ を取得
embedding_vector = fm.embedding.weight[feature_idx]
```

## FMが有用な理由

1. **スパースデータに強い**: 高次元で疎なデータでも、低次元の潜在ベクトルで交互作用を表現
2. **効率的**: 交互作用を O(nk) で計算可能（k は埋め込み次元）
3. **過学習を抑制**: 低ランク分解により、パラメータ数を抑えて汎化性能を向上

## 参考

- [Factorization Machines (Rendle, 2010)](https://www.csie.ntu.edu.tw/~b97053/paper/Rendle2010FM.pdf)
- [DeepFM: A Factorization-Machine based Neural Network for CTR Prediction](https://arxiv.org/abs/1703.04247)

---
Generated: 2026-01-28
