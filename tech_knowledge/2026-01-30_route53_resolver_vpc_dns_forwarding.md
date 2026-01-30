# Route53 Resolver Outbound Endpointを使ったVPC間DNS転送

## 概要

Route53 Resolver Outbound Endpointを使うことで、VPCピアリング先のDNSサーバーにDNSクエリを転送できる。重要なのは、これはVPCレベルで動作するため、ECS TaskやEC2インスタンスなどの個別リソースに特別な設定は不要という点。

## 仕組み

### アーキテクチャ

```
VPC A (自環境)
  ├─ ECS Task
  ├─ Route53 Resolver Outbound Endpoint
  └─ VPC Peering → VPC B (外部環境)
                      └─ DNS Server
```

### DNS解決フロー

1. **ECS TaskがDNS名で接続**: アプリケーションコード内で `database.example.local` のようなDNS名を使用
2. **VPCのDNS Resolver**: VPC内のDNSリゾルバが自動的にクエリを処理
3. **Resolver Ruleのマッチング**: 特定ドメイン（`.example.local`）に対するルールが適用
4. **Outbound Endpointが転送**: VPCピアリング経由で外部DNSサーバーにクエリを転送
5. **応答の返却**: DNSサーバーからの応答がECS Taskに返される

## ECS Taskに特別な設定が不要な理由

### VPCレベルの透過的な動作

- **Route53 ResolverはVPCに紐づく**: Resolver Rule AssociationでVPCに関連付けられる
- **VPC内のすべてのリソースが恩恵を受ける**: ECS Task、EC2、Lambda（VPC内）など
- **DNSクエリは自動的にルーティング**: アプリケーションからは通常のDNS解決と同じ

### 必要な設定は環境変数のみ

```bash
# ECS Task Definition の環境変数
DATABASE_HOST=database.example.local  # DNS名を指定するだけ
DATABASE_PORT=1433
```

## Terraform実装のポイント

### 1. Security Group

```hcl
resource "aws_security_group" "route53_resolver" {
  vpc_id = module.vpc.vpc_id
  
  # DNS (UDP/TCP 53) を外部に転送可能にする
  egress {
    from_port   = 53
    to_port     = 53
    protocol    = "udp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  egress {
    from_port   = 53
    to_port     = 53
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

### 2. Outbound Endpoint

```hcl
resource "aws_route53_resolver_endpoint" "outbound" {
  direction = "OUTBOUND"
  security_group_ids = [aws_security_group.route53_resolver.id]
  
  # 高可用性のため複数のサブネットにIPアドレスを配置
  dynamic "ip_address" {
    for_each = module.vpc.private_subnet_ids
    content {
      subnet_id = ip_address.value
    }
  }
}
```

### 3. Resolver Rule

```hcl
resource "aws_route53_resolver_rule" "forward" {
  domain_name          = "example.local"  # 転送対象ドメイン
  rule_type            = "FORWARD"
  resolver_endpoint_id = aws_route53_resolver_endpoint.outbound.id
  
  target_ip {
    ip = "10.0.1.53"  # 外部DNSサーバーのIPアドレス
  }
}
```

### 4. VPCへの関連付け

```hcl
resource "aws_route53_resolver_rule_association" "vpc" {
  resolver_rule_id = aws_route53_resolver_rule.forward.id
  vpc_id           = module.vpc.vpc_id
}
```

## 必要な前提条件

1. **VPCピアリング**: 外部VPCとの接続が確立されていること
2. **ネットワーク到達性**: ピアリング経由でDNSサーバーにアクセス可能なこと
3. **Security Group**: ECS Task側でデータベースポート（例: 1433）への接続が許可されていること

## 注意点

- **Resolver Endpointのコスト**: ENIごとに時間課金が発生（複数AZ配置で複数ENI）
- **DNSサーバーIPの管理**: 外部DNSサーバーのIPアドレスを `locals.tf` などで管理
- **ドメイン名の正確性**: Resolver Ruleのドメイン名は正確に指定する必要がある

## まとめ

Route53 Resolver Outbound Endpointを使えば：
- ✅ VPCピアリング先のプライベートDNSを解決できる
- ✅ ECS Taskなどの個別リソースに特別な設定は不要
- ✅ 環境変数でDNS名を指定するだけで動作
- ✅ VPCレベルの透過的な動作で運用負荷が低い

---
Generated: 2026-01-30
