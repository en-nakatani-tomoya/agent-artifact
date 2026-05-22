# 導入ロードマップ案

優先度は `Change Tracking` を最初にし, 次に `Performance Risks Inbox`, 並行して `Cloud Cost Intelligence` のタグ前提を確認する
APM OTel API support は新規計測を追加するタイミングで小さく検証するのがよい

## Phase 1: Change Tracking の最小導入

目的

- deploy / infrastructure / model change を New Relic 上で時系列に残す
- APM latency, error, logs と変更イベントを同じ調査導線に乗せる

最小イベント

| change_type | source | 対象 | 属性 |
| --- | --- | --- | --- |
| deploy | GitHub Actions | ECS service | `environment`, `service`, `commit_sha`, `pull_request`, `image_tag` |
| infrastructure | GitHub Actions | Terraform workflow | `environment`, `workspace`, `commit_sha`, `pull_request` |
| model | batch / deploy workflow | model version | `environment`, `model_name`, `model_version`, `s3_uri` |
| monitoring | Terraform workflow | alert condition | `environment`, `policy`, `condition`, `commit_sha` |

実装候補

- GitHub Actions の deploy workflow 後に NerdGraph `changeTrackingCreateEvent` を呼ぶ
- Terraform workflow 後に apply 結果を change event として送る
- model version を切り替える処理がある場合, 成功後に change event を送る

設計メモ

- `environment` は `dev`, `test`, `stg`, `prod` で揃える
- `service` は New Relic app name と照合可能な値にする
- `entity.guid` を解決できるなら付与する
- workflow 失敗は change event ではなく GitHub Actions 側の failure として扱う

確認したいファイル

- `.github/workflows/`
- `infra/envs/test/main_recommend_work_api.tf`
- `infra/envs/stg/main_recommend_work_api.tf`
- `infra/envs/prod/main_recommend_work_api.tf`
- `api/recommend_work_api/newrelic.ini`

## Phase 2: Performance Risks Inbox の試用

目的

- `recommend_work_api` の latency 悪化要因を alert 前に拾えるか確認する
- datastore / external call / payload 系のリスク検出が有効か確認する

試用対象

- `enten-recommend-work-api-test`
- `enten-recommend-work-api-stg`

見る観点

- slow query / slow external call がどの function trace に紐づくか
- `valkey_handler`, `aos_handler`, `dynamodb_handler` 周辺が識別できるか
- Group performance risks view で transactionName や API path で絞れるか
- 閾値を環境ごとに変える必要があるか

必要なら改善する計測

- `agent.function_trace()` の name を処理単位で明確にする
- datastore 呼び出し回数, payload size, result count を custom attribute として追加する
- access log と business log の `trace.id` 連携を維持する

判断基準

- QA や障害調査で, 既存 logs / APM より早く原因候補に到達できる
- false positive が少なく, triage の定例確認に耐える
- Public Preview の制約を踏まえても test / stg で価値がある

## Phase 3: Cloud Cost Intelligence の前提整備

目的

- CCI を有効にしたときに `service`, `environment`, `team` 単位で費用が見える状態にする
- New Relic ingest cost と AWS cost の両方を operational metric として扱う

先に確認すること

- `infra/modules/default_tags/` のタグが主要 resource に行き渡っているか
- ECS, OpenSearch, ElastiCache, DynamoDB, S3, Glue Job に `service` / `environment` 相当の tag があるか
- New Relic AWS integration が CCI の要求する AWS cost data に対応できているか
- Cost and Usage Report, Cost Explorer, Trusted Advisor 連携の権限が足りるか

見るべきコストテーマ

- OpenSearch node / storage sizing
- ElastiCache memory と node type
- ECS Fargate CPU / memory
- Glue Job の実行時間
- CloudWatch Metric Streams と Firehose
- New Relic Logs ingest

導入時の注意

- タグが整っていない状態で有効化すると, チーム向けの説明材料として弱い
- CCI は FinOps 観点の UI なので, engineering action に落ちる分類を先に作る
- Kubernetes Cost Allocation は現状不要

## Phase 4: APM OTel API support の小規模検証

目的

- 新規計測コードを New Relic API 直書きから少しずつ離す
- 既存 APM 体験を壊さず OpenTelemetry API を使えるか確認する

検証候補

- 新しい usecase / datastore adapter に OTel span を追加する
- OTel span attributes が New Relic APM / distributed tracing で期待どおり見えるか確認する
- `agent.function_trace()` と OTel span が二重計測にならないか確認する

対象としてよさそうな箇所

- `api/recommend_work_api/app/infrastructure/datastore/retriever/`
- `api/recommend_work_api/app/infrastructure/datastore/domain_service/`
- 新規追加予定の外部 service client

やらない方がよいこと

- 既存の `agent.function_trace()` を一括置換する
- preview / agent version 未確認のまま prod に入れる
- OTel collector 運用まで一気に広げる

## 優先順位

1. Change Tracking の event schema を決める
2. test / stg deploy workflow から change event を送る
3. Performance Risks Inbox を `recommend_work_api` で確認する
4. CCI 用のタグ充足度を棚卸しする
5. 新規計測追加時に OTel API を検証する

## 未確認事項

- Highspot 資料内の日本語版説明と公式 docs の差分
- 現在の New Relic 契約で CCI / Performance Risks Inbox / Mobile session replay が利用可能か
- New Relic Python agent の OTel API support 対応 version
- GitHub Actions の deploy workflow 名と New Relic API key の保存場所
- CCI に必要な AWS 権限と現行 IAM role の差分
