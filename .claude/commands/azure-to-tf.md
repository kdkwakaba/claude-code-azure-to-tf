# /azure-to-tf — 既存 Azure リソースを Terraform テンプレートに変換する

既存の Azure リソースグループを aztfexport でエクスポートし、azurerm/azapi ベースの Terraform コードに変換・インポートして Terraform 管理下に置くまでの一連の作業を実行する。

## 使い方

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

引数が省略された場合はユーザーに対話形式で確認する。

---

## 実行フロー

引数をパースして以下の順に 2 つのサブエージェントを起動する。**メイン会話は各サブエージェントの完了を待ってから次に進む。**

### Phase 1: 分析・分類サブエージェント

Agent ツール（description: "Azure リソース分析・分類"）を起動し、以下のプロンプトを渡す（`<...>` には実際の値を代入する）：

```
`.claude/rules/aztf-phase1.md` を Read ツールで読み込み、記載の手順に従って処理を実行してください。

実行引数：
- RG_LIST: <rg1,rg2,...>
- SUBSCRIPTION: <subscription-id または 空文字>
- BACKEND: <local|azblob|hcp>
- INCLUDE_DEFAULTS: <true|false>
- IMPORT_RG: <true|false>
- EXCLUDE_RESOURCE_TYPES: <types または 空文字>
- OUTPUT_DIR: <output-dir>
```

Phase 1 完了後、サブエージェントが返した `work_dir`（classification.json の格納先）を確認する。

### Phase 2: コード生成サブエージェント

Phase 1 完了後、Agent ツール（description: "Terraform コード生成"）を起動し、以下のプロンプトを渡す（`<work_dir>` には Phase 1 の返却値を代入する）：

```
`.claude/rules/aztf-phase2.md` を Read ツールで読み込み、記載の手順に従って処理を実行してください。

Phase 1 の出力:
- classification_json_path: <work_dir>/classification.json
```

Phase 2 完了後、サブエージェントの返却内容をユーザーに提示する。

### Phase 2 完了後のクリーンアップ

Phase 2 のサブエージェントが正常完了したら、Phase 1 で aztfexport が生成した一時ファイルを削除する：

```bash
rm -rf <work_dir>
```

`<work_dir>` は Phase 1 サブエージェントが返した絶対パス（`/tmp/aztfexport-*` 形式）を使用する。`infra/` 配下の生成済み Terraform ファイルは削除しない。

---

## STEP 10: plan 結果のレビュー支援

`No changes` または `X to import, 0 to add, 0 to change, 0 to destroy` が理想。

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
cd <output-dir>/<component>
terraform apply tfplan
```

---

## STEP 12: Backend の設定・移行

Backend 設定の詳細手順は `.claude/rules/aztf-backend.md` を参照してください。

- `--backend local`：ローカル Backend のまま完了。必要に応じてリモート移行を提案する
- `--backend azblob`：`terraform init` または `terraform init -migrate-state` を案内する
- `--backend hcp`：Workspace 作成確認後に `terraform login` と `terraform init` を案内する

---

## エラーハンドリング早見表

基本方針：エラー発生時は処理を停止しエラー内容とユーザーへの対処法を表示する。以下は方針が異なるケースのみ：

| 状況 | 対応 |
|---|---|
| コンポーネント分類不明・環境検出不可 | ユーザーに確認して続行（Phase 1 サブエージェントが対応） |
| import 不可リソース | ユーザーに報告・スキップして続行 |
| plan で差分発生 | 内容をユーザーに提示・コードを修正して解消を試みる |
| plan で意図しない destroy/replace | 処理停止・ユーザーに確認 |
| validate/apply エラー | 内容を確認してコードを修正し案内 |
| Backend 移行失敗 | 処理停止・ローカル Backend のまま維持 |
