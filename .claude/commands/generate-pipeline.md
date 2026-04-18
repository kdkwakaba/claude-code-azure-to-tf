# /generate-pipeline — Terraform CI/CD パイプラインを生成する

`infra/` のコンポーネント構成を読み取り、正しい実行順序で Terraform を実行する CI/CD パイプラインを GitHub Actions または Azure DevOps 向けに生成する。

## 使い方

```
/generate-pipeline [--platform github|azuredevops|both] [--output-dir <path>]
```

| オプション | 説明 | デフォルト |
|-----------|------|-----------|
| `--platform` | 対象プラットフォーム（`github` / `azuredevops` / `both`） | 対話形式で確認 |
| `--output-dir` | 生成ファイルの出力先 | リポジトリルート |

引数が省略された場合はユーザーに対話形式で確認する。

---

## 実行フロー

### STEP 1: 引数のパースと対話確認

`--platform` が未指定の場合はユーザーに確認する：

```
どのプラットフォーム向けにパイプラインを生成しますか？
  1. GitHub Actions
  2. Azure DevOps
  3. 両方
```

### STEP 2: infra/ 構造のスキャン

`infra/` ディレクトリを読み取ってコンポーネントと環境を検出する：

```bash
# コンポーネント検出（modules/ と environments/ は除外）
ls -d infra/*/  | grep -Ev 'modules/|environments/' | xargs -I{} basename {}

# 環境検出
ls infra/environments/
```

`infra/` が存在しない、またはコンポーネントが 1 つも検出されない場合は処理を停止し、先に `/azure-to-tf` を実行するよう案内する。

### STEP 3: コンポーネント実行順序の決定

検出されたコンポーネントを以下の依存順でソートする。リストにないコンポーネントは末尾にアルファベット順で追加する：

```
network → dns → ope → monitoring → data → backend → api → app → front
```

決定した順序をユーザーに提示して確認を取る：

```
検出したコンポーネントと実行順序：
  1. network
  2. data
  3. api
  4. app

この順序で問題ありませんか？（yes / no — no の場合は正しい順序を入力してください）
```

### STEP 4: 本番環境の判定

`infra/environments/` 配下の環境名を読み取り、以下のパターン（大文字小文字を区別しない）に一致する環境を**本番環境**として扱い、`terraform apply` に手動承認ゲートを必須とする：

- `prd`、`prod`、`production`

上記以外（`stg`、`staging`、`dev`、`development`、`uat`、`qa`、`test` 等）は `terraform apply` を自動実行する。

判定結果をユーザーに提示して確認を取る：

```
検出した環境：
  本番環境（手動承認必須）: prd
  非本番環境（自動 apply）: stg

この判定で問題ありませんか？（yes / no）
```

### STEP 5: 単一テンプレートモードの検出

同一コンポーネントに対して複数環境の tfvars（`environments/<env>/<component>.tfvars`）が存在する場合は単一テンプレートモードと判断し、環境ごとに Backend ステート分離が必要な旨をパイプラインにコメントとして記載する。

### STEP 6: パイプラインファイルの生成

`.claude/rules/pipeline-templates.md` を Read ツールで読み込み、テンプレートと規則に従ってファイルを生成する。

STEP 2〜5 で収集した情報（コンポーネント一覧・実行順序・環境一覧・本番判定結果）を代入して生成すること。

生成対象ファイル：

| プラットフォーム | 出力ファイル |
|---|---|
| GitHub Actions | `.github/workflows/terraform.yml` |
| Azure DevOps | `azure-pipelines.yml` |

出力先にファイルがすでに存在する場合は上書きする前にユーザーに確認する。

### STEP 7: 事前準備の案内

生成完了後、ユーザーに以下を案内する（`.claude/rules/pipeline-templates.md` の「事前準備」セクションを参照）：

- GitHub Actions：GitHub Environment の作成・Secrets の登録
- Azure DevOps：変数グループ・Environments の作成と承認チェックの設定

---

## エラーハンドリング早見表

基本方針：エラー発生時は処理を停止しエラー内容とユーザーへの対処法を表示する。以下は方針が異なるケースのみ：

| 状況 | 対応 |
|---|---|
| `infra/` が存在しない | 処理停止・`/azure-to-tf` を先に実行するよう案内する |
| 環境が検出されない | var-file 指定なしで生成し、警告として報告する |
| 本番判定が曖昧 | ユーザーに確認して続行 |
| `secret.tfvars` が存在するコンポーネントあり | 該当箇所にコメントを記載し、ユーザーに Secrets 追加を案内する |
| 出力ファイルが既存 | 上書き前にユーザーに確認する |
