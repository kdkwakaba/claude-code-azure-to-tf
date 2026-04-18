# Phase 1: 分析・分類サブエージェント手順

`/azure-to-tf` コマンドの Phase 1 サブエージェントが読み込む手順書。ユーザーへの確認が必要な場面では **AskUserQuestion ツール**を使用してください。

## 実行引数の受け取り

プロンプトに含まれる以下の引数を変数として保持する：

- `RG_LIST`：カンマ区切りのリソースグループ名リスト
- `SUBSCRIPTION`：Azure サブスクリプション ID（空の場合は ARM_SUBSCRIPTION_ID を使用）
- `BACKEND`：`local` / `azblob` / `hcp`
- `INCLUDE_DEFAULTS`：`true` / `false`
- `IMPORT_RG`：`true` / `false`
- `EXCLUDE_RESOURCE_TYPES`：追加除外するリソース種別（空の場合は追加なし）
- `OUTPUT_DIR`：Terraform ファイルの出力先ディレクトリ

---

## STEP 0: 前提条件チェック

以下をすべて確認し、満たされていない場合は処理を停止してユーザーに案内する。

以下を **1 コマンドずつ個別の Bash 呼び出し** で実行する。`&&` や `||` によるチェーン実行は行わない。各 printenv コマンドは exit code で判定する（0 = 設定あり、非 0 = 未設定）。値そのものはログに残さないため `>/dev/null` でリダイレクトする。

```bash
# 必須環境変数（1 行ずつ個別の Bash 呼び出しで実行する）
printenv ARM_CLIENT_ID >/dev/null
printenv ARM_CLIENT_SECRET >/dev/null
printenv ARM_TENANT_ID >/dev/null
```

```bash
# 必須ツール（1 行ずつ個別の Bash 呼び出しで実行する）
aztfexport -v
```

```bash
terraform version
```

```bash
az version
```

- `ARM_CLIENT_ID` / `ARM_CLIENT_SECRET` / `ARM_TENANT_ID` のいずれかが未設定の場合（exit code 非 0）、設定方法を案内して停止する
- ツールが見つからない場合はインストール方法を案内して停止する

### Backend 別の追加チェック

**`--backend azblob` の場合**

```bash
# exit code 0 なら設定あり、非 0 なら未設定
printenv ARM_SUBSCRIPTION_ID >/dev/null
```

**`--backend hcp` の場合**（1 行ずつ個別の Bash 呼び出しで実行する）

```bash
printenv HCP_CLIENT_ID >/dev/null
```

```bash
printenv HCP_CLIENT_SECRET >/dev/null
```

```bash
# または TFE_TOKEN でも可
printenv TFE_TOKEN >/dev/null
```

- `HCP_CLIENT_ID` + `HCP_CLIENT_SECRET` の両方、または `TFE_TOKEN` のいずれかが設定されていない場合（exit code 非 0）、案内して停止する

### プロバイダーバージョンの検出

STEP 4.6 の classification.json に含めるバージョン制約文字列を変数に保持する。**複合コマンドは使わず、以下のコマンドを 1 つずつ個別の Bash 呼び出しで実行し、サブエージェントが結果を判断して値を組み立てる。**

**1. Terraform レジストリへの到達性確認**

```bash
curl -s --max-time 5 -o /dev/null https://registry.terraform.io
```

- exit code 0（到達可能）の場合：`AZURERM_VERSION="~> 4.0"` / `AZAPI_VERSION="~> 2.0"` を採用し、以降の手順はスキップする
- 非 0（到達不可）の場合：ミラー環境と判断し、以降の手順を実行する

**2. ミラー側の azurerm バージョン一覧を取得**

```bash
ls ~/.terraform.d/mirror/registry.terraform.io/hashicorp/azurerm/
```

Read ツールでも可。取得した一覧からセマンティックバージョン形式（`^[0-9]+\.` で始まるもの）だけを抽出し、最大のバージョンを選択して `AZURERM_VERSION="= <version>"` に設定する。一覧が空・コマンド失敗時は `AZURERM_VERSION="~> 4.0"` をデフォルト値とする。

**3. ミラー側の azapi バージョン一覧を取得**

```bash
ls ~/.terraform.d/mirror/registry.terraform.io/azure/azapi/
```

同様に最大バージョンを選択して `AZAPI_VERSION="= <version>"` に設定する。空・失敗時は `AZAPI_VERSION="~> 2.0"` をデフォルト値とする。

ミラー環境を検出した旨と採用した `AZURERM_VERSION` / `AZAPI_VERSION` をユーザーに提示する。

---

## STEP 0.5: Backend 詳細の事前収集

`--backend local` 以外の場合のみ実行する。**AskUserQuestion ツール**で必要情報を収集し変数として保持する。

**`--backend azblob` の場合**：
- Backend 用ストレージアカウントの RG 名
- ストレージアカウント名
- コンテナー名（デフォルト: tfstate）

**`--backend hcp` の場合**：
- HCP Terraform Organization 名
- Workspace 名プレフィックス（デフォルト: プロジェクト名）

---

## STEP 1: Azure ログイン

`--password` にシークレットを直接渡すとプロセスリスト（`ps aux`）に平文で露出するため、環境変数経由でログインする（Azure CLI は `AZURE_CLIENT_SECRET` を自動参照する）。**以下のコマンドを 1 つずつ個別の Bash 呼び出しで実行する。**

```bash
export AZURE_CLIENT_ID="$ARM_CLIENT_ID"
```

```bash
export AZURE_CLIENT_SECRET="$ARM_CLIENT_SECRET"
```

```bash
export AZURE_TENANT_ID="$ARM_TENANT_ID"
```

```bash
az login --service-principal --username "$AZURE_CLIENT_ID" --tenant "$AZURE_TENANT_ID"
```

```bash
# ログイン後すぐにシェル変数からシークレットを除去する
unset AZURE_CLIENT_SECRET
```

```bash
# SUBSCRIPTION が指定されている場合のみ実行する
az account set --subscription "<SUBSCRIPTION>"
```

```bash
# ログイン確認
az account show
```

ログインに失敗した場合は処理を停止する。

---

## STEP 1.5: 類似 RG グループの検出

複数 RG が指定された場合のみ実行する。単一 RG の場合はスキップする。

各 RG 名から既知の環境識別子（`dev`, `development`, `stg`, `staging`, `uat`, `qa`, `test`, `prd`, `prod`, `production`、区切り文字 `-` を含む）を除去したベース名を算出し、ベース名が一致する RG を類似グループとしてまとめる。

検出対象のパターン：

| パターン | 例 |
|---|---|
| プレフィックス形式 `<env>-<base>` | `stg-rg-system-01`, `dev-rg-system-01` → ベース `rg-system-01` |
| サフィックス形式 `<base>-<env>` | `rg-system-01-stg`, `rg-system-01-dev` → ベース `rg-system-01` |
| 中間形式 `<prefix>-<env>-<suffix>` | `rg-myapp-prd-01`, `rg-myapp-dev-01` → ベース `rg-myapp-01` |

類似グループが 1 つも検出されなかった場合はスキップして通常フローに進む。

類似グループが検出された場合は **AskUserQuestion ツール**で確認する：

```
以下の RG は環境識別子のみが異なる類似グループとして検出されました：

  グループ 1（ベース: rg-system-01）
    stg → stg-rg-system-01
    dev → dev-rg-system-01

単一テンプレート＋環境別 tfvars として生成しますか？
  yes : 単一セットのテンプレートを生成し、environments/stg/ と environments/dev/ にそれぞれ tfvars を分割する
  no  : 環境ごとに別テンプレートを生成する（通常フロー）
```

`yes` の場合は **単一テンプレートモード**として処理する：

| 項目 | 方針 |
|---|---|
| テンプレート RG | グループ内で `prd`/`prod` 環境の RG を優先し、存在しない場合は最初の RG を使用する |
| コンポーネント分類 | テンプレート RG のリソースのみを基準に分類する |
| tfvars 生成 | 全環境の RG から設定値を取得して `environments/<env>/<component>.tfvars` を個別生成する |

---

## STEP 2: aztfexport でリソースをエクスポート

`aztfexport resource-group` は 1 回に 1 RG しか受け付けないため、**RG ごとに順次実行**する。

作業ディレクトリの作成は **2 つの個別の Bash 呼び出し** で実行する。`mktemp` でランダムなディレクトリ名を生成し、`chmod 700` で他プロセスからの読み取りを防ぐ（`date +%s` は秒単位で予測可能。`/tmp` はデフォルト 755 のため他プロセスから読み取れる）。

```bash
mktemp -d /tmp/aztfexport-XXXXXX
```

出力されたパスを `WORK_DIR` としてサブエージェントの変数に保持し、次のコマンドで使用する：

```bash
chmod 700 "<WORK_DIR の値>"
```

**`INCLUDE_DEFAULTS` が false（デフォルト）の場合**

除外リストファイルは **Write ツール**で作成する。パスは `$WORK_DIR/aztfexport-skip-defaults.txt`（$WORK_DIR は 700 パーミッションで保護済み。/tmp 直下の固定ファイル名は symlink attack や並列実行時の競合リスクがあるため使用しない）。

Write ツールに渡す内容：

```
azurerm_postgresql_flexible_server_configuration
azurerm_mysql_flexible_server_configuration
azurerm_mariadb_configuration
azurerm_postgresql_flexible_server_backup
azurerm_mysql_flexible_server_backup
azurerm_storage_container
azurerm_log_analytics_workspace_table_custom_log
azurerm_log_analytics_saved_search
azurerm_app_service_custom_hostname_binding
```

`EXCLUDE_RESOURCE_TYPES` が指定されている場合は、上記の 9 行に続けてユーザー指定の種別を 1 行ずつ追加したものを Write ツールで書き出す（bash の追記処理は行わない）。

除外リストファイルのパスは `$WORK_DIR/aztfexport-skip-defaults.txt`（以降 `<EXCLUDE_FILE>` と表記）。サブエージェントは `RG_LIST` をカンマで分割し、各 RG（以降 `<RG>` と表記）について以下のコマンドを **1 つずつ個別の Bash 呼び出し** で実行する。for ループや `&&` によるチェーンは使わない。

```bash
mkdir -p "<WORK_DIR>/<RG>"
```

```bash
aztfexport resource-group --output-dir "<WORK_DIR>/<RG>" --non-interactive --plain-ui --generate-import-block --continue --parallelism 30 --exclude-terraform-resource-file "<EXCLUDE_FILE>" "<RG>"
```

この 2 コマンドを RG_LIST の要素数だけ繰り返す。

**`INCLUDE_DEFAULTS` が true の場合**

同様に各 RG について以下を **1 つずつ個別の Bash 呼び出し** で実行する：

```bash
mkdir -p "<WORK_DIR>/<RG>"
```

```bash
aztfexport resource-group --output-dir "<WORK_DIR>/<RG>" --non-interactive --plain-ui --generate-import-block --continue --parallelism 30 "<RG>"
```

全 RG のエクスポート完了後、RG ごとのリソース数と合計をユーザーに提示する。いずれかの RG でエクスポートが失敗した場合はエラー内容を表示して停止する。

### エクスポート後リソース要約の生成

aztfexport の生 HCL はリソース数に比例して大きくなるため、要約ファイル `$WORK_DIR/summary.txt` を生成する。以降の STEP では原則このファイルを参照する。**パイプを使わず、Grep ツールと Write ツールの組み合わせで実行する。**

1. **Grep ツール**で `<WORK_DIR>` を再帰検索する：
   - `pattern`: `^resource "|^import \{|^\s+(id|to)\s*=`
   - `path`: `<WORK_DIR>`
   - `output_mode`: `content`
   - `-n`: `false`
   - `head_limit`: `0`（全件取得）
2. Grep の結果から `<ファイルパス>:` プレフィックスおよび行頭の空白を除去する（サブエージェント側で文字列処理を行う）
3. 整形後の内容を **Write ツール** で `<WORK_DIR>/summary.txt` に書き出す

---

## STEP 3: リソースのコンポーネント分類

`$WORK_DIR/summary.txt` を Read ツールで読んでリソース種別・ラベルを把握する。aztfexport の生 HCL ファイルは読まない。

分類ロジックの詳細（優先順位・リソース種別マッピング表・名前パターン定義）は `.claude/rules/aztf-classification.md` を Read ツールで読んで参照すること。

分類結果には必ず **所属 RG 名（`source_resource_group`）** を記録する。

分類が判断できないリソースがある場合は **AskUserQuestion ツール**でユーザーに確認して続行する。

**単一テンプレートモードの場合**：テンプレート RG のリソースのみを分類対象とする。同一グループの他 RG リソースは STEP 4.6 の similar_rg_groups に記録し、Phase 2 の tfvars 値抽出にのみ使用する。

---

## STEP 4: プロバイダーの選択

AVM モジュールは使用しない。以下の優先順位でリソースを実装する：

1. **`azurerm` プロバイダー**：対応するリソースタイプが存在する場合は常に `azurerm` を使用する
2. **`azapi` プロバイダー**：`azurerm` に対応するリソースタイプが存在しない場合のみ使用する

`azapi` を使用するリソースがある場合は classification.json の `needs_azapi` を `true` にする。

---

## STEP 4.5: 類似リソースのモジュール化

STEP 3 の分類結果を横断し、以下のいずれかに該当するリソースを特定してモジュール化を検討する（繰り返しが 1 箇所のみ、または設定が大きく異なる場合はモジュール化しない）：

- 同一リソース種別が 2 コンポーネント以上で使用され、設定の大半が共通
- 同一コンポーネント内で同一種別のリソースが 3 つ以上あり、差異がパラメーターのみ
- コンポーネントをまたいで必ず一緒に出現するリソース群（サブネット＋NSG など）

優先的に確認するリソース種別：`azurerm_monitor_diagnostic_setting` / `azurerm_private_endpoint` / `azurerm_private_dns_zone` + `azurerm_private_dns_zone_virtual_network_link` / `azurerm_subnet` + `azurerm_network_security_group` + `azurerm_subnet_network_security_group_association` / `azurerm_role_assignment`

候補が見つかった場合は **AskUserQuestion ツール**で「モジュール名・対象リソース種別・対象コンポーネント・共通パラメーター・差異パラメーター」を列挙してユーザーに確認する（yes / no / 一部のみ）。

モジュール名はリソースの**役割**をスネークケースで表現する：`diagnostic_setting` / `private_endpoint` / `subnet_with_nsg` / `private_dns_zone`

モジュールのファイル構成：`modules/<module-name>/` 以下に `main.tf` / `variables.tf` / `outputs.tf` の 3 ファイル（README.md は生成しない）。

---

## STEP 4.6: 分類結果のファイル書き出し

STEP 0-4.5 の結果を `$WORK_DIR/classification.json` に書き出す：

```json
{
  "output_dir": "<OUTPUT_DIR の値>",
  "backend": "<BACKEND の値>",
  "backend_config": {},
  "import_rg": false,
  "include_defaults": false,
  "work_dir": "<$WORK_DIR の絶対パス>",
  "single_template_mode": false,
  "components": {
    "front": ["azurerm_static_web_app.main"],
    "api":   ["azurerm_function_app_flex_consumption.func", "azurerm_storage_account.func"],
    "data":  ["azurerm_postgresql_flexible_server.main", "azurerm_key_vault.main"]
  },
  "source_resource_groups": {
    "front": "rg-myapp-prd-front",
    "api":   "rg-myapp-prd-api",
    "data":  "rg-myapp-prd-data"
  },
  "modules": ["diagnostic_setting", "private_endpoint"],
  "needs_azapi": false,
  "azurerm_version": "<AZURERM_VERSION>",
  "azapi_version": "<AZAPI_VERSION>",
  "detected_env": "prd",
  "similar_rg_groups": []
}
```

`backend_config` には STEP 0.5 で収集した Backend 設定値を格納する（`--backend local` の場合は空オブジェクト）。

---

## 返却内容

Phase 1 完了後、以下をメイン会話に返却する：

- `work_dir`：`$WORK_DIR` の絶対パス（Phase 2 サブエージェントへの引き渡しに使用）
- 処理した RG 一覧とリソース数
- 検出したコンポーネント一覧
- 単一テンプレートモードの有無と対象環境
- モジュール化したリソースの一覧（ある場合）
- エラーや警告があれば内容
