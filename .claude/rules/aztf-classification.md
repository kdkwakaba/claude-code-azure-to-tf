# STEP 3: リソースのコンポーネント分類（詳細）

`/azure-to-tf` コマンドの STEP 3 で参照する分類ロジック。

## 分類の優先順位

全 RG のエクスポート結果を統合し、各リソースについて以下の優先順位でコンポーネントを決定する。  
分類結果には必ず **所属 RG 名（`source_resource_group`）** を記録する。これは STEP 5 での変数生成・import ブロック構築に使用する。

1. **リソースのタグ**：`Component`・`Layer`・`Tier` タグが付いている場合はその値を使用する
2. **リソース名のパターン**：プレフィックス・サフィックスからコンポーネントを推定する
   - `-front`・`-web`・`-ui`・`-cdn`・`-static` → `front`
   - `-api`・`-func`・`-apim` → `api`
   - `-app`・`-svc`・`-worker`・`-bus`・`-queue` → `backend`
   - `-db`・`-sql`・`-cosmos`・`-redis`・`-pg` → `data`
3. **リソース種別のデフォルトマッピング**（下表参照）
4. **判断できない場合**：ユーザーに確認する

## リソース種別マッピング

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

## 単一テンプレートモード時の分類（類似 RG グループ）

STEP 1.5 で単一テンプレートモードが選択されたグループでは以下の方針で分類する：

- **分類対象**：テンプレート RG のリソースのみを対象とする
- **他 RG のリソース**：分類・コード生成には使用しない。STEP 6 での tfvars 値抽出（`az resource list` による設定値取得）にのみ使用する
- **`source_resource_group`**：テンプレート RG 名を記録する
- **リソース名のパターン認識**：環境識別子を含む RG 名・リソース名からコンポーネントを推定する際、環境識別子部分（`stg`・`prd` など）は除いてパターンマッチする

例: テンプレート RG が `prd-rg-system-01` で、Function App 名が `func-myapp-prd-01` の場合、
`-api`・`-func` サフィックスパターンにより `api` コンポーネントに分類する（`prd` は環境識別子として除外して判断）。
