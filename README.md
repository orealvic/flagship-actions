# flagship-actions

Reusable GitHub Actions workflows for the Flagship Azure Landing Zone project.

## Available workflows

### `terraform.yml` — Plan and apply

A reusable workflow that handles the complete Terraform lifecycle for any IaC stack. Used by `flagship-platform`, `flagship-landing-zone`, and `flagship-ai` as their core CI/CD primitive.

**What it does:**

1. OIDC login to Azure (no stored secrets)
2. `terraform fmt -check -recursive`
3. `terraform init` with remote backend (Azure Storage, AAD auth)
4. `terraform validate`
5. `tfsec` security scan (informational on Day 2; tighten later)
6. `terraform plan` (when `apply: false`) — output posted as a PR comment
7. `terraform apply` (when `apply: true`) — gated by GitHub Environment approval

**Usage from a caller workflow:**

```yaml
name: Platform IaC

on:
  pull_request:
    paths: ['platform/**']
  push:
    branches: [main]
    paths: ['platform/**']

permissions:
  id-token: write
  contents: read
  pull-requests: write

jobs:
  plan:
    if: github.event_name == 'pull_request'
    uses: orealvic/flagship-actions/.github/workflows/terraform.yml@main
    with:
      working_directory: platform
      state_key: platform.tfstate
      apply: false

  apply:
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    needs: []
    uses: orealvic/flagship-actions/.github/workflows/terraform.yml@main
    with:
      working_directory: platform
      state_key: platform.tfstate
      environment: platform-prod   # triggers approval gate
      apply: true
```

**Required repo variables** (set per consuming repo):

| Variable | Set in bootstrap output |
|---|---|
| `AZURE_CLIENT_ID` | Repo-specific managed identity client ID |
| `AZURE_TENANT_ID` | Entra tenant ID |
| `AZURE_SUBSCRIPTION_ID` | Target Azure subscription |
| `TFSTATE_RG` | Resource group holding state SA |
| `TFSTATE_SA` | State storage account name |
| `TFSTATE_CONTAINER` | Blob container holding state files |

## Design principles

- **Reusable, not copy-pasted.** A bug fix or pattern improvement here propagates to every consumer on the next workflow run.
- **OIDC by default.** Every job assumes federated identity; no service principal secrets supported.
- **State backend always remote.** Never commits state to a runner's local filesystem.
- **PR feedback is rich.** Plan output goes into the PR conversation, not buried in action logs.
- **Apply requires approval.** The `environment:` input ties apply jobs to GitHub Environments with required reviewers.

## See also

- [flagship-docs](https://github.com/orealvic/flagship-docs) — architecture and ADRs
- [flagship-platform](https://github.com/orealvic/flagship-platform) — first consumer of these workflows
