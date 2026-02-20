# gcp workload identity federation and service account

## 概要
GCP の Workload Identity Federation（WIF）と Service Account（SA）の概念、および GitHub Actions からキーレスで GCP を操作する認証フローを整理。

## 詳細
## Service Account（SA）

アプリケーションやシステムが GCP を操作するための専用アカウント。AWS の IAM Role に相当する。

- 人間は Google アカウント（user:xxx@gmail.com）で認証する
- プログラムは SA（sa-name@project-id.iam.gserviceaccount.com）で認証する
- SA 単体には権限がなく、IAM ロールを付与して初めて操作が可能になる

### IAM ロール

SA や人間に「何ができるか」を定義するもの。AWS の IAM Policy に相当。

代表的なロール例:
- roles/bigquery.dataOwner: BigQuery のデータセット・テーブルを管理
- roles/bigquery.jobUser: BigQuery のクエリジョブを実行
- roles/iam.serviceAccountAdmin: 他の SA を作成・削除・更新
- roles/resourcemanager.projectIamAdmin: プロジェクト内の IAM バインディングを管理

## Workload Identity Federation（WIF）

外部の ID プロバイダー（GitHub, GitLab, AWS 等）のトークンを GCP の認証情報に変換する仕組み。SA キー（JSON）を不要にする「キーレス認証」を実現する。

### 従来方式の課題
- SA キー（JSON ファイル）を GitHub Secrets 等に保存 → 漏洩リスク
- キーのローテーション管理が煩雑
- 長期間有効な認証情報が外部に存在するリスク

### WIF の 3 つの構成要素

1. **Workload Identity Pool**: 外部 ID をまとめて管理する入れ物。1 つのプールに複数のプロバイダーを設定可能。
2. **Workload Identity Provider**: プール内に作る検証ルール。OIDC トークンの発行元（issuer）の検証と、attribute_condition によるリポジトリ・ブランチ等の制約を行う。attribute_mapping でトークン内情報を GCP 属性にマッピングする。
3. **SA への WIF バインディング**: 「このプール経由で認証された外部 ID は、この SA として振る舞ってよい」という紐付け。google_service_account_iam_member で roles/iam.workloadIdentityUser を付与する。

### GitHub Actions での認証フロー

1. GitHub Actions が OIDC トークンを発行（リポジトリ名、ブランチ名等を含む）
2. google-github-actions/auth アクションが GCP に送信
3. GCP WIF がトークンを検証（issuer、repository、ref 等の条件）
4. 検証 OK → SA の一時認証情報を発行（有効期限: 約1時間）
5. アプリケーション（Terraform 等）がその認証情報で GCP リソースを操作

## AWS との対応表

| GCP | AWS | 役割 |
|---|---|---|
| Service Account | IAM Role | プログラムの認証 ID |
| IAM Role（権限） | IAM Policy | 何ができるかの定義 |
| Workload Identity Federation | OIDC Provider + IAM Role Trust Policy | 外部からのキーレス認証 |
| Workload Identity Pool | OIDC Provider | 外部 ID の受け入れ口 |
| attribute_condition | Trust Policy の Condition | どの外部 ID を信頼するかの制約 |

## セキュリティのベストプラクティス

- IAM ロールは最小権限の原則に従い、必要最小限を付与する
- WIF の attribute_condition でリポジトリだけでなくブランチも制限する（例: main のみ）
- roles/resourcemanager.projectIamAdmin + roles/iam.serviceAccountAdmin の組み合わせは権限昇格パスになりうるため注意

## 参考
- https://cloud.google.com/iam/docs/workload-identity-federation
- https://cloud.google.com/iam/docs/service-accounts
- https://docs.github.com/en/actions/security-for-github-actions/security-hardening-your-deployments/configuring-openid-connect-in-google-cloud-platform

---
Generated: 2026-02-20
