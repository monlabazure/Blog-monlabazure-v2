---
title: "LAB-15 — Continuous Integration Terraform avec GitHub Actions"
date: 2026-10-15
draft: false
description: "Construire une CI Terraform avec GitHub Actions : format, init sans backend, validate et TFLint avant toute authentification Azure."
summary: "Automatiser les contrôles statiques Terraform sur chaque Pull Request sans donner encore de permissions Azure à la pipeline."
slug: "terraform-azure-lab-15-ci-github-actions"
categories:
  - "Automatisation"
tags:
  - "Terraform"
  - "GitHub Actions"
  - "CI"
  - "TFLint"
  - "Validation"
series:
  - "Terraform Azure — De zéro à production"
weight: 16
ShowToc: true
TocOpen: true
---

# LAB-15 — Continuous Integration Terraform avec GitHub Actions

Le LAB-14 a introduit la Pull Request comme unité de changement. Nous allons maintenant ajouter une première CI.

La CI de ce Lab ne possède **aucun accès Azure**.

Elle vérifie ce qui peut être contrôlé sans credential cloud :

```text
Pull Request
    │
    ▼
Checkout
    │
    ▼
Terraform setup
    │
    ├── fmt
    ├── init -backend=false
    ├── validate
    └── TFLint
```

Le `terraform plan` distant sera ajouté au LAB-16 avec OIDC.

## 1. Pourquoi commencer sans accès Azure ?

Une pipeline CI ne doit pas obtenir des permissions cloud simplement parce qu’elle vérifie le format HCL.

Nous appliquons :

```text
besoin
  │
  ▼
permission minimale
```

La CI statique a besoin de lire le code. Pas de modifier Azure.

## 2. Arborescence

Créer :

```text
.github/
└── workflows/
    └── terraform-ci.yml
```

Nous utiliserons :

- `actions/checkout@v7` ;
- `hashicorp/setup-terraform@v4` ;
- `terraform-linters/setup-tflint@v6`.

Les versions majeures doivent être régulièrement revues. Pour une politique de supply-chain plus stricte, les actions peuvent être épinglées sur un commit SHA complet.

## 3. Workflow CI

Créer :

```yaml
name: Terraform CI

on:
  pull_request:
    branches:
      - main
    paths:
      - "**/*.tf"
      - "**/*.tfvars.example"
      - ".github/workflows/terraform-ci.yml"
      - ".tflint.hcl"

permissions:
  contents: read

jobs:
  terraform-format:
    name: Terraform Format
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v7

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v4
        with:
          terraform_version: ">=1.16.0,<2.0.0"
          terraform_wrapper: false

      - name: Terraform fmt
        run: terraform fmt -check -recursive -diff

  terraform-validate:
    name: Validate ${{ matrix.environment }}
    runs-on: ubuntu-latest

    strategy:
      fail-fast: false

      matrix:
        environment:
          - dev
          - qual
          - prod

    defaults:
      run:
        working-directory: environments/${{ matrix.environment }}

    steps:
      - name: Checkout
        uses: actions/checkout@v7

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v4
        with:
          terraform_version: ">=1.16.0,<2.0.0"
          terraform_wrapper: false

      - name: Terraform init without backend
        run: terraform init -backend=false

      - name: Terraform validate
        run: terraform validate

  tflint:
    name: TFLint
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v7

      - name: Setup TFLint
        uses: terraform-linters/setup-tflint@v6
        with:
          cache: true

      - name: TFLint init
        run: tflint --init

      - name: TFLint
        run: tflint --recursive -f compact
```

## 4. Pourquoi `terraform init -backend=false` ?

`terraform validate` a besoin des providers et modules.

Mais la CI statique n’a aucune raison d’accéder au Remote State.

Nous initialisons donc :

```text
providers
modules
```

sans initialiser :

```text
backend
```

Cette séparation est particulièrement utile pour les modules réutilisables et les contrôles de Pull Request.

## 5. `.tflint.hcl`

Créer à la racine :

```hcl
config {
  call_module_type = "all"
}

plugin "terraform" {
  enabled = true
  preset  = "recommended"
}
```

TFLint peut être enrichi plus tard avec des plugins spécifiques.

Le premier objectif est de détecter des problèmes de code que `terraform validate` ne cherche pas nécessairement.

## 6. `fmt` et `validate` ne font pas la même chose

```text
terraform fmt
      │
      └── style HCL

terraform validate
      │
      └── syntaxe + cohérence interne

TFLint
      │
      └── règles de lint supplémentaires
```

Une configuration peut être bien formatée mais incorrecte.

Une configuration peut être valide mais présenter des problèmes de qualité.

## 7. Pourquoi une matrix ?

Nous avons trois Root Modules :

```text
environments/dev
environments/qual
environments/prod
```

Une modification d’un module partagé peut casser les trois.

La matrix vérifie donc chaque consommateur.

## 8. `fail-fast: false`

Si DEV échoue, nous voulons quand même savoir si QUALIF et PROD échouent aussi.

Cela donne davantage d’informations dans une seule exécution de CI.

## 9. Tester une erreur de format

Modifier volontairement :

```hcl
variable "environment" { type=string }
```

Pousser la branche.

Le job `Terraform Format` doit échouer.

Corriger localement :

```powershell
terraform fmt -recursive
```

Puis committer.

## 10. Tester une erreur Terraform

Créer temporairement :

```hcl
resource "azurerm_resource_group" "broken" {
  property_that_does_not_exist = "x"
}
```

`terraform validate` doit échouer.

Retirer immédiatement ce code après le test.

## 11. Branch Protection

Une fois le workflow fonctionnel, configurer `main` pour exiger les checks :

```text
Terraform Format
Validate dev
Validate qual
Validate prod
TFLint
```

La branche principale ne dépend plus seulement de la discipline humaine.

## 12. Pas de `terraform apply` en PR

Une Pull Request provenant d’un code non encore accepté ne doit pas modifier l’infrastructure.

Le pipeline CI reste en lecture du code.

Même au LAB-16, le PR workflow ajoutera `terraform plan`, pas `apply`.

## 13. Pas encore de `terraform plan`

Un plan réaliste doit accéder :

- au backend ;
- au state ;
- aux APIs Azure ;
- à l’identité de l’environnement.

Nous n’allons pas ajouter un Client Secret uniquement pour obtenir un plan.

Nous allons d’abord construire l’authentification OIDC.

## 14. Caching

Les premières versions de la CI restent simples.

Nous n’ajoutons pas immédiatement :

```text
cache providers
cache plugins
cache modules
```

Une optimisation doit venir après une mesure du temps de pipeline, pas avant la compréhension du workflow.

## 15. Épinglage des actions

Pour un Lab :

```yaml
uses: actions/checkout@v7
```

est lisible.

Pour une organisation avec exigences supply-chain élevées, préférer :

```yaml
uses: actions/checkout@<FULL_COMMIT_SHA>
```

et automatiser les mises à jour via Dependabot ou Renovate.

Le même principe s’applique à :

```text
azure/login
setup-terraform
setup-tflint
```

## 16. Résultat attendu

Une Pull Request doit afficher quelque chose proche de :

```text
✓ Terraform Format
✓ Validate dev
✓ Validate qual
✓ Validate prod
✓ TFLint
```

Si l’un des checks échoue, le merge est bloqué par la protection de branche.

## 17. Validation

```text
[ ] workflow terraform-ci.yml créé
[ ] permissions contents: read uniquement
[ ] checkout configuré
[ ] setup-terraform configuré
[ ] fmt -check exécuté
[ ] init -backend=false exécuté
[ ] validate exécuté sur DEV/QUAL/PROD
[ ] TFLint exécuté
[ ] erreur volontaire détectée
[ ] branch protection préparée
[ ] aucun credential Azure utilisé
```

## 18. Suite

La CI est correcte mais incapable de produire le plan Azure.

Nous allons maintenant connecter GitHub et Microsoft Entra ID sans Client Secret.

**LAB-16 — GitHub Actions + Azure OIDC pour Terraform**

## Références officielles

- HashiCorp — `terraform validate`
- HashiCorp — setup-terraform
- GitHub Actions documentation
- TFLint documentation
