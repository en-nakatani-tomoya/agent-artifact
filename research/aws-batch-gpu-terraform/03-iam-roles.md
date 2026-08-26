# 03. IAM 設計 — 4 つのロールが誰の権限か

対象: 信頼ポリシーと権限ポリシーの違いは分かる人。Batch 特有の「ロールが多すぎる」問題を整理する。

## 1. 前提の 30 秒整理

IAM ロールは常に 2 枚のポリシーで出来ている。

- **信頼ポリシー (assume role policy)** = 「**誰が**このロールを名乗れるか」
- **権限ポリシー** = 「そのロールで**何ができるか**」

**サービスリンクロール (SLR)** は特殊で、AWS サービスが自分用に持つ、AWS 側が定義・管理するロール。ユーザーは作らない（初回利用時に自動作成される）。Batch では `AWSServiceRoleForBatch`。

## 2. 登場するロール一覧

| # | ロール | 定義場所 | 信頼する principal | 誰が使うか | 何ができるか |
| --- | --- | --- | --- | --- | --- |
| 1 | Batch サービスロール | **作らない**（SLR に委譲） | — | Batch サービス本体 | EC2 の起動/終了、ECS クラスタ操作 |
| 2 | ECS インスタンスロール | compute モジュール | `ec2.amazonaws.com` | **EC2 インスタンス**（の ECS エージェント） | ECS クラスタへの登録、タスクの受領 |
| 3 | Spot Fleet ロール | compute モジュール（SPOT 時のみ） | `spotfleet.amazonaws.com` | Spot Fleet | Spot インスタンスの起動とタグ付け |
| 4 | 実行ロール (Execution Role) | job モジュール | `ecs-tasks.amazonaws.com` | **ECS エージェント**（コンテナ起動の前準備） | ECR pull、ロググループへの書き込み、Secrets 取得 |
| 5 | ジョブロール (Job Role) | job モジュール | `ecs-tasks.amazonaws.com` | **コンテナの中のアプリコード** | S3 の読み書きなど |

**混同しやすいのは 4 と 5**。両方 `ecs-tasks.amazonaws.com` を信頼するが、使うタイミングが違う。

- 実行ロール = **コンテナが起動する前**。イメージを取ってきてログの出口を用意するのは ECS エージェントの仕事。足りないと `STARTING` で落ち、アプリのログは 1 行も出ない。
- ジョブロール = **コンテナが起動した後**。SDK（boto3 等）が拾う認証情報はこれ。足りないとアプリ側の `AccessDenied` として現れる。

## 3. Batch サービスロールは SLR に委譲する

CE で `service_role` を指定しない、が現在の AWS 推奨。自前のサービスロールを作る方式はレガシー扱いで、マネージドポリシーの更新に追随する手間もなくなる。

**副作用（要注意）**: SLR は「そのアカウントでまだ一度も Batch を使っていない場合」に自動作成されるが、**Terraform の apply がこれを待たずに進んで失敗するケース**がある。新規アカウントへの初回 apply で踏む。`aws_iam_service_linked_role` リソースで明示的に作る手もあり、「そこまでやるか」は判断。

## 4. マネージドポリシーを使う所と自作する所

**マネージドで済ませてよい**:
- `AmazonEC2ContainerServiceforEC2Role`（インスタンスロール）— ECS の定番。自作する理由が薄い。
- `AmazonEC2SpotFleetTaggingRole`（Spot Fleet ロール）— 同上。
- `AmazonSSMManagedInstanceCore`（オプトイン）— Session Manager で GPU インスタンスに入るため。**GPU ドライバ周りの切り分けで `nvidia-smi` を叩きたい場面は初期に必ず来る**ので、少なくとも検証環境では有効化を勧める。

**あえて自作する価値があるもの**:

```
AmazonECSTaskExecutionRolePolicy は ECR / Logs を Resource "*" で許可するため使わず、
必要な権限のみ自前ポリシーで組む
```

実行ロールでマネージドポリシーを避け、ECR は**そのステージのリポジトリだけ**、Logs は**そのステージのロググループだけ**に絞る。手間の割に効く最小権限化。

例外は 1 つだけで、これは正当:
```hcl
# ecr:GetAuthorizationToken は API 仕様上リソース指定ができない
resources = ["*"]
```

## 5. ジョブロールの S3 権限 — 知らないと見落とす 6 点

**(a) バケット ARN とオブジェクト ARN は別物**

`s3:ListBucket` は `arn:aws:s3:::bucket`（バケット自体）に対する権限。`s3:GetObject` は `arn:aws:s3:::bucket/*`（中身）に対する権限。混ぜると動かない。

**(b) プレフィックス絞り込みは 2 箇所に効かせる**

```hcl
# ListBucket 側は condition の s3:prefix で絞る
condition { test = "StringLike"  variable = "s3:prefix"  values = ["${prefix}*"] }
# オブジェクト側は ARN そのもので絞る
"${bucket_arn}/${prefix}*"
```
片方だけだと絞ったつもりで絞れていない。

**(c) マルチパートアップロードの 3 点セット**

```hcl
"s3:PutObject", "s3:AbortMultipartUpload", "s3:ListMultipartUploadParts"
```
boto3 は既定で 8 MiB を超えるとマルチパートに切り替わる。**`PutObject` だけ許可して「小さいファイルは通るのに本番データで落ちる」のが典型的な事故**。大きい parquet を書くパイプラインでは必ず必要。

**(d) `s3:ListBucketMultipartUploads` は別ステートメントに分ける**

このアクションはバケット ARN 向けかつ **`s3:prefix` を条件キーに持たない**。`ListBucket` と同じステートメントに入れると条件が付いてしまい機能しない。

**(e) 削除権限は既定オフにしてフラグで開ける**

`allow_delete = false` を既定にすると、再実行時に古い出力を消せない。パイプラインが「同一キーへの上書き（PutObject）」で回る前提かどうかで要否が決まる。上書きなら `PutObject` だけで足りる。

**(f) KMS は SSE 方式次第**

SSE-S3（AES256）なら KMS 権限は不要。SSE-KMS に切り替えるなら `kms:Decrypt` / `kms:GenerateDataKey` が要る。**入力変数（KMS キー ARN のリスト、既定は空）で切替可能にしておく**と、方針変更に追従できる。

## 6. IAM 観点のレビュー項目

- 実行ロールとジョブロールが**別のロールとして分かれているか**（兼用実装をよく見る）
- ジョブロールに ECR や Logs の権限が**混ざっていないか**
- `Resource = "*"` が残っているのは `ecr:GetAuthorizationToken` **だけ**か
- 逃げ道（`additional_policy_arns`）を全ロールに用意するのは便利だが、**env 層でこれを使い始めると最小権限がモジュール外に漏れる**。使い始めたら設計を見直す合図。
