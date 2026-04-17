# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [Unreleased]

### Added
- `SECURITY.md` — security policy and vulnerability reporting guidelines
- `CHANGELOG.md` — this file

### Changed
- `/azure-to-tf`: Temporary working directory is now created with `mktemp -d` and `chmod 700` to prevent other processes from reading exported Azure resource configurations
- `/azure-to-tf`: Service principal login no longer passes `--password` as a CLI argument; credentials are passed via environment variables to avoid exposure in process lists
- `/azure-to-tf`: Credential existence checks now use `[ -n "$VAR" ]` instead of `echo $VAR` to prevent secrets from appearing in terminal output and logs
- `/azure-to-tf`: `EXCLUDE_FILE` is now placed inside the secured working directory instead of `/tmp` directly, eliminating symlink attack risk
- `aztf-templates.md`: Added `outputs.tf` template section with explicit rules for `sensitive = true` on outputs containing connection strings, keys, and URIs
- `/azure-to-tf`: Temporary files under `/tmp/aztfexport-*` are now deleted automatically after Phase 2 completes

## [0.1.0] - 2026-04-09

### Added
- Initial release
- `/azure-to-tf` slash command — onboards existing Azure resource groups into Terraform management via aztfexport
  - Phase 1 subagent: prerequisite checks, Azure login, aztfexport export, component classification, module detection
  - Phase 2 subagent: Terraform code generation, `terraform validate`, environment-specific tfvars generation
  - Single-template mode for similar resource groups that differ only by environment identifier
  - `--backend` option supporting `local`, `azblob`, and `hcp`
  - `--import-rg` option to bring resource groups themselves under Terraform management
  - `--include-defaults` option to include Azure-generated default resources
- `.claude/rules/terraform-style.md` — Terraform coding conventions (file structure, naming, variables, locals, outputs, iteration, providers)
- `.claude/rules/azure-practices.md` — Azure-specific practices (identity, secrets, networking, state management, cost, security)
- `.claude/rules/aztf-classification.md` — component classification logic for resource group exports
- `.claude/rules/aztf-phase1.md` — Phase 1 subagent instructions
- `.claude/rules/aztf-phase2.md` — Phase 2 subagent instructions
- `.claude/rules/aztf-templates.md` — Terraform code generation templates
- `.claude/rules/aztf-backend.md` — backend configuration and migration instructions
- Dev Container (`.devcontainer/`) with Azure CLI, Terraform, aztfexport, and Claude CLI
- `terraform fmt` hook — automatically formats `.tf` files on save
- `.env.example` — template for required environment variables

[Unreleased]: https://github.com/KazukiYamabe/claude-code-azure-to-tf/compare/v0.1.0...HEAD
[0.1.0]: https://github.com/KazukiYamabe/claude-code-azure-to-tf/releases/tag/v0.1.0
