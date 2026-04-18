# claude-code-azure-to-tf

※このリポジトリは現在開発中のリソースとなります

Claude Code を使って Azure インフラ向けの Terraform コードを生成・管理するためのプロンプト・指示セットです。

## 概要

このリポジトリは、Azure 上の既存リソースを Terraform で管理するための Claude Code スラッシュコマンド・ルール・設定を提供します。

主な機能：

- **`/azure-to-tf`** — 既存の Azure リソースグループを Terraform 管理下に取り込むスラッシュコマンド
- **`/generate-pipeline`** — 生成した Terraform コードを実行する CI/CD パイプラインを生成するスラッシュコマンド
- **Terraform スタイルルール** — 命名規則・ファイル構成・変数定義などのコーディング規約
- **Azure プラクティスルール** — セキュリティ・ネットワーク・シークレット管理などのベストプラクティス
- **自動フォーマット** — `.tf` ファイルの編集時に `terraform fmt` を自動実行する Hook
- **MCP サーバー** — Microsoft Learn ドキュメントと Terraform Registry を参照可能

## 前提条件

### 開発環境

Dev Container（`.devcontainer/`）を使用することを推奨します。以下のツールが含まれています：

- Azure CLI
- Terraform（>= 1.5.0）
- aztfexport
- Claude CLI

### 環境変数

`.env.example` をコピーして `.env` を作成し、各値を設定してください。

```bash
ARM_TENANT_ID=your-tenant-id
ARM_SUBSCRIPTION_ID=your-subscription-id
ARM_CLIENT_ID=your-client-id
ARM_CLIENT_SECRET=your-client-secret
```

HCP Terraform を使用する場合は以下も設定します：

```bash
TFE_TOKEN=your-tfe-token
```

> `.env` は `.gitignore` によりコミット対象外です。クレデンシャルをリポジトリにコミットしないでください。

---

## `/azure-to-tf` — Azure リソースの Terraform オンボーディング

既存の Azure リソースグループを aztfexport でエクスポートし、azurerm/azapi ベースの Terraform コードに変換・インポートするまでの一連の作業を自動化します。

### 使い方

```
/azure-to-tf --resource-group <rg1>[,<rg2>,...] [--subscription <sub-id>] [--backend local|azblob|hcp] [--include-defaults] [--import-rg] [--exclude-resource-type <type,...>] [--output-dir <path>]
```

| オプション | 説明 | デフォルト |
|-----------|------|-----------|
| `--resource-group` | エクスポート対象 RG（カンマ区切りで複数可） | （必須） |
| `--subscription` | Azure サブスクリプション ID | `ARM_SUBSCRIPTION_ID` 環境変数 |
| `--backend` | Backend 種別（`local` / `azblob` / `hcp`） | `local` |
| `--include-defaults` | Azure 自動生成デフォルトリソースをエクスポート対象に含める | 省略時は除外 |
| `--import-rg` | リソースグループ自体を Terraform 管理下に置く | 省略時は除外 |
| `--exclude-resource-type` | 追加除外するリソース種別（カンマ区切り） | 省略時は追加なし |
| `--output-dir` | Terraform ファイル出力先 | `infra/` |

### 実行フロー

| フェーズ | 内容 |
|---------|------|
| Phase 1 | 前提条件チェック・Azure ログイン・aztfexport によるエクスポート・コンポーネント分類・モジュール化検討 |
| Phase 2 | Terraform コード生成（`main.tf` / `variables.tf` / `outputs.tf` / `terraform.tf` / `locals.tf` / `import.tf`）・環境別 tfvars 生成・`terraform validate` |

複数 RG に環境識別子のみ異なる類似グループ（例: `prd-rg-app` と `stg-rg-app`）が検出された場合、単一テンプレート＋環境別 tfvars として生成するか対話形式で確認します。

### 生成されるプロジェクト構造例

```
infra/
├── network/
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   ├── terraform.tf
│   ├── locals.tf
│   └── import.tf
├── api/
├── data/
├── app/
├── modules/
│   └── private_dns_zone/
└── environments/
    ├── prd/
    │   ├── network.tfvars
    │   ├── api.tfvars
    │   └── api.secret.tfvars   # 機密変数（.gitignore 対象）
    └── stg/
        ├── network.tfvars
        └── api.tfvars
```

### 注意事項

- `terraform plan` / `terraform apply` は実行前にユーザーへの確認が必須です
- `*.secret.tfvars` は `.gitignore` に自動追加されます。リポジトリにコミットしないでください
- AVM（Azure Verified Modules）は使用しません。既存リソースの import 時に意図しない設定変更が発生するためです
- import できなかったリソースはスキップされ、手動対応が必要な旨が報告されます

---

## `/generate-pipeline` — CI/CD パイプラインの生成

`/azure-to-tf` で生成した `infra/` 構造を読み取り、コンポーネントを正しい実行順序で Terraform 実行する CI/CD パイプラインを生成します。

### 使い方

```
/generate-pipeline [--platform github|azuredevops|both] [--output-dir <path>]
```

| オプション | 説明 | デフォルト |
|-----------|------|-----------|
| `--platform` | 対象プラットフォーム（`github` / `azuredevops` / `both`） | 対話形式で確認 |
| `--output-dir` | 生成ファイルの出力先 | リポジトリルート |

### 生成されるファイル

| プラットフォーム | ファイル |
|---|---|
| GitHub Actions | `.github/workflows/terraform.yml` |
| Azure DevOps | `azure-pipelines.yml` |

### パイプラインの動作

- PR 時：`terraform fmt -check` → `terraform validate` → `terraform plan`
- `main` ブランチ push 時：上記に加えて `terraform apply` を実行
- 本番環境（`prd` / `prod` / `production`）の apply には手動承認ゲートを設定
- コンポーネントは依存関係の順序（`network → data → api → app → front` 等）で順次実行

---

## リポジトリ構成

```
.
├── .claude/
│   ├── commands/
│   │   ├── azure-to-tf.md          # /azure-to-tf スラッシュコマンド定義
│   │   └── generate-pipeline.md    # /generate-pipeline スラッシュコマンド定義
│   ├── rules/
│   │   ├── terraform-style.md      # Terraform コードスタイルルール
│   │   ├── azure-practices.md      # Azure 固有のプラクティスルール
│   │   ├── aztf-phase1.md          # /azure-to-tf Phase 1 サブエージェント手順
│   │   ├── aztf-phase2.md          # /azure-to-tf Phase 2 サブエージェント手順
│   │   ├── aztf-templates.md       # Terraform コード生成テンプレート集
│   │   ├── aztf-classification.md  # リソースのコンポーネント分類ロジック
│   │   ├── aztf-backend.md         # Backend 設定・移行手順
│   │   └── pipeline-templates.md   # CI/CD パイプライン生成テンプレート集
│   └── settings.json               # 権限・MCP サーバー・Hook 設定
├── .devcontainer/                  # Dev Container 設定
├── .env.example                    # 環境変数テンプレート
├── CLAUDE.md                       # Claude Code へのプロジェクト指示
├── CHANGELOG.md                    # 変更履歴
├── SECURITY.md                     # セキュリティポリシー・脆弱性報告先
└── README.md
```
