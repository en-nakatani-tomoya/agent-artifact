# 05. ジョブ定義モジュールの実装詳細

## 1. `container_properties` を `jsonencode` で組む

Batch のジョブ定義は、API 上「コンテナ設定を JSON 文字列で持つ」構造をしている。Terraform でもそうなる。

```hcl
container_properties = jsonencode(local.container_properties)
```

**落とし穴が 2 つある。**

**(a) map の順序揺れによる無限差分**

```hcl
# キー名でソートし、map の順序揺れによる不要な差分を防ぐ
environment = [for name in sort(keys(var.environment_variables)) : { name = ..., value = ... }]
secrets     = [for name in sort(keys(var.secrets)) : { name = ..., valueFrom = ... }]
```

`environment` と `secrets` は JSON の**配列**なので順序が意味を持つ。AWS 側から返る順序と一致しないと毎回差分が出る。`sort` で正規化するのが定石。

**(b) 空の値を出さない**

```hcl
merge(
  { ...必須項目... },
  length(var.command) > 0 ? { command = var.command } : {},
  var.shared_memory_size_mebibytes == null ? {} : { linuxParameters = { sharedMemorySize = ... } },
)
```

`command = []` を送ると API が受け付けない / 差分が出る。`merge` でキーごと落とす。

**副作用**: `jsonencode` 方式は plan の差分が「JSON 文字列全体の置換」として表示されるので**読みにくい**。EC2 コンテナジョブでは代替がないので受け入れる。

## 2. イメージタグ戦略 — `:latest` か不変タグか

| | `:latest` を参照 | 不変タグ（`:sha-abc123`） |
| --- | --- | --- |
| デプロイ | **イメージ push だけ**。Terraform apply 不要 | apply が必要 = CI で Terraform を回す |
| 何が動いているか | **ジョブ定義からは分からない**。ECR を見に行く必要がある | ジョブ定義を見れば分かる |
| ロールバック | 前のイメージを `latest` として push し直す | 前のリビジョンを投入 |
| 障害調査 | 「先週失敗したジョブはどのイメージだったか」が**追えない** | 追える |

`:latest` を選ぶなら、ECR 側を `IMMUTABLE_WITH_EXCLUSION`（latest だけ上書き可）にして **latest 以外は不変**という担保を置くのが最低条件。

それでも「実行時にどの digest だったか」の記録は残らない（Batch のジョブ詳細の保持は短い）。**イメージ digest / ビルド ID をジョブ定義のタグや環境変数に載せる、あるいはアプリの起動ログに出す**といった補完がないと、運用フェーズの障害調査が成立しない。

なお `propagate_tags = true` はジョブ定義のタグを ECS タスクに伝播するもので、イメージの同定には効かない。

## 3. リトライとタイムアウト

```hcl
retry_strategy { attempts = 2 }               # 総試行回数（1 = 再試行なし）
timeout        { attempt_duration_seconds = 7200 }   # 1 回の試行あたり
```

- `attempts` は**総試行回数**であって「リトライ回数」ではない。2 = 1 回だけやり直す。
- `timeout` は **1 回の試行あたり**。3 時間 × 2 回 = 最悪 6 時間占有しうる。
- **`evaluate_on_exit` を書かないと、あらゆる失敗が等しくリトライされる**。

```hcl
retry_strategy {
  attempts = 2
  evaluate_on_exit {
    action           = "RETRY"
    on_status_reason = "Host EC2*"   # Spot 中断など
  }
  evaluate_on_exit {
    action = "EXIT"                   # それ以外（アプリの決定的失敗）は即終了
  }
}
```

**SPOT を選んだなら、この切り分けはほぼ必須**。GPU の時間単価を考えると、決定的に失敗するジョブを 2 回走らせるのは純粋な浪費。副作用のある処理なら冪等性の担保も要る。

## 4. リソース要求とインスタンスの整合（最頻出の事故）

```hcl
# ジョブ定義
vcpus = 4, memory_mebibytes = 30720, gpu = 1
# インスタンス: 4 vCPU / 32 GiB (32768 MiB) / GPU 1
```

**ECS on EC2 では、インスタンスの物理メモリのうち OS とカーネルが使う分を除いた「ECS が認識する利用可能メモリ」が上限**で、32768 MiB より確実に小さい（実測で 31000 MiB 前後、AMI とカーネルバージョンに依存）。

**超えていると、ジョブは `RUNNABLE` から永遠に進まず、しかもエラーメッセージがほとんど出ない**（Batch のジョブ状態理由に短く出るだけ）。GPU バッチ基盤で最も嵌まりやすいポイント。

vCPU も同様にオーバーサブスクライブできない。4 vCPU インスタンスに `vcpus = 4` はぴったりで余裕ゼロ。

**対策**: 構築後にまず**ダミージョブを 1 本投げて `RUNNABLE` を抜けるか確認する**手順を運用フローに入れる。plan が通ることは何の保証にもならない。

## 5. `/dev/shm` の設定

Docker の既定 `/dev/shm` は **64 MB**。PyTorch の `DataLoader` を `num_workers > 0` で使うと共有メモリを食い、`Bus error` や `DataLoader worker (pid xxx) is killed` で落ちる。

```hcl
linuxParameters = { sharedMemorySize = 2048 }   # MiB
```

ML ワークロードなら**入力変数として用意しておく**。既定 64 MB のまま本番投入すると踏む。

## 6. ログ

```hcl
resource "aws_cloudwatch_log_group" "this" {
  name              = "${local.name_prefix}-cwlg"
  retention_in_days = 30
}

logConfiguration = {
  logDriver = "awslogs"
  options   = {
    "awslogs-group"         = aws_cloudwatch_log_group.this.name
    "awslogs-stream-prefix" = local.log_stream_prefix
  }
}
```

- **ロググループを Terraform で明示的に作るのが重要**。作らないと ECS が自動作成し、**保持期間が「無期限」になってコストが青天井**になる。
- **ステージごとにロググループを分離**すると、実行ロールの Logs 権限もそのグループだけに絞れる（→ 03）。最小権限と整合する。
- **ログストリーム名は `{prefix}/{container-name}/{ecs-task-id}`**。ジョブ ID とストリーム名の対応は Batch のジョブ詳細から辿る。**ジョブ ID や実行日時からログに到達する導線**が運用設計に要る。

## 7. シークレットの扱い

```hcl
variable "secrets" {
  # 環境変数名 => SSM Parameter Store のパラメータ ARN
  type = map(string)
  validation {
    condition     = alltrue([for v in values(var.secrets) : can(regex(":ssm:", v))])
    error_message = "secrets の値は SSM Parameter Store のパラメータ ARN を指定する"
  }
}
```

- この map を渡すと、ジョブ定義の `secrets` に載り、**実行ロールにその ARN だけの `ssm:GetParameters` が自動で付く**。権限と設定が 1 箇所で完結する良い形。
- validation が「渡し間違えると実行時まで気づけない」ミスを plan 時に潰す。
- **ただし Secrets Manager の ARN は弾かれる**。DB 接続情報などを Secrets Manager で管理するステージがあるなら、対応が要る。モジュールの適用範囲の境界として認識しておく。
