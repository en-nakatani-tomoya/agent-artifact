# aws agent plugins for aws

## 概要
AWSがClaude CodeやCursorなどのAIエージェントにAWSのベストプラクティス・デプロイ・コードレビューなどのスキルを組み込むプラグイン「Agent Plugins for AWS」を公開した

## 詳細
Agent Plugins for AWSは、Agent Skills・MCPサーバ・Hooks・関連ドキュメントから構成されるプラグイン。Claude CodeやCursorに導入すると、「Deploy this app to AWS」のような指示だけで、(1)アプリの依存関係・フレームワーク解析、(2)App Runner/RDS/CloudFront/Secrets Managerなど適切なAWSサービスの推奨、(3)最新価格表に基づくコスト見積もり、(4)AWS CDK・Dockerfileなどインフラコード生成、(5)プロビジョニング・ビルド・DBマイグレーションまでのデプロイ実行、を一気通貫で自動実行できる。個別機能の呼び出しも可能。

## 参考
- https://www.publickey1.jp/blog/26/awsclaude_codeagent_plugins_for_aws.html

---
Generated: 2026-03-24
