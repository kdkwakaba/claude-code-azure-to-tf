# claude-code-azure-terraform

Claude Code を使って Azure インフラ向けの Terraform コードを生成・管理するためのプロンプト・指示セットです。

## 概要

このリポジトリは、Azure 上の既存リソースを Terraform で管理するための Claude Code Skill・ルール・設定を提供します。

主な機能：

- **`/azure-to-tf`** — 既存の Azure リソースを Terraform 管理下に取り込むスラッシュコマンド
- **Terraform スタイルルール** — AVM 優先・命名規則・ファイル構成などのコーディング規約
- **Azure プラクティスルール** — セキュリティ・ネットワーク・シークレット管理などのベストプラクティス
- **自動フォーマット** — `.tf` ファイルの編集時に `terraform fmt` を自動実行する Hook

## 前提条件

### 開発環境

Dev Container（`.devcontainer/`）を使用することを推奨します。以下のツールが含まれています：

- Azure CLI
- Terraform（>= 1.5.0）
- aztfexport
- Claude CLI

### 環境変数

```bash
export ARM_CLIENT_ID="<サービスプリンシパルのクライアント ID>"
export ARM_CLIENT_SECRET="<サービスプリンシパルのクライアントシークレット>"
export ARM_TENANT_ID="<テナント ID>"
export ARM_SUBSCRIPTION_ID="<サブスクリプション ID>"
```

## `/azure-to-tf` — Azure リソースの Terraform オンボーディング

既存の Azure リソースグループを aztfexport でエクスポートし、Azure Verified Modules（AVM）対応の Terraform コードに変換・インポートするまでの一連の作業を自動化します。

### 使い方

```
/azure-to-tf --resource-group <rg-name> [--subscription <sub-id>]
```

### 実行ステップ

| STEP | 内容 |
|------|------|
| 0 | 前提条件チェック（環境変数・ツール） |
| 1 | サービスプリンシパルで Azure にログイン |
| 2 | aztfexport でリソースグループをエクスポート |
| 3 | リソースをコンポーネント（front / api / backend / database）に分類 |
| 4 | AVM モジュールの存在を Terraform Registry API で動的確認 |
| 5 | プロジェクト構造（`infra/`）を作成し Terraform コードを生成 |
| 6 | 環境別 tfvars を生成（production / staging / development） |
| 7 | `terraform fmt`（`.tf` 編集時に Hook で自動実行） |
| 8 | `terraform init` + import ブロックの処理 |
| 9 | `terraform validate`（全ファイル生成後に実行） |
| 10 | `terraform plan` で内容を検証 |
| 11 | Development 環境へ `terraform apply`（ユーザー確認後） |
| 12 | Azure Storage Backend へ移行（ユーザー確認後） |

### 生成されるプロジェクト構造例

```
infra/
├── front/                  # フロントエンド関連リソース
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   ├── terraform.tf
│   └── locals.tf
├── api/                    # API 関連リソース
├── backend/                # バックエンド関連リソース
├── data/               # データベース関連リソース
└── environments/
    ├── production.tfvars
    ├── staging.tfvars
    └── development.tfvars
```

### 環境別スケール方針

| リソース | production | staging | development |
|---------|-----------|---------|-------------|
| App Service Plan | P2v3 | B2 | B1 |
| Azure SQL Database | GP_S_Gen5_4 | GP_S_Gen5_2 | Basic |
| Azure PostgreSQL Flexible | GP_Standard_D4s_v3 | GP_Standard_D2s_v3 | Burstable_B1ms |
| Azure Cache for Redis | Standard_C1 | Basic_C0 | Basic_C0 |
| AKS ノード SKU | Standard_D4s_v3 | Standard_D2s_v3 | Standard_B2s |

### 注意事項

- `terraform plan` / `terraform apply` / `terraform init -migrate-state` は実行前にユーザーへの確認が必要です
- Backend 用 Azure Storage Account はユーザーが事前に準備し、権限を付与してください
- import できないリソースはスキップされ、手動対応が必要な旨が報告されます

## リポジトリ構成

```
.
├── .claude/
│   ├── rules/
│   │   ├── terraform-style.md   # Terraform コードスタイルルール
│   │   └── azure-practices.md   # Azure 固有のプラクティスルール
│   ├── skills/
│   │   └── azure-to-tf.md        # /azure-to-tf スラッシュコマンド定義
│   └── settings.json            # MCP サーバー・Hook 設定
├── CLAUDE.md                    # Claude Code へのプロジェクト指示
└── README.md
```
