# CloudWatch Metric Streams Include Filter Cardinality Reduction

## 概要
CloudWatch Metric Streams の include_filter を Terraform で管理し、namespace と metric name を明示的に絞ることで、外部監視基盤へ流すメトリクス量とカーディナリティを制御する方法を整理する。

## 詳細
# CloudWatch Metric Streams の include_filter でメトリクス量を制御する

## 背景

監視基盤に送るメトリクスは、多ければ多いほどよいとは限らない。クラウド環境では、サービスごとに大量のメトリクスが自動生成される。全namespace・全metricを外部監視基盤へストリーミングすると、実際には使っていないメトリクスまで取り込み対象になり、可視化やアラートに使う前からデータ量が増える。

この問題に対して有効なのが、CloudWatch Metric Streams の include filter である。AWSの発表では、Metric Streams はもともとnamespace単位でinclude/excludeでき、現在はmetric name単位のフィルタリングにも対応している。つまり、「AWS/ECS は送るが CPUUtilization と MemoryUtilization だけにする」「AWS/RDS は必要な数個のメトリクスだけ送る」といった制御ができる。

## Terraformでの基本形

Terraformでは `aws_cloudwatch_metric_stream` に `include_filter` を指定する。以下は一般化した例である。

```hcl
resource "aws_cloudwatch_metric_stream" "observability" {
  name          = "observability-metric-stream"
  role_arn      = aws_iam_role.metric_stream_to_firehose.arn
  firehose_arn  = aws_kinesis_firehose_delivery_stream.metrics.arn
  output_format = "opentelemetry1.0"

  include_filter {
    namespace = "AWS/ECS"
    metric_names = [
      "CPUUtilization",
      "MemoryUtilization",
    ]
  }

  include_filter {
    namespace = "AWS/ApplicationELB"
    metric_names = [
      "HTTPCode_Target_5XX_Count",
      "TargetResponseTime",
    ]
  }

  include_filter {
    namespace = "AWS/RDS"
    metric_names = [
      "CPUUtilization",
      "DatabaseConnections",
      "FreeStorageSpace",
    ]
  }
}
```

この書き方のポイントは、送信対象を「監視基盤に存在するすべてのCloudWatchメトリクス」ではなく、「運用上使うnamespaceとmetric nameのリスト」に限定することにある。

## include filter を locals で管理する

実運用では、`include_filter` を直接並べるよりも、対象namespaceとmetric nameを `locals` で管理した方がレビューしやすい。

```hcl
locals {
  metric_stream_include_filters = {
    "AWS/ECS" = [
      "CPUUtilization",
      "MemoryUtilization",
    ]

    "AWS/ApplicationELB" = [
      "HTTPCode_Target_5XX_Count",
      "TargetResponseTime",
    ]

    "AWS/RDS" = [
      "CPUUtilization",
      "DatabaseConnections",
      "FreeStorageSpace",
    ]
  }
}

resource "aws_cloudwatch_metric_stream" "observability" {
  name          = "observability-metric-stream"
  role_arn      = aws_iam_role.metric_stream_to_firehose.arn
  firehose_arn  = aws_kinesis_firehose_delivery_stream.metrics.arn
  output_format = "opentelemetry1.0"

  dynamic "include_filter" {
    for_each = local.metric_stream_include_filters

    content {
      namespace    = include_filter.key
      metric_names = include_filter.value
    }
  }
}
```

この形にすると、Pull Requestでは「どのnamespaceを追加したか」「そのnamespaceでどのmetric nameを送るか」だけを確認しやすい。監視基盤の設定変更を、コードレビューで扱える単位に分解できる。

## なぜメトリクス量が減るのか

Metric Streams の出力量は、概念的には次の要素で増える。

```text
出力メトリクス量 = namespace数 * metric name数 * dimension組み合わせ数 * 統計値数
```

CloudWatchメトリクスはdimensionを持つ。たとえばALBであればLoadBalancerやTargetGroup、ECSであればClusterNameやServiceName、RDSであればDBInstanceIdentifierのような軸がある。namespaceを丸ごと送ると、そのnamespaceに含まれる多くのmetric nameとdimensionの組み合わせが対象になる。

include filter は、この掛け算のうち `namespace数` と `metric name数` を先に絞る。dimensionの数そのものを直接消すわけではないが、不要なmetric nameを送らないだけで、外部監視基盤へ流れる系列数を大きく減らせる。

たとえば、あるnamespaceに100個のmetric nameがあり、そのうち運用で実際に使うものが10個だけなら、metric nameの段階で90%を除外できる。さらに、アラートやダッシュボードに必要なnamespaceだけに限定できる場合、全体の削減幅はさらに大きくなる。

## 一般的な削減幅の考え方

削減率は環境によって変わるため、固定値では語れない。ただし、公開情報とメトリクスの構造から、次のように見積もれる。

- 未使用namespaceを送らない: 使っていないAWSサービス分のメトリクスを丸ごと削減できる。
- namespace内でmetric nameを絞る: 利用metric name数 / 全metric name数 に近い比率まで削減できる。
- 高dimensionのmetric nameを除外する: dimension組み合わせが多いmetricほど削減効果が大きい。
- histogramやdistribution系の外部監視メトリクスを減らす: 監視基盤側で複数系列に展開される場合、倍率分の効果が出る。

Datadogは、未使用メトリクスにMetrics without Limitsを適用した顧客で、可視性を大きく損なわず最大70%程度のcustom metrics usage削減が見られるとしている。これはCloudWatch Metric Streamsそのものの削減率ではないが、「使っていないメトリクスやタグを取り込み・インデックス対象から外すと数十%規模で減る」ことの参考になる。

一方で、APMのトランザクション粒度や高dimensionのカスタムメトリクスを、サービス単位・リソース単位の代表メトリクスに置き換える場合は、90%以上削減されるケースもあり得る。これは、100個のエンドポイントを数個のサービスメトリクスに集約するような場合に、系列数が桁で変わるためである。

## 設計の勘所

include filter を使うときは、単に少なくするのではなく、監視の目的から逆算する。

- アラートに使うmetric nameを先に決める。
- ダッシュボードで見るmetric nameを次に決める。
- 調査時だけ必要な詳細メトリクスは、常時streamする必要があるかを確認する。
- 高dimensionのmetric nameは、送る前に本当に使うか確認する。
- include filter の変更はTerraformでレビューし、手作業で対象を増やさない。
- 出力量そのものをCloudWatchや監視SaaS側で定期的に確認する。

特に、include filter は「今見えているものを削る」操作ではなく、「監視基盤へ送る入口を設計する」操作として扱うとよい。入口で絞ると、後段の保存、インデックス、クエリ、アラート評価のすべてが軽くなる。

## まとめ

CloudWatch Metric Streams の `include_filter` は、監視コストとメトリクスカーディナリティを制御するための入口設計である。Terraformでnamespaceとmetric nameを明示すれば、監視対象をコードレビュー可能にしながら、不要なメトリクスのストリーミングを避けられる。

一般的には、未使用namespace・未使用metric nameの除外だけでも数十%規模の削減が期待できる。高dimensionのメトリクスやAPM由来の細かい系列を、サービス単位・リソース単位の代表メトリクスへ寄せる場合は、90%以上の削減も起こり得る。

ただし、削減率は環境依存である。安全に進めるには、まず現在のMetric Streams出力量、namespace別の利用状況、metric name別の参照有無、dimensionの多さを確認し、そのうえで `include_filter` を段階的に導入するのがよい。

## 参考
- https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch-Metric-Streams.html
- https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/cloudwatch_metric_stream
- https://aws.amazon.com/about-aws/whats-new/2023/08/amazon-cloudwatch-metric-streams-filtering-metric-name/
- https://www.datadoghq.com/blog/custom-metrics-governance/
- https://docs.datadoghq.com/account_management/billing/custom_metrics/

---
Generated: 2026-05-28
