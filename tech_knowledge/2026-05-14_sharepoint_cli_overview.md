# sharepoint cli overview

## 概要
SharePoint を CLI から操作する代表的なツール (CLI for Microsoft 365 / PnP PowerShell) の概要と、CLI for Microsoft 365 の認証方式まとめ

## 詳細
## ツール選択肢

### 1. CLI for Microsoft 365 (m365)
- PnP コミュニティ提供の OSS。クロスプラットフォーム (Windows / macOS / Linux、bash / zsh / PowerShell どこでも動く)
- npm パッケージ: `npm i -g @pnp/cli-microsoft365`
- コマンド体系は `m365 <service> <verb>` 形式 (例: `m365 spo site list`, `m365 spo file get`)
- SharePoint だけでなく Teams / OneDrive / Planner / Power Platform / Viva など横断管理可能

### 2. PnP PowerShell
- 元 Windows 専用、現在はクロスプラットフォーム化済み
- PowerShell モジュール `PnP.PowerShell`
- 2024 以降、app-only 認証必須に強化されユーザー資格情報での自動化は厳しい

既存スクリプトが PowerShell なら PnP PowerShell、新規でクロスプラットフォーム自動化なら m365 CLI が現代的な選択。

## CLI for Microsoft 365 でできる SharePoint 操作
- サイト / サブサイトの CRUD、設定変更
- リスト・ライブラリ・アイテム・ファイル操作
- アクセス権、サイトコレクション App Catalog 管理
- SharePoint Framework (SPFx) プロジェクトのスキャフォールド / 管理

## 認証方法 (m365 CLI)

`m365 login --authType <type>` で指定。

| authType | 用途 | SharePoint |
|---|---|---|
| deviceCode | デフォルト。ブラウザでデバイスコード入力 | 対応 |
| browser | 対話ブラウザ認証。条件付きアクセス環境で推奨 | 対応 |
| certificate | 証明書ベース。CI/CD 向け非対話 | 対応 |
| password | username/password。MFA 不可 | 対応 |
| identity | Azure Cloud Shell 等の Managed Identity | 対応 |
| federatedIdentity | GitHub Actions の OIDC フェデレーション | 対応 |
| secret | クライアントシークレット (app-only) | SharePoint API は不可 |

## 使い分けの目安
- ローカル開発: deviceCode / browser が手軽
- CI/CD で SharePoint を触る: 証明書認証 or federated identity (OIDC)
- Entra app ID + Secret は SharePoint API 呼び出しが失敗するので NG

## インストールと初期セットアップ
```bash
npm i -g @pnp/cli-microsoft365
m365 setup        # 初期構成
m365 login        # ログイン (デフォルト deviceCode)
m365 help         # コマンド一覧
```

## 参考
- https://pnp.github.io/cli-microsoft365/
- https://pnp.github.io/cli-microsoft365/user-guide/connecting-microsoft-365/
- https://pnp.github.io/cli-microsoft365/cmd/login/
- https://github.com/pnp/cli-microsoft365
- https://pnp.github.io/blog/post/pnp-powershell-or-cli-for-microsoft-365-or-both-or-other/

---
Generated: 2026-05-14
