# aws private subnet nat gateway alb security group

## 概要
プライベートサブネットのECSタスクがinternet-facing ALBにアクセスする際、NAT GatewayによるIPアドレス変換が発生するため、ALBのセキュリティグループにNAT GatewayのCIDRを追加する必要がある。SG参照が使えない理由と代替手段を整理する。

## 詳細
## 前提: プライベートサブネットとNAT Gateway

プライベートサブネットのリソース（ECSタスク等）はインターネットに直接アクセスできない。
外部サービス（S3、外部API等）との通信にはNAT Gatewayを経由する。

```
ECSタスク(private IP: 10.x.x.x) → NAT Gateway(public IP: 54.x.x.x) → インターネット
```

NAT Gatewayは送信元IPをプライベートIPから自身のパブリックIPに変換（SNAT）する。

## 問題: VPC内サービス間のALB経由通信

internet-facing ALBはパブリックIPを持つ。VPC内のECSタスクがALBのパブリックDNS名でアクセスすると:

1. DNSはALBのパブリックIPに解決される
2. プライベートサブネットのルートテーブルは `0.0.0.0/0 → NAT Gateway` のため、NAT Gatewayを経由
3. NAT Gatewayが送信元IPを変換（ECSのprivate IP → NAT GatewayのパブリックIP）
4. ALBに到達した時点で、送信元IPはNAT GatewayのパブリックIP

そのため、ALBのセキュリティグループにNAT GatewayのパブリックIPを許可する必要がある。

## なぜセキュリティグループ参照（SG-to-SG）が使えないのか

AWSのセキュリティグループでは、Ingressルールに別のSGを指定してアクセスを許可できる（SG参照）。
しかし、internet-facing ALBの場合は使えない。

理由: NAT Gatewayを経由する時点で送信元IPが変換されており、元のECSタスクのSGとの紐付けが失われるため。
ALBのSGから見ると、リクエストの送信元はECSタスクのSGではなくNAT GatewayのパブリックIPとなる。

## 代替手段

| 方法 | SG参照可能 | 概要 |
|------|-----------|------|
| NAT Gateway CIDR追加 | No | パブリックALBにNAT GatewayのIPを許可。最もシンプル |
| Internal ALB | Yes | 内部用ALBを別途作成。プライベートIPで通信するためSG参照が有効 |
| ECS Service Connect / Cloud Map | - | ALBを経由せずサービス間で直接通信 |
| VPC PrivateLink | Yes | NLB + VPC Endpointでプライベート接続 |

### 各方式の使い分け

- **NAT Gateway CIDR追加**: 1つのパブリックALBで外部・内部の両方をカバーしたい場合。構成がシンプルだが、NAT GatewayのIPが変わると更新が必要
- **Internal ALB**: 外部アクセスと内部アクセスを明確に分離したい場合。ALBが2つになるためコスト増
- **Service Connect / Cloud Map**: ALB不要でサービス間通信を行いたい場合。ECS環境に閉じた通信向き
- **PrivateLink**: 異なるVPC間やAWSアカウント間でのプライベート接続が必要な場合

## 実例: enten-ai-recommend-platform (stg環境)

- **ranking_work_api**: NAT Gateway CIDRを追加。recommend_work_apiなど他のECSタスクからALB経由でアクセスされるため
- **recommend_work_api**: NAT Gateway CIDRなし。外部（VPN、en転職サーバー）からのみアクセスされるため

## 参考
- https://docs.aws.amazon.com/vpc/latest/userguide/vpc-nat-gateway.html
- https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-subnets.html

---
Generated: 2026-03-10
