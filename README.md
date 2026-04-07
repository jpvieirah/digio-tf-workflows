# digio-tf-workflows

Repositorio centralizado de **reusable workflows** para pipelines Terraform com Azure.

Zero hardcode — backend, container e state key sao derivados automaticamente do nome do repositorio do workload.

---

## Arquitetura

```
  Workload Repo                        Reusable Workflow Repo
  (terraform-workload-loan)            (digio-tf-workflows)
  ┌──────────────────────┐             ┌───────────────────────────────────┐
  │ .github/workflows/   │             │ .github/workflows/               │
  │   deploy.yml (caller)│────uses────>│   terraform-plan-apply.yml       │
  │                      │             │   module-ci.yml                  │
  │ main.tf              │             │   module-release.yml             │
  │ variables.tf         │             └───────────────────────────────────┘
  │ backend.tf           │                        │
  └──────────────────────┘                        ▼
                                       ┌───────────────────────────────────┐
                                       │ Resolve backend automaticamente   │
                                       │                                   │
                                       │ Repo: terraform-workload-loan     │
                                       │ Env:  prd                         │
                                       │   → container: tfstate-sub-prd-loan│
                                       │   → key:       loan.tfstate       │
                                       │                                   │
                                       │ Container nao existe?             │
                                       │   → Cria automaticamente          │
                                       └───────────────────────────────────┘
```

## Workflows disponiveis

### `terraform-plan-apply.yml` — Workloads

Pipeline completa de provisionamento:

| Fase | Ferramenta | Descricao |
|------|------------|-----------|
| Resolve | Auto | Deriva container + key do nome do repo |
| Login | Azure OIDC | Autenticacao federada sem secrets estaticos |
| Container | `az cli` | Cria container no Storage Account se nao existir |
| Init | `terraform init` | Inicializa com backend remoto |
| Lint | `terraform fmt` | Verifica formatacao |
| Validate | `terraform validate` | Valida sintaxe HCL |
| Policy | **Checkov** v12 | Scan de policy-as-code (CIS, SOC2, etc.) |
| Security | **Trivy** 0.35.0 | Scan de misconfiguracoes IaC |
| Plan | `terraform plan` | Plano de execucao com detailed exit code |
| Cost | **Infracost** v3 | Estimativa de custo mensal |
| Summary | GitHub | Dashboard no job summary + comentario no PR |
| Artifacts | GitHub | Upload de tfplan, scan results (30 dias) |
| Apply | `terraform apply` | Deploy com environment gate (approval) |

### `module-ci.yml` — Modulos

Validacao de modulos Terraform (sem backend, sem state):

`fmt` → `init -backend=false` → `validate` → `checkov` → `terraform test`

### `module-release.yml` — Modulos

Versionamento semantico automatico com GitHub Releases.

Detecta bump pelo prefixo do commit: `fix:` → patch, `feat:` → minor, `breaking:` → major.

---

## Como usar

### Workload (provisionamento)

No repositorio do workload, crie `.github/workflows/deploy.yml`:

```yaml
name: Deploy
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  workflow_dispatch:

permissions:
  id-token: write
  contents: read
  pull-requests: write

jobs:
  deploy:
    uses: cbssdigital/digio-tf-workflows/.github/workflows/terraform-plan-apply.yml@main
    with:
      environment: prd
      apply: ${{ github.ref == 'refs/heads/main' && github.event_name == 'push' }}
    secrets: inherit
```

Isso e tudo. O workflow resolve automaticamente:
- **Container**: `tfstate-sub-prd-{nome-do-workload}`
- **Key**: `{nome-do-workload}.tfstate`
- Se o container nao existir, **cria automaticamente**

### Convencao de nomes

O nome do workload e extraido do nome do repositorio removendo prefixos comuns:

| Repositorio | Workload | Container (env=prd) | Key |
|-------------|----------|---------------------|-----|
| `terraform-workload-loan` | `loan` | `tfstate-sub-prd-loan` | `loan.tfstate` |
| `terraform-workload-creditcard` | `creditcard` | `tfstate-sub-prd-creditcard` | `creditcard.tfstate` |
| `workload-account` | `account` | `tfstate-sub-prd-account` | `account.tfstate` |
| `landing-zone-connectivity` | `landing-zone-connectivity` | `tfstate-sub-prd-landing-zone-connectivity` | `landing-zone-connectivity.tfstate` |

### Override manual

Se algum workload fugir da convencao:

```yaml
jobs:
  deploy:
    uses: cbssdigital/digio-tf-workflows/.github/workflows/terraform-plan-apply.yml@main
    with:
      environment: prd
      backend_container: tfstate-sub-shd-management    # override explicito
      backend_key: custom-name.tfstate                  # override explicito
    secrets: inherit
```

### Modulo (CI)

```yaml
name: CI
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

permissions:
  contents: read
  pull-requests: write

jobs:
  ci:
    uses: cbssdigital/digio-tf-workflows/.github/workflows/module-ci.yml@main
    with:
      terraform_version: "1.14.7"
```

### Modulo (Release)

```yaml
name: Release
on:
  workflow_dispatch:
    inputs:
      bump:
        description: "Version bump"
        default: "auto"
        type: choice
        options: [auto, patch, minor, major]

permissions:
  contents: write

jobs:
  release:
    uses: cbssdigital/digio-tf-workflows/.github/workflows/module-release.yml@main
    with:
      bump: ${{ inputs.bump }}
    secrets: inherit
```

---

## Inputs — terraform-plan-apply.yml

| Input | Obrigatorio | Default | Descricao |
|-------|:-----------:|---------|-----------|
| `environment` | Sim | — | Ambiente alvo (prd, uat, dev, pprd) |
| `backend_container` | | auto | Override do container. Default = `tfstate-sub-{env}-{workload}` |
| `backend_key` | | auto | Override da key. Default = `{workload}.tfstate` |
| `terraform_dir` | | `.` | Caminho do root module |
| `terraform_version` | | `1.14.7` | Versao do Terraform CLI |
| `apply` | | `false` | Executar apply apos plan |
| `runner` | | `ubuntu-latest` | Label do runner (suporta self-hosted) |
| `checkov_enabled` | | `true` | Habilitar Checkov |
| `infracost_enabled` | | `true` | Habilitar Infracost |
| `trivy_enabled` | | `true` | Habilitar Trivy |

## Secrets necessarios

Configurar no repositorio caller ou na org (herdados via `secrets: inherit`):

| Secret | Obrigatorio | Descricao |
|--------|:-----------:|-----------|
| `AZURE_TENANT_ID` | Sim | Tenant ID do Azure AD |
| `AZURE_CLIENT_ID` | Sim | Client ID da App Registration (OIDC) |
| `AZURE_SUBSCRIPTION_ID` | Sim | Subscription ID alvo |
| `TF_STATE_RG` | Sim | Resource Group do Storage Account de state |
| `TF_STATE_SA` | Sim | Nome do Storage Account de state |
| `INFRACOST_API_KEY` | Nao | API key do Infracost (opcional se `infracost_enabled=false`) |

---

## State Management

```
 Storage Account (stshdmgmttfstatebrsz)
 ├── tfstate-sub-architecture/        ← Landing Zone: infra.tfstate
 ├── tfstate-sub-shd-connectivity/    ← Shared: connectivity.tfstate
 ├── tfstate-sub-shd-identity/        ← Shared: iam.tfstate
 ├── tfstate-sub-shd-management/      ← Shared: mgmt.tfstate
 ├── tfstate-sub-dev-loan/            ← Dev: loan.tfstate, app2.tfstate
 ├── tfstate-sub-uat-loan/            ← UAT: loan.tfstate
 ├── tfstate-sub-pprd-loan/           ← Pre-Prod: loan.tfstate
 ├── tfstate-sub-prd-loan/            ← Prod: loan.tfstate
 ├── tfstate-sub-prd-creditcard/      ← Prod: creditcard.tfstate
 ├── tfstate-sub-prd-account/         ← Prod: account.tfstate
 └── ...
```

Cada workload repo gera exatamente 1 arquivo `.tfstate` no container do seu ambiente. Se o container nao existir, o workflow cria automaticamente na primeira execucao.

---

## Governanca

| Regra | Implementacao |
|-------|---------------|
| Plan falhou → apply bloqueado | `detailed-exitcode` + condicional no job apply |
| Approval antes do apply | GitHub Environment protection rules |
| Security scan obrigatorio | Checkov + Trivy antes do plan |
| Custo visivel | Infracost no summary e PR comment |
| Auditoria | Artifacts retidos por 30 dias |
| Zero secrets no codigo | Tudo via `secrets: inherit` |
| Zero config no caller | Backend derivado do nome do repo |

---

## Versionamento

```yaml
# Branch (desenvolvimento)
uses: cbssdigital/digio-tf-workflows/.github/workflows/terraform-plan-apply.yml@main

# Tag (producao — recomendado)
uses: cbssdigital/digio-tf-workflows/.github/workflows/terraform-plan-apply.yml@v1

# SHA (maximo controle)
uses: cbssdigital/digio-tf-workflows/.github/workflows/terraform-plan-apply.yml@abc1234
```

## Tooling baseline

| Ferramenta | Versao | Notas |
|-----------|--------|-------|
| Terraform CLI | 1.14.7 | Configuravel via input |
| AzureRM Provider | 4.65.0 | Definido no workload |
| `actions/checkout` | v4 | |
| `hashicorp/setup-terraform` | v3 | |
| `azure/login` | v2 | OIDC |
| `bridgecrewio/checkov-action` | v12 | Desligavel |
| `aquasecurity/trivy-action` | 0.35.0 | Desligavel |
| `infracost/actions/setup` | v3 | Desligavel |
| `actions/upload-artifact` | v4 | |
