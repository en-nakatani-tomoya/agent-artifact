# AWS Batch (EC2 / GPU) を Terraform で組むときの設計知見

GPU バッチ処理基盤を AWS Batch（ECS on EC2）+ Terraform で構築した際の、レビュー観点と設計判断の記録。
具体的なプロジェクト名・アカウント情報は抽象化してある。

## 想定シナリオ

ML パイプラインの 2 ステージ（前処理 / 埋め込み生成）を GPU コンテナで実行する基盤を、
Terraform の child module 2 つ + 環境ルート（env 層）の結線として構成する。

```text
modules/batch_gpu_compute   … コンピューティング環境 + ジョブキュー + SG + 起動テンプレート + インスタンスロール
modules/batch_job           … ジョブ定義 + ジョブロール + 実行ロール + ロググループ
envs/{env}/*.tf             … 上記を呼び、S3 / ECR / VPC と結線
```

## 読む順序

| ファイル | 内容 |
| --- | --- |
| [01-aws-batch-mental-model.md](01-aws-batch-mental-model.md) | Batch の正体（ECS on EC2 + キュー + オートスケーラ）、ジョブのライフサイクルと障害切り分け |
| [02-module-boundary-and-wiring.md](02-module-boundary-and-wiring.md) | モジュール境界の引き方、ステージ固有 IAM をどこで組むかの比較 |
| [03-iam-roles.md](03-iam-roles.md) | 登場する 4 ロールの役割分担、S3 ポリシーの非自明な落とし穴 |
| [04-compute-environment-details.md](04-compute-environment-details.md) | CE 置換の罠、AMI 解決、SPOT、起動テンプレート、タグ、SG |
| [05-job-definition-details.md](05-job-definition-details.md) | ジョブ定義の組み立て、イメージタグ戦略、リトライ、メモリ整合、ログ |
| [06-review-checklist.md](06-review-checklist.md) | レビュー観点チェックリスト（そのまま流用可） |

## 要点だけ 10 行で

1. Batch はコンテナを動かさない。動かすのは ECS で、資源は EC2 Fleet。Batch はキューとオートスケーラ。
2. CE / キュー / ジョブ定義は**ライフサイクルが違う**ので、モジュール境界はそこで引く。
3. CE は `name_prefix` + `create_before_destroy` + IAM への `depends_on` の三点セットが必須。書かないと置換も削除も詰む。
4. ロールは 4 種類（インスタンス / Spot Fleet / 実行 / ジョブ）。**実行ロールとジョブロールの混同**が最頻出。
5. `AmazonECSTaskExecutionRolePolicy` は `Resource "*"` なので、最小権限にするなら自作する。
6. S3 の書き込み権限は `PutObject` だけでは足りない。**マルチパートの 3 アクション**を揃える。
7. IMDS の `http_put_response_hop_limit` は **2 以上**。1 だとコンテナから認証情報が取れない。
8. ジョブのメモリ要求は**インスタンス物理メモリより小さい「ECS 利用可能メモリ」**が上限。超えると `RUNNABLE` で無言停止。
9. SPOT を選ぶなら `evaluate_on_exit` で「中断のみリトライ」に絞る。でないとアプリの決定的失敗も再実行される。
10. ロググループは Terraform で明示的に作る。自動作成任せだと**保持期間が無期限**になる。
