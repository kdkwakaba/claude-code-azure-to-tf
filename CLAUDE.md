# CLAUDE.md

このファイルは、リポジトリ内のコードを操作する際に Claude Code (claude.ai/code) へ提供するガイダンスです。

## 目的

このリポジトリは、Claude Code が Azure インフラ向けの Terraform コードを生成・修正するためのプロンプト・指示セットです。

## 開発環境

dev container（`.devcontainer/`）には **Azure CLI**、**Terraform**、**Claude CLI** が含まれています。

```bash
# ツールの動作確認
claude --version && az version && terraform version

# フォーマット（確認不要）
terraform fmt -recursive

# 構文検証（確認不要）
terraform validate

# プラン・適用（実行前にユーザー確認必須）
terraform plan   # ARM_SUBSCRIPTION_ID 環境変数が必要
terraform apply
```

## プロジェクト構成

コンポーネントごとに独立した Terraform ルートモジュールを `infra/` 配下に配置する。環境の差異は `.tfvars` で管理する。ブランチ・リポジトリ・フォルダーを環境ごとに分けない。

## 追加ルール

@.claude/rules/terraform-style.md
@.claude/rules/azure-practices.md
