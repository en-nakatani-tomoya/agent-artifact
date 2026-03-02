# HIP3 ventuals valuation unit

## 概要
Hyperliquid HIP-3上のVentualsでは、企業の評価額を10億（1 billion）で割った値を1取引単位（1 contract）の価格として採用している。

## 詳細
Ventuals（HIP-3）は未上場・プレIPO企業の評価額に対する無期限先物（Perpetual Futures）取引プラットフォーム。取引単位は「Valuation Units」と呼ばれ、企業評価額を10億で割った値が1contractの価格となる。例: SpaceXの評価額が$420.69Bの場合、1 SPACEXの価格は$420.69。これは読みやすさ・取引しやすさのための設計。オラクル価格はオフチェーンの二次市場データと8時間EMAのマーク価格を50/50で加重平均し、マーク価格はオラクルの±20%以内に制限される。

## 参考
- https://docs.ventuals.com/perp-specifications/private-companies
- https://docs.ventuals.com/overview/markets

---
Generated: 2026-03-02
