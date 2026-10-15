---
title: "LAB-16 — GitHub Actions + Azure OIDC pour Terraform"
date: 2026-10-15
draft: false
description: "Authentifier Terraform depuis GitHub Actions vers Azure avec OpenID Connect, User Assigned Managed Identity et Workload Identity Federation sans Client Secret."
summary: "Configurer la fédération GitHub OIDC, isoler les backends DEV/QUALIF/PROD et générer un terraform plan Azure depuis une Pull Request."
slug: "terraform-azure-lab-16-github-actions-azure-oidc"
categories:
  - "Automatisation"
tags:
  - "Terraform"
  - "GitHub Actions"
  - "OIDC"
  - "Microsoft Entra ID"
  - "Managed Identity"
  - "Azure"
series:
  - "Terraform Azure — De zéro à production"
weight: 17
ShowToc: true
TocOpen: true
---

# LAB-16 — GitHub Actions + Azure OIDC pour Terraform

Nous avons volontairement attendu avant de donner un accès Azure à GitHub Actions.

La cible est maintenant :

```text
GitHub Actions
      │
      │ OIDC ID Token court
      ▼
Microsoft Entra ID
      │
      ▼
Federated Identity Credential
      │
      ▼
UAMI Terraform DEV
      │
      ├── Contributor sur RG DEV
      └── Storage Blob Data Contributor sur state DEV
```

Aucun Client Secret Azure permanent ne sera stocké dans GitHub.

## 1. Objectifs

Nous allons :

- créer la confiance GitHub → Entra ;
- comprendre issuer, subject et audience ;
- ajouter un Federated Identity Credential aux UAMI ;
- isoler le backend par container ;
- attribuer `Storage Blob Data Contributor` au scope du container ;
- configurer GitHub Environments ;
- utiliser `id-token: write` ;
- authentifier `azure/login@v3` ;
- configurer AzureRM Provider et Backend avec OIDC ;
- exécuter un vrai `terraform plan` DEV depuis une Pull Request.

## 2. Point important en 2026 : le `sub` GitHub

Ne copiez pas aveuglément une ancienne valeur comme :

```text
repo:ORG/REPO:environment:dev
```

Depuis juillet 2026, GitHub a introduit des changements pour les claims OIDC par défaut de certains repositories nouvellement créés, renommés ou transférés, avec des identifiants immuables.

La bonne méthode est donc :

> définir explicitement la politique OIDC du repository et utiliser la valeur `sub` exacte correspondant à ce repository et à cet environnement.

Dans le Terraform de ce Lab, nous traiterons donc les subjects comme des **inputs explicites**.

## 3. Variables du bootstrap IAM

Ajouter au `identity-bootstrap` :

```hcl
variable "github_oidc_subjects" {
  description = "Subject OIDC GitHub exact autorisé pour chaque environnement."
  type        = map(string)

  validation {
    condition = alltrue([
      for environment in ["dev", "qual", "prod"] :
      contains(keys(var.github_oidc_subjects), environment)
    ])

    error_message = "Un subject GitHub OIDC est requis pour dev, qual et prod."
  }
}
```

Exemple de `terraform.tfvars.example` :

```hcl
github_oidc_subjects = {
  dev  = "<EXACT_GITHUB_OIDC_SUB_DEV>"
  qual = "<EXACT_GITHUB_OIDC_SUB_QUAL>"
  prod = "<EXACT_GITHUB_OIDC_SUB_PROD>"
}
```

## 4. Federated Identity Credentials

Ajouter :

```hcl
resource "azurerm_federated_identity_credential" "github" {
  for_each = var.github_oidc_subjects

  name = "fic-github-${each.key}"

  user_assigned_identity_id = azurerm_user_assigned_identity.terraform[each.key].id

  audience = [
    "api://AzureADTokenExchange"
  ]

  issuer  = "https://token.actions.githubusercontent.com"
  subject = each.value
}
```

Nous créons une relation de confiance. Nous ne stockons aucun token GitHub dans Azure.

## 5. Audience

Pour l’échange standard Azure Workload Identity Federation :

```text
api://AzureADTokenExchange
```

est l’audience attendue.

## 6. Pourquoi un credential par environnement ?

Nous voulons que le contexte autorisé pour DEV ne soit pas automatiquement accepté par PROD.

```text
GitHub Environment dev
   │
   ▼
FIC DEV
   │
   ▼
UAMI DEV
```

Même logique pour QUALIF et PROD.

## 7. Corriger l’isolation du Remote State

Le LAB-09 avait identifié une limite :

```text
un container tfstate partagé
```

avec trois identities.

Nous allons créer :

```text
tfstate-dev
tfstate-qual
tfstate-prod
```

Ainsi, les permissions Data Plane peuvent être attribuées au scope exact d’un container.

## 8. Créer les containers

Dans le bootstrap du backend ou un stack IAM privilégié :

```hcl
resource "azurerm_storage_container" "environment_state" {
  for_each = toset([
    "dev",
    "qual",
    "prod"
  ])

  name = "tfstate-${each.key}"

  storage_account_id = data.azurerm_storage_account.tfstate.id

  container_access_type = "private"
}
```

Si le Storage Account désactive Shared Key, configurer le provider du bootstrap pour utiliser Microsoft Entra ID sur le data plane selon l’architecture du LAB-06.

## 9. Attribuer le rôle Data Plane

```hcl
resource "azurerm_role_assignment" "terraform_state" {
  for_each = toset([
    "dev",
    "qual",
    "prod"
  ])

  scope = azurerm_storage_container.environment_state[each.key].id

  role_definition_name = "Storage Blob Data Contributor"

  principal_id = azurerm_user_assigned_identity.terraform[each.key].principal_id

  principal_type = "ServicePrincipal"

  skip_service_principal_aad_check = true
}
```

Le scope exact peut dépendre de la ressource/provider utilisés pour matérialiser le container. Vérifier le Resource ID ARM du container dans votre version AzureRM et ne pas élargir automatiquement au Storage Account si ce n’est pas nécessaire.

## 10. Migrer les states

DEV passe de :

```text
container = tfstate
key       = terraform-azure-platform/dev/platform.tfstate
```

à :

```text
container = tfstate-dev
key       = platform.tfstate
```

Créer le nouveau fichier backend :

```hcl
storage_account_name = "<STORAGE_ACCOUNT>"
container_name       = "tfstate-dev"
key                  = "platform.tfstate"

use_azuread_auth = true
use_cli          = true
```

Puis localement :

```powershell
terraform init `
  -migrate-state `
  -backend-config="backend.azurerm.tfbackend"
```

Répéter pour QUALIF et PROD.

Ne jamais supprimer l’ancien blob avant d’avoir vérifié la migration et obtenu :

```text
terraform state list correct
terraform plan = No changes
```

## 11. GitHub Environments

Créer dans GitHub :

```text
dev
qual
prod
```

Ajouter dans chaque Environment les variables non secrètes :

```text
AZURE_CLIENT_ID
AZURE_TENANT_ID
AZURE_SUBSCRIPTION_ID
TF_BACKEND_STORAGE_ACCOUNT
TF_BACKEND_CONTAINER
TF_BACKEND_KEY
```

Les Client ID, Tenant ID et Subscription ID sont des identifiants, pas des Client Secrets.

## 12. Permissions GitHub Actions

Pour demander un OIDC token :

```yaml
permissions:
  contents: read
  id-token: write
```

Sans :

```yaml
id-token: write
```

GitHub ne permet pas au job de demander le jeton OIDC nécessaire.

## 13. Workflow PR Plan DEV

Créer :

```text
.github/workflows/terraform-plan-dev.yml
```

```yaml
name: Terraform Plan DEV

on:
  pull_request:
    branches:
      - main
    paths:
      - "modules/**"
      - "environments/dev/**"
      - ".github/workflows/terraform-plan-dev.yml"

permissions:
  contents: read
  id-token: write

jobs:
  plan:
    name: Terraform Plan DEV
    runs-on: ubuntu-latest

    environment: dev

    defaults:
      run:
        working-directory: environments/dev

    env:
      ARM_USE_OIDC: "true"
      ARM_USE_AZUREAD: "true"
      ARM_CLIENT_ID: ${{ vars.AZURE_CLIENT_ID }}
      ARM_TENANT_ID: ${{ vars.AZURE_TENANT_ID }}
      ARM_SUBSCRIPTION_ID: ${{ vars.AZURE_SUBSCRIPTION_ID }}

    steps:
      - name: Checkout
        uses: actions/checkout@v7

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v4
        with:
          terraform_version: ">=1.16.0,<2.0.0"
          terraform_wrapper: false

      - name: Azure login with OIDC
        uses: azure/login@v3
        with:
          client-id: ${{ vars.AZURE_CLIENT_ID }}
          tenant-id: ${{ vars.AZURE_TENANT_ID }}
          subscription-id: ${{ vars.AZURE_SUBSCRIPTION_ID }}

      - name: Terraform init
        run: |
          terraform init \
            -backend-config="storage_account_name=${{ vars.TF_BACKEND_STORAGE_ACCOUNT }}" \
            -backend-config="container_name=${{ vars.TF_BACKEND_CONTAINER }}" \
            -backend-config="key=${{ vars.TF_BACKEND_KEY }}" \
            -backend-config="use_oidc=true" \
            -backend-config="use_azuread_auth=true" \
            -backend-config="client_id=${ARM_CLIENT_ID}" \
            -backend-config="tenant_id=${ARM_TENANT_ID}"

      - name: Terraform validate
        run: terraform validate

      - name: Terraform plan
        run: terraform plan -no-color -out=tfplan

      - name: Show plan
        run: terraform show -no-color tfplan
```

## 14. Pourquoi `azure/login` si Terraform supporte directement OIDC ?

AzureRM Provider et Backend peuvent utiliser directement les variables du runtime GitHub OIDC.

Nous conservons néanmoins :

```yaml
azure/login@v3
```

dans ce Lab pour :

- obtenir un contexte Azure CLI utile au diagnostic ;
- utiliser `az account show` si nécessaire ;
- rendre l’identité utilisée visible dans les logs.

Terraform lui-même n’utilise pas un Client Secret fourni par cette action.

## 15. OIDC du Provider

Le provider peut détecter le mécanisme GitHub OIDC lorsque nous avons :

```text
ARM_USE_OIDC=true
ARM_CLIENT_ID
ARM_TENANT_ID
ARM_SUBSCRIPTION_ID
```

et les variables de runtime GitHub permettant de demander le token.

## 16. OIDC du Backend

Le backend doit lui aussi être configuré :

```text
use_oidc = true
use_azuread_auth = true
```

et l’identité doit posséder :

```text
Storage Blob Data Contributor
```

sur le container correspondant.

Avoir `Contributor` sur le Resource Group DEV ne suffit pas pour écrire le Blob State.

## 17. Tester l’identité

Ajouter temporairement avant `terraform init` :

```yaml
      - name: Show Azure context
        run: |
          az account show \
            --query "{Subscription:name,SubscriptionId:id,TenantId:tenantId}" \
            --output table
```

Nous ne devons jamais afficher un access token.

## 18. Tester une permission insuffisante

Retirer temporairement `Storage Blob Data Contributor` de l’UAMI DEV.

Le job doit échouer lors de l’accès backend.

Remettre ensuite le Role Assignment.

Ce test démontre que :

```text
Azure resource Contributor
≠
Blob data access
```

## 19. Tester une mauvaise fédération

Mettre volontairement un subject ne correspondant pas au workflow.

`azure/login` doit échouer avant même Terraform.

La sécurité OIDC repose précisément sur cette correspondance de claims.

Restaurer immédiatement le subject correct.

## 20. Pull Request provenant d’un fork

Un workflow avec accès cloud doit être conçu avec beaucoup de prudence pour les PR externes.

Ne pas exécuter automatiquement des opérations privilégiées sur du code non fiable provenant d’un fork.

Pour un repository privé d’infrastructure, limiter les contributeurs et protéger les workflows sensibles.

## 21. Ne pas utiliser `pull_request_target` naïvement

`pull_request_target` s’exécute dans un contexte privilégié par rapport au repository cible.

Exécuter du code provenant de la PR dans ce contexte peut créer un chemin d’exfiltration des credentials ou tokens.

Pour Terraform infrastructure, garder un modèle simple et explicitement revu.

## 22. OIDC n’élimine pas tous les risques

OIDC retire le Client Secret permanent.

Il reste nécessaire de sécuriser :

```text
repository
branches
GitHub Environments
workflow files
subject claims
RBAC
backend
review process
```

Une fédération trop permissive peut être aussi dangereuse qu’un secret mal protégé.

## 23. Supply chain des GitHub Actions

Pour une plateforme de production, considérer le pinning :

```yaml
uses: azure/login@<FULL_COMMIT_SHA>
```

plutôt qu’un tag mobile.

Même logique pour checkout et setup-terraform.

## 24. Validation

```text
[ ] FIC créé pour DEV
[ ] FIC créé pour QUALIF
[ ] FIC créé pour PROD
[ ] subject exact vérifié
[ ] audience AzureADTokenExchange configurée
[ ] containers backend séparés
[ ] states migrés
[ ] Storage Blob Data Contributor limité par environnement
[ ] GitHub Environments créés
[ ] variables Azure configurées
[ ] id-token: write configuré
[ ] azure/login OIDC fonctionne
[ ] terraform init OIDC fonctionne
[ ] terraform plan DEV fonctionne
[ ] aucun Client Secret Azure utilisé
```

## 25. Suite

Nous savons produire un plan Azure sécurisé depuis GitHub.

Il reste à construire la promotion et les `apply`.

**LAB-17 — Continuous Deployment DEV → QUALIF → PROD**

## Références officielles

- GitHub Docs — Configuring OpenID Connect in Azure
- Microsoft Learn — GitHub Actions Workload Identity Federation
- HashiCorp AzureRM — OIDC Authentication
- HashiCorp — AzureRM Backend OIDC
- Terraform Registry — `azurerm_federated_identity_credential`
