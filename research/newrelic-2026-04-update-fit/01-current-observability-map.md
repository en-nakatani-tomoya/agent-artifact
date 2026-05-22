# 現状の New Relic 利用状況

このリポジトリは New Relic をすでに APM, 構造化ログ, AWS infrastructure metrics, Terraform 管理の alert に使っている
そのため, 追加機能の評価では「ゼロ導入」ではなく, 既存テレメトリに何を重ねると効果が出るかを重視する

## アプリケーション APM

`api/recommend_work_api/app/main.py` で New Relic Python agent を初期化している

- `api/recommend_work_api/newrelic.ini`
- `backend/service/count_user_imp/newrelic.ini`
- `backend/service/user_embed_updater/newrelic.ini`
- `backend/service/user_ranking_feature_updater/newrelic.ini`
- `backend/service/work_embed_updater/newrelic.ini`
- `backend/service/work_ranking_feature_updater/newrelic.ini`

`api/recommend_work_api` と複数 batch service に `newrelic` Python package が入っている
`agent.function_trace()` と `agent.background_task()` が API 処理, feature 取得, ranking, recall, batch 処理に広く付与されている

代表例

- `api/recommend_work_api/app/presentations/rest/route/works_v1.py`
- `api/recommend_work_api/app/domain/domain_service/recall/recall.py`
- `api/recommend_work_api/app/infrastructure/datastore/handlers/valkey_handler.py`
- `backend/service/user_ranking_feature_updater/app/main.py`
- `backend/service/work_embed_updater/app/entrypoint/register.py`

## ログ連携

ECS の FireLens 経由で New Relic Logs に送っている

- `infra/modules/ecs/ecs_task_definition.tf`
- `shared/firelens/envs/test/fluent-bit.conf`
- `shared/firelens/envs/stg/fluent-bit.conf`
- `shared/firelens/envs/prod/fluent-bit.conf`

`api/recommend_work_api/app/config/logging_config.py` では `trace.id`, `span.id`, `entity.guid`, `entity.name` など New Relic linking metadata を構造化ログに付与している
QA 証跡でも `trace.id` から New Relic Logs を辿る運用がある

## AWS infrastructure metrics

AWS CloudWatch Metric Streams と Kinesis Data Firehose で New Relic に push している

- `infra/modules/newrelic_aws_integration/README.md`
- `infra/modules/newrelic_aws_integration/metric_stream.tf`
- `infra/envs/test/main_newrelic_aws_integration.tf`
- `infra/envs/stg/main_newrelic_aws_integration.tf`
- `infra/envs/prod/main_newrelic_aws_integration.tf`

現状の `aws_cloudwatch_metric_stream.output_format` は `opentelemetry0.7`
コメント上は `json / opentelemetry0.7 / opentelemetry1.0` が選択肢として認識されている

## Terraform monitoring

`monitoring/` で New Relic provider を使い, alert policy と conditions を管理している

- `monitoring/envs/test/main_recommend_work_api.tf`
- `monitoring/modules/api_alert/response_time.tf`
- `monitoring/modules/api_alert/anomaly_response_time.tf`
- `monitoring/modules/api_alert/error_rate.tf`
- `monitoring/modules/elasticache/`
- `monitoring/modules/opensearch/`
- `monitoring/modules/api_ecs_cluster/`
- `monitoring/modules/batch_ecs_cluster/`

API は transaction duration, error rate, alive monitoring を見る構成
ECS cluster, OpenSearch, ElastiCache も New Relic alert 化されている

## 実行基盤

主な workload は ECS Fargate と Glue Job
`infra/modules/ecs/ecs_task_definition.tf` では `requires_compatibilities = ["FARGATE"]`, `operating_system_family = "LINUX"` を使う

Kubernetes manifest, Helm values, EKS node group などは主要構成として見当たらない
そのため Kubernetes Windows node monitoring は現状の本リポジトリには直接効きにくい

## 相性評価で重視する観点

- 既存 APM / logs / metrics をそのまま使って効果が出るか
- Terraform / GitHub Actions / ECS deploy と連携できるか
- 推薦 API の latency, recall, ranking, datastore 呼び出しの調査時間を短縮できるか
- New Relic ingest cost の増加に見合うか
- 個人 QA 調査ではなくチーム運用に乗せられるか
