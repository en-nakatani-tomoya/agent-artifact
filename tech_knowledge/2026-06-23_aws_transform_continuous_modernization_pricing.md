# aws transform continuous modernization pricing

## 概要
AWS Transform continuous modernization は、技術的負債の継続検出と修正PR生成を狙うプレビュー機能。料金は AWS Transform のカスタム変換エージェント分課金が参考になり、公式料金は 0.035 USD / agent minute とされている。

## 詳細
Publickey の 2026-06-22 記事と AWS 公式情報から、AWS Transform continuous modernization は、従来の一度きりの大規模移行支援ではなく、複数リポジトリを継続的にスキャンして、古いランタイム、非推奨 API、古い依存関係、組織標準から外れたコード、ドキュメント不足、脆弱性などを検出し、優先順位付けや修正 Pull Request 生成まで支援する機能として理解できる。CI/CD に Continuous Modernization を加える考え方に近い。料金については、AWS Transform の公式料金ページではカスタム変換エージェントが 0.035 USD / agent minute とされ、例として Node.js SDK アップグレード約 20 分で 0.70 USD、Java バージョンアップ約 72 分で 2.52 USD、Python ランタイムアップグレード約 37 分で 1.30 USD が示されている。ただし continuous modernization 個別のリポジトリ単価や固定料金は明示されておらず、プレビュー中のため、分析・検出・PR生成のどこが課金対象になるかは利用画面や契約条件で確認する必要がある。導入時は、自動生成PRをCIとレビュー前提にすること、組織ルールを設計してノイズを抑えること、接続リポジトリや権限、機密情報、監査ログの扱いを確認することが重要。

## 参考
- https://www.publickey1.jp/blog/26/awsaiaws_transform_continuous_modernization.html
- https://aws.amazon.com/jp/transform/pricing/
- https://aws.amazon.com/jp/transform/continuous-modernization/
- https://aws.amazon.com/jp/blogs/aws/proactively-reduce-tech-debt-autonomously-with-aws-transform-continuous-modernization-preview/

---
Generated: 2026-06-23
