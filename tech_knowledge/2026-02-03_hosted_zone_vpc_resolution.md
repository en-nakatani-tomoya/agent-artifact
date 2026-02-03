# Hosted Zone And VPC DNS Resolution

## 概要
VPC の DNS は、関連付いた Private Hosted Zone を自動で解決し、別系統の DNS に対しては Resolver のフォワード設定が必要になる。

## 詳細
- Private Hosted Zone は VPC に関連付けると、VPC 内の AmazonProvidedDNS が自動で解決する。
- Cross-account の Private Hosted Zone も、ゾーン共有/関連付けを行えば同様に解決される。
- Public Hosted Zone はインターネット公開 DNS であり、VPC から外向き DNS 解決が可能であれば通常の経路で解決できる。
- ルート53ではない社内 DNS（AD など）を解決する場合、Route53 Resolver Outbound + フォワードルールで明示的に転送する必要がある。

## コード例
```hcl
resource "aws_route53_zone" "example_zone" {
  name = "example.internal"
  vpc {
    vpc_id = module.vpc.vpc_id
  }
}
```

## 参考
- https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/hosted-zones-private.html
- https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resolver-forwarding-outbound-queries.html

---
Generated: 2026-02-03
