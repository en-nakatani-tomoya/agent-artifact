# 04. コンピューティング環境モジュールの実装詳細 — 非自明な 7 点

CE 側には「知らないと読み飛ばすが、知ると重要」な仕掛けが集中する。

## 1. `name_prefix` + `create_before_destroy` + `depends_on` の三点セット

```hcl
resource "aws_batch_compute_environment" "this" {
  name_prefix = "${local.name_prefix}-batch-compute-environment-"
  ...
  lifecycle { create_before_destroy = true }
  depends_on = [ ...インスタンスロール / Spot Fleet ロールの attachment 全部... ]
}
```

**`name` でなく `name_prefix` を使う理由**: CE の名前は変更不可（ForceNew）。CE の設定を変えて置換が必要になると「同名の新 CE を作る」ことになり、名前が衝突して失敗する。`name_prefix` なら provider がランダムサフィックスを付けるので新旧が共存できる。

**`create_before_destroy` の理由**: 上とセット。さらに「**ジョブキューから参照されている CE は削除できない**」という Batch の制約があるため、`新 CE 作成 → キューの参照を新 CE に付け替え → 旧 CE 削除` の順に流す必要がある。破棄が先だと詰む。

**IAM への `depends_on` の理由**: ポリシーが先に剥がれると **CE の削除処理が進まず `DELETING` で stuck する**。CE の削除は Batch が EC2 を落とす処理を伴い、それにインスタンスロールの権限が要る。Terraform の依存グラフは削除時に逆順になるので、`depends_on` を書くと「IAM は CE より後に消える」が保証される。

この 3 つは **Batch を Terraform で扱った人が必ず踏む定番の罠**への対処。書かれていなければ指摘、書かれていれば追認でよい。

なお、**この置換シナリオは apply しないと検証できない**。初回構築後に一度 CE の設定を変えて apply し、置換が通ることを確かめておく価値がある。

## 2. AMI を SSM パブリックパラメータから引く

```hcl
data "aws_ssm_parameter" "gpu_ami" {
  name = "/aws/service/ecs/optimized-ami/amazon-linux-2023/gpu/recommended"
}
# → JSON なのでデコードして image_id を取る
gpu_ami_id = jsondecode(data.aws_ssm_parameter.gpu_ami.insecure_value)["image_id"]
```

AWS 公開のパラメータで、**最新の ECS GPU 最適化 AMI（NVIDIA ドライバ + ECS エージェント + nvidia-container-toolkit 入り）の情報が JSON で入っている**。

**`value` でなく `insecure_value` を使う理由**: `aws_ssm_parameter` の `value` は provider が sensitive でマークするため、`jsondecode` の結果を他所で使うと出力が伏せられ plan が読みにくくなる。`insecure_value` は SecureString 以外に限り生の値を返す。AMI ID は機密ではないので問題ない。

**トレードオフ（重要）**: `recommended` を参照すると、**AWS が AMI を更新した日に plan に差分が出て CE が置換される**。

| | `recommended` を追う | AMI ID を固定して手動更新 |
| --- | --- | --- |
| セキュリティパッチ | 自動で追随 | 放置されがち |
| plan の安定性 | **AWS の都合で勝手に差分が出る** | 安定 |
| 実行中ジョブ | 更新タイミングを制御できない | 制御できる |

CI が定期的に apply する構成なら、**バッチ実行中に CE が置換される**可能性を検討しておく。緩和策が次の `update_policy`。

## 3. `update_policy` の二段構え

```hcl
update_policy = optional(object({
  job_execution_timeout_minutes = optional(number, 30)
  terminate_jobs_on_update      = optional(bool, false)
}), null)   # ← オブジェクト自体の既定は null
```

`update_policy` は「CE のインフラ更新時に、実行中ジョブを何分待つか / 強制終了するか」を決める。**オブジェクト内には既定値があるのに、オブジェクト自体の既定が `null`** なので、何も渡さなければブロックごと出力されず Batch 既定が使われる。意図的だが読み違えやすい書き方。

タイムアウトの長いジョブ（数時間）がある場合、「30 分待って強制終了しない」だと更新が長時間ブロックされうる。

## 4. SPOT 関連の条件分岐

```hcl
is_spot = var.capacity_type == "SPOT"
allocation_strategy = coalesce(var.allocation_strategy,
  local.is_spot ? "SPOT_PRICE_CAPACITY_OPTIMIZED" : "BEST_FIT_PROGRESSIVE")
spot_bid_percentage = local.is_spot ? var.spot_bid_percentage : null
```

- **`bid_percentage` は SPOT のときだけ有効な引数**。オンデマンドで値が入っていると API エラーになるので `null` に潰す。
- 配置戦略の既定はこれで妥当。`SPOT_PRICE_CAPACITY_OPTIMIZED` は価格と中断リスクの両方を見る現行推奨。`BEST_FIT_PROGRESSIVE` はオンデマンドで複数インスタンスタイプを段階的に使う。
- Spot Fleet ロールは `count = local.is_spot ? 1 : 0`。`SPOT_PRICE_CAPACITY_OPTIMIZED` では必須ではないが、`BEST_FIT` に切り替えても動くよう常に作っておく判断もある（未使用リソース vs 切替時の事故防止）。

**GPU × SPOT は実運用上いちばん効く選択**。オンデマンド比で 6〜7 割安くなる一方、**GPU インスタンスの Spot 中断は現実に起きる**。数時間かかるジョブが中断されると最初からやり直しになるので、**冪等性とチェックポイントが前提**になる。リトライ戦略とセットで設計する（→ 05 §3）。

## 5. 起動テンプレート

```hcl
root_volume_size_gibibytes = 300
encrypted = true
metadata_options { http_tokens = "required"  http_put_response_hop_limit = 2 }
update_default_version = true
# CE 側は version = aws_launch_template.this.latest_version を参照
```

- **EBS 暗号化 ON、IMDSv2 必須** はベースライン。
- **`hop_limit = 2` は必須**。コンテナ（bridge ネットワーク）から IMDS を引くとホップが 1 つ増えるため、1 だとインスタンスロールの認証情報が取れない。よく踏む罠。
- **`update_default_version = true` と `version = latest_version` は組で使う**。CE が最新版を見るので、テンプレート変更が CE の差分になる。
- **ルートボリュームサイズ**: GPU コンテナイメージ（数 GB〜十数 GB）+ モデル重みが同じボリュームに載るので、既定の 30 GiB では足りない。200〜300 GiB を見込む。`delete_on_termination = true` かつ `min_vcpus = 0` なら、実際のコストは稼働時間分だけ。

## 6. タグの例外措置

**事実**: Batch は自身の API 経由で EC2 を起動するため、**Terraform provider の `default_tags` が Batch 起動の EC2 / EBS に届かない**。放置するとタグなしインスタンスが立ち、コスト配賦が壊れる。

**対処**: 起動テンプレートの `tag_specifications`（`instance` と `volume` の 2 種）でタグを付ける。

```hcl
dynamic "tag_specifications" {
  for_each = ["instance", "volume"]
  content {
    resource_type = tag_specifications.value
    tags = merge({ Name = "..." }, var.tags)
  }
}
```

「モジュール内に共通タグを書かない」というルールがある場合の整理: **モジュールは `var.tags` を透過するだけで中身を決めない**（決めるのは env 層）ので、ルールの精神には反していない。

**むしろ危険なのは**、`var.tags` の既定が `{}` だと **env 層で渡し忘れても plan が通り、静かにタグなしインスタンスが立つ**こと。必須引数にするか `validation` で非空を強制する検討余地がある。

## 7. セキュリティグループ

- **インバウンドは 0 本**。ジョブは外向き通信しかしない。
- **egress は 443/tcp のみ**を `0.0.0.0/0` に対して開ける。ECR / S3 / CloudWatch Logs の宛先 IP を固定できないため。セキュリティスキャナ（trivy の `AVD-AWS-0104` 等）に引っかかるので、**理由を明記した ignore 登録**とセットにする。
- **より良い代替**: VPC エンドポイント（S3 Gateway、ECR API / DKR、Logs の Interface エンドポイント）を置けば、egress をエンドポイントの SG / プレフィックスリストに絞れる。さらに **GPU コンテナイメージ（数 GB）の pull が NAT ゲートウェイを経由しなくなり、データ処理料金が下がる**。セキュリティとコストの両方で効くので、初期スコープ外にしても issue として残す価値がある。

**単一ルールリソースを使う際の定石**:

```hcl
# aws_vpc_security_group_egress_rule は CIDR を 1 つしか取らないため、宛先ごとに 1 ルールへ展開する
egress_rules = { for rule in flatten([...]) : "${rule.port}-${rule.cidr_ipv4}" => rule }
```

キーを `"${port}-${cidr}"` にすると、同じポート・同じ CIDR を description 違いで 2 回渡したときにキー衝突する。実害は薄いが認識しておく。
