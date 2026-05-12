# tensorflow serving parallelism parameters

## 概要
tensorflow_model_server の rest_api_port / model_config_file / tensorflow_intra_op_parallelism / tensorflow_inter_op_parallelism / rest_api_num_threads がモデル推論にどう作用するかを計算グラフレベルで整理した。

## 詳細
## 各パラメータの定義（公式 main.cc の help より）

- rest_api_port: HTTP/REST API のリッスンポート。0 なら無効化。--port (gRPC) と別値必須。
- model_config_file: ASCII ModelServerConfig protobuf。複数モデル/バージョンポリシーを宣言。指定時 --model_name / --model_base_path は無視。
- tensorflow_intra_op_parallelism: 1 つの op の実行を並列化するスレッド数。デフォルト auto。--platform_config_file 指定時は無視。
- tensorflow_inter_op_parallelism: 同時に実行可能な op の数。デフォルト auto。
- rest_api_num_threads: HTTP/REST API 処理スレッド数。未設定なら CPU 数から自動設定。

## レイヤ別の役割

- ネットワーク層: rest_api_port
- モデル管理層: model_config_file
- HTTP サーバ層: rest_api_num_threads (JSON パース・Tensor 化・Session.Run 呼び出しのスレッドプール、同時受付の上限)
- TF ランタイム層: tensorflow_intra_op_parallelism / tensorflow_inter_op_parallelism

## op (Operation) とは

計算グラフを構成する演算ノード単位。MatMul / Conv2D / Add / Relu / BiasAdd / Softmax など。Dense 層は MatMul + BiasAdd + Activation の組み合わせで表現される。

## intra-op と inter-op の違い

- intra-op: 1 つの op の内部計算を複数スレッドで分割。大きな MatMul の出力次元をスレッド分割するイメージ。MatMul / Conv2D / BatchMatMul / Reduce 系で効く。EmbeddingLookup や Reshape では効きにくい。
- inter-op: 依存関係のない複数の op を同時並行で実行。マルチタワーや並列ブランチを持つモデル (DeepFM, 2-tower) で効く。

## モデル推論における作用（DeepFM 例）

1. リクエスト到着 → rest_api_num_threads のプールがハンドル (JSON→Tensor)
2. Session.Run 起動
3. TF ランタイムが計算グラフを実行:
   - 独立な EmbeddingLookup 群を inter_op の範囲で並列実行
   - FM 部と Deep 部 (Concat 前まで独立) を inter_op で並列実行
   - 各 MatMul / Conv2D を intra_op スレッドで内部分割
4. Concat → 出力 MatMul → Sigmoid → レスポンス

## 経緯

元々 --tensorflow_session_parallelism 1 つで両方制御していたが、intra と inter の最適値が異なるため issue #1237 を経て分離された。

## チューニング観点

- 効果はワークロード依存。auto-config がベースライン、手動指定する場合は実測で最適値を探るのが公式推奨 (g3doc/performance.md)。
- 同時稼働スレッドのピークは概ね intra × inter。CPU コア数を超えるとコンテキストスイッチで遅くなる。
- rest_api_num_threads × (intra × inter) が CPU 数を大幅に超えるとスレッド争奪が起きる。CPU コア数 ≈ intra × inter に揃え、rest はバースト吸収の役割で設計するのが定石。

## モデル特性別の効き方

- マルチタワー / 並列ブランチ多: inter ◎ / intra ○
- 1 本道の深い MLP: inter △ / intra ◎
- 巨大行列演算支配的: inter △ / intra ◎
- 小さい op 大量: inter ○ / intra △
- EmbeddingLookup 中心 (I/O 律速): どちらも伸びにくい

## 参考
- https://github.com/tensorflow/serving/blob/master/tensorflow_serving/model_servers/main.cc
- https://github.com/tensorflow/serving/blob/master/tensorflow_serving/g3doc/performance.md
- https://github.com/tensorflow/serving/issues/1237

---
Generated: 2026-05-12
