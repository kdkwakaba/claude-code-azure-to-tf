# /azure-to-tf — 既存 Azure リソースを Terraform テンプレートに変換する

既存の Azure リソースグループを `aztfexport` でエクスポートし、azurerm/azapi ベースの Terraform コードに変換・インポートして Terraform 管理下に置くまでの一連の作業を実行する。

## 使い方

```
/azure-to-tf --resource-group <rg-name> [--subscription <sub-id>] [--backend local|azblob|hcp]
```

| オプション | 説明 | デフォルト |
|-----------|------|-----------|
| `--resource-group` | エクスポート対象のリソースグループ名 | （必須） |
| `--subscription` | Azure サブスクリプション ID | `ARM_SUBSCRIPTION_ID` 環境変数を使用 |
| `--backend` | Terraform Backend の種類（`local` / `azblob` / `hcp`） | `local` |

引数が省略された場合はユーザーに対話形式で確認する。

---

## STEP 0: 前提条件チェック

以下をすべて確認し、満たされていない場合は処理を停止してユーザーに案内する。

```bash
# 必須環境変数
echo $ARM_CLIENT_ID
echo $ARM_CLIENT_SECRET
echo $ARM_TENANT_ID

# 必須ツール
aztfexport version
terraform version
az version
```

- `ARM_CLIENT_ID` / `ARM_CLIENT_SECRET` / `ARM_TENANT_ID` のいずれかが未設定の場合、設定方法を案内して停止する
- ツールが見つからない場合はインストール方法を案内して停止する

### Backend 別の追加チェック

**`--backend azblob` の場合**

```bash
echo $ARM_SUBSCRIPTION_ID  # 未設定の場合はユーザーに確認
```

- `ARM_SUBSCRIPTION_ID` が未設定の場合は設定方法を案内する
- Backend 用ストレージアカウントの詳細（RG名・SA名・コンテナー名）を STEP 12 で入力するよう案内する

**`--backend hcp` の場合**

```bash
echo $HCP_CLIENT_ID
echo $HCP_CLIENT_SECRET
# または
echo $TFE_TOKEN
```

- `HCP_CLIENT_ID` + `HCP_CLIENT_SECRET` または `TFE_TOKEN` のいずれかが設定されていない場合、案内して停止する
- HCP Terraform の Organization 名・Workspace 名を STEP 12 で入力するよう案内する

---

## STEP 1: Azure ログイン

```bash
az login --service-principal \
  --username "$ARM_CLIENT_ID" \
  --password "$ARM_CLIENT_SECRET" \
  --tenant "$ARM_TENANT_ID"

# サブスクリプション指定がある場合
az account set --subscription "<sub-id>"

# ログイン確認
az account show
```

ログインに失敗した場合は処理を停止する。

---

## STEP 2: aztfexport でリソースをエクスポート

```bash
WORK_DIR="/tmp/aztfexport-$(date +%s)"
mkdir -p "$WORK_DIR"
cd "$WORK_DIR"

aztfexport resource-group "<rg-name>" --non-interactive
```

- `--non-interactive` フラグで対話なしに実行する
- エクスポート失敗時はエラー内容を表示して停止する
- エクスポート完了後、生成されたファイル一覧をユーザーに提示する

---

## STEP 3: リソースのコンポーネント分類

エクスポートされた各リソースについて、以下の優先順位でコンポーネントを決定する。

### 分類の優先順位

1. **リソースのタグ**：`Component`・`Layer`・`Tier` タグが付いている場合はその値を使用する
2. **リソース名のパターン**：プレフィックス・サフィックスからコンポーネントを推定する
   - `-front`・`-web`・`-ui`・`-cdn`・`-static` → `front`
   - `-api`・`-func`・`-apim` → `api`
   - `-app`・`-svc`・`-worker`・`-bus`・`-queue` → `backend`
   - `-db`・`-sql`・`-cosmos`・`-redis`・`-pg` → `database`
3. **リソース種別のデフォルトマッピング**（下表参照）
4. **判断できない場合**：ユーザーに確認する

### リソース種別マッピング

| Azure リソース種別 | 分類先 |
|---|---|
| `Microsoft.Web/staticSites` | `front` |
| `Microsoft.Cdn/profiles`・`Microsoft.Network/frontDoors` | `front` |
| `Microsoft.Network/applicationGateways` | `front` |
| `Microsoft.Web/sites`（kind: functionapp） | `api` |
| `Microsoft.App/containerApps` | `api` |
| `Microsoft.ApiManagement/service` | `api` |
| `Microsoft.Web/sites`（kind: app） | `backend` |
| `Microsoft.ServiceBus/namespaces`・`Microsoft.EventHub/namespaces` | `backend` |
| `Microsoft.Storage/storageAccounts` | `backend`（静的コンテンツ用途なら `front`） |
| `Microsoft.Sql/servers`・`Microsoft.Sql/servers/databases` | `database` |
| `Microsoft.DocumentDB/databaseAccounts` | `database` |
| `Microsoft.DBforPostgreSQL/flexibleServers` | `database` |
| `Microsoft.DBforMySQL/flexibleServers` | `database` |
| `Microsoft.Cache/Redis` | `database` |
| `Microsoft.KeyVault/vaults` | コンテキストで判断・不明時はユーザーに確認 |
| `Microsoft.Network/virtualNetworks`・`Microsoft.Network/networkSecurityGroups` | コンテキストで判断・不明時はユーザーに確認 |
| `Microsoft.OperationalInsights/workspaces`・`Microsoft.Insights/*` | コンテキストで判断・不明時はユーザーに確認 |

---

## STEP 4: プロバイダーの選択

AVM モジュールは使用しない。以下の優先順位でリソースを実装する：

1. **`azurerm` プロバイダー**：対応するリソースタイプが存在する場合は常に `azurerm` を使用する
2. **`azapi` プロバイダー**：`azurerm` に対応するリソースタイプが存在しない場合のみ使用する

`azapi` を使用するリソースの直前には以下のコメントを追加する：

```hcl
# Note: No azurerm resource available for this resource type. Using azapi.
```

### terraform.tf の required_providers

使用するプロバイダーに応じて `required_providers` を設定する：

```hcl
# azurerm のみ使用する場合
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 4.0"
    }
  }

  backend "local" {}
}

provider "azurerm" {
  features {}
  subscription_id = var.subscription_id
}
```

```hcl
# azapi も必要な場合（azurerm に対応タイプが存在しないリソースがある場合のみ追加）
required_providers {
  azurerm = {
    source  = "hashicorp/azurerm"
    version = "~> 4.0"
  }
  azapi = {
    source  = "Azure/azapi"
    version = "~> 2.0"
  }
}
```

---

## STEP 5: プロジェクト構造の作成とコード生成

### ディレクトリ構造の作成

分類結果に基づき、リソースが存在するコンポーネントのみ作成する：

```
infra/
├── front/
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   ├── terraform.tf
│   └── locals.tf
├── api/
│   └── ...
├── backend/
│   └── ...
├── data/
│   └── ...
└── environments/
    └── production.tfvars   # 検出した環境のみ作成
```

### terraform.tf（Backend 別テンプレート）

STEP 4 のテンプレートを使用する。`--backend` オプションに応じて backend ブロックを切り替える。

#### `--backend azblob`

```hcl
backend "azurerm" {
  resource_group_name  = "<入力値>"
  storage_account_name = "<入力値>"
  container_name       = "<入力値>"
  key                  = "<component>.tfstate"
}
```

#### `--backend hcp`

```hcl
cloud {
  organization = "<organization>"

  workspaces {
    name = "<project>-<component>"
  }
}
```

### variables.tf（共通変数）

```hcl
variable "subscription_id" {
  type        = string
  description = "Azure サブスクリプション ID"
}

variable "resource_group_name" {
  type        = string
  description = "リソースグループ名"
}

variable "location" {
  type        = string
  description = "Azure リージョン"
}

# リソース種別固有の変数（aztfexport の出力から抽出）
```

### パラメーター化の対象

以下のような環境ごとに変更が予測される値は、固定値でなく変数として定義する：

| 対象 | 変数例 |
|---|---|
| リソース名（環境名・プレフィックス含む） | `function_app_name`、`sql_server_name` |
| SKU・価格レベル | `app_service_sku_name`、`sql_database_sku_name` |
| スケーリング設定（インスタンス数・容量等） | `app_service_worker_count`、`redis_cache_capacity` |
| 付属ストレージアカウント名・SKU | `storage_account_name`、`storage_account_replication_type` |
| その他の付属リソース（サービスプラン等）の SKU | `service_plan_sku_name` |

これらは `variables.tf` に定義し、実際の値は `environments/*.tfvars` に記載する。

### locals.tf（タグ・名前の管理）

**既存リソースのタグをそのまま複製する。新しいタグキーは追加しない。**
タグがないリソースには `tags` 引数を設定しない。

```hcl
locals {
  resource_prefix = "<aztfexport から抽出した実際のプレフィックス>"

  # 既存リソースのタグ構造をそのまま複製する
  # 例: 既存タグが { env = "prd", component = "api" } の場合
  common_tags = {
    env       = var.env_tag
    component = "<コンポーネント固定値>"
  }
}
```

タグが環境依存の値を含む場合（例: `env = "prd"`）は変数化する。それ以外は固定値でよい。

### main.tf の例

既存リソースの設定値を反映する。Terraform のデフォルト値と異なる設定は明示的に記述する。

**コメントのガイドライン**：運用・保守の観点から、以下の箇所にコメントを追加する：

- リソースブロックの直前：そのリソースの役割・用途を 1 行で説明する
- 非直感的な設定値（`false` にしている理由、特定の SKU を選択した理由など）：インラインコメントで補足する
- `azapi` を使用している理由（後述）
- 関連リソース間の依存関係が明示的でない場合

```hcl
# Function App が使用するストレージアカウント（実行ログ・コード格納用）
resource "azurerm_storage_account" "func" {
  name                            = var.storage_account_name
  resource_group_name             = var.resource_group_name
  location                        = var.location
  account_tier                    = "Standard"
  account_kind                    = "StorageV2"
  account_replication_type        = var.storage_account_replication_type # 環境により LRS/GRS を切り替え
  allow_nested_items_to_be_public = false # パブリックアクセスは禁止（セキュリティポリシー）

  tags = local.common_tags
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

### import ブロック（環境変数を使用）

aztfexport が生成した import ブロックをコンポーネントごとの `import.tf` に配置する。
`id` 内の subscription ID とリソースグループ名は**必ず変数補間**を使用し、環境ごとに切り替えられるようにする：

```hcl
import {
  id = "/subscriptions/${var.subscription_id}/resourceGroups/${var.resource_group_name}/providers/Microsoft.Web/sites/func-myapp-01"
  to = azurerm_function_app_flex_consumption.func
}
```

#### リソース名が環境ごとに異なる場合

リソース名に環境名・プレフィックス・サフィックスが含まれ環境ごとに変わる場合は、変数に抽出して補間する：

```hcl
# variables.tf に追加
variable "function_app_name" {
  type        = string
  description = "Function App 名"
}

# import.tf
import {
  id = "/subscriptions/${var.subscription_id}/resourceGroups/${var.resource_group_name}/providers/Microsoft.Web/sites/${var.function_app_name}"
  to = azurerm_function_app_flex_consumption.func
}

# main.tf のリソース定義も同じ変数を参照する
resource "azurerm_function_app_flex_consumption" "func" {
  name                = var.function_app_name
  resource_group_name = var.resource_group_name
  ...
}
```

#### リソース名が全環境で共通の場合

リソース名が変わらない場合は補間不要（ハードコードで可）：

```hcl
import {
  id = "/subscriptions/${var.subscription_id}/resourceGroups/${var.resource_group_name}/providers/Microsoft.Storage/storageAccounts/stsharedcontent01"
  to = azurerm_storage_account.shared
}
```

#### 判断基準

| リソース名のパターン | 対応 |
|---|---|
| 環境名・プレフィックスを含む（例: `func-myapp-prd-01`） | 変数化して補間 |
| 全環境で固定（例: `st-shared-config`） | ハードコードで可 |
| 不明・確認できない | 変数化しておく（後から固定化も容易） |

---

## STEP 6: 環境別 tfvars の生成

### 環境の検出

既存リソースのタグから環境を特定し、**その環境の tfvars 1 ファイルのみ**作成する：

| タグ値 | 作成するファイル |
|---|---|
| `env = "prd"` / `environment = "production"` | `environments/production.tfvars` |
| `env = "stg"` / `environment = "staging"` | `environments/staging.tfvars` |
| `env = "dev"` / `environment = "development"` | `environments/development.tfvars` |
| タグなし・判断不可 | ユーザーに確認する |

### tfvars の値

**既存リソースの実際の設定値を使用する。ベストプラクティスの SKU 推奨値には変更しない。**

```hcl
# environments/production.tfvars（例）
subscription_id     = "<既存リソースの subscription_id>"
resource_group_name = "<既存リソースグループ名>"
location            = "<既存リソースの location>"
env_tag             = "prd"           # 既存タグ値

# 環境ごとに異なるリソース名（import.tf で変数補間しているもの）
function_app_name     = "func-myapp-prd-01"
sql_server_name       = "sqldb-myapp-prd-01"

# 既存リソースの実際の設定値をそのまま反映
sql_database_sku_name = "GP_S_Gen5_1"   # 既存の SKU をそのまま使用
```

複数環境がある場合は環境ごとに tfvars を用意し、同じ変数に異なる値を設定する：

```hcl
# environments/staging.tfvars（例）
subscription_id     = "<stg の subscription_id>"
resource_group_name = "rg-myapp-stg"
location            = "japaneast"
env_tag             = "stg"

function_app_name     = "func-myapp-stg-01"
sql_server_name       = "sqldb-myapp-stg-01"
sql_database_sku_name = "GP_S_Gen5_1"
```

### 追加環境の tfvars が必要な場合

ユーザーから追加環境（staging・development など）への展開を求められた場合：

1. ユーザーに確認する：
   ```
   <environment> 環境の tfvars を作成します。
   対象リソースグループ名を教えてください。
   （未作成の場合は「なし」と回答してください）
   ```
2. **リソースグループ名が提供された場合**：`az resource list --resource-group <rg>` で該当 RG の既存リソース設定値を取得してから tfvars を作成する
3. **「なし」の場合**：既存環境の値をベースにコメントとして変更候補を記載した雛形を作成し、ユーザーに確認を求める

---

## STEP 7: terraform fmt（Hook により自動実行）

`.tf` ファイルの書き込み・編集のたびに `settings.json` の Hook が自動的に `terraform fmt` を実行する。
このステップでの手動実行は不要。

---

## STEP 8: import ブロックの確認

各コンポーネントの `import.tf` を確認し、import ブロックが正しく生成されていることを確認する。

> **注意**: `terraform init` はここでは実行しない。`terraform init` を Claude Code 側で実行すると、実行環境に依存した `.terraform.lock.hcl` が生成され、ユーザーが別環境で `terraform init` を実行した際にプロバイダーの checksum が一致せずエラーになる。`terraform init` と `terraform plan` はユーザーが手動で実行すること。

`terraform plan` が import ブロックを検出し、既存リソースのインポートを処理する。

### import 不可リソースの処理

`terraform plan` でエラーになったリソースは以下の手順で処理する：

1. エラーとなったリソースの import ブロックを `main.tf` から削除する
2. 以下の情報をユーザーに報告する：
   ```
   [import スキップ] 以下のリソースは import できませんでした：
   - リソース名: <name>
   - リソース種別: <type>
   - 理由: <error message>
   手動での対応が必要です。
   ```
3. ユーザーの確認を取ってから続行する

---

## STEP 9: コード生成の最終確認

すべての `.tf` ファイルの生成が完了した後、以下を確認する：

- 各コンポーネントに必要なファイル（`main.tf`、`variables.tf`、`outputs.tf`、`terraform.tf`、`locals.tf`）が揃っているか
- `import.tf` の import ブロックが正しく生成されているか
- `variables.tf` に定義した変数が `environments/*.tfvars` に対応する値として記載されているか

> **注意**: `terraform validate` は `terraform init` が完了していないと実行できないため、ここでは実行しない。ユーザーが `terraform init` を実行した後に手動で確認する。

生成完了後、ユーザーに以下を案内する：

```
Terraform コードの生成が完了しました。
次の手順で初期化・検証・インポートを行ってください：

  cd infra/<component>
  terraform init
  terraform validate
  terraform plan -var-file="../../environments/<detected_env>.tfvars"

plan の結果に "X to import, 0 to add, 0 to change, 0 to destroy" と表示されれば想定通りです。
差分が発生した場合はその内容を共有してください。
```

---

## STEP 10: plan 結果のレビュー支援

ユーザーが `terraform plan` を実行し、その結果を共有してきた場合に内容をレビューする。

### 期待する結果

`No changes` または import のみ（`X to import, 0 to add, 0 to change, 0 to destroy`）が理想。

### 差分が発生した場合

差分が発生した場合は内容をユーザーに提示し、確認を求める。以下は対応方針：

| 差分の種類 | 対応 |
|---|---|
| タグの追加・変更 | Terraform コードを既存タグ構造に合わせて修正する |
| 設定値の変更（SKU・フラグ等） | Terraform コードに既存設定値を明示して差分を解消する |
| `destroy` または `replace` | 処理を停止し、ユーザーに原因を報告して確認を取る |
| `add`（import 以外の新規作成） | 不要なリソース定義がないか確認し、ユーザーに報告する |

---

## STEP 11: apply（ユーザー確認必須）

**STEP 10 でユーザーの承認を得た後にのみ実行する。**

```bash
cd infra/<component>
terraform apply -var-file="../../environments/<detected_env>.tfvars"
```

すべてのコンポーネントの apply が成功したら STEP 12 に進む。

---

## STEP 12: Backend の設定・移行

`--backend` オプションの値に応じて以下の処理を行う。

---

### `--backend local`（デフォルト）

ローカル Backend のまま完了とする。必要に応じてリモート Backend への移行を提案する：

```
現在はローカル Backend を使用しています。
リモート Backend（azblob または hcp）への移行が必要な場合は /azure-to-tf を
--backend azblob または --backend hcp オプション付きで再実行してください。
```

---

### `--backend azblob`

対話形式でストレージアカウント情報を収集する：

```
Azure Blob Storage Backend の設定を入力してください。

? リソースグループ名（Backend ストレージが存在する RG）:
? ストレージアカウント名:
? コンテナー名（デフォルト: tfstate）:
```

入力値を STEP 5 で生成した `terraform.tf` の `backend "azurerm"` ブロックに反映する。

`terraform.tf` の更新後、以下をユーザーに案内する（Claude Code 側では実行しない）：

```
terraform.tf の backend 設定を更新しました。
以下のコマンドを手動で実行してください：

  cd infra/<component>
  terraform init          # 新規の場合
  # または
  terraform init -migrate-state   # --backend local から移行する場合
```

- `--backend local` から変更した場合（再実行時）は `-migrate-state` を付けるよう案内し、移行成功後にローカルの `terraform.tfstate` が不要になる旨を通知する（削除はユーザー判断に委ねる）

---

### `--backend hcp`

対話形式で HCP Terraform 情報を収集する：

```
HCP Terraform Backend の設定を入力してください。

? Organization 名:
? Workspace 名プレフィックス（デフォルト: <project-name>）:
  ※ コンポーネントごとに <prefix>-front / <prefix>-api などが作成されます
```

HCP Terraform ポータルで各コンポーネントの Workspace が存在しない場合は事前に作成するよう案内する：

```
以下の Workspace を HCP Terraform ポータルで作成してください：
- <prefix>-front
- <prefix>-api
- <prefix>-backend
- <prefix>-database
（存在するコンポーネントのみ）
```

`terraform.tf` の更新後、以下をユーザーに案内する（Claude Code 側では実行しない）：

```
terraform.tf の backend 設定を更新しました。
以下のコマンドを手動で実行してください：

  cd infra/<component>
  terraform login   # 未ログインの場合のみ
  terraform init
```

---

## エラーハンドリング早見表

| 状況 | 対応 |
|---|---|
| 環境変数未設定 | 処理停止・設定方法を案内 |
| ツール未インストール | 処理停止・インストール方法を案内 |
| Azure ログイン失敗 | 処理停止・エラー内容を表示 |
| aztfexport 失敗 | 処理停止・エラー内容を表示 |
| コンポーネント分類不明 | ユーザーに確認して続行 |
| 環境検出不可 | ユーザーに確認して続行 |
| import 不可リソース | ユーザーに報告・スキップして続行 |
| plan で差分発生 | 内容をユーザーに提示・Terraform コードを修正して解消を試みる |
| plan で意図しない destroy/replace | 処理停止・ユーザーに確認 |
| validate エラー（ユーザーが実行後に報告） | 内容を確認してコードを修正しユーザーに案内 |
| apply 失敗 | 処理停止・エラー内容を表示 |
| Backend 移行失敗（azblob） | 処理停止・ローカル Backend のまま維持 |
| Backend 移行失敗（hcp） | 処理停止・ローカル Backend のまま維持 |
| HCP Terraform 認証エラー | `terraform login` の実行を案内して停止 |
| HCP Terraform Workspace 未作成 | ポータルでの Workspace 作成を案内して停止 |
