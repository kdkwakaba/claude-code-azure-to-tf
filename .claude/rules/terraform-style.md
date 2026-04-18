# Terraform コードスタイル

## ファイル構成

各 Terraform モジュールは以下のファイルに論理的に分割する:

| ファイル | 用途 |
|---------|------|
| `main.tf` | コアリソース定義 |
| `variables.tf` | 入力変数 |
| `outputs.tf` | 出力値 |
| `terraform.tf` | プロバイダーおよびバックエンド設定 |
| `locals.tf` | ローカル値と計算式 |
| `*.tfvars` | 環境別パラメーター値 |

`main.tf` や `variables.tf` が大きくなった場合はリソース種別ごとにファイルを分割する（例: `main.networking.tf`、`main.storage.tf`）。

## 命名規則

- 変数名・モジュール名・ローカル値はすべて `snake_case` を使用する
- Azure リソース名は [Azure 命名規則](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/azure-best-practices/resource-naming) に従う

## 変数定義

- `type` と `description` を必ず明示する
- 機密変数には `sensitive = true` を設定する（`sensitive = false` の明示は避ける）
- コレクション型のデフォルト値は原則 nullable にしない

## ローカル値

複雑な式・繰り返し出現する式・共通タグは `locals.tf` に抽出する:

```hcl
locals {
  resource_prefix = "<プロジェクト名>"

  # 既存リソースのタグ構造をそのまま複製する
  # 新しいタグキー（ManagedBy など）は追加しない
  common_tags = {
    env       = var.env_tag
    component = "<コンポーネント固定値>"
  }
}
```

既存 Azure リソースのタグをインポートする場合は、既存のタグ構造・キー名・値をそのまま使用する。Terraform 管理であることを示す新しいタグ（`ManagedBy = "terraform"` など）は追加しない。

## 出力

- 他の構成で実際に使用する値のみ出力する
- すべての出力に `description` を付ける
- 機密を含む出力には `sensitive = true` を設定する

## 反復処理・動的ブロック

- 0–1 個のリソースには `count`、複数リソースには map を使った `for_each` を使用する
- オプションのネストブロックには `dynamic` ブロックを使い、`coalesce` や `try` でデフォルト値を扱う

## 依存関係

- 暗黙の参照で依存が確立されている場合は `depends_on` を削除する
- モジュール出力への `depends_on` は避ける

## プロバイダーの選択（既存リソースのインポート時）

既存 Azure リソースを Terraform 管理下に置く場合：

1. **`azurerm` プロバイダーを優先して使用する**：対応するリソースタイプが存在する場合は常に `azurerm` を使用する
2. **`azapi` プロバイダーはフォールバックとして使用する**：`azurerm` に対応するリソースタイプが存在しない場合のみ `azapi_resource` を使用し、その旨をコメントで明示する
3. **AVM（Azure Verified Modules）は使用しない**（既存リソースの import 時に意図しない設定変更・リソース追加生成が発生するため）

## フォーマット

`terraform fmt -recursive` で常に一貫したフォーマットを維持する。
