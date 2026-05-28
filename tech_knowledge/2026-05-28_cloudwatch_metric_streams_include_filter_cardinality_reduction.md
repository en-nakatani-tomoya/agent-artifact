# CloudWatch Metric Streams Metric Name Filtering Cost Reduction

## 概要
CloudWatch Metric Streams の include_filter / exclude_filter を Terraform で管理し、namespace 単位ではなく metric name 単位で送信対象を絞ることで、CloudWatch metric update と外部監視基盤への取り込み量を入口で削減する方法を整理する。

## 詳細
### CloudWatch Metric Streams のコストは「入口」で決まる

CloudWatch Metric Streams は、CloudWatch メトリクスを外部監視基盤やデータレイクへ継続的に送る仕組みである。設定を広く取りすぎると、実際には見ていないメトリクスまでストリーミング対象になり、CloudWatch の metric update と外部監視基盤側の取り込み量が増える。

ここで重要なのは、単に namespace 単位で対象を選ぶだけでは粗いという点である。AWS は Metric Streams のフィルタリングを metric name 単位にも拡張しており、特定 namespace の中から送る metric name だけを include したり、逆に不要な metric name だけを exclude したりできる。

つまり、コスト削減の主戦場は「どのAWSサービスのメトリクスを送るか」だけではなく、「その namespace のどの metric name を送るか」にある。

### Terraform で metric name 単位に include する

Terraform では `aws_cloudwatch_metric_stream` の `include_filter` に `metric_names` を指定する。これにより、namespace 全体ではなく、必要な metric name だけをストリーミングできる。

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
}
```

この例では、`AWS/ECS` namespace を丸ごと送っていない。ECS のメトリクスのうち、運用上見る必要がある `CPUUtilization` と `MemoryUtilization` だけを送っている。同様に、Application Load Balancer も 5xx と応答時間に限定している。

この差は大きい。namespace 単位の include は「そのAWSサービスに属するメトリクスを広く送る」設定になりやすい。一方、metric name 単位の include は「実際にアラートやダッシュボードで使うメトリクスだけを送る」設定になる。

### exclude_filter は「大半は送るが一部だけ落とす」ときに使う

送信対象のほとんどを使う namespace では、`include_filter` で列挙するより `exclude_filter` の方が読みやすい場合がある。

```hcl
resource "aws_cloudwatch_metric_stream" "observability" {
  name          = "observability-metric-stream"
  role_arn      = aws_iam_role.metric_stream_to_firehose.arn
  firehose_arn  = aws_kinesis_firehose_delivery_stream.metrics.arn
  output_format = "opentelemetry1.0"

  exclude_filter {
    namespace = "AWS/Usage"
    metric_names = [
      "CallCount",
    ]
  }
}
```

`include_filter` は「必要なものだけを送る」設計に向いている。`exclude_filter` は「基本的には送るが、ノイズや高コストな一部だけ落とす」設計に向いている。

注意点として、CloudWatch Metric Streams のAPIでは include filters と exclude filters を同じ操作に混在させない前提になっている。設計としても、基本方針はどちらかに寄せる方がよい。コスト削減を目的に始めるなら、通常は allowlist としての `include_filter` から始める方が安全である。

### locals で「送る metric name」をレビュー可能にする

実運用では、`include_filter` を直接並べるより、namespace と metric name の対応を `locals` に寄せると意図が見えやすい。

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

この形にすると、Pull Request では「どの namespace の、どの metric name を送るのか」だけを確認できる。レビューの論点が、監視基盤の細かい配線ではなく、送信対象メトリクスの妥当性に集中する。

### metric update は metric name の段階で大きく減らせる

Metric Streams の出力量は、概念的には次の掛け算で増える。

```text
metric update 量 = namespace数 * metric name数 * dimension組み合わせ数 * 統計値数
```

CloudWatch メトリクスは dimension を持つ。たとえば ALB であれば LoadBalancer や TargetGroup、ECS であれば ClusterName や ServiceName、RDS であれば DBInstanceIdentifier のような軸がある。namespace を丸ごと送ると、その namespace に含まれる多数の metric name と dimension の組み合わせが stream 対象になる。

`include_filter.metric_names` は、この掛け算のうち `metric name数` を入口で減らす。dimension が多い metric name を送らなければ、その metric name に付随する dimension 組み合わせも後段へ流れない。

たとえば、ある namespace に 100 個の metric name があり、実際にアラートやダッシュボードで使うものが 10 個だけなら、metric name の段階で 90% を除外できる。dimension が多い metric name を除外できる場合、metric update の削減効果はさらに大きくなる。

ここでのポイントは、外部監視基盤に入った後でクエリやダッシュボードを整理するのでは遅いということだ。stream の入口で送信対象を絞れば、CloudWatch 側の metric update、Firehose 経由の転送、外部監視基盤側の取り込み、保存、インデックス、アラート評価のすべてが軽くなる。

### namespace 単位の制限だけでは不十分な理由

namespace 単位のフィルタは、最初の整理としては有効である。使っていない AWS サービスの namespace を送らないだけでも、不要な metric update は減る。

しかし、実際のコストを押し下げるには、namespace 内部の metric name まで見た方がよい。多くの運用では、ある AWS サービスのすべてのメトリクスを見るわけではない。アラートに使う数個のメトリクス、ダッシュボードに使う数個のメトリクス、調査時だけたまに見るメトリクスが混在している。

Metric Streams は常時送信の仕組みなので、「調査時だけ見るかもしれない」メトリクスまで常時 stream するかは慎重に決めるべきである。必要になったときに CloudWatch 側で直接確認できるメトリクスまで、常時外部へ転送する必要はない場合が多い。

### 設計の勘所

Metric Streams のフィルタ設計では、次の順で考えるとよい。

- まずアラートに使う metric name を列挙する。
- 次に主要ダッシュボードで常時見る metric name を追加する。
- 調査時だけ見る metric name は、常時 stream する必要があるかを確認する。
- dimension 組み合わせが多い metric name は、送信対象に入れる前に用途を確認する。
- 基本方針は `include_filter` による allowlist とし、例外的に落としたいものが明確な場合だけ `exclude_filter` を検討する。
- namespace と metric name の対応は Terraform の locals などで管理し、レビュー可能にする。

この設計にすると、監視基盤の入口で「送るべきメトリクス」と「送らなくてよいメトリクス」を明確に分けられる。

### まとめ

CloudWatch Metric Streams のコスト削減で重要なのは、namespace 単位の制御に留めず、metric name 単位で stream 対象を制限することである。Terraform の `aws_cloudwatch_metric_stream` では、`include_filter` や `exclude_filter` の `metric_names` を使って、この制御をコード化できる。

特に `include_filter.metric_names` を allowlist として使うと、外部監視基盤へ送る CloudWatch metric update を入口で削減できる。ある namespace のうち実際に使う metric name が 10% なら、metric name の段階で 90% を除外できる。高 dimension の metric name を除外できる場合、後段に流れる系列数と取り込み量への効果はさらに大きい。

監視コストを下げるためには、後段のダッシュボードやアラートを整理するだけではなく、Metric Streams の入口で何を送るかを明示することが重要である。

## 参考
- https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch-Metric-Streams.html
- https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/cloudwatch_metric_stream
- https://aws.amazon.com/about-aws/whats-new/2023/05/amazon-cloudwatch-metric-streams-filtering-name/

---
Generated: 2026-05-28
