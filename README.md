# digio-tf-workflows

Repositório centralizado de **reusable workflows** para pipelines Terraform com Azure.

## Workflows disponíveis

### `terraform-plan-apply.yml`

Workflow reutilizável para plan & apply com governança integrada:

| Fase | Ferramenta | Descrição |
|------|-----------|-----------|
| Lint | `terraform fmt` | Verifica formatação |
| Validate | `terraform validate` | Valida sintaxe HCL |
| Policy | **Checkov** (v12) | Scan de policy-as-code (CIS, SOC2, etc.) |
| Security | **Trivy** (0.35.0) | Scan de misconfigurações IaC |
| Plan | `terraform plan` | Plano de execução com detailed exit code |
| Cost | **Infracost** (v3) | Estimativa de custo mensal |
| Apply | `terraform apply` | Deploy com environment gate |

### Fluxo

```
PR/Push → Checkout → TF Init → fmt → validate → Checkov → Trivy → Plan → Infracost → Summary → Artifacts
                                                                                                    │
                                                                              (if apply=true)       │
                                                                              Environment Approval ←─┘
                                                                                       │
                                                                                    Apply
```

## Como usar (caller)

No repositório do workload, crie `.github/workflows/deploy.yml`:

```yaml
name: Deploy
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  workflow_dispatch:

jobs:
  deploy:
    uses: jpvieirah/digio-tf-workflows/.github/workflows/terraform-plan-apply.yml@v1
    with:
      environment: prd
      terraform_dir: "."
      backend_key: "sub-prod/workloads/my-app.tfstate"
      apply: ${{ github.ref == 'refs/heads/main' && github.event_name == 'push' }}
    secrets: inherit
```

## Inputs

| Input | Obrigatório | Default | Descrição |
|-------|:-----------:|---------|-----------|
| `environment` | ✅ | — | Ambiente alvo (prd, nprd, dev) |
| `terraform_dir` | | `.` | Caminho do root module |
| `terraform_version` | | `1.14.7` | Versão do Terraform CLI |
| `apply` | | `false` | Executar apply após plan |
| `backend_container` | | `tfstate` | Container do Azure Storage |
| `backend_key` | ✅ | — | Chave do state file |
| `checkov_enabled` | | `true` | Habilitar Checkov |
| `infracost_enabled` | | `true` | Habilitar Infracost |
| `trivy_enabled` | | `true` | Habilitar Trivy |

## Secrets necessários

Os secrets devem ser configurados no repositório caller (ou herdados via `secrets: inherit`):

| Secret | Descrição |
|--------|-----------|
| `AZURE_TENANT_ID` | Tenant ID do Azure AD |
| `AZURE_CLIENT_ID` | Client ID da App Registration (OIDC) |
| `AZURE_SUBSCRIPTION_ID` | Subscription ID do Azure |
| `TF_STATE_RG` | Resource Group do Storage Account de state |
| `TF_STATE_SA` | Nome do Storage Account de state |
| `INFRACOST_API_KEY` | API key do Infracost (opcional se infracost_enabled=false) |

## Versionamento

Este repositório é versionado por tags semânticas. Callers devem pinar em tag (`@v1`) ou SHA para estabilidade.

```yaml
# Tag (recomendado)
uses: jpvieirah/digio-tf-workflows/.github/workflows/terraform-plan-apply.yml@v1

# SHA (mais seguro)
uses: jpvieirah/digio-tf-workflows/.github/workflows/terraform-plan-apply.yml@abc1234
```

## Tooling baseline

| Ferramenta | Versão |
|-----------|--------|
| Terraform CLI | 1.14.7 |
| AzureRM Provider | 4.65.0 |
| `actions/checkout` | v4 |
| `hashicorp/setup-terraform` | v3 |
| `azure/login` | v2 |
| `bridgecrewio/checkov-action` | v12 |
| `aquasecurity/trivy-action` | 0.35.0 |
| `infracost/actions/setup` | v3 |
| `actions/upload-artifact` | v4 |
