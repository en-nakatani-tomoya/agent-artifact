# terraform application coupled modules

## 概要
アプリ構成に強く依存した Terraform モジュール（バッチパイプラインの 1 ステージ = タスク定義 + ログ + IAM + SG）を作ることの是非を公式ガイダンスと照合した結論。薄いラッパーでなく抽象度を上げているなら、アプリ依存であること自体は推奨側の設計に含まれる。

## 詳細
## 判断基準は「再利用回数」ではなく「新しい概念を記述しているか」

HashiCorp は単一リソースの薄いラッパーを明確に非推奨としているが、その理由は再利用性ではなく抽象度。
「A good module should raise the level of abstraction by describing a new concept in your architecture」
「モジュール名が中のリソース型と同じにしかならないなら、新しい抽象を作れていない」という命名テストが判定に使える。

タスク定義 + ロググループ + タスクロール + 実行ロール + セキュリティグループをまとめて
「パイプライン 1 ステージの実行単位」という概念を作っているなら、この基準は満たす。

## アプリ依存は分割軸として正しい

HashiCorp のモジュール設計チュートリアルは、カプセル化 / 権限 / 変更頻度（volatility）の 3 軸を挙げ、
アプリ固有モジュール（web tier / app tier）は「リリースごとに変わる高 volatility な資源」、
ネットワーク・DB・セキュリティは「高権限・低 volatility」として明示的に区別している。
「モジュールは opinionated で、一つのことをうまくやるべき」とも書かれている。

つまり「アプリ依存モジュール = アンチパターン」ではなく、低 volatility の基盤層と混ぜるのがアンチパターン。

## Gruntwork の 2 層モデル

- modules: VPC / ECS クラスタのような汎用ビルディングブロック
- services: それらを組み合わせた、組織固有で再利用性は低いが直接デプロイできる単位

「チェーンは 2 段に留めろ」が推奨。env ルート → アプリ層モジュールの 2 段構成はこれに一致する。

## ARN / ID を入力で受けるのは推奨パターン

モジュール内で data 参照して依存を自分で引かず、呼び出し側から ID / ARN を受け取る形は
dependency inversion として公式に推奨されている（「モジュールはルートモジュールから依存を受け取るので、
同じモジュールを別の繋ぎ方で使える」）。バケット ARN を入力で渡す設計はこの点では正解側。

## 早すぎる抽象化の線引き

1 箇所でしか使わないならモジュール化しない、2 箇所目が必要になってから切り出す、が定石
（rule of three の緩い適用）。同一モジュールを 2 インスタンス以上作っているなら閾値は超えている。

## 実際に問題になるのは「値」ではなく「型」への焼き込み

アプリ依存が害になるのは、入力スキーマの型に固有名詞が入るケース。
例: storage_access = { pipeline_output_bucket_arn, models_bucket_arn } のように用途別の
バケット名をフィールド名にすると、用途が増えるたびにモジュール本体の IAM を編集することになる。

中立化する形:

  variable "bucket_access" {
    type = map(object({
      bucket_arn = string
      access     = string           # "read" | "read_write"
      key_prefix = optional(string) # null ならバケット全体
    }))
  }

得: 用途追加でモジュールを触らない / 複数モジュールに重複した S3 ポリシーを 1 本化 /
プレフィックス絞り込みが自然に入る。
損: 呼び出し側の記述が数行増える / 既存モジュールの書き換えとレビューが要る。

環境変数を map(string) で受けている箇所と一貫させる、という観点でも判断できる。

## 最終的な決め手は「利用者が誰か」

同一リポジトリ内の相対パス参照でモジュールを持ち、利用者がそのリポジトリだけなら、
インターフェース変更のコストが実質ゼロなのでアプリ依存の副作用は小さい。
バージョン付きの共有モジュールとして他チームに配る話が出た時点で、
入力スキーマの中立化は「やった方がいい」から「やらないと詰む」に変わる。

## 参考
- https://developer.hashicorp.com/terraform/language/modules/develop
- https://developer.hashicorp.com/terraform/language/modules/develop/composition
- https://developer.hashicorp.com/terraform/tutorials/modules/pattern-module-creation
- https://docs.gruntwork.io/library/overview/modules/
- https://blog.gruntwork.io/how-to-create-reusable-infrastructure-with-terraform-modules-25526d65f73d

---
Generated: 2026-08-26
