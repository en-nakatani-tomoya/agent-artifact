# ecs task definition revision behavior

## 概要
ECS タスク定義の revision が、内容変更の有無ではなく RegisterTaskDefinition API の呼び出し回数で進む仕組みを整理

## 詳細
ECS のタスク定義は family 単位でまとまり、my-app:1 / my-app:2 のように revision 番号が付く。revision は内容が変わったかどうかではなく、RegisterTaskDefinition API が呼ばれるたびに +1 される。中身を 1 文字も変えずに再登録しても新しい revision が発行され、ECS 側で差分検知は行われない。一度発行された revision はイミュータブルで上書き編集はできない。

revision が進む操作: マネジメントコンソールの「新しいリビジョンの作成」、aws ecs register-task-definition の実行、差分のない再登録、CI/CD でイメージタグだけ差し替えた register。

Terraform 運用の場合: aws_ecs_task_definition リソースは属性に差分があるときだけ Terraform が RegisterTaskDefinition を呼び直すため revision が進む。よくある差分要因はコンテナイメージタグの変更、環境変数・シークレット参照の追加変更、CPU/メモリ/ポートマッピングの変更、task role / execution role の ARN 変更、container_definitions JSON のキー順序や空白による実質差分。逆に :latest タグ運用などで Terraform 上の属性が変わらなければ、新イメージを push しても revision は進まず、サービス更新で新イメージを引き直すだけになる。

## コード例
```
# 中身が同じでも revision は進む
aws ecs register-task-definition --cli-input-json file://task-def.json
# -> my-app:5
aws ecs register-task-definition --cli-input-json file://task-def.json
# -> my-app:6 (内容同一でも +1)
```

## 参考
- https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task_definitions.html
- https://docs.aws.amazon.com/AmazonECS/latest/APIReference/API_RegisterTaskDefinition.html

---
Generated: 2026-05-18
