---
title: "LAB-17 — Continuous Deployment DEV → QUALIF → PROD"
date: 2026-10-15
draft: false
description: "Construire un pipeline GitHub Actions de déploiement Terraform avec promotion DEV, QUALIF et PROD, plans sauvegardés, OIDC, GitHub Environments et approbations."
summary: "Promouvoir le même commit Terraform entre trois environnements isolés avec un contrôle renforcé avant la production."
slug: "terraform-azure-lab-17-cd-dev-qual-prod"
categories:
  - "Automatisation"
tags:
  - "Terraform"
  - "GitHub Actions"
  - "CD"
  - "DEV"
  - "QUALIF"
  - "PROD"
  - "OIDC"
series:
  - "Terraform Azure — De zéro à production"
weight: 18
ShowToc: true
TocOpen: true
---

# LAB-17 — Continuous Deployment DEV → QUALIF → PROD

Nous avons maintenant :

- Pull Requests ;
- CI statique ;
- OIDC ;
- plan Azure depuis GitHub ;
- identité et state séparés par environnement.

Nous pouvons construire la chaîne de déploiement.

La cible est :

```text
Merge sur main
      │
      ▼
DEV plan/apply
      │
      ▼
DEV validation
      │
      ▼
QUAL plan/apply
      │
      ▼
QUAL validation
      │
      ▼
PROD approval
      │
      ▼
PROD plan/apply
```

Le même commit traverse les environnements.

## 1. Principe

Nous ne voulons pas :

```text
commit A en DEV
commit B en QUAL
commit C en PROD
```

pendant une même promotion.

Nous voulons promouvoir un SHA précis :

```text
commit A
   │
   ├── DEV
   ├── QUAL
   └── PROD
```

Cela rend les résultats beaucoup plus interprétables.

## 2. GitHub Environments

Nous avons déjà :

```text
dev
qual
prod
```

Configurer dans GitHub :

### DEV

- pas d’approbation manuelle ;
- identité DEV ;
- backend DEV.

### QUALIF

- éventuellement approbation ou règles de déploiement ;
- identité QUALIF ;
- backend QUALIF.

### PROD

- reviewers obligatoires ;
- identité PROD ;
- backend PROD ;
- éventuellement délai ou restrictions supplémentaires.

Les protections GitHub Environments sont une couche distincte de Terraform.

## 3. Concurrency

Deux déploiements du même environnement ne doivent pas s’exécuter simultanément.

Terraform possède déjà son state locking.

Nous ajoutons également une protection CI :

```yaml
concurrency:
  group: terraform-${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: false
```

Pour une gestion par environnement, chaque job peut également utiliser une logique de concurrency dédiée selon la stratégie du repository.

L’objectif est de ne pas considérer le state lock comme notre seul ordonnanceur.

## 4. Workflow CD

Créer :

```text
.github/workflows/terraform-cd.yml
```

## 5. Trigger

```yaml
name: Terraform CD

on:
  push:
    branches:
      - main
    paths:
      - "modules/**"
      - "environments/**"
      - ".github/workflows/terraform-cd.yml"

permissions:
  contents: read
  id-token: write
```

Nous déployons uniquement du code déjà intégré dans `main`.

## 6. Job DEV

```yaml
jobs:
  deploy-dev:
    name: Deploy DEV
    runs-on: ubuntu-latest

    environment: dev

    concurrency:
      group: terraform-dev
      cancel-in-progress: false

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

      - name: Azure login
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
        run: terraform plan -out=tfplan

      - name: Terraform apply
        run: terraform apply -auto-approve tfplan
```

Le `-auto-approve` est acceptable ici parce que nous appliquons **le plan sauvegardé du job**, dans une pipeline contrôlée. L’approbation humaine éventuelle est gérée au niveau GitHub Environment, pas via stdin Terraform.

## 7. Job QUALIF

Ajouter :

```yaml
  deploy-qual:
    name: Deploy QUALIF
    runs-on: ubuntu-latest

    needs:
      - deploy-dev

    environment: qual

    concurrency:
      group: terraform-qual
      cancel-in-progress: false

    defaults:
      run:
        working-directory: environments/qual

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

      - name: Azure login
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

      - name: Terraform plan
        run: terraform plan -out=tfplan

      - name: Terraform apply
        run: terraform apply -auto-approve tfplan
```

QUALIF ne commence que si DEV réussit.

## 8. Job PROD

```yaml
  deploy-prod:
    name: Deploy PROD
    runs-on: ubuntu-latest

    needs:
      - deploy-qual

    environment: prod

    concurrency:
      group: terraform-prod
      cancel-in-progress: false

    defaults:
      run:
        working-directory: environments/prod

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

      - name: Azure login
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

      - name: Terraform plan
        run: terraform plan -out=tfplan

      - name: Show plan
        run: terraform show -no-color tfplan

      - name: Terraform apply
        run: terraform apply -auto-approve tfplan
```

La protection réelle est configurée sur GitHub Environment `prod` :

```text
Required reviewers
```

Le job s’arrête avant l’exécution jusqu’à approbation.

## 9. Pourquoi refaire un plan à chaque environnement ?

DEV, QUALIF et PROD possèdent :

- states différents ;
- Resource IDs différents ;
- paramètres différents ;
- éventuellement subscriptions différentes.

Nous ne pouvons pas générer un plan DEV puis l’appliquer en PROD.

Un plan Terraform est lié au contexte dans lequel il a été calculé.

## 10. Plan PR vs plan de déploiement

Le plan de Pull Request donne une prévisualisation avant merge.

Le plan de CD est recalculé après merge sur le commit exact et juste avant l’apply.

Il peut exister une différence si l’infrastructure a changé entre les deux moments.

C’est pourquoi le plan d’apply doit être généré dans le même job que son application.

## 11. Tests entre environnements

Dans une vraie chaîne, ajouter entre :

```text
DEV apply
```

et :

```text
QUAL apply
```

des tests :

```text
Smoke test
Health check
Integration test
Policy verification
```

Même principe avant PROD.

## 12. Exemple de smoke test DEV

Après l’apply DEV :

```yaml
      - name: Smoke test
        run: |
          echo "Ajouter ici le test fonctionnel de la plateforme DEV"
```

Lorsque l’Application Gateway est disponible :

```bash
curl --fail --retry 5 "https://<dev-domain>/health"
```

Éviter de coder en dur le domaine. Il peut venir d’un output Terraform ou d’une variable d’environnement.

## 13. Utiliser les outputs

Exemple :

```bash
APP_URL=$(terraform output -raw application_url)
curl --fail "$APP_URL/health"
```

Attention à ne pas publier des outputs sensibles dans les logs.

## 14. Échec DEV

Si DEV échoue :

```text
deploy-dev = failed
```

alors :

```text
deploy-qual = skipped
deploy-prod = skipped
```

La promotion s’arrête.

## 15. Échec QUALIF

Même principe :

```text
DEV = success
QUAL = failed
PROD = skipped
```

Nous ne promouvons pas un changement qui n’a pas passé l’environnement précédent.

## 16. Approval PROD

Le reviewer PROD doit vérifier :

- SHA déployé ;
- résultat DEV ;
- résultat QUALIF ;
- plan PROD ;
- actions de destruction ;
- remplacements ;
- impact applicatif ;
- fenêtre de changement si nécessaire.

L’approbation ne doit pas être un simple clic automatique.

## 17. Drift avant PROD

Si quelqu’un modifie PROD manuellement après la PR, le plan PROD peut contenir un drift inattendu.

La pipeline ne doit pas ignorer ce changement.

Le reviewer doit comprendre le plan réellement généré.

## 18. Rollback

Ne pas construire un bouton qui fait simplement :

```text
terraform apply ancien tfplan
```

Un plan sauvegardé peut devenir périmé si l’état distant a changé.

Le rollback normal consiste à :

```text
revenir à une configuration Git connue
      │
      ▼
nouveau terraform plan
      │
      ▼
review
      │
      ▼
apply
```

## 19. Destroy en pipeline

Ne pas intégrer automatiquement :

```text
terraform destroy
```

au workflow CD normal.

Une destruction d’environnement doit disposer d’un workflow dédié, déclenché manuellement et fortement protégé.

## 20. Séparer déploiement et bootstrap IAM

Le workflow workload n’utilise pas une identité capable d’administrer librement Azure RBAC.

Les changements IAM privilégiés restent dans un pipeline/bootstrap séparé.

Nous préservons la séparation introduite au LAB-09.

## 21. GitHub Actions Environment PROD

Le nom du GitHub Environment est également utilisé dans le modèle de confiance OIDC selon la politique choisie.

Ne pas renommer un Environment sans vérifier :

- Federated Identity Credential ;
- OIDC subject ;
- workflow ;
- protections.

## 22. Concurrency et Terraform Locking

Deux niveaux :

```text
GitHub concurrency
      │
      ▼
évite de lancer plusieurs pipelines du même env

Terraform state lock
      │
      ▼
protège les écritures du state
```

Ils sont complémentaires.

## 23. Validation

```text
[ ] CD déclenchée uniquement après merge main
[ ] DEV utilise identité DEV
[ ] QUAL utilise identité QUAL
[ ] PROD utilise identité PROD
[ ] states différents
[ ] jobs chaînés avec needs
[ ] concurrency par environnement
[ ] plan généré juste avant apply
[ ] plan sauvegardé appliqué
[ ] prod GitHub Environment protégé
[ ] approval PROD testé
[ ] rollback par nouveau plan compris
[ ] destroy séparé du pipeline normal
```

## 24. Suite

La plateforme sait maintenant se déployer automatiquement.

Nous allons ajouter les contrôles qualité qui empêchent un code dangereux d’arriver jusqu’au plan.

**LAB-18 — Quality Gates, linting, sécurité et tests Terraform**

## Références officielles

- GitHub Docs — Environments
- GitHub Docs — Deployment protection rules
- GitHub Docs — Workflow concurrency
- HashiCorp — Terraform Automation
- HashiCorp — State Locking
