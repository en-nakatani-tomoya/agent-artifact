# elasticache cpu utilization vs engine cpu utilization

## 概要
ElastiCache (Valkey/Redis) の CloudWatch メトリクス CPUUtilization と EngineCPUUtilization の違いと、両方を見るべき理由を整理

## 詳細
CPUUtilization はホスト（インスタンス）全体の全 vCPU の使用率を示す。Valkey エンジン本体に加え、バックグラウンドタスクや OS の処理など、ノード上で動く全プロセスの合計。

EngineCPUUtilization は Valkey/Redis のメインエンジンスレッド 1 本だけの CPU 使用率を示す。Valkey/Redis OSS は基本シングルスレッドでコマンドを処理するため、エンジン専用の指標として用意されている。

両方を見るべき理由:

1. CPUUtilization だけだと過小評価する
   メインエンジンはシングルスレッドなので、マルチ vCPU インスタンスでは 1 コアしか使わない。例えば 4 vCPU インスタンスでエンジンが 1 コアを 100% 使い切っていても、ホスト全体の CPUUtilization は 25% にしか見えず「まだ余裕」と誤認しがち。

2. EngineCPUUtilization がリクエスト処理の真のボトルネックを示す
   コマンド処理が詰まる本当の上限はエンジンスレッドの飽和。スケールアップ判断はこの指標で行うのが AWS 公式の推奨。

3. CPUUtilization はバックグラウンド負荷の検知に使う
   バックアップ、レプリケーション、メンテナンス等の非エンジン処理のコストはこちらに出る。エンジンは暇でもホストが高い、という状況の検知に有効。

つまり「エンジン本体の処理限界」と「ノード全体の負荷」を別々に観測するための 2 軸。AWS 公式ドキュメントも両方の併用を推奨している。

注意点: 1 vCPU の小型ノード（cache.t2/t3.micro 等）では両者の値はほぼ一致する。差が出るのはマルチ vCPU ノードのみ。

## 参考
- https://docs.aws.amazon.com/AmazonElastiCache/latest/dg/CacheMetrics.Redis.html
- https://docs.aws.amazon.com/AmazonElastiCache/latest/dg/CacheMetrics.WhichShouldIMonitor.html
- https://repost.aws/knowledge-center/elasticache-redis-high-cpu-usage
- https://aws.amazon.com/about-aws/whats-new/2018/04/amazon-elastiCache-for-redis-introduces-new-cpu-utilization-metric-for-better-visibility-into-redis-workloads/

---
Generated: 2026-04-27
