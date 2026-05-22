# アップデート別の相性評価

結論として, このリポジトリと相性が良い順は `Change Tracking`, `Performance Risks Inbox`, `Cloud Cost Intelligence`, `APM OTel API support`
`Kubernetes` と `Mobile` は重要な機能ではあるが, このリポジトリ単体では直接の導入面が薄い

## サマリ

| 項目 | 相性 | 理由 | 最初に見る場所 |
| --- | --- | --- | --- |
| Change Tracking | 高 | Terraform, ECS deploy, batch 実行, model version 更新など変更起点が多く, APM / logs と相関しやすい | `infra/`, `.github/workflows/`, `api/recommend_work_api` |
| Performance Risks Inbox | 高 | API と datastore 呼び出しに New Relic agent trace が入り, latency 調査と相性が良い | `api/recommend_work_api/app/infrastructure/datastore/` |
| Cloud Cost Intelligence | 中から高 | AWS integration が既にあり, ECS, OpenSearch, ElastiCache, Glue, S3 などコスト対象が多い | `infra/modules/newrelic_aws_integration/` |
| APM OTel API support | 中 | 既存 New Relic Python agent を維持しつつ OTel 標準化の入口になる | `api/recommend_work_api/newrelic.ini` |
| Kubernetes Linux/Windows node monitoring | 低 | 現状は ECS Fargate 中心で Kubernetes node が見当たらない | なし |
| Mobile session replay | 低 | このリポジトリには mobile app のコードが見当たらない | なし |

## Change Tracking

相性は最も高い
理由は, このシステムの障害調査では「いつ何が変わったか」が APM, logs, metric と同じくらい重要になるため

このリポジトリで change event にしたいもの

- ECS service deploy
- batch task definition 更新
- Terraform apply 相当の infrastructure change
- model artifact / model version の切り替え
- OpenSearch index, DynamoDB table, ElastiCache 設定の変更
- QA や検証用の一時的なデータ操作
- alert threshold の変更

既存の接続点

- New Relic APM entity がある
- `monitoring/` に New Relic alert が Terraform 管理されている
- `docs/qa/evidence/` で `trace.id` を使った調査証跡が残っている
- GitHub Actions 経由で Terraform を動かす運用がある

期待効果

- latency 悪化や error rate 変化を deploy, config, model 更新と同一画面で照合できる
- QA 証跡に「この時刻の前後で何が変わったか」を残しやすい
- New Relic UI で `trace.id` と change event の両方から原因に寄せられる

注意点

- 変更イベントの粒度を細かくしすぎるとノイズになる
- `team`, `environment`, `service`, `change_type`, `source`, `commit_sha`, `pull_request`, `model_version` などの属性設計が先に必要
- Terraform はローカル実行禁止なので, GitHub Actions workflow から送る設計に寄せる

## Performance Risks Inbox

API latency と datastore 呼び出しの調査に効く可能性が高い

特に相性が良い箇所

- `api/recommend_work_api/app/infrastructure/datastore/handlers/valkey_handler.py`
- `api/recommend_work_api/app/infrastructure/datastore/handlers/aos_handler.py`
- `api/recommend_work_api/app/infrastructure/datastore/handlers/dynamodb_handler.py`
- `api/recommend_work_api/app/infrastructure/factory/user/user_factory_impl.py`
- `api/recommend_work_api/app/domain/domain_service/recall/recall.py`
- `api/recommend_work_api/app/domain/domain_service/ranking/ranking.py`

この API は推薦処理の中で Valkey, OpenSearch, DynamoDB, feature decode, ranking / reranking が絡む
既に `agent.function_trace()` が広く入っているため, Performance Risks Inbox が検出する slow HTTP, large payload, database / datastore 的な高コスト処理を見つけやすい土台がある

期待効果

- 95 percentile response time alert の発火前に, 遅い呼び出しや過剰な呼び出し傾向を拾える
- `monitoring/modules/api_alert/` の閾値監視だけでは分からないコード上の反復処理や外部呼び出しを見つけやすい
- QA の不具合調査で「ログを読む前に見る画面」として使える

注意点

- 2026年5月14日時点では Public Preview とされているため, 本番運用の必須導線にするのは早い
- Python agent の datastore 自動計測で New Relic がどこまで分類できるかは実画面確認が必要
- カスタム `function_trace()` の名前が粗いと原因箇所の粒度が足りない可能性がある

## Cloud Cost Intelligence

相性は中から高
既に New Relic AWS Cloud Integration があるため, 導入面の距離は近い

特に見る価値があるコスト対象

- ECS Fargate
- OpenSearch
- ElastiCache
- DynamoDB
- S3
- Glue Job
- Kinesis Data Firehose / CloudWatch Metric Streams
- New Relic Logs ingest

期待効果

- observability と infrastructure cost を同じ New Relic 上で見られる
- latency 改善や batch 処理増加が AWS cost にどう効いたかを追いやすい
- OpenSearch, ElastiCache, Fargate sizing の見直し候補をチームで共有しやすい
- AI Costs は, 将来 LLM API 利用や AI 推薦処理を入れる場合に見る余地がある

注意点

- CCI はタグ設計が弱いと team / service / environment 別の集計がぼやける
- `infra/modules/default_tags/` と AWS resource tag の整合を先に見るべき
- Kubernetes Cost Allocation はこのリポジトリでは直接効きにくい
- New Relic への取り込みが増える場合, New Relic 側の ingest cost も同時に見る

## APM OTel API support

相性は中
短期の改善というより, vendor lock-in を下げつつ既存 New Relic agent を活かす選択肢

このリポジトリでの使いどころ

- 新規の共通計測 API を作るときに OpenTelemetry API を採用する
- `agent.function_trace()` の直接依存を増やしすぎない
- 将来 New Relic 以外の collector や backend にも trace を流せる形に寄せる
- AWS Metric Streams の `opentelemetry1.0` 対応可否を別途確認する

期待効果

- New Relic 依存の計測コードを増やさずに telemetry を拡張できる
- OTel instrumented library を採用した場合にも APM 体験を保ちやすい
- 長期的に collector や metric pipeline の選択肢を残せる

注意点

- 既存コードは New Relic Python agent API にかなり依存している
- 一気に置き換えるより, 新規追加計測から OTel API を検証するのが現実的
- 公式サポート状況は agent version ごとの差があり得るため, 実際の Python agent release notes 確認が必要

## Kubernetes Linux/Windows node monitoring

相性は低
現状の主要実行基盤は ECS Fargate であり, Kubernetes node を運用している形跡がない

将来効くケース

- 推薦 API や batch を EKS に移す
- Windows container / mixed OS cluster を持つ
- Kubernetes Cost Allocation を使いたい

現時点では, このアップデートを起点に何か実装する優先度は低い

## Mobile session replay

相性は低
このリポジトリには iOS, Android, React Native など mobile app 実装が見当たらない

ただし, プロダクト全体で mobile app が別リポジトリにあるなら価値はある
推薦 API の `trace.id` や request id と mobile session replay を結べると, ユーザー操作から API latency / error まで追える

このリポジトリ側で関係する可能性があるもの

- API response に correlation id を返す設計
- mobile client から送られる request id を構造化ログに入れる設計
- PII / masking 方針とログ属性の見直し
