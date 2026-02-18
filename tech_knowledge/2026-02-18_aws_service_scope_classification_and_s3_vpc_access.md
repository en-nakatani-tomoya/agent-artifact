# aws service scope classification and s3 vpc access

## 概要
AWSサービスはグローバル・リージョナル・AZの3つのスコープに分類される。S3はリージョナルサービス（VPC外）であるため、VPC PeeringではなくVPC Endpointでアクセスする。

## 詳細
## サービス分類

### 1. グローバルサービス（Global Service）
リージョンに依存せず全世界で共通のサービス。
- 例: IAM, Route 53, CloudFront, AWS Organizations

### 2. リージョナルサービス（Regional Service）
特定のAWSリージョン内で動作するサービス。VPCの外側で動作する。
- 例: S3, DynamoDB, SQS, Lambda
- S3バケットは特定のリージョンに作成されるが、他リージョンからもアクセス可能

### 3. AZサービス（Availability Zone Service）
特定のAZ内で動作するサービス。
- 例: EC2インスタンス, EBSボリューム, サブネット

## S3アクセスとVPC Peering

S3はVPCに属さないリージョナルサービスのため、VPC Peeringは不要。

| 方式 | コスト | インターネット経由 | VPC Peering必要 |
|------|--------|-------------------|----------------|
| Gateway Endpoint | 無料 | No | No |
| Interface Endpoint (PrivateLink) | 有料 | No | No |
| NAT Gateway | 有料 | Yes | No |

ベストプラクティスはVPC Gateway Endpoint。無料かつインターネットを経由しない。

## 構造イメージ

AWS Global
├── IAM, Route 53, CloudFront ← グローバル
└── Region (ap-northeast-1)
    ├── S3, DynamoDB, SQS ← リージョナル（VPC外）
    └── VPC
        ├── ALB, NAT Gateway ← VPC内
        ├── AZ-a → EC2, EBS
        └── AZ-c → EC2, EBS

---
Generated: 2026-02-18
