# Phase 2: コード生成サブエージェント手順

`/azure-to-tf` コマンドの Phase 2 サブエージェントが読み込む手順書。

## 最初に読み込むファイル

1. `<classification_json_path>` を Read ツールで読む（コンポーネント構成・オプションを把握する）
2. `.claude/rules/aztf-templates.md` を Read ツールで読む（テンプレートと生成ルールを確認する）
3. `<work_dir>/summary.txt` を Read ツールで読む（リソース一覧の確認）

---

## ディレクトリ構造の作成

`output_dir/` 以下を作成する：

- コンポーネントごとのディレクトリ（classification.json の `components` キー一覧）
- `modules` が空でない場合のみ `modules/` ディレクトリ（モジュール名サブディレクトリ含む）
- `environments/<detected_env>/`（単一テンプレートモードの場合は `similar_rg_groups` の全環境分）

---

## .tf ファイルの生成

aztf-templates.md のテンプレートと規則に従い、各コンポーネントに以下を生成する：
`main.tf` / `variables.tf` / `outputs.tf` / `terraform.tf` / `locals.tf` / `import.tf`

各リソースの属性値は生 HCL を grep で部分参照する（ファイル全体は読まない）：

```bash
grep -A 50 'resource "<type>" "<label>"' <work_dir>/<source_rg>/main.tf
```

`terraform.tf` の backend ブロックは classification.json の `backend` と `backend_config` の値を使って aztf-templates.md のテンプレートに従い生成する。

---

## --import-rg の処理

`import_rg` が `true` の場合のみ：

- `import.tf` に `azurerm_resource_group.main` の import ブロックを追加する
- `main.tf` に `lifecycle { prevent_destroy = true }` を追加する
- 返却内容に以下の警告を含める：「⚠️ --import-rg が指定されています。terraform destroy でリソースグループごと削除されるリスクがあります」

---

## secret.tfvars の自動生成

`sensitive = true` の変数を含むコンポーネントがある場合のみ：

- `environments/<detected_env>/<component>.secret.tfvars` を生成する（変数名・ガイドコメントのみ、値は空文字列）
- `.gitignore` に `*.secret.tfvars` が未登録の場合は追記する

---

## environments/\<env\>/\<component\>.tfvars の生成

- 既存リソースの実際の設定値を使用する（推奨 SKU への変更は行わない）
- `subscription_id` は tfvars に記載しない（ARM_SUBSCRIPTION_ID 環境変数から取得）
- 複数 RG にまたがる場合は識別子付き変数を列挙する（`resource_group_name_front` など）
- 単一テンプレートモードの場合：`similar_rg_groups` の各 RG から設定値を取得して全環境分の tfvars を生成する

```bash
az resource list --resource-group <env-rg> --query "[].{name:name,type:type,sku:sku,kind:kind}" -o json
```

---

## import ブロックの検証

生成した各 `import.tf` を確認し、subscription ID とリソースグループ名が変数補間（`${var.subscription_id}` / `${var.resource_group_name}`）になっていることを検証する。

---

## terraform validate の実行

全コンポーネントで以下を順次実行する（並列実行しない）：

```bash
export TF_PLUGIN_CACHE_DIR="$HOME/.terraform.d/plugin-cache"
mkdir -p "$TF_PLUGIN_CACHE_DIR"
cd <output_dir>/<component>
terraform init -backend=false
terraform validate
```

`terraform init -backend=false` は validate 専用。`.terraform.lock.hcl` はユーザーが terraform init（本番用）を実行すると上書きされるため問題ない。

---

## 返却内容

Phase 2 完了後、以下をメイン会話に返却する：

- 生成したコンポーネント一覧と各コンポーネントの .tf ファイル数
- validate の結果（エラーがあれば内容と該当ファイル）
- secret.tfvars が必要なコンポーネントの一覧
- 単一テンプレートモードの場合は生成した環境一覧
- `--import-rg` が true の場合は警告メッセージ
- 実行時情報（コンポーネント名・環境名・Backend 種別）を使った `terraform init` / `plan` の手順案内：
  - プラグインキャッシュ設定（`TF_PLUGIN_CACHE_DIR`）を初回のみ実行するよう案内する
  - 複数コンポーネントの並列実行は禁止。推奨実行順（network → ope → data → api → front）を明示する
  - secret.tfvars が存在するコンポーネントのみ `-var-file` を追加する
  - 単一テンプレートモードの場合は Backend 種別ごとのステート分離方法を含める
    - `local`：`terraform workspace new <env>` でワークスペースを環境ごとに作成
    - `azblob`：`terraform init -reconfigure -backend-config="key=<component>-<env>.tfstate"`
    - `hcp`：HCP Terraform ポータルでの Workspace 作成を案内
  - plan 結果の期待値（`X to import, 0 to add, 0 to change, 0 to destroy`）と差分発生時の対応を伝える
