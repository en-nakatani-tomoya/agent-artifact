# 外部ソースメモ

Highspot の共有 URL はこの環境から本文取得できなかった
このメモでは, ユーザー共有の抜粋と New Relic 公式ページで確認できた内容を分けて残す

## ユーザー共有の抜粋

対象は「2026年4月分の New Relic アップデート」

- Cloud Cost Intelligence: クラウドコスト最適化の判断を迅速化
- Change Tracking: システムに影響を与えるすべての変更イベントを把握
- Mobile: モバイルアプリのユーザー操作を可視化
- Performance Risks Inbox: 深刻なパフォーマンスリスクを自動で検出し調査時間を削減
- APM: OTel API をサポートし最小コストで vendor 依存から脱却
- Kubernetes: Linux / Windows node を統合監視し一貫した運用を実現

## New Relic 公式で確認した内容

### Cloud Cost Intelligence

URL: https://docs.newrelic.com/whats-new/2026/04/whats-new-04-28-cci-ga/

確認日: 2026-05-22

確認できた要点

- 2026-04-28 に General Availability
- AWS, Azure, Google Cloud の cloud cost visibility を New Relic に統合
- Cost overview, drill-down analysis, Kubernetes cost allocation, budget management, cost optimization recommendations, AI cost management, alerts support が含まれる
- Kubernetes cost allocation は既存 New Relic Kubernetes telemetry を活用する説明

このリポジトリへの読み替え

- AWS integration が既にあるため入口は近い
- Kubernetes cost allocation より, ECS / OpenSearch / ElastiCache / Glue / S3 / Firehose の費用可視化の方が本命

### Change Tracking

URL: https://docs.newrelic.com/whats-new/2026/03/whats-new-03-12-change-tracking/

確認日: 2026-05-22

確認できた要点

- deploy 以外の system change も change event として扱える
- feature flag, configuration change, business event, custom change event を送れる
- NerdGraph mutation `changeTrackingCreateEvent` が案内されている
- change event を timeseries data と並べて root cause analysis に使う

このリポジトリへの読み替え

- GitHub Actions deploy, Terraform workflow, model version change, alert threshold change と相性が良い
- New Relic APM / logs / alert が既にあるため, change event の効果が出やすい

### Mobile session replay

URL: https://docs.newrelic.com/whats-new/2026/04/whats-new-04-28-mobile-session-replay-ga/

確認日: 2026-05-22

確認できた要点

- 2026-04-28 に General Availability
- mobile app session replay を breadcrumbs / logs などの telemetry と同期して確認できる
- PII は標準では収集されず, masking / sampling を server side で設定可能
- iOS UIKit, Android XML layouts, React Native Views, iOS SwiftUI, Android Jetpack Compose に対応
- GB ingest pricing に寄与する

このリポジトリへの読み替え

- mobile app 実装が見当たらないため直接導入対象ではない
- 別リポジトリに mobile app がある場合, API の correlation id 設計はこのリポジトリ側でも関係する

### Performance Risks Inbox

URL: https://docs.newrelic.com/whats-new/2026/05/whats-new-05-14-performance-risks-inbox/

URL: https://docs.newrelic.com/docs/errors-inbox/performance-risks/access-and-use/

確認日: 2026-05-22

確認できた要点

- 2026-05-14 時点で Public Preview
- N+1 queries, slow SQL queries, excessive database queries, sequential database queries, slow HTTP requests, large HTTP payloads などを検出対象としている
- APM entity view と account-level Errors Inbox からアクセスできる
- Triage view と Group performance risks view がある
- threshold は account / entity level で設定できる
- New Relic MCP server の `list_performance_risks()` でも取得可能とされている

このリポジトリへの読み替え

- `recommend_work_api` は datastore / ranking / recall 処理が多く, latency risk 検出と相性が良い
- Public Preview なので test / stg で確認してから運用判断する

### APM OTel API support

URL: https://newrelic.com/press-release/20260224-2

確認日: 2026-05-22

確認できた要点

- New Relic APM agents が OTel API compatibility と instrumentation support を既存 language agent に組み込む方針
- 既存 New Relic dashboard / alert との backward compatibility を保ちながら OTel 採用を進める説明
- Infra NRDOT と Collector Observability も案内されている

このリポジトリへの読み替え

- 既存 New Relic agent を捨てずに, 新規計測から OTel API を検証するのが現実的
- agent version ごとの Python 対応状況は別途確認が必要

### Kubernetes Windows node monitoring

URL: https://docs.newrelic.com/whats-new/2026/04/whats-new-4-16-windows-ga/

確認日: 2026-05-22

確認できた要点

- 2026-04-16 に General Availability
- `newrelic-infrastructure` chart で Linux / Windows node を統合監視
- `enableWindows: true` で Windows node monitoring を有効化
- Windows Server 2019 / 2022, EKS / AKS / GKE に対応
- HostProcess containers で CPU, memory, disk, network などの node level metrics を収集

このリポジトリへの読み替え

- 現状は ECS Fargate 中心で Kubernetes node は見当たらない
- 将来 EKS 移行や mixed OS cluster が出るまでは優先度が低い

## リポジトリ側で確認した主な根拠

- `api/recommend_work_api/app/main.py`: New Relic Python agent 初期化
- `api/recommend_work_api/app/config/logging_config.py`: New Relic linking metadata を構造化ログへ付与
- `api/recommend_work_api/newrelic.ini`: APM 設定
- `infra/modules/ecs/ecs_task_definition.tf`: ECS Fargate / FireLens 構成
- `shared/firelens/envs/test/fluent-bit.conf`: New Relic Logs output
- `infra/modules/newrelic_aws_integration/metric_stream.tf`: CloudWatch Metric Streams を New Relic へ push
- `monitoring/modules/api_alert/`: API response time / error rate alert
- `monitoring/modules/elasticache/`: ElastiCache alert
- `monitoring/modules/opensearch/`: OpenSearch alert
