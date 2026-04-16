# murr columnar cache for ai workloads

## 概要
Murr (murrdb) は AI/ML 推論向けに特化した Rust 製 columnar キャッシュ層。Redis/Valkey の汎用代替ではなく、feature serving・batch inference など AI ワークロード専用のポジション。

## 詳細
## 概要
- Rust 製 (86%) + Python bindings (7.4%)
- Apache 2.0 / v0.2.0-rc1 (2026-04 時点 early-stage)
- インストール: pip install murrdb
- プロトコル: HTTP REST (port 8080) + Arrow Flight gRPC (port 8081)

## 特徴
- Columnar batch I/O: 行ごとのオーバーヘッドなしでバッチ読み書き
- Zero-copy protocol: np.ndarray / pd.DataFrame / torch.Tensor に変換なしで構築可能
- Tiered storage: hot はメモリ、cold はディスク + S3 レプリケーション
- Stateless 設計: 状態は S3 に永続化、ノードは起動時に自己復元 (self-bootstrap)
- Immutable segment モデル (Apache Lucene 着想、.seg ファイル、memmap2 で mmap 読み取り)
- Last-write-wins での key resolution (インクリメンタル更新)

## 性能主張
- HTTP + Arrow IPC で 104μs レイテンシ / 9.63M keys/sec
- 最適化された Redis レイアウトより 約 2.3〜2.5 倍速い raw read
- Python ingestion で Redis 比 17x、Feast 比 36x
- DynamoDB 比で ~10x 安価 (CPU/RAM 課金ベース vs per-query)

## 向き・不向き
### 向いている用途
- ML ランキングワークロード
- Feature serving (オンライン推論の feature store)
- バッチ推論データアクセス
- embedding キャッシュ系

### 向いていない用途
- OLTP ワークロード
- 集計分析 (analytics aggregations)
- 汎用キャッシュ (セッション、rate limit など)
- feature governance シナリオ

## 競合ポジショニング
- vs Redis: 永続化あり、cold data を NVMe にオフロード可能
- vs RocksDB: 分散機能がビルトイン、手動同期不要
- vs DynamoDB: CPU/RAM 課金で ~10x 安価との主張
- vs Postgres: zero-copy protocol でパース不要

## 所感・導入判断
「Redis/Valkey を使いたい場面で murr」は用途次第。汎用キャッシュ用途 (セッション、分散ロック、pub/sub) は対象外なので、recommend-platform 文脈なら feature store や embedding キャッシュ系で検討価値あり。early-stage (v0.2.0-rc1) なので本番投入は慎重に、まずは PoC レベルから。

## 参考
- https://github.com/murrdb/murr
- http://murrdb.io/

---
Generated: 2026-04-16
