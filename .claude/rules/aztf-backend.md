# STEP 12: Backend の設定・移行（詳細）

`/azure-to-tf` コマンドの STEP 12 で参照する Backend 設定手順。

---

## `--backend local`（デフォルト）

ローカル Backend のまま完了とする。必要に応じてリモート Backend への移行を提案する：

```
現在はローカル Backend を使用しています。
リモート Backend（azblob または hcp）への移行が必要な場合は /azure-to-tf を
--backend azblob または --backend hcp オプション付きで再実行してください。
```

---

## `--backend azblob`

STEP 0.5 で収集した情報を使用して `terraform.tf` の backend ブロックを更新する（対話収集が必要な場合は以下の質問を行う）：

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

## `--backend hcp`

STEP 0.5 で収集した情報を使用する（対話収集が必要な場合は以下の質問を行う）：

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
