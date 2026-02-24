# aws acm certificate and dns validation

## 概要
AWS ACM証明書の役割と、DNS検証（CNAME）による所有権確認の仕組みを整理。TerraformでのACM証明書発行フローとトラブルシューティングも含む。

## 詳細
## ACM (AWS Certificate Manager) とは
SSL/TLS証明書を無料で発行・管理するAWSサービス。ALBやCloudFrontにアタッチしてHTTPS通信を実現する。通常は認証局(CA)から有料で購入するが、ACMならAWSサービスで使う限り無料。

## 証明書の役割
「このサーバーは本当にexample.comです」を第三者（AWS）が保証するもの。これによりブラウザが安全な通信と判断し、HTTPS接続が確立される。

## DNS検証の仕組み
1. ACM証明書をリクエスト
2. AWSが検証用のCNAMEレコード情報を発行（例: _abc123.example.com → _def456.acm-validations.aws）
3. そのCNAMEをDNS（Route 53等）に登録
4. AWSがDNSを確認し、レコードが存在すれば「ドメイン所有者」と判断して証明書を検証済みにする

CNAMEを登録できる = DNS管理権限を持つ = ドメイン所有者、という論理。

## 典型的な構成
ユーザー → HTTPS → ALB（ACM証明書アタッチ） → HTTP → アプリケーション

## Terraformでの構成
- aws_acm_certificate: 証明書リクエスト
- aws_route53_record: 検証用CNAMEレコード作成（domain_validation_optionsを参照）
- aws_acm_certificate_validation: 検証完了を待機するリソース

## トラブルシューティング
- ExpiredTokenException: 検証待機中にAWSセッショントークンが期限切れ。terraform applyを再実行するか、セッション期間を延長する。
- GitHub ActionsのOIDCセッションはデフォルト1時間で、証明書検証の待機と合わせると超過しやすい。

## コード例
```
resource "aws_acm_certificate" "cert" {\n  domain_name       = "example.com"\n  validation_method = "DNS"\n}\n\nresource "aws_route53_record" "cert_validation" {\n  for_each = aws_acm_certificate.cert.domain_validation_options\n  zone_id  = aws_route53_zone.main.zone_id\n  name     = each.value.resource_record_name\n  type     = each.value.resource_record_type\n  records  = [each.value.resource_record_value]\n  ttl      = 60\n}\n\nresource "aws_acm_certificate_validation" "cert" {\n  certificate_arn         = aws_acm_certificate.cert.arn\n  validation_record_fqdns = [for r in aws_route53_record.cert_validation : r.fqdn]\n}
```

## 参考
- https://docs.aws.amazon.com/acm/latest/userguide/dns-validation.html
- https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/acm_certificate_validation

---
Generated: 2026-02-24
