# Route53 Resolver DNS Basics

## 概要
Route53 Resolver のアウトバウンド構成では、DNSフォワード用の各リソース役割と UDP/TCP 53 の許可が重要となる。

## 詳細
- **Security Group** は Resolver Endpoint が外部DNSへ問い合わせるための送信(egress)を許可する。DNSは通常 UDP/53 を使い、応答が大きい場合などに TCP/53 を使うため両方の許可が推奨。
- **Outbound Resolver Endpoint** は VPC 内のリソースからのDNSクエリを外部DNSへ転送するための出口。複数サブネット(複数AZ)にIPを持たせることで冗長化し、AZ障害時も名前解決を継続できる。
- **Resolver Rule** は特定ドメイン(例: `.ensya13-lan.local`)のクエリを指定DNSサーバーへフォワードする。
- **Resolver Rule Association** はルールをVPCに関連付け、VPC内のリソースに適用する。

## コード例
```hcl
resource "aws_route53_resolver_endpoint" "outbound" {
  name      = "example-resolver-outbound"
  direction = "OUTBOUND"

  # 複数サブネットで冗長化
  dynamic "ip_address" {
    for_each = var.private_subnet_ids
    content {
      subnet_id = ip_address.value
    }
  }
}
```

## 参考
- https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resolver.html

---
Generated: 2026-02-02
# Route53 Resolver DNS Basics

## 概要
Route53 Resolver のアウトバウンド構成では、DNSフォワード用の各リソース役割と UDP/TCP 53 の許可が重要となる。

## 詳細
- **Security Group** は Resolver Endpoint が外部DNSへ問い合わせるための送信(egress)を許可する。DNSは通常 UDP/53 を使い、応答が大きい場合などに TCP/53 を使うため両方の許可が推奨。
- **Outbound Resolver Endpoint** は VPC 内のリソースからのDNSクエリを外部DNSへ転送するための出口。複数サブネット(複数AZ)にIPを持たせることで冗長化し、AZ障害時も名前解決を継続できる。
- **Resolver Rule** は特定ドメイン(例: `.ensya13-lan.local`)のクエリを指定DNSサーバーへフォワードする。
- **Resolver Rule Association** はルールをVPCに関連付け、VPC内のリソースに適用する。

## コード例
```hcl
resource "aws_route53_resolver_endpoint" "outbound" {
  name      = "example-resolver-outbound"
  direction = "OUTBOUND"

  # 複数サブネットで冗長化
  dynamic "ip_address" {
    for_each = var.private_subnet_ids
    content {
      subnet_id = ip_address.value
    }
  }
}
```

## 参考
- https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resolver.html

---
Generated: 2026-02-02
