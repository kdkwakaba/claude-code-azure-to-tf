# STEP 5: コード生成テンプレート集

`/azure-to-tf` コマンドの STEP 5 で参照するテンプレートと規則。

## terraform.tf（Backend 別テンプレート）

`--backend` オプションに応じて backend ブロックのみ切り替える。**`--backend` オプションを省略した場合（デフォルト）は必ず `backend "local" {}` を使用する。**

プロバイダーバージョン（`AZURERM_VERSION`・`AZAPI_VERSION`）は STEP 0 で検出した値を使用する。

### `--backend local`（デフォルト・省略時）

```hcl
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "<AZURERM_VERSION>"  # STEP 0 で検出した値
    }
  }

  backend "local" {}
}

provider "azurerm" {
  features {}
  # subscription_id は ARM_SUBSCRIPTION_ID 環境変数から自動取得する
  # 明示的に指定する場合は var.subscription_id を使用する
  subscription_id = var.subscription_id
}
```

### `--backend azblob`

```hcl
backend "azurerm" {
  resource_group_name  = "<STEP 0.5 で収集した値>"
  storage_account_name = "<STEP 0.5 で収集した値>"
  container_name       = "<STEP 0.5 で収集した値>"
  key                  = "<component>.tfstate"
}
```

### `--backend hcp`

```hcl
cloud {
  organization = "<STEP 0.5 で収集した organization 名>"

  workspaces {
    name = "<STEP 0.5 で収集した prefix>-<component>"
  }
}
```

### azapi も必要な場合（azurerm に対応タイプが存在しないリソースがある場合のみ追加）

```hcl
azapi = {
  source  = "Azure/azapi"
  version = "<AZAPI_VERSION>"  # STEP 0 で検出した値
}
```

---

## variables.tf（共通変数）

全コンポーネント共通の変数を定義する：

```hcl
variable "subscription_id" {
  type        = string
  description = "Azure サブスクリプション ID（省略時は ARM_SUBSCRIPTION_ID 環境変数を使用）"
  default     = null
}

variable "resource_group_name" {
  type        = string
  description = "リソースグループ名"
}

variable "location" {
  type        = string
  description = "Azure リージョン"
}

variable "env" {
  type        = string
  description = "環境識別子（例: prd / stg / dev）。タグ値・リソース名プレフィックスの一部として使用する"
}

# リソース種別固有の変数（aztfexport の出力から抽出）
```

コンポーネント内のリソースが **複数 RG にまたがる** 場合は、`resource_group_name` を削除し RG ごとに識別子付きの変数に置き換える：

```hcl
variable "resource_group_name_front" {
  type        = string
  description = "front リソースグループ名"
}

variable "resource_group_name_data" {
  type        = string
  description = "data リソースグループ名"
}
```

**識別子の導出ルール**：RG 名の末尾セグメントを使用する（`rg-myapp-front` → `front`、`rg-myapp-prd-data` → `data`）。末尾が環境識別子のみの場合（`rg-myapp-prd`）はスネークケース全体を使う（`rg_myapp_prd`）。

`env` 変数はタグの `env` キーと、リソース名プレフィックスの環境部分（例: `prd-func-myapp-01` の `prd`）の両方に対応する。`locals.tf` で参照し、リソース名の変数はすべて tfvars で完全な名前を定義する（`env` を使って動的に組み立てない）。

---

## パラメーター化の対象

以下の値はすべて `main.tf` にハードコードせず、`variables.tf` に変数として定義し実際の値を `environments/<env>/<component>.tfvars` に記載する。機密変数（`sensitive = true`）は `<component>.secret.tfvars` に記載する。

| カテゴリ | 対象 | 変数例 |
|---|---|---|
| 命名 | リソース名（環境名・プレフィックスを含む） | `function_app_name`、`sql_server_name`、`vnet_name` |
| 命名 | サブネット名 | `front_subnet_name`、`api_subnet_name` |
| コンピュート | SKU・価格レベル | `app_service_sku_name`、`sql_database_sku_name`、`redis_sku_name` |
| コンピュート | スケーリング設定（インスタンス数・容量等） | `app_service_worker_count`、`redis_cache_capacity` |
| コンピュート | ストレージレプリケーション種別 | `storage_account_replication_type` |
| ネットワーク | VNet アドレス空間 | `vnet_address_space`（`list(string)`） |
| ネットワーク | サブネットアドレスプレフィックス | `front_subnet_prefix`、`api_subnet_prefix`（`string`） |
| ネットワーク | サービスエンドポイント | `api_subnet_service_endpoints`（`list(string)`） |
| ネットワーク | NSG ルール（CIDR・ポート等） | `allowed_ip_ranges`、`allowed_port` |
| ネットワーク | プライベートエンドポイント IP | `psql_private_ip` |
| ネットワーク | DNS ゾーン名・TTL | `private_dns_zone_name`、`dns_record_ttl` |
| DB | バージョン | `psql_version`、`mysql_version` |
| DB | ストレージサイズ（MB） | `psql_storage_mb` |
| DB | バックアップ保持期間（日数） | `psql_backup_retention_days` |
| DB | 高可用性モード・ゾーン番号 | `psql_high_availability_mode`、`psql_zone` |
| アプリ | App Settings の環境依存値 | `storage_blob_uri`、`app_insights_connection_string` |
| アプリ | CORS 許可オリジン | `cors_allowed_origins`（`list(string)`） |
| アプリ | HTTPS のみ強制フラグ | `https_only`（`bool`） |
| アプリ | VNet 統合サブネット ID（他コンポーネント参照） | `terraform_remote_state` 経由で output 参照 |
| 機密 | ストレージアクセスキー | `storage_account_access_key` |
| 機密 | Function App 暗号化キー | `function_app_decryption_key` |
| 機密 | DB 管理者パスワード | `psql_admin_password` |

**パラメーター化の判断基準**：環境（dev / stg / prd）で値が異なる可能性がある、運用中に変更されうるネットワーク設定（IP/CIDR/ポート）、セキュリティ上ハードコードを避けるべき値（接続文字列等）のいずれかに該当する場合はパラメーター化する。Azure のリソース種別固有の固定値（`account_kind = "StorageV2"` 等）・全環境で変わらない設定値はハードコードで可。

---

## locals.tf（タグ・名前の管理）

**既存リソースのタグをそのまま複製する。新しいタグキーは追加しない。**
タグがないリソースには `tags` 引数を設定しない。

```hcl
locals {
  resource_prefix = "<aztfexport から抽出した実際のプレフィックス>"

  # 既存リソースのタグ構造をそのまま複製する
  # 例: 既存タグが { env = "prd", component = "api" } の場合
  common_tags = {
    env       = var.env   # 環境識別子は var.env で統一
    component = "<コンポーネント固定値>"
  }
}
```

タグの `env` キーには `var.env` を使用する。

---

## outputs.tf（出力値）

他コンポーネントから `terraform_remote_state` 経由で参照される値のみ出力する。すべての output に `description` を付ける。

**接続文字列・キー・URI など機密性のある値には必ず `sensitive = true` を設定する。** これを省略すると Terraform の plan ログや state ファイルに平文で記録される。

### sensitive = true が必要な output の例

```hcl
output "app_insights_connection_string" {
  value       = azurerm_application_insights.main.connection_string
  description = "Application Insights 接続文字列"
  sensitive   = true
}

output "app_insights_instrumentation_key" {
  value       = azurerm_application_insights.main.instrumentation_key
  description = "Application Insights インストルメンテーションキー"
  sensitive   = true
}

output "key_vault_uri" {
  value       = azurerm_key_vault.main.vault_uri
  description = "Key Vault の URI"
  sensitive   = true
}
```

### sensitive = true が不要な output の例（非機密の参照値）

```hcl
output "subnet_id" {
  value       = azurerm_subnet.api.id
  description = "API サブネットのリソース ID"
}

output "log_analytics_workspace_id" {
  value       = azurerm_log_analytics_workspace.main.workspace_id
  description = "Log Analytics ワークスペース ID"
}
```

### sensitive = true の判断基準

| 値の種類 | sensitive |
|---|---|
| 接続文字列（`connection_string`） | true |
| アクセスキー・シークレット・パスワード | true |
| インストルメンテーションキー | true |
| Key Vault URI | true |
| リソース ID・サブネット ID | false |
| ホスト名・FQDN（パブリックに解決可能） | false |
| ワークスペース ID・リソース名 | false |

---

## 動的な値の参照

Azure がリソース作成時に自動生成する値（ランダムサフィックスを含むホスト名・エンドポイント・ID 等）は、aztfexport の出力にその時点の値がハードコードされているが、**リソースが再作成されると値が変わる**。これらは可能な限り他リソースの output 参照に置き換える。

### 参照に置き換えるべき値のパターン

> **注意**: `azurerm_app_service_custom_hostname_binding` の `*.azurewebsites.net` デフォルトバインディングは STEP 2 の除外リストに含まれているため、除外済みです。App Service / Function App のデフォルトホスト名は他リソースの `app_settings` 等から参照する場合のみ下表の `default_hostname` を使用してください。

| ハードコードされがちな値 | 置き換え後の参照 |
|---|---|
| App Service / Function App のデフォルトホスト名（`*.azurewebsites.net`） | `azurerm_windows_web_app.main.default_hostname` / `azurerm_function_app_flex_consumption.main.default_hostname`（他リソースから参照する場合のみ） |
| Storage Account のエンドポイント | `azurerm_storage_account.func.primary_blob_endpoint` / `primary_queue_endpoint` / `primary_table_endpoint` |
| User Assigned Identity の client_id | `azurerm_user_assigned_identity.main.client_id` |
| Application Insights の接続文字列・インストルメンテーションキー | `azurerm_application_insights.main.connection_string` / `instrumentation_key` |
| Log Analytics ワークスペース ID | `azurerm_log_analytics_workspace.main.workspace_id` |
| PostgreSQL / MySQL の FQDN | `azurerm_postgresql_flexible_server.main.fqdn` |
| Key Vault の URI | `azurerm_key_vault.main.vault_uri` |
| Container Registry のログインサーバー | `azurerm_container_registry.main.login_server` |

### 適用例

```hcl
# NG: Storage エンドポイントをハードコード
app_settings = {
  AzureWebJobsStorage__blobServiceUri = "https://prdsttfconvtestdata01.blob.core.windows.net"
  AzureWebJobsStorage__clientId       = "ff5d9c6b-3e55-44e4-a762-0a07ffbf9f31"
}

# OK: リソース参照を使用
app_settings = {
  AzureWebJobsStorage__blobServiceUri = azurerm_storage_account.func.primary_blob_endpoint
  AzureWebJobsStorage__clientId       = azurerm_user_assigned_identity.main.client_id
}
```

### クロスコンポーネント参照の場合

参照先リソースが別コンポーネントにある場合（例: `api` コンポーネントから `ope` コンポーネントの Application Insights を参照）は、`terraform_remote_state` 経由で参照先コンポーネントの output から取得する：

```hcl
# variables.tf に追加（remote_state の接続情報はコンポーネント・環境ごとに異なる）
variable "ope_remote_state_resource_group_name" {
  type        = string
  description = "ope コンポーネントの tfstate が格納されている Storage Account のリソースグループ名"
}

variable "ope_remote_state_storage_account_name" {
  type        = string
  description = "ope コンポーネントの tfstate が格納されている Storage Account 名"
}

# 同様に ope_remote_state_container_name と ope_remote_state_key も定義する

# main.tf に追加
data "terraform_remote_state" "ope" {
  backend = "azurerm"
  config = {
    resource_group_name  = var.ope_remote_state_resource_group_name
    storage_account_name = var.ope_remote_state_storage_account_name
    container_name       = var.ope_remote_state_container_name
    key                  = var.ope_remote_state_key
  }
}

app_settings = {
  APPLICATIONINSIGHTS_CONNECTION_STRING = data.terraform_remote_state.ope.outputs.app_insights_connection_string
}
```

参照先コンポーネントの `outputs.tf` に必要な値が出力されていることを確認する。`--backend local` の場合は `backend = "local"` と `path` を使用する：

```hcl
# --backend local の場合
data "terraform_remote_state" "ope" {
  backend = "local"
  config = {
    path = var.ope_remote_state_path
  }
}
```

remote_state の接続情報は環境ごとに異なるため、すべて `environments/<env>/<component>.tfvars` に定義する。

---

## main.tf の例

既存リソースの設定値を反映する。Terraform のデフォルト値と異なる設定は明示的に記述する。

**コメントのガイドライン**：運用・保守の観点から、以下の箇所にコメントを追加する：

- リソースブロックの直前：そのリソースの役割・用途を 1 行で説明する
- 非直感的な設定値（`false` にしている理由、特定の SKU を選択した理由など）：インラインコメントで補足する
- `azapi` を使用している理由
- 関連リソース間の依存関係が明示的でない場合

```hcl
# アプリケーション全体を収容する仮想ネットワーク
resource "azurerm_virtual_network" "main" {
  name                = var.vnet_name
  address_space       = var.vnet_address_space
  location            = var.location
  resource_group_name = var.resource_group_name
  tags                = local.common_tags
}

# Function App が使用するストレージアカウント（実行ログ・コード格納用）
resource "azurerm_storage_account" "func" {
  name                            = var.storage_account_name
  resource_group_name             = var.resource_group_name
  location                        = var.location
  account_tier                    = "Standard"       # StorageV2 では Standard 固定
  account_replication_type        = var.storage_account_replication_type
  allow_nested_items_to_be_public = false             # パブリックアクセスは禁止（セキュリティポリシー）
  tags                            = local.common_tags
}

# azurerm に対応タイプが存在しない場合のみ azapi を使用
# Note: No azurerm resource available for this resource type. Using azapi.
resource "azapi_resource" "example" {
  type      = "Microsoft.Example/resources@2024-01-01"
  name      = local.resource_name
  parent_id = var.resource_group_id
  body = {
    properties = {
      # 既存リソースの設定値
    }
  }
}
```

---

## import ブロック（import.tf）

aztfexport が生成した import ブロックをコンポーネントごとの `import.tf` に配置する。`id` 内の subscription ID とリソースグループ名は**必ず変数補間**を使用する。

### --import-rg が指定された場合のみ追加

```hcl
# ⚠️ リソースグループを Terraform 管理下に置きます。terraform destroy でグループごと削除されるリスクがあります。
import {
  id = "/subscriptions/${var.subscription_id}/resourceGroups/${var.resource_group_name}"
  to = azurerm_resource_group.main
}
```

対応する `main.tf` のリソースブロック：

```hcl
resource "azurerm_resource_group" "main" {
  name     = var.resource_group_name
  location = var.location
  tags     = local.common_tags

  lifecycle {
    prevent_destroy = true
  }
}
```

### 単一 RG の場合（リソース名が環境ごとに異なる場合は変数補間を使用）

```hcl
import {
  id = "/subscriptions/${var.subscription_id}/resourceGroups/${var.resource_group_name}/providers/Microsoft.Web/sites/${var.function_app_name}"
  to = azurerm_function_app_flex_consumption.func
}
```

全環境で固定のリソース名（例: `st-shared-config`）はハードコードで可。

### 複数 RG にまたがる場合

各リソースの所属 RG に対応する変数を補間する：

```hcl
# front RG のリソース
import {
  id = "/subscriptions/${var.subscription_id}/resourceGroups/${var.resource_group_name_front}/providers/Microsoft.Web/staticSites/stapp-myapp-01"
  to = azurerm_static_web_app.main
}

# data RG のリソース
import {
  id = "/subscriptions/${var.subscription_id}/resourceGroups/${var.resource_group_name_data}/providers/Microsoft.DBforPostgreSQL/flexibleServers/psql-myapp-prd-01"
  to = azurerm_postgresql_flexible_server.main
}
```

リソース名が不明・確認できない場合は変数化しておく（後から固定化も容易）。

---

## 単一テンプレートモード（類似 RG グループ）の追加規則

STEP 1.5 で単一テンプレートモードが選択されたグループに対して適用する。

### 環境別 tfvars の生成

グループ内の各 RG から実際の設定値を取得し、**同一の変数セット・異なる値**で環境別 tfvars をそれぞれ生成する。

```hcl
# environments/prd/api.tfvars（テンプレート RG: prd-rg-system-01 から生成）
resource_group_name   = "prd-rg-system-01"
location              = "japaneast"
env                   = "prd"
function_app_name     = "func-myapp-prd-01"
app_service_sku_name  = "P1v3"
sql_database_sku_name = "GP_S_Gen5_2"

# environments/stg/api.tfvars（類似 RG: stg-rg-system-01 から実際の値を取得して生成）
resource_group_name   = "stg-rg-system-01"
location              = "japaneast"
env                   = "stg"
function_app_name     = "func-myapp-stg-01"
app_service_sku_name  = "B2"
sql_database_sku_name = "GP_S_Gen5_1"
```

すべての tfvars ファイルの変数名は `variables.tf` と完全に一致させる。

### `--backend azblob` 使用時の terraform.tf

単一テンプレートモードでは `key` を init 時に外部から渡すため、`terraform.tf` の backend ブロックから `key` を省略する：

```hcl
backend "azurerm" {
  resource_group_name  = "<STEP 0.5 で収集した値>"
  storage_account_name = "<STEP 0.5 で収集した値>"
  container_name       = "<STEP 0.5 で収集した値>"
  # key は terraform init -backend-config="key=<component>-<env>.tfstate" で環境ごとに指定する
}
```

### `--backend hcp` 使用時の terraform.tf

単一テンプレートモードでは Workspace 名を環境ごとに切り替えるため、`workspaces` ブロックの `name` を省略する：

```hcl
cloud {
  organization = "<STEP 0.5 で収集した organization 名>"

  workspaces {
    # name は省略し、terraform init 実行前に環境変数 TF_WORKSPACE で指定する
    # 例: TF_WORKSPACE=<prefix>-<component>-stg terraform init
    tags = ["<prefix>-<component>"]
  }
}
```
