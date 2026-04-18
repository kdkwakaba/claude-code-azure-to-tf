# Security Policy

## Supported Versions

| Version | Supported |
|---------|-----------|
| latest (main branch) | :white_check_mark: |

## Reporting a Vulnerability

If you discover a security vulnerability in this repository, **please do not open a public GitHub Issue.**

Report it privately via one of the following channels:

- **GitHub Private Vulnerability Reporting**: Use [Security Advisories](../../security/advisories/new) in this repository
- **Email**: Contact the maintainer at the email address listed on their GitHub profile

Please include the following in your report:

- A description of the vulnerability and its potential impact
- Steps to reproduce the issue
- Any suggested mitigations or fixes

You can expect an acknowledgment within **5 business days** and a resolution or status update within **30 days**.

## Security Considerations for Users

This toolset handles Azure credentials and infrastructure configuration. Please follow these practices when using it:

### Credential Handling

- **Never commit credentials** to source control. Use `.env` (already in `.gitignore`) to store `ARM_CLIENT_SECRET` and other secrets locally.
- Use a **service principal with the minimum required permissions** (e.g., `Reader` for export, `Contributor` for import).
- Prefer **certificate-based or federated identity authentication** over client secret where possible.
- Rotate service principal credentials regularly.

### Generated Terraform Files

- **`*.secret.tfvars`** files contain sensitive variable values and are excluded from git by default. Verify your `.gitignore` before committing.
- **`*.tfvars`** files may contain environment-specific configuration. Avoid committing them to public repositories.
- **`*.tfstate`** and **`*.tfstate.backup`** files must never be committed — they may contain secrets in plaintext.
- Use a **remote backend with encryption** (e.g., Azure Blob Storage with customer-managed keys) for production state files.

### Temporary Files

- The `/azure-to-tf` command creates temporary files under `/tmp/aztfexport-*` during execution. These files contain exported Azure resource configurations and are automatically deleted after code generation completes.
- If the process is interrupted, manually remove any remaining `/tmp/aztfexport-*` directories.

### Azure Permissions

- Follow the **principle of least privilege** when assigning roles to the service principal.
- Avoid granting `Owner` or broad `Contributor` roles at the subscription level unless necessary.
- Review generated `azurerm_role_assignment` resources in Terraform plans before applying.

## Out of Scope

The following are considered out of scope for vulnerability reports:

- Security issues in third-party tools (aztfexport, Azure CLI, Terraform) — please report these to their respective projects
- Issues in Azure infrastructure deployed using the generated Terraform code — these are the user's responsibility
