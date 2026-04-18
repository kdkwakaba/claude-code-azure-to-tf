# パイプラインテンプレート集

`/generate-pipeline` コマンドの STEP 6 で参照するテンプレートと生成規則。

---

## 共通設計方針

- コンポーネントは `/generate-pipeline` の STEP 3 で決定した順序で順次実行する（並列実行しない）
- `terraform fmt -check` → `terraform validate` → `terraform plan` は PR 時に実行する
- `terraform apply` は `main` ブランチへの push 時のみ実行する
- 本番環境（`prd` / `prod` / `production`）の apply には必ず手動承認ゲートを設ける
- クレデンシャルはコードにハードコードせず、各プラットフォームのシークレット機構から参照する
- `secret.tfvars` が存在するコンポーネントは、該当箇所にコメントを記載してユーザーに対応を促す

---

## GitHub Actions テンプレート

出力ファイル：`.github/workflows/terraform.yml`

### 設計上の注意点

- 本番環境の apply job には `environment:` に GitHub Environment 名を指定する（手動承認は GitHub Environment の Required reviewers で実現）
- 非本番環境は `environment:` を省略して自動 apply にする
- コンポーネント間の依存は `needs:` で表現し、同一環境内で順次実行する
- Terraform プラグインキャッシュは `actions/cache` で共有する
- 単一テンプレートモードの場合は `terraform init` の `-backend-config` オプション箇所にコメントを記載する

### テンプレート

```yaml
name: Terraform

on:
  push:
    branches:
      - main
    paths:
      - 'infra/**'
      - '.github/workflows/terraform.yml'
  pull_request:
    paths:
      - 'infra/**'
      - '.github/workflows/terraform.yml'

env:
  TF_PLUGIN_CACHE_DIR: ${{ github.workspace }}/.terraform.d/plugin-cache
  ARM_CLIENT_ID: ${{ secrets.ARM_CLIENT_ID }}
  ARM_CLIENT_SECRET: ${{ secrets.ARM_CLIENT_SECRET }}
  ARM_TENANT_ID: ${{ secrets.ARM_TENANT_ID }}
  ARM_SUBSCRIPTION_ID: ${{ secrets.ARM_SUBSCRIPTION_ID }}

jobs:
  # ============================================================
  # コンポーネント × 環境 の組み合わせ分だけ job を生成する。
  # 実行順は STEP 3 で決定した順序に従い needs: で制御する。
  # ============================================================

  # 例: network / stg（非本番 → environment: 指定なし）
  network-stg:
    name: "network / stg"
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: infra/network

    steps:
      - uses: actions/checkout@v4

      - name: Terraform プラグインキャッシュ
        uses: actions/cache@v4
        with:
          path: .terraform.d/plugin-cache
          key: terraform-${{ runner.os }}-${{ hashFiles('infra/network/.terraform.lock.hcl') }}
          restore-keys: terraform-${{ runner.os }}-

      - uses: hashicorp/setup-terraform@v3

      - name: terraform fmt
        run: terraform fmt -check -recursive

      - name: terraform init
        run: terraform init
        # 単一テンプレートモード（azblob backend）の場合は以下に差し替える：
        # run: terraform init -backend-config="key=network-stg.tfstate"

      - name: terraform validate
        run: terraform validate

      - name: terraform plan
        run: |
          terraform plan \
            -var-file="../../environments/stg/network.tfvars" \
            -out=tfplan
        # secret.tfvars が存在する場合は以下を追加する：
        # -var-file="../../environments/stg/network.secret.tfvars" \
        # → GitHub Actions Secrets に機密変数を登録し -var で渡すか、
        #   secret.tfvars の内容を Actions Secret として個別に追加してください。

      - name: terraform apply
        if: github.ref == 'refs/heads/main' && github.event_name == 'push'
        run: terraform apply tfplan

  # 例: network / prd（本番 → environment: production で手動承認）
  network-prd:
    name: "network / prd"
    runs-on: ubuntu-latest
    needs: [network-stg]            # 同コンポーネントの非本番が成功した後に実行
    environment: production         # GitHub Environment "production" に Required reviewers を設定すること
    defaults:
      run:
        working-directory: infra/network

    steps:
      - uses: actions/checkout@v4

      - name: Terraform プラグインキャッシュ
        uses: actions/cache@v4
        with:
          path: .terraform.d/plugin-cache
          key: terraform-${{ runner.os }}-${{ hashFiles('infra/network/.terraform.lock.hcl') }}
          restore-keys: terraform-${{ runner.os }}-

      - uses: hashicorp/setup-terraform@v3

      - name: terraform fmt
        run: terraform fmt -check -recursive

      - name: terraform init
        run: terraform init

      - name: terraform validate
        run: terraform validate

      - name: terraform plan
        run: |
          terraform plan \
            -var-file="../../environments/prd/network.tfvars" \
            -out=tfplan

      - name: terraform apply
        if: github.ref == 'refs/heads/main' && github.event_name == 'push'
        run: terraform apply tfplan

  # 以降、STEP 3 の実行順に従い次のコンポーネントの job を追加する。
  # 各 job の needs: には直前のコンポーネントの同環境 job を指定する。
  # 例: data-stg の needs: → [network-stg]
  #     data-prd の needs: → [data-stg, network-prd]
```

### GitHub Actions 事前準備

ユーザーに以下の設定を案内する（Claude Code 側では実行しない）：

```
GitHub Actions パイプラインを使用するには、以下の設定をリポジトリで行ってください：

1. GitHub Environment の作成
   Settings → Environments → New environment
   - 名前: production
   - Required reviewers に承認者を追加する

2. Repository Secrets の登録
   Settings → Secrets and variables → Actions → New repository secret
   - ARM_CLIENT_ID
   - ARM_CLIENT_SECRET
   - ARM_TENANT_ID
   - ARM_SUBSCRIPTION_ID

3. secret.tfvars が存在するコンポーネントがある場合
   該当する機密変数の値を追加の Repository Secret として登録し、
   plan ステップの -var フラグで渡してください。
```

---

## Azure DevOps テンプレート

出力ファイル：`azure-pipelines.yml`

### 設計上の注意点

- 本番環境の apply ステップには `environment:` に Azure DevOps Environment 名を指定する（手動承認は Pipelines > Environments の Approval チェックで実現）
- 非本番環境は `environment:` を省略したスクリプトステップとして定義し自動 apply にする
- ステージ間の依存は `dependsOn:` で表現し、同一環境内で順次実行する
- クレデンシャルは変数グループ `terraform-credentials` から参照する
- Terraform はスクリプトでインストールするか、セルフホストエージェントを使用する
- 単一テンプレートモードの場合は `terraform init` の `-backend-config` オプション箇所にコメントを記載する

### テンプレート

```yaml
trigger:
  branches:
    include:
      - main
  paths:
    include:
      - infra/**
      - azure-pipelines.yml

pr:
  paths:
    include:
      - infra/**
      - azure-pipelines.yml

variables:
  - group: terraform-credentials   # ARM_CLIENT_ID / ARM_CLIENT_SECRET / ARM_TENANT_ID / ARM_SUBSCRIPTION_ID を登録すること
  - name: TF_PLUGIN_CACHE_DIR
    value: $(Pipeline.Workspace)/.terraform.d/plugin-cache

stages:
  # ============================================================
  # コンポーネント × 環境 の組み合わせ分だけ stage を生成する。
  # 実行順は STEP 3 で決定した順序に従い dependsOn: で制御する。
  # ============================================================

  # 例: network / stg（非本番 → deployment job なし・自動 apply）
  - stage: Network_Stg
    displayName: "network / stg"
    jobs:
      - job: terraform
        displayName: "Terraform"
        pool:
          vmImage: ubuntu-latest
        steps:
          - checkout: self

          - task: Cache@2
            displayName: Terraform プラグインキャッシュ
            inputs:
              key: 'terraform | $(Agent.OS) | infra/network/.terraform.lock.hcl'
              restoreKeys: terraform | $(Agent.OS)
              path: $(TF_PLUGIN_CACHE_DIR)

          - script: |
              wget -O- https://apt.releases.hashicorp.com/gpg \
                | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
              echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] \
                https://apt.releases.hashicorp.com $(lsb_release -cs) main" \
                | sudo tee /etc/apt/sources.list.d/hashicorp.list
              sudo apt-get update && sudo apt-get install -y terraform
            displayName: Terraform インストール

          - script: terraform fmt -check -recursive
            displayName: terraform fmt
            workingDirectory: infra/network

          - script: terraform init
            displayName: terraform init
            workingDirectory: infra/network
            env:
              ARM_CLIENT_ID: $(ARM_CLIENT_ID)
              ARM_CLIENT_SECRET: $(ARM_CLIENT_SECRET)
              ARM_TENANT_ID: $(ARM_TENANT_ID)
              ARM_SUBSCRIPTION_ID: $(ARM_SUBSCRIPTION_ID)
            # 単一テンプレートモード（azblob backend）の場合は以下に差し替える：
            # script: terraform init -backend-config="key=network-stg.tfstate"

          - script: terraform validate
            displayName: terraform validate
            workingDirectory: infra/network

          - script: |
              terraform plan \
                -var-file="../../environments/stg/network.tfvars" \
                -out=tfplan
            displayName: terraform plan
            workingDirectory: infra/network
            env:
              ARM_CLIENT_ID: $(ARM_CLIENT_ID)
              ARM_CLIENT_SECRET: $(ARM_CLIENT_SECRET)
              ARM_TENANT_ID: $(ARM_TENANT_ID)
              ARM_SUBSCRIPTION_ID: $(ARM_SUBSCRIPTION_ID)
            # secret.tfvars が存在する場合は以下を追加する：
            # -var-file="../../environments/stg/network.secret.tfvars" \
            # → 変数グループ terraform-credentials に機密変数を追加し -var で渡してください。

          - script: terraform apply tfplan
            displayName: terraform apply
            condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
            workingDirectory: infra/network
            env:
              ARM_CLIENT_ID: $(ARM_CLIENT_ID)
              ARM_CLIENT_SECRET: $(ARM_CLIENT_SECRET)
              ARM_TENANT_ID: $(ARM_TENANT_ID)
              ARM_SUBSCRIPTION_ID: $(ARM_SUBSCRIPTION_ID)

  # 例: network / prd（本番 → deployment job + Approval チェック）
  - stage: Network_Prd
    displayName: "network / prd"
    dependsOn:
      - Network_Stg                 # 同コンポーネントの非本番が成功した後に実行
    jobs:
      - deployment: terraform
        displayName: "Terraform"
        pool:
          vmImage: ubuntu-latest
        environment: prd            # Pipelines > Environments で "prd" を作成し Approval チェックを追加すること
        strategy:
          runOnce:
            deploy:
              steps:
                - checkout: self

                - task: Cache@2
                  displayName: Terraform プラグインキャッシュ
                  inputs:
                    key: 'terraform | $(Agent.OS) | infra/network/.terraform.lock.hcl'
                    restoreKeys: terraform | $(Agent.OS)
                    path: $(TF_PLUGIN_CACHE_DIR)

                - script: |
                    wget -O- https://apt.releases.hashicorp.com/gpg \
                      | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
                    echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] \
                      https://apt.releases.hashicorp.com $(lsb_release -cs) main" \
                      | sudo tee /etc/apt/sources.list.d/hashicorp.list
                    sudo apt-get update && sudo apt-get install -y terraform
                  displayName: Terraform インストール

                - script: terraform fmt -check -recursive
                  displayName: terraform fmt
                  workingDirectory: infra/network

                - script: terraform init
                  displayName: terraform init
                  workingDirectory: infra/network
                  env:
                    ARM_CLIENT_ID: $(ARM_CLIENT_ID)
                    ARM_CLIENT_SECRET: $(ARM_CLIENT_SECRET)
                    ARM_TENANT_ID: $(ARM_TENANT_ID)
                    ARM_SUBSCRIPTION_ID: $(ARM_SUBSCRIPTION_ID)

                - script: terraform validate
                  displayName: terraform validate
                  workingDirectory: infra/network

                - script: |
                    terraform plan \
                      -var-file="../../environments/prd/network.tfvars" \
                      -out=tfplan
                  displayName: terraform plan
                  workingDirectory: infra/network
                  env:
                    ARM_CLIENT_ID: $(ARM_CLIENT_ID)
                    ARM_CLIENT_SECRET: $(ARM_CLIENT_SECRET)
                    ARM_TENANT_ID: $(ARM_TENANT_ID)
                    ARM_SUBSCRIPTION_ID: $(ARM_SUBSCRIPTION_ID)

                - script: terraform apply tfplan
                  displayName: terraform apply
                  condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
                  workingDirectory: infra/network
                  env:
                    ARM_CLIENT_ID: $(ARM_CLIENT_ID)
                    ARM_CLIENT_SECRET: $(ARM_CLIENT_SECRET)
                    ARM_TENANT_ID: $(ARM_TENANT_ID)
                    ARM_SUBSCRIPTION_ID: $(ARM_SUBSCRIPTION_ID)

  # 以降、STEP 3 の実行順に従い次のコンポーネントの stage を追加する。
  # 各 stage の dependsOn: には直前のコンポーネントの同環境 stage を指定する。
  # 例: Data_Stg の dependsOn: → [Network_Stg]
  #     Data_Prd の dependsOn: → [Data_Stg, Network_Prd]
```

### Azure DevOps 事前準備

ユーザーに以下の設定を案内する（Claude Code 側では実行しない）：

```
Azure DevOps パイプラインを使用するには、以下の設定をプロジェクトで行ってください：

1. 変数グループの作成
   Pipelines → Library → + Variable group
   - グループ名: terraform-credentials
   - 以下の変数を追加する（ARM_CLIENT_SECRET はシークレットとしてマークする）：
     ARM_CLIENT_ID / ARM_CLIENT_SECRET / ARM_TENANT_ID / ARM_SUBSCRIPTION_ID

2. Environment の作成と承認チェックの設定
   Pipelines → Environments → New environment
   - 本番環境（例: prd）を作成し、Approvals & checks → Approvals で承認者を追加する
   - 非本番環境（例: stg）は作成不要（job: を使用するため）

3. secret.tfvars が存在するコンポーネントがある場合
   該当する機密変数の値を変数グループ terraform-credentials に追加し（シークレットとしてマーク）、
   plan ステップの -var フラグで渡してください。

4. パイプラインの登録
   Pipelines → New pipeline → Azure Repos Git → リポジトリを選択
   → Existing Azure Pipelines YAML file → /azure-pipelines.yml を選択
```
