---
title: "LAB-18 — Quality Gates, linting, sécurité et tests Terraform"
date: 2026-10-15
draft: false
description: "Ajouter des quality gates Terraform avec fmt, validate, TFLint, terraform test, Checkov et contrôles de plan dans GitHub Actions."
summary: "Transformer la CI Terraform en barrière de qualité reproductible avant le plan et le déploiement Azure."
slug: "terraform-azure-lab-18-quality-gates-tests-securite"
categories:
  - "Automatisation"
tags:
  - "Terraform"
  - "GitHub Actions"
  - "TFLint"
  - "Checkov"
  - "Terraform Test"
  - "Security"
series:
  - "Terraform Azure — De zéro à production"
weight: 19
ShowToc: true
TocOpen: true
---

# LAB-18 — Quality Gates, linting, sécurité et tests Terraform

Une pipeline qui exécute correctement `terraform apply` peut encore déployer une mauvaise architecture.

Nous voulons maintenant ajouter des barrières avant le déploiement :

```text
Code
 │
 ▼
fmt
 │
 ▼
validate
 │
 ▼
TFLint
 │
 ▼
terraform test
 │
 ▼
Security scan
 │
 ▼
terraform plan
 │
 ▼
Review
 │
 ▼
Apply
```

L’objectif n’est pas d’empiler des outils. Chaque contrôle doit répondre à une classe de risque précise.

## 1. Les Quality Gates

Nous retenons :

- `terraform fmt -check` pour le style HCL ;
- `terraform validate` pour la cohérence Terraform ;
- TFLint pour le linting ;
- `terraform test` pour les contrats de modules ;
- Checkov pour les mauvaises configurations de sécurité ;
- `terraform plan` pour l’impact réel sur un environnement ;
- revue humaine pour le contexte architectural.

Aucun outil ne remplace tous les autres.

## 2. Architecture CI

```text
Pull Request
   │
   ├── Format
   ├── Validate
   ├── TFLint
   ├── Unit tests Terraform
   ├── Checkov
   └── Terraform Plan DEV
          │
          ▼
       Reviewer
```

## 3. TFLint

Conserver :

```hcl
config {
  call_module_type = "all"
}

plugin "terraform" {
  enabled = true
  preset  = "recommended"
}
```

Puis localement :

```powershell
tflint --init
tflint --recursive -f compact
```

Un warning TFLint ne doit pas être automatiquement ignoré.

Décider :

```text
corriger
ou
justifier explicitement une exception
```

## 4. Terraform Test

Terraform fournit un framework de test natif.

Créer dans :

```text
modules/network/tests/
└── network.tftest.hcl
```

## 5. Test du module réseau avec provider mocké

```hcl
mock_provider "azurerm" {}

run "network_plan" {
  command = plan

  variables {
    resource_group_name = "rg-mla-test-fce-001"
    location            = "francecentral"

    virtual_network_name = "vnet-mla-test-fce-001"

    address_spaces = [
      "10.99.0.0/16"
    ]

    app_subnet_name = "snet-app-test-fce-001"

    app_subnet_prefixes = [
      "10.99.1.0/24"
    ]

    private_endpoints_subnet_name = "snet-pe-test-fce-001"

    private_endpoints_subnet_prefixes = [
      "10.99.2.0/24"
    ]

    container_apps_subnet_name = "snet-aca-test-fce-001"

    container_apps_subnet_prefixes = [
      "10.99.8.0/21"
    ]

    application_gateway_subnet_name = "snet-appgw-test-fce-001"

    application_gateway_subnet_prefixes = [
      "10.99.3.0/24"
    ]

    tags = {
      environment = "test"
    }
  }

  assert {
    condition = azurerm_virtual_network.this.name == "vnet-mla-test-fce-001"

    error_message = "Le nom du VNet n'est pas conforme à l'input fourni."
  }

  assert {
    condition = azurerm_subnet.app.name == "snet-app-test-fce-001"

    error_message = "Le Subnet application attendu n'est pas présent."
  }

  assert {
    condition = length(azurerm_virtual_network.this.address_space) == 1

    error_message = "Le VNet doit contenir un address space dans ce test."
  }
}
```

Le provider est mocké : nous pouvons tester la logique du module sans créer un VNet Azure réel.

## 6. Lancer les tests

Depuis le module :

```powershell
terraform init -backend=false
terraform test
```

Terraform cherche par défaut les fichiers :

```text
*.tftest.hcl
```

dans le dossier `tests`.

## 7. Tests avec infrastructure réelle

`terraform test` peut également exécuter des opérations qui créent réellement des ressources.

Cela peut :

- coûter de l’argent ;
- nécessiter des credentials ;
- laisser des ressources en cas de cleanup incomplet.

Nous séparons donc :

```text
tests mockés
```

et :

```text
tests d’intégration Azure
```

Les seconds doivent utiliser une subscription de test dédiée et une procédure de cleanup.

## 8. Checkov

Pour le Lab, nous épinglons une version connue au moment de la rédaction :

```powershell
python -m pip install "checkov==3.3.21"
```

Puis :

```powershell
checkov `
  -d . `
  --framework terraform `
  --compact
```

Avant publication ou utilisation ultérieure, vérifier la version maintenue par le projet Checkov.

## 9. Ce que cherche un scanner IaC

Exemples :

```text
Public network access
Weak TLS
Admin credentials
Missing encryption
Unsafe NSG rules
Missing logging
Insecure storage
Key Vault protections
```

Un finding n’est pas automatiquement une vulnérabilité exploitable.

Il doit être évalué dans le contexte de l’architecture.

## 10. Gérer les exceptions

Une mauvaise pratique serait :

```text
scanner échoue
      │
      ▼
désactiver tous les checks
```

Une exception doit être :

- ciblée ;
- justifiée ;
- tracée ;
- revue.

Exemple `.checkov.yml` :

```yaml
framework:
  - terraform

compact: true

soft-fail: false
```

Les exclusions spécifiques ne doivent être ajoutées qu’avec une justification documentée.

## 11. Workflow Quality

Créer :

```text
.github/workflows/terraform-quality.yml
```

```yaml
name: Terraform Quality

on:
  pull_request:
    branches:
      - main
    paths:
      - "**/*.tf"
      - "**/*.tftest.hcl"
      - ".tflint.hcl"
      - ".checkov.yml"
      - ".github/workflows/terraform-quality.yml"

permissions:
  contents: read

jobs:
  fmt:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v7

      - uses: hashicorp/setup-terraform@v4
        with:
          terraform_version: ">=1.16.0,<2.0.0"
          terraform_wrapper: false

      - run: terraform fmt -check -recursive -diff

  validate:
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
      - uses: actions/checkout@v7

      - uses: hashicorp/setup-terraform@v4
        with:
          terraform_version: ">=1.16.0,<2.0.0"
          terraform_wrapper: false

      - run: terraform init -backend=false
      - run: terraform validate

  tflint:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v7

      - uses: terraform-linters/setup-tflint@v6
        with:
          cache: true

      - run: tflint --init
        env:
          GITHUB_TOKEN: ${{ github.token }}

      - run: tflint --recursive -f compact

  terraform-test-network:
    runs-on: ubuntu-latest

    defaults:
      run:
        working-directory: modules/network

    steps:
      - uses: actions/checkout@v7

      - uses: hashicorp/setup-terraform@v4
        with:
          terraform_version: ">=1.16.0,<2.0.0"
          terraform_wrapper: false

      - run: terraform init -backend=false
      - run: terraform test

  checkov:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v7

      - uses: actions/setup-python@v7
        with:
          python-version: "3.13"

      - run: python -m pip install "checkov==3.3.21"

      - run: checkov -d . --framework terraform --compact
```

## 12. Versions des outils

Le workflow épingle volontairement Checkov à une version précise.

Pour les GitHub Actions, nous utilisons des tags majeurs dans le Lab pour la lisibilité.

Une politique production plus stricte peut épingler toutes les actions sur SHA.

Les mises à jour doivent alors être automatisées et revues.

## 13. Tests de validation métier

Les tests ne doivent pas uniquement vérifier que Terraform compile.

Exemple :

```hcl
assert {
  condition = (
    azurerm_container_registry.main.admin_enabled == false
  )

  error_message = "L'admin user ACR doit rester désactivé."
}
```

Autre exemple :

```hcl
assert {
  condition = (
    azurerm_key_vault.main.enable_rbac_authorization == true
  )

  error_message = "Key Vault doit utiliser Azure RBAC."
}
```

## 14. Preconditions

Terraform permet également de protéger certaines ressources avec des conditions.

Exemple conceptuel :

```hcl
lifecycle {
  precondition {
    condition = (
      var.environment != "prod" ||
      var.enable_public_access == false
    )

    error_message = "PROD ne peut pas activer l'accès public."
  }
}
```

Cela transforme une règle architecturale en contrainte exécutable.

## 15. `check` blocks

Pour des validations qui ne correspondent pas directement à une seule variable ou ressource, les `check` blocks peuvent exprimer des assertions sur la configuration ou l’infrastructure.

Ils ne doivent pas devenir un substitut à Azure Policy pour la gouvernance globale.

## 16. `terraform plan -detailed-exitcode`

Une commande utile en automatisation :

```bash
terraform plan -detailed-exitcode
```

Codes principaux :

```text
0 = aucun changement
1 = erreur
2 = changements présents
```

Attention : un code `2` n’est pas un échec Terraform.

Le workflow doit le traiter explicitement.

## 17. Quality Gate vs Azure Policy

Un scanner CI protège le chemin Git.

Azure Policy protège le périmètre Azure même si quelqu’un déploie autrement :

```text
Portal
CLI
Bicep
Terraform
autre pipeline
```

La cible mature combine :

```text
CI Quality Gates
+
Azure Policy
```

## 18. False positives

Un WAF public ou une Public IP Application Gateway peuvent être intentionnels.

Le scanner peut les signaler.

La bonne réponse est :

```text
comprendre
documenter
accepter explicitement
```

et non :

```text
ignorer silencieusement
```

## 19. Required checks

Sur `main`, rendre obligatoires :

```text
fmt
validate
tflint
terraform-test-network
checkov
terraform-plan-dev
```

Le code n’arrive plus sur `main` tant que les gates ne sont pas verts.

## 20. Qualité des modules

Chaque module partagé devrait progressivement avoir :

```text
README
Inputs documentés
Outputs documentés
Tests
Examples
Provider constraints
Versioning
Changelog si partagé
```

La qualité n’est pas uniquement une affaire de pipeline.

## 21. Validation

```text
[ ] TFLint actif
[ ] terraform test ajouté
[ ] provider mocké compris
[ ] assertions de module créées
[ ] Checkov exécuté
[ ] findings revus
[ ] exceptions documentées
[ ] detailed-exitcode compris
[ ] quality workflow créé
[ ] required checks configurés
[ ] distinction CI / Azure Policy comprise
```

## 22. Suite

Nos contrôles sont maintenant solides.

Le prochain Lab va volontairement casser la plateforme et construire une méthode de diagnostic reproductible.

**LAB-19 — Troubleshooting Terraform en conditions réelles**

## Références officielles

- HashiCorp — `terraform test`
- HashiCorp — Provider Mocking
- HashiCorp — `terraform validate`
- TFLint
- Checkov
