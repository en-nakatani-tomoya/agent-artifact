# New Relic 2026年4月アップデート相性整理

New Relic 2026年4月分アップデートのうち, `enten-ai-recommend-platform` と相性が良いものを整理するワークスペース
現状の New Relic 利用実態, 相性評価, 試す順番を複数ドキュメントに分けて残す

## 目的

- New Relic 2026年4月アップデートの中で, このリポジトリに効きそうな機能を選別する
- 既存の New Relic APM, ログ, AWS integration, Terraform monitoring との接続点を明確にする
- すぐ試せるもの, 設計が必要なもの, 今は優先度が低いものを分ける

## 背景

共有されたアップデート抜粋では以下が主な対象

- Cloud Cost Intelligence
- Change Tracking
- Mobile
- Performance Risks Inbox
- APM OTel API support
- Kubernetes Linux/Windows node monitoring

Highspot の資料 URL は直接取得できなかったため, ユーザー共有の抜粋と New Relic 公式 What's New / docs を照合して整理した

## ファイル構成

- `README.md`: 本ファイル
- `01-current-observability-map.md`: このリポジトリの New Relic 利用状況
- `02-fit-assessment.md`: アップデート別の相性評価
- `03-adoption-roadmap.md`: 試す順番と実装候補
- `04-source-notes.md`: 確認した外部情報と根拠メモ

## 進め方

1. 既存の New Relic 利用箇所をリポジトリから確認する
2. 公式情報から各アップデートの機能境界を確認する
3. このリポジトリでの効果, 導入コスト, リスクを評価する
4. 次に試すなら何から始めるかをロードマップ化する

## 関連

- `infra/modules/newrelic_aws_integration/README.md`
- `monitoring/README.md`
- `api/recommend_work_api/newrelic.ini`
- `shared/firelens/envs/test/fluent-bit.conf`
- New Relic 公式 What's New: https://docs.newrelic.com/whats-new/
