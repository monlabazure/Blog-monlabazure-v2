---
title: "LAB-10 — Azure Key Vault, secrets et Terraform"
date: 2026-10-15
draft: false
description: "Déployer Azure Key Vault avec Terraform, comprendre Management Plane, Data Plane, RBAC, données sensibles et éviter de stocker inutilement les secrets dans le Terraform State."
summary: "Créer un Key Vault en mode RBAC, séparer création de la ressource et accès aux secrets, puis adopter une stratégie qui limite l’exposition des secrets dans Terraform."
slug: "terraform-azure-lab-10-key-vault-secrets"
categories:
  - "Automatisation"
tags:
  - "Terraform"
  - "Azure"
  - "Key Vault"
  - "Secrets"
  - "RBAC"
  - "Microsoft Entra ID"
series:
  - "Terraform Azure — De zéro à production"
weight: 11
ShowToc: true
TocOpen: true
---

# LAB-10 — Azure Key Vault, secrets et Terraform

Dans le LAB-09, nous avons séparé identité humaine, identité Terraform et autorisations Azure RBAC. Nous allons maintenant appliquer ce modèle à une ressource qui oblige à comprendre une différence essentielle : **créer un Key Vault n’est pas la même chose que lire un secret dans ce Key Vault**.

Le premier besoin relève du **Management Plane**. Le second relève du **Data Plane**.

L’objectif du Lab n’est donc pas seulement de créer un coffre-fort. Nous allons surtout construire une manière propre de travailler avec les secrets sans transformer Terraform en gestionnaire de mots de passe.

## Architecture

```text
Terraform
   │
   │ Management Plane
   ▼
Azure Resource Manager
   │
   ▼
Key Vault
   │
   │ Data Plane
   ├── Secrets
   ├── Keys
   └── Certificates

Identité Terraform
   └── Contributor sur le Resource Group

Identité opérateur
   └── Key Vault Secrets Officer pendant le Lab
```

Le coffre sera créé dans le Resource Group DEV :

```text
rg-mla-dev-fce-001
```

Le nom du Key Vault doit être globalement unique. Nous allons donc le calculer à partir du projet, de l’environnement et d’un suffixe dérivé de la subscription.

## 1. Objectifs

À la fin de ce Lab, nous devons savoir :

- créer un Azure Key Vault avec Terraform ;
- activer le modèle d’autorisation Azure RBAC ;
- distinguer Management Plane et Data Plane ;
- comprendre `Key Vault Contributor`, `Key Vault Secrets User` et `Key Vault Secrets Officer` ;
- comprendre pourquoi `sensitive = true` ne chiffre pas le state ;
- éviter d’inscrire un vrai secret directement dans un fichier `.tf` ou `.tfvars` ;
- injecter un secret de démonstration hors Terraform ;
- vérifier les permissions avec Azure CLI ;
- préparer le passage à Private Endpoint dans le LAB-11.

## 2. Pourquoi ne pas utiliser `azurerm_key_vault_secret` partout ?

Terraform sait gérer une ressource :

```hcl
resource "azurerm_key_vault_secret" "example" {
  name         = "api-key"
  value        = var.api_key
  key_vault_id = azurerm_key_vault.main.id
}
```

Techniquement, cela fonctionne.

Mais la valeur du secret doit alors être connue de Terraform. Elle peut donc être persistée dans le plan ou dans le state selon l’usage.

Le fait d’écrire :

```hcl
variable "api_key" {
  type      = string
  sensitive = true
}
```

ne signifie pas que la valeur disparaît du state.

Pour ce Lab, nous allons donc appliquer une règle simple :

> Terraform crée le coffre, sa configuration et ses autorisations. La valeur du secret de démonstration est injectée en dehors de Terraform.

Cette stratégie n’est pas universelle, mais elle oblige à séparer proprement la gestion de l’infrastructure et la gestion de la valeur secrète.

## 3. Arborescence

Créer :

```text
lab-10/
├── versions.tf
├── providers.tf
├── variables.tf
├── locals.tf
├── main.tf
├── outputs.tf
└── terraform.tfvars.example
```

Le Lab suppose que le Resource Group DEV créé par le bootstrap du LAB-09 existe déjà.

## 4. `versions.tf`

```hcl
terraform {
  required_version = ">= 1.16.0, < 2.0.0"

  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 5.0"
    }
  }

  backend "azurerm" {}
}
```

## 5. `providers.tf`

```hcl
provider "azurerm" {
  features {}

  subscription_id = var.subscription_id
}
```

## 6. `variables.tf`

```hcl
variable "subscription_id" {
  description = "Subscription Azure cible."
  type        = string
  nullable    = false
}

variable "environment" {
  description = "Environnement cible."
  type        = string
  default     = "dev"

  validation {
    condition     = contains(["dev", "qual", "prod"], lower(var.environment))
    error_message = "environment doit être dev, qual ou prod."
  }
}

variable "location" {
  description = "Région Azure."
  type        = string
  default     = "francecentral"
}

variable "project_name" {
  description = "Nom court du projet."
  type        = string
  default     = "mla"
}

variable "owner" {
  description = "Équipe responsable."
  type        = string
  default     = "cloud-platform"
}
```

## 7. `locals.tf`

```hcl
locals {
  environment = lower(var.environment)
  location    = lower(var.location)

  location_codes = {
    francecentral = "fce"
    westeurope    = "weu"
  }

  location_code = local.location_codes[local.location]

  resource_group_name = "rg-${var.project_name}-${local.environment}-${local.location_code}-001"

  subscription_hash = substr(
    md5(var.subscription_id),
    0,
    8
  )

  key_vault_name = substr(
    "kv-${var.project_name}-${local.environment}-${local.subscription_hash}",
    0,
    24
  )

  common_tags = {
    environment = local.environment
    project     = var.project_name
    managed_by  = "terraform"
    owner       = var.owner
    lab         = "lab-10"
  }
}
```

Le suffixe n’est pas un secret. Il sert uniquement à réduire le risque de collision sur le nom global du Key Vault.

## 8. `main.tf`

```hcl
data "azurerm_client_config" "current" {}

data "azurerm_resource_group" "main" {
  name = local.resource_group_name
}

resource "azurerm_key_vault" "main" {
  name                = local.key_vault_name
  location            = data.azurerm_resource_group.main.location
  resource_group_name = data.azurerm_resource_group.main.name
  tenant_id           = data.azurerm_client_config.current.tenant_id

  sku_name = "standard"

  enable_rbac_authorization     = true
  public_network_access_enabled = true

  soft_delete_retention_days = 7
  purge_protection_enabled   = false

  network_acls {
    bypass         = "AzureServices"
    default_action = "Allow"
  }

  tags = local.common_tags
}
```

### Pourquoi RBAC ?

Nous utilisons :

```hcl
enable_rbac_authorization = true
```

afin que les permissions Data Plane soient accordées avec Azure RBAC plutôt qu’avec l’ancien modèle Access Policies.

Cela permet d’unifier davantage le modèle d’autorisation autour de :

```text
Principal
+
Role
+
Scope
```

## 9. Purge Protection

Dans le Lab :

```hcl
purge_protection_enabled = false
```

pour faciliter le nettoyage.

Pour une plateforme de production, nous évaluerons généralement :

```hcl
purge_protection_enabled = true
```

Une fois activée, cette protection ne doit pas être considérée comme un simple toggle de Lab. Son cycle de vie doit être pris en compte avant déploiement.

## 10. `outputs.tf`

```hcl
output "key_vault_name" {
  description = "Nom du Key Vault."
  value       = azurerm_key_vault.main.name
}

output "key_vault_id" {
  description = "Resource ID du Key Vault."
  value       = azurerm_key_vault.main.id
}

output "key_vault_uri" {
  description = "URI Data Plane du Key Vault."
  value       = azurerm_key_vault.main.vault_uri
}
```

## 11. `terraform.tfvars.example`

```hcl
subscription_id = "<SUBSCRIPTION_ID>"
environment     = "dev"
location        = "francecentral"
project_name    = "mla"
owner           = "cloud-platform"
```

Créer ensuite un `terraform.tfvars` local avec la bonne subscription.

## 12. Backend

Réutiliser le Storage Account du LAB-06 et utiliser une key dédiée :

```hcl
storage_account_name = "<BACKEND_STORAGE_ACCOUNT>"
container_name       = "tfstate"
key                  = "terraform-azure-platform/dev/key-vault.tfstate"

use_azuread_auth = true
use_cli          = true
```

Sauvegarder ce contenu dans :

```text
backend.azurerm.tfbackend
```

## 13. Initialiser et planifier

```powershell
terraform init `
  -backend-config="backend.azurerm.tfbackend"

terraform fmt -recursive
terraform validate
terraform plan -out="lab-10.tfplan"
```

Le plan doit créer un Key Vault, pas un secret.

Appliquer :

```powershell
terraform apply ".\lab-10.tfplan"
```

## 14. Vérifier le Management Plane

```powershell
$kvName = terraform output -raw key_vault_name

az keyvault show `
  --name $kvName `
  --query "{Name:name,Location:location,RBAC:properties.enableRbacAuthorization,PublicAccess:properties.publicNetworkAccess}" `
  --output json
```

La ressource existe.

Cela ne signifie pas encore que notre utilisateur peut lire ou écrire un secret.

## 15. Tester le Data Plane sans rôle

```powershell
az keyvault secret list `
  --vault-name $kvName `
  --output table
```

Selon les rôles déjà hérités dans votre environnement, la commande peut échouer avec une erreur d’autorisation.

C’est précisément la distinction recherchée : une identité capable de voir ou administrer la ressource Key Vault via ARM n’obtient pas automatiquement tous les droits sur son contenu.

## 16. Accorder temporairement le rôle Data Plane à l’opérateur

Récupérer l’Object ID de l’identité connectée :

```powershell
$operatorObjectId = az ad signed-in-user show `
  --query id `
  -o tsv
```

Récupérer le Resource ID du coffre :

```powershell
$kvId = terraform output -raw key_vault_id
```

Créer la Role Assignment :

```powershell
az role assignment create `
  --assignee-object-id $operatorObjectId `
  --assignee-principal-type User `
  --role "Key Vault Secrets Officer" `
  --scope $kvId
```

Attendre la propagation RBAC si nécessaire.

## 17. Injecter un secret de démonstration hors Terraform

Utiliser une valeur non sensible créée uniquement pour le Lab :

```powershell
az keyvault secret set `
  --vault-name $kvName `
  --name "demo-api-key" `
  --value "lab-only-value-change-me"
```

Dans une vraie exploitation, la valeur ne devrait pas être saisie de cette manière dans un terminal partagé ou enregistrée dans l’historique de commandes.

Elle viendrait par exemple d’un processus d’initialisation sécurisé, d’un outil de gestion des secrets ou d’une rotation applicative.

## 18. Lire le secret

```powershell
az keyvault secret show `
  --vault-name $kvName `
  --name "demo-api-key" `
  --query "{Name:name,Enabled:attributes.enabled,Id:id}" `
  --output json
```

Nous n’affichons pas la valeur.

Pour démontrer explicitement l’accès Data Plane :

```powershell
az keyvault secret show `
  --vault-name $kvName `
  --name "demo-api-key" `
  --query value `
  -o tsv
```

Ne pas utiliser cette variante dans une pipeline qui journalise automatiquement stdout.

## 19. Pourquoi ne pas exposer un secret en output Terraform ?

À éviter :

```hcl
output "secret_value" {
  value     = data.azurerm_key_vault_secret.example.value
  sensitive = true
}
```

Même si Terraform masque l’affichage dans certains contextes, nous avons fait entrer la valeur dans le graphe d’exécution et potentiellement dans le state.

Un output `sensitive` n’est pas un coffre-fort.

## 20. Role Matrix

| Besoin | Rôle typique |
|---|---|
| Créer/configurer le Key Vault | Contributor ou rôle management dédié |
| Lire la valeur d’un secret | Key Vault Secrets User |
| Créer/modifier/supprimer des secrets | Key Vault Secrets Officer |
| Administrer les rôles RBAC du coffre | rôle IAM privilégié approprié |

Le rôle exact doit toujours être adapté au besoin réel et au scope le plus petit possible.

## 21. Retirer le rôle temporaire

À la fin du test :

```powershell
az role assignment delete `
  --assignee-object-id $operatorObjectId `
  --role "Key Vault Secrets Officer" `
  --scope $kvId
```

Le secret reste dans le coffre.

L’opérateur perd son droit Data Plane.

## 22. Vérifier le state

```powershell
terraform state list
```

Nous devons voir le Key Vault mais aucun :

```text
azurerm_key_vault_secret
```

Le state connaît l’infrastructure du coffre, pas la valeur de notre secret de démonstration.

## 23. Limite actuelle

Le Key Vault reste accessible via le réseau public :

```hcl
public_network_access_enabled = true
```

Il est protégé par authentification et RBAC, mais son endpoint Data Plane est toujours joignable depuis Internet.

Dans une architecture plus stricte, nous voulons :

```text
Client
  │
  ▼
VNet
  │
  ▼
Private Endpoint
  │
  ▼
Key Vault
```

C’est l’objet du LAB-11.

## 24. Validation du LAB-10

Le Lab est terminé lorsque :

```text
[ ] Key Vault créé avec Terraform
[ ] enable_rbac_authorization = true
[ ] aucun secret réel n'est stocké dans Terraform
[ ] Management Plane compris
[ ] Data Plane compris
[ ] Key Vault Secrets Officer attribué temporairement
[ ] secret de démonstration créé hors Terraform
[ ] rôle temporaire supprimé
[ ] terraform plan retourne No changes
[ ] state inspecté
[ ] public network access identifié comme amélioration suivante
```

## 25. À retenir

Le point principal du Lab n’est pas le bloc `azurerm_key_vault`.

Il faut retenir :

```text
Terraform peut administrer le coffre
sans avoir besoin de connaître le contenu du coffre.
```

Et :

```text
sensitive
≠
chiffré
≠
absent du state
```

Le prochain Lab va rendre l’accès au Key Vault privé au niveau réseau.

**LAB-11 — Private Endpoint, Private DNS et réseau privé Azure**

## Références officielles

- Microsoft Learn — Azure Key Vault RBAC
- Microsoft Learn — Azure Key Vault security
- HashiCorp AzureRM — `azurerm_key_vault`
- HashiCorp — Sensitive Data in Terraform State
