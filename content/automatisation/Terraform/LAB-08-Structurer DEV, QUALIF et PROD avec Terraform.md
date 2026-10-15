---
title: "LAB-08 — Structurer DEV, QUALIF et PROD avec Terraform"
date: 2026-10-08
draft: false
description: "Concevoir une architecture Terraform multi-environnements avec DEV, QUALIF et PROD séparés, modules partagés, Remote States distincts et configurations Azure dédiées."
summary: "Comparer workspaces, tfvars, dossiers et repositories puis construire une architecture Terraform DEV, QUALIF et PROD avec un Root Module et un Remote State par environnement."
slug: "terraform-azure-lab-08-dev-qual-prod"
categories:
  - "Automatisation"
tags:
  - "Terraform"
  - "Azure"
  - "Environments"
  - "DEV"
  - "QUALIF"
  - "PROD"
  - "Remote State"
  - "Modules"
  - "AzureRM"
series:
  - "Terraform Azure — De zéro à production"
weight: 9
ShowToc: true
TocOpen: true
---

# LAB-08 — Structurer DEV, QUALIF et PROD avec Terraform

Jusqu'ici, nous avons essentiellement travaillé sur un seul environnement :

```text
DEV
```

Notre plateforme commence pourtant à contenir les briques d'une architecture plus sérieuse :

```text
Modules
Remote State
Azure Storage
State Locking
Microsoft Entra ID
RBAC
```

Il faut maintenant résoudre une question structurante :

> Comment utiliser le même code Terraform pour DEV, QUALIF et PROD sans mélanger leurs states, leurs paramètres et leurs déploiements ?

Une mauvaise stratégie multi-environnements peut provoquer des incidents beaucoup plus graves qu'une erreur HCL.

Le risque n'est plus simplement :

```text
Terraform syntax error
```

mais :

```text
Code DEV
   │
   ▼
State PROD
   │
   ▼
Infrastructure PROD
```

Nous allons donc comparer plusieurs approches avant de retenir celle utilisée pour la suite de MonLabAzure.

---

# 1. Objectifs du Lab

Nous allons comprendre et comparer :

- un seul répertoire avec plusieurs `.tfvars` ;
- Terraform CLI Workspaces ;
- un répertoire par environnement ;
- plusieurs repositories ;
- plusieurs Root Modules ;
- plusieurs Remote States ;
- séparation des backend keys ;
- séparation des variables ;
- séparation des subscriptions Azure ;
- modules partagés ;
- promotion DEV → QUALIF → PROD ;
- blast radius ;
- isolation des environnements ;
- verrouillage des versions des providers ;
- préparation des futures pipelines CI/CD.

Notre architecture cible sera :

```text
1 environnement
      =
1 Root Module
      =
1 Remote State
      =
1 backend key
      =
1 frontière de déploiement
      =
1 future pipeline
```

---

# 2. Les environnements

Nous utiliserons :

```text
DEV
```

pour le développement et les premiers tests.

```text
QUALIF
```

pour la validation avant production.

```text
PROD
```

pour la plateforme de production.

Nous utiliserons dans Terraform les identifiants :

```text
dev
qual
prod
```

---

# 3. Architecture Azure cible

Pour le Lab, les trois environnements peuvent être placés dans la même subscription Azure.

Nous créerons :

```text
Subscription Azure
│
├── rg-mla-dev-fce-001
│    │
│    └── vnet-mla-dev-fce-001
│         ├── 10.20.1.0/24
│         └── 10.20.2.0/24
│
├── rg-mla-qual-fce-001
│    │
│    └── vnet-mla-qual-fce-001
│         ├── 10.30.1.0/24
│         └── 10.30.2.0/24
│
└── rg-mla-prod-fce-001
     │
     └── vnet-mla-prod-fce-001
          ├── 10.40.1.0/24
          └── 10.40.2.0/24
```

Les address spaces seront :

```text
DEV
10.20.0.0/16

QUALIF
10.30.0.0/16

PROD
10.40.0.0/16
```

---

# 4. Architecture de production possible

Dans une vraie Landing Zone Azure, nous pourrions aller plus loin :

```text
Management Group
│
├── Non-Production
│    │
│    ├── DEV Subscription
│    └── QUALIF Subscription
│
└── Production
     │
     └── PROD Subscription
```

voire :

```text
DEV
│
└── identité Terraform DEV

QUALIF
│
└── identité Terraform QUALIF

PROD
│
└── identité Terraform PROD
```

avec des droits distincts.

Le Lab fonctionne avec une seule subscription afin de ne pas exiger trois subscriptions Azure.

La structure Terraform sera toutefois compatible avec une séparation future.

---

# 5. Première stratégie — un seul Root Module + plusieurs `.tfvars`

Une organisation simple pourrait être :

```text
terraform/
│
├── main.tf
├── variables.tf
├── providers.tf
│
└── environments/
     ├── dev.tfvars
     ├── qual.tfvars
     └── prod.tfvars
```

Nous exécuterions :

```powershell
terraform plan `
  -var-file="environments\dev.tfvars"
```

ou :

```powershell
terraform plan `
  -var-file="environments\prod.tfvars"
```

Cette approche est simple.

Mais les `.tfvars` ne séparent pas le state.

---

# 6. Le risque des `.tfvars` seuls

Imaginons :

```text
terraform.tfstate
```

contenant DEV.

Puis nous exécutons :

```powershell
terraform plan `
  -var-file="prod.tfvars"
```

Terraform peut interpréter cela comme :

> transformer l'infrastructure actuellement suivie par le state DEV pour qu'elle corresponde aux paramètres PROD.

Un fichier `.tfvars` ne constitue donc pas une frontière de state.

---

# 7. Règle importante

```text
tfvars
≠
state isolation
```

Les `.tfvars` permettent de modifier les inputs.

Ils ne créent pas automatiquement une nouvelle unité de déploiement.

---

# 8. Deuxième stratégie — Terraform CLI Workspaces

Terraform CLI propose :

```powershell
terraform workspace list
```

Par défaut :

```text
* default
```

Nous pourrions créer :

```powershell
terraform workspace new dev
terraform workspace new qual
terraform workspace new prod
```

Chaque workspace possède alors son propre state logique.

---

# 9. Architecture avec CLI Workspaces

Conceptuellement :

```text
Même configuration
       │
       ├── workspace dev
       │      └── state DEV
       │
       ├── workspace qual
       │      └── state QUALIF
       │
       └── workspace prod
              └── state PROD
```

Cela peut être utile dans certains scénarios.

---

# 10. Pourquoi nous ne retenons pas les CLI Workspaces

Les CLI Workspaces utilisent toujours la même configuration et le même backend.

Ils ne constituent donc pas une frontière forte lorsqu'on souhaite avoir :

```text
credentials DEV

credentials QUALIF

credentials PROD
```

ou :

```text
permissions différentes

backend différent

approbations différentes

pipelines différentes
```

Nous voulons pouvoir traiter PROD comme une cible explicitement distincte.

Nous ne retiendrons donc pas les CLI Workspaces comme mécanisme principal de séparation.

---

# 11. CLI Workspace et HCP Terraform Workspace

Attention à une confusion fréquente.

Le terme :

```text
workspace
```

est également utilisé dans HCP Terraform et Terraform Enterprise.

Un HCP Terraform Workspace représente une unité opérationnelle plus complète avec notamment :

- state ;
- variables ;
- runs ;
- permissions ;
- configuration.

Ce n'est pas exactement le même modèle que :

```powershell
terraform workspace new
```

utilisé avec Terraform CLI.

Dans cette série, nous travaillons avec :

```text
Terraform CLI
+
Azure Storage Backend
```

Nous parlons donc ici des **CLI Workspaces**.

---

# 12. Troisième stratégie — un répertoire par environnement

Architecture :

```text
terraform/
│
├── modules/
│
├── environments/
│   ├── dev/
│   ├── qual/
│   └── prod/
```

Chaque environnement possède :

```text
backend
provider
variables
configuration
state
```

indépendants.

C'est l'approche que nous allons retenir.

---

# 13. Quatrième stratégie — repositories séparés

Nous pourrions également avoir :

```text
terraform-platform-dev

terraform-platform-qual

terraform-platform-prod
```

Cela apporte une isolation importante.

Mais cela peut également provoquer :

- duplication ;
- divergence entre repositories ;
- promotion plus complexe ;
- davantage de maintenance.

Cette stratégie peut être pertinente lorsque les environnements appartiennent à des équipes ou périmètres de sécurité complètement différents.

Ce n'est pas nécessaire pour notre plateforme actuelle.

---

# 14. Architecture retenue

Nous choisissons :

```text
terraform-azure-platform/
│
├── modules/
│
│   ├── network/
│
│   └── network-security/
│
└── environments/
    │
    ├── dev/
    │
    ├── qual/
    │
    └── prod/
```

Les modules sont partagés.

Les Root Modules sont séparés.

---

# 15. Notre règle d'architecture

À partir de ce Lab :

```text
DEV
=
environments/dev
=
Root Module DEV
=
State DEV

QUALIF
=
environments/qual
=
Root Module QUALIF
=
State QUALIF

PROD
=
environments/prod
=
Root Module PROD
=
State PROD
```

---

# 16. Pourquoi un Root Module par environnement ?

Le Root Module représente une unité de déploiement.

DEV et PROD ne devraient pas être simplement deux valeurs dans le même state.

Nous voulons pouvoir lancer :

```powershell
terraform plan
```

dans DEV sans que Terraform puisse modifier le state PROD.

---

# 17. Blast radius

La notion de **blast radius** représente ici l'étendue potentielle des conséquences d'une opération.

Avec :

```text
1 énorme state
```

une erreur peut affecter une grande partie de la plateforme.

Avec :

```text
state DEV

state QUALIF

state PROD
```

nous limitons davantage les frontières d'impact.

La séparation des states n'élimine pas tous les risques.

Elle réduit la surface concernée par une opération Terraform donnée.

---

# 18. Backend cible

Le Storage Account créé dans LAB-06 est conservé.

Nous utiliserons trois blobs :

```text
tfstate/
│
└── terraform-azure-platform/
     │
     ├── dev/
     │    └── platform.tfstate
     │
     ├── qual/
     │    └── platform.tfstate
     │
     └── prod/
          └── platform.tfstate
```

---

# 19. Trois environnements, trois locks

Puisque chaque environnement possède un blob différent :

```text
DEV state
```

peut être verrouillé indépendamment de :

```text
QUAL state
```

et :

```text
PROD state
```

Un `terraform apply` DEV n'a donc pas besoin de verrouiller le state PROD.

---

# 20. Préparer LAB-08

Depuis PowerShell :

```powershell
Set-Location "C:\Labs\MonLabAzure\terraform-azure-platform"
```

Créer :

```powershell
New-Item `
  -ItemType Directory `
  -Name "lab-08"

Set-Location ".\lab-08"
```

Puis :

```powershell
New-Item `
  -ItemType Directory `
  -Path ".\modules\network" `
  -Force

New-Item `
  -ItemType Directory `
  -Path ".\modules\network-security" `
  -Force

New-Item `
  -ItemType Directory `
  -Path ".\environments\dev" `
  -Force

New-Item `
  -ItemType Directory `
  -Path ".\environments\qual" `
  -Force

New-Item `
  -ItemType Directory `
  -Path ".\environments\prod" `
  -Force
```

---

# 21. Arborescence finale

Nous allons obtenir :

```text
lab-08/
│
├── modules/
│   │
│   ├── network/
│   │   ├── versions.tf
│   │   ├── variables.tf
│   │   ├── main.tf
│   │   └── outputs.tf
│   │
│   └── network-security/
│       ├── versions.tf
│       ├── variables.tf
│       ├── main.tf
│       └── outputs.tf
│
└── environments/
    │
    ├── dev/
    │   ├── backend.tf
    │   ├── versions.tf
    │   ├── providers.tf
    │   ├── variables.tf
    │   ├── locals.tf
    │   ├── main.tf
    │   ├── outputs.tf
    │   └── terraform.tfvars.example
    │
    ├── qual/
    │   └── mêmes fichiers
    │
    └── prod/
        └── mêmes fichiers
```

Localement, chaque environnement possédera aussi :

```text
terraform.tfvars
backend.azurerm.tfbackend
.terraform/
.terraform.lock.hcl
```

---

# 22. Modules partagés

Nous allons reprendre les deux modules du LAB-07 :

```text
network

network-security
```

Ils seront identiques pour DEV, QUALIF et PROD.

Nous ne créerons pas :

```text
network-dev

network-qual

network-prod
```

La différence doit venir des inputs.

---

# 23. `modules/network/versions.tf`

```hcl
terraform {
  required_version = ">= 1.16.0, < 2.0.0"

  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = ">= 5.0, < 6.0"
    }
  }
}
```

---

# 24. `modules/network/variables.tf`

```hcl
variable "resource_group_name" {
  description = "Resource Group contenant les ressources réseau."
  type        = string
  nullable    = false
}

variable "location" {
  description = "Région Azure."
  type        = string
  nullable    = false
}

variable "virtual_network_name" {
  description = "Nom du Virtual Network."
  type        = string
  nullable    = false
}

variable "address_spaces" {
  description = "Address spaces du Virtual Network."
  type        = list(string)
  nullable    = false

  validation {
    condition     = length(var.address_spaces) > 0
    error_message = "address_spaces doit contenir au moins une plage."
  }
}

variable "app_subnet_name" {
  description = "Nom du Subnet application."
  type        = string
  nullable    = false
}

variable "app_subnet_prefixes" {
  description = "Préfixes du Subnet application."
  type        = list(string)
  nullable    = false
}

variable "private_endpoints_subnet_name" {
  description = "Nom du Subnet Private Endpoints."
  type        = string
  nullable    = false
}

variable "private_endpoints_subnet_prefixes" {
  description = "Préfixes du Subnet Private Endpoints."
  type        = list(string)
  nullable    = false
}

variable "tags" {
  description = "Tags Azure."
  type        = map(string)
  default     = {}
  nullable    = false
}
```

---

# 25. `modules/network/main.tf`

```hcl
resource "azurerm_virtual_network" "this" {
  name                = var.virtual_network_name
  location            = var.location
  resource_group_name = var.resource_group_name
  address_space       = var.address_spaces

  tags = var.tags
}

resource "azurerm_subnet" "app" {
  name = var.app_subnet_name

  resource_group_name  = var.resource_group_name
  virtual_network_name = azurerm_virtual_network.this.name

  address_prefixes = var.app_subnet_prefixes
}

resource "azurerm_subnet" "private_endpoints" {
  name = var.private_endpoints_subnet_name

  resource_group_name  = var.resource_group_name
  virtual_network_name = azurerm_virtual_network.this.name

  address_prefixes = var.private_endpoints_subnet_prefixes
}
```

---

# 26. `modules/network/outputs.tf`

```hcl
output "virtual_network_id" {
  description = "Resource ID du Virtual Network."
  value       = azurerm_virtual_network.this.id
}

output "virtual_network_name" {
  description = "Nom du Virtual Network."
  value       = azurerm_virtual_network.this.name
}

output "subnet_ids" {
  description = "Resource IDs des Subnets."

  value = {
    app               = azurerm_subnet.app.id
    private_endpoints = azurerm_subnet.private_endpoints.id
  }
}
```

---

# 27. `modules/network-security/versions.tf`

```hcl
terraform {
  required_version = ">= 1.16.0, < 2.0.0"

  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = ">= 5.0, < 6.0"
    }
  }
}
```

---

# 28. `modules/network-security/variables.tf`

```hcl
variable "resource_group_name" {
  description = "Resource Group contenant le NSG."
  type        = string
  nullable    = false
}

variable "location" {
  description = "Région Azure."
  type        = string
  nullable    = false
}

variable "network_security_group_name" {
  description = "Nom du Network Security Group."
  type        = string
  nullable    = false
}

variable "app_subnet_id" {
  description = "Resource ID du Subnet application."
  type        = string
  nullable    = false
}

variable "tags" {
  description = "Tags Azure."
  type        = map(string)
  default     = {}
  nullable    = false
}
```

---

# 29. `modules/network-security/main.tf`

```hcl
resource "azurerm_network_security_group" "this" {
  name                = var.network_security_group_name
  location            = var.location
  resource_group_name = var.resource_group_name

  tags = var.tags
}

resource "azurerm_subnet_network_security_group_association" "app" {
  subnet_id = var.app_subnet_id

  network_security_group_id = azurerm_network_security_group.this.id
}
```

---

# 30. `modules/network-security/outputs.tf`

```hcl
output "network_security_group_id" {
  description = "Resource ID du Network Security Group."
  value       = azurerm_network_security_group.this.id
}

output "network_security_group_name" {
  description = "Nom du Network Security Group."
  value       = azurerm_network_security_group.this.name
}
```

---

# 31. Construire le Root Module DEV

Nous allons d'abord créer le Root Module DEV.

Les fichiers statiques seront ensuite copiés dans QUALIF et PROD.

---

# 32. `environments/dev/backend.tf`

```hcl
terraform {
  backend "azurerm" {}
}
```

Chaque environnement possède son propre répertoire `.terraform` et sa propre configuration backend.

---

# 33. `environments/dev/versions.tf`

```hcl
terraform {
  required_version = ">= 1.16.0, < 2.0.0"

  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 5.0"
    }
  }
}
```

---

# 34. `environments/dev/providers.tf`

```hcl
provider "azurerm" {
  features {}

  subscription_id = var.subscription_id
}
```

Nous rendons la subscription explicite dans le Root Module.

Cela réduit la dépendance au :

```text
current default subscription
```

d'Azure CLI.

---

# 35. Pourquoi expliciter la subscription ?

Une infrastructure professionnelle ne devrait pas dépendre uniquement de :

```powershell
az account set
```

effectué plusieurs heures auparavant sur le poste d'un ingénieur.

Nous voulons que la configuration nous dise explicitement :

```text
Cette instance cible cette subscription.
```

---

# 36. `environments/dev/variables.tf`

```hcl
variable "subscription_id" {
  description = "Subscription Azure cible."
  type        = string
  nullable    = false

  validation {
    condition = can(
      regex(
        "^[0-9a-fA-F]{8}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{12}$",
        var.subscription_id
      )
    )

    error_message = "subscription_id doit être un GUID Azure valide."
  }
}

variable "project_name" {
  description = "Nom court du projet."
  type        = string
  default     = "mla"
  nullable    = false
}

variable "environment" {
  description = "Environnement cible."
  type        = string
  nullable    = false

  validation {
    condition = contains(
      ["dev", "qual", "prod"],
      lower(var.environment)
    )

    error_message = "environment doit être dev, qual ou prod."
  }
}

variable "location" {
  description = "Région Azure."
  type        = string
  default     = "francecentral"
  nullable    = false

  validation {
    condition = contains(
      ["francecentral", "westeurope"],
      lower(var.location)
    )

    error_message = "location doit être francecentral ou westeurope."
  }
}

variable "resource_instance" {
  description = "Numéro d'instance."
  type        = number
  default     = 1
  nullable    = false

  validation {
    condition = (
      var.resource_instance >= 1 &&
      var.resource_instance <= 999 &&
      floor(var.resource_instance) == var.resource_instance
    )

    error_message = "resource_instance doit être un entier compris entre 1 et 999."
  }
}

variable "owner" {
  description = "Équipe responsable."
  type        = string
  default     = "cloud-platform"
  nullable    = false
}

variable "network" {
  description = "Configuration réseau."

  type = object({
    vnet_address_spaces = list(string)
    app_subnet_prefixes = list(string)
    pe_subnet_prefixes  = list(string)
  })

  nullable = false
}

variable "additional_tags" {
  description = "Tags supplémentaires."
  type        = map(string)
  default     = {}
  nullable    = false
}
```

---

# 37. `environments/dev/locals.tf`

```hcl
locals {
  project_name = lower(trimspace(var.project_name))
  environment  = lower(trimspace(var.environment))
  location     = lower(trimspace(var.location))

  location_codes = {
    francecentral = "fce"
    westeurope    = "weu"
  }

  location_code = local.location_codes[local.location]

  instance_suffix = format(
    "%03d",
    var.resource_instance
  )

  resource_names = {
    resource_group = "rg-${local.project_name}-${local.environment}-${local.location_code}-${local.instance_suffix}"

    virtual_network = "vnet-${local.project_name}-${local.environment}-${local.location_code}-${local.instance_suffix}"

    subnet_app = "snet-app-${local.environment}-${local.location_code}-${local.instance_suffix}"

    subnet_private_endpoints = "snet-pe-${local.environment}-${local.location_code}-${local.instance_suffix}"

    network_security_group_app = "nsg-app-${local.environment}-${local.location_code}-${local.instance_suffix}"
  }

  mandatory_tags = {
    environment = local.environment
    project     = local.project_name
    managed_by  = "terraform"
    owner       = trimspace(var.owner)
    lab         = "lab-08"
  }

  common_tags = merge(
    var.additional_tags,
    local.mandatory_tags
  )
}
```

---

# 38. `environments/dev/main.tf`

```hcl
resource "azurerm_resource_group" "main" {
  name     = local.resource_names.resource_group
  location = local.location

  tags = local.common_tags
}

module "network" {
  source = "../../modules/network"

  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location

  virtual_network_name = local.resource_names.virtual_network
  address_spaces       = var.network.vnet_address_spaces

  app_subnet_name     = local.resource_names.subnet_app
  app_subnet_prefixes = var.network.app_subnet_prefixes

  private_endpoints_subnet_name     = local.resource_names.subnet_private_endpoints
  private_endpoints_subnet_prefixes = var.network.pe_subnet_prefixes

  tags = local.common_tags
}

module "network_security" {
  source = "../../modules/network-security"

  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location

  network_security_group_name = local.resource_names.network_security_group_app

  app_subnet_id = module.network.subnet_ids["app"]

  tags = local.common_tags
}
```

---

# 39. `environments/dev/outputs.tf`

```hcl
output "environment" {
  description = "Environnement déployé."
  value       = local.environment
}

output "subscription_id" {
  description = "Subscription Azure cible."
  value       = var.subscription_id
}

output "resource_group_name" {
  description = "Nom du Resource Group."
  value       = azurerm_resource_group.main.name
}

output "virtual_network_name" {
  description = "Nom du Virtual Network."
  value       = module.network.virtual_network_name
}

output "virtual_network_id" {
  description = "Resource ID du Virtual Network."
  value       = module.network.virtual_network_id
}

output "app_subnet_id" {
  description = "Resource ID du Subnet application."
  value       = module.network.subnet_ids["app"]
}

output "private_endpoints_subnet_id" {
  description = "Resource ID du Subnet Private Endpoints."
  value       = module.network.subnet_ids["private_endpoints"]
}

output "app_nsg_id" {
  description = "Resource ID du NSG application."
  value       = module.network_security.network_security_group_id
}
```

---

# 40. `environments/dev/terraform.tfvars.example`

```hcl
subscription_id = "<SUBSCRIPTION_ID>"

project_name      = "mla"
environment       = "dev"
location          = "francecentral"
resource_instance = 1

owner = "cloud-platform"

network = {
  vnet_address_spaces = [
    "10.20.0.0/16"
  ]

  app_subnet_prefixes = [
    "10.20.1.0/24"
  ]

  pe_subnet_prefixes = [
    "10.20.2.0/24"
  ]
}

additional_tags = {
  workload    = "terraform-platform"
  criticality = "low"
}
```

---

# 41. Copier le Root Module vers QUALIF et PROD

Les fichiers suivants sont volontairement identiques :

```text
backend.tf
versions.tf
providers.tf
variables.tf
locals.tf
main.tf
outputs.tf
```

Depuis :

```text
lab-08/
```

exécuter :

```powershell
$files = @(
  "backend.tf",
  "versions.tf",
  "providers.tf",
  "variables.tf",
  "locals.tf",
  "main.tf",
  "outputs.tf"
)

foreach ($file in $files) {

  Copy-Item `
    ".\environments\dev\$file" `
    ".\environments\qual\$file"

  Copy-Item `
    ".\environments\dev\$file" `
    ".\environments\prod\$file"
}
```

---

# 42. Cette duplication est-elle acceptable ?

Oui, dans ce cas.

Les vraies implémentations réutilisables sont dans :

```text
modules/
```

Les dossiers :

```text
environments/dev
environments/qual
environments/prod
```

représentent des frontières de déploiement.

Un petit niveau de duplication dans les Root Modules peut être préférable à une abstraction supplémentaire difficile à comprendre.

---

# 43. Ne pas créer un méta-module uniquement pour éviter sept petits fichiers

Nous pourrions créer :

```text
module/platform
```

uniquement pour supprimer la duplication entre les trois Root Modules.

Mais nous ajouterions :

```text
Root
  ↓
Platform Module
  ↓
Network Module
  ↓
Resources
```

Nous augmenterions la profondeur de composition pour relativement peu de valeur à ce stade.

Nous conservons donc une architecture volontairement lisible.

---

# 44. `environments/qual/terraform.tfvars.example`

Créer :

```hcl
subscription_id = "<SUBSCRIPTION_ID>"

project_name      = "mla"
environment       = "qual"
location          = "francecentral"
resource_instance = 1

owner = "cloud-platform"

network = {
  vnet_address_spaces = [
    "10.30.0.0/16"
  ]

  app_subnet_prefixes = [
    "10.30.1.0/24"
  ]

  pe_subnet_prefixes = [
    "10.30.2.0/24"
  ]
}

additional_tags = {
  workload    = "terraform-platform"
  criticality = "medium"
}
```

---

# 45. `environments/prod/terraform.tfvars.example`

Créer :

```hcl
subscription_id = "<SUBSCRIPTION_ID>"

project_name      = "mla"
environment       = "prod"
location          = "francecentral"
resource_instance = 1

owner = "cloud-platform"

network = {
  vnet_address_spaces = [
    "10.40.0.0/16"
  ]

  app_subnet_prefixes = [
    "10.40.1.0/24"
  ]

  pe_subnet_prefixes = [
    "10.40.2.0/24"
  ]
}

additional_tags = {
  workload    = "terraform-platform"
  criticality = "high"
}
```

---

# 46. Créer les `terraform.tfvars`

Pour ce Lab, nous pouvons utiliser la subscription Azure actuellement sélectionnée pour les trois environnements.

Depuis :

```text
lab-08/
```

récupérer :

```powershell
$subscriptionId = az account show --query id -o tsv
```

Créer DEV :

```powershell
(Get-Content `
  ".\environments\dev\terraform.tfvars.example" `
  -Raw
).Replace(
  "<SUBSCRIPTION_ID>",
  $subscriptionId
) | Set-Content `
  -Encoding utf8 `
  ".\environments\dev\terraform.tfvars"
```

Même opération pour QUALIF :

```powershell
(Get-Content `
  ".\environments\qual\terraform.tfvars.example" `
  -Raw
).Replace(
  "<SUBSCRIPTION_ID>",
  $subscriptionId
) | Set-Content `
  -Encoding utf8 `
  ".\environments\qual\terraform.tfvars"
```

Et PROD :

```powershell
(Get-Content `
  ".\environments\prod\terraform.tfvars.example" `
  -Raw
).Replace(
  "<SUBSCRIPTION_ID>",
  $subscriptionId
) | Set-Content `
  -Encoding utf8 `
  ".\environments\prod\terraform.tfvars"
```

---

# 47. Avec trois vraies subscriptions

Dans une architecture réelle, nous pourrions générer :

```text
DEV terraform.tfvars
subscription_id = DEV

QUAL terraform.tfvars
subscription_id = QUAL

PROD terraform.tfvars
subscription_id = PROD
```

Le code Terraform ne change pas.

Seule l'instance de configuration change.

---

# 48. Préparer le backend Azure

Nous allons réutiliser le Storage Account créé au LAB-06.

Récupérer son nom :

```powershell
$backendStorageAccount = terraform `
  -chdir="..\lab-06\bootstrap" `
  output -raw storage_account_name
```

Si nous sommes déjà dans :

```text
lab-08/
```

utiliser :

```powershell
$backendStorageAccount = terraform `
  -chdir="..\lab-06\bootstrap" `
  output -raw storage_account_name
```

---

# 49. Backend DEV

Créer :

```text
environments/dev/backend.azurerm.tfbackend
```

avec :

```powershell
@"
storage_account_name = "$backendStorageAccount"
container_name       = "tfstate"
key                  = "terraform-azure-platform/dev/platform.tfstate"

use_azuread_auth = true
use_cli          = true
"@ | Set-Content `
  -Encoding utf8 `
  ".\environments\dev\backend.azurerm.tfbackend"
```

---

# 50. Backend QUALIF

```powershell
@"
storage_account_name = "$backendStorageAccount"
container_name       = "tfstate"
key                  = "terraform-azure-platform/qual/platform.tfstate"

use_azuread_auth = true
use_cli          = true
"@ | Set-Content `
  -Encoding utf8 `
  ".\environments\qual\backend.azurerm.tfbackend"
```

---

# 51. Backend PROD

```powershell
@"
storage_account_name = "$backendStorageAccount"
container_name       = "tfstate"
key                  = "terraform-azure-platform/prod/platform.tfstate"

use_azuread_auth = true
use_cli          = true
"@ | Set-Content `
  -Encoding utf8 `
  ".\environments\prod\backend.azurerm.tfbackend"
```

---

# 52. Comparer les backend keys

Nous avons :

```text
DEV
terraform-azure-platform/dev/platform.tfstate
```

```text
QUALIF
terraform-azure-platform/qual/platform.tfstate
```

```text
PROD
terraform-azure-platform/prod/platform.tfstate
```

Les trois environnements utilisent :

```text
même Storage Account

même Container
```

mais :

```text
trois blobs différents
```

---

# 53. Pourquoi ne pas utiliser le même blob ?

À éviter absolument :

```text
DEV
QUALIF
PROD
   │
   ▼
platform.tfstate
```

Sinon Terraform considérerait les trois environnements comme appartenant au même ensemble de ressources gérées.

---

# 54. Préparer l'authentification backend

Dans PowerShell :

```powershell
$env:ARM_USE_AZUREAD = "true"
$env:ARM_USE_CLI     = "true"
$env:ARM_TENANT_ID   = az account show --query tenantId -o tsv
```

Le backend utilisera Microsoft Entra ID.

---

# 55. Initialiser DEV

Depuis :

```text
lab-08/
```

exécuter :

```powershell
terraform `
  -chdir="environments/dev" `
  init `
  -backend-config="backend.azurerm.tfbackend"
```

Terraform initialise :

```text
Root Module DEV
```

avec :

```text
Remote State DEV
```

---

# 56. Pourquoi utiliser `-chdir` ?

Nous pourrions faire :

```powershell
cd environments/dev
terraform plan
```

Mais :

```powershell
terraform -chdir="environments/dev" plan
```

rend la cible plus explicite dans nos commandes.

Cela sera également pratique dans les futures pipelines.

---

# 57. Vérifier le workspace

Exécuter :

```powershell
terraform `
  -chdir="environments/dev" `
  workspace show
```

Résultat :

```text
default
```

C'est volontaire.

Nous n'utilisons pas un CLI Workspace nommé :

```text
dev
```

La séparation DEV vient du :

```text
Root Module DEV
+
Backend Key DEV
```

---

# 58. Valider DEV

```powershell
terraform `
  -chdir="environments/dev" `
  fmt `
  -check `
  -recursive
```

Puis :

```powershell
terraform `
  -chdir="environments/dev" `
  validate
```

---

# 59. Plan DEV

```powershell
terraform `
  -chdir="environments/dev" `
  plan `
  -out="dev.tfplan"
```

Nous devons obtenir six créations :

```text
Resource Group
Virtual Network
Subnet APP
Subnet PE
NSG
Association
```

---

# 60. Vérifier la cible avant `apply`

Exécuter :

```powershell
terraform `
  -chdir="environments/dev" `
  output
```

Avant le premier apply, les outputs ne seront pas encore disponibles.

Lire donc attentivement le plan et vérifier :

```text
environment = dev

rg-mla-dev-fce-001

vnet-mla-dev-fce-001

10.20.0.0/16
```

---

# 61. Appliquer DEV

```powershell
terraform `
  -chdir="environments/dev" `
  apply `
  "dev.tfplan"
```

Puis :

```powershell
terraform `
  -chdir="environments/dev" `
  plan
```

Résultat attendu :

```text
No changes.
```

---

# 62. Vérifier DEV

```powershell
terraform `
  -chdir="environments/dev" `
  output
```

Nous devons notamment retrouver :

```text
environment = dev

resource_group_name = rg-mla-dev-fce-001

virtual_network_name = vnet-mla-dev-fce-001
```

---

# 63. Partager le même Dependency Lock File

L'initialisation DEV a créé :

```text
environments/dev/.terraform.lock.hcl
```

Nous voulons éviter que QUALIF ou PROD sélectionnent involontairement une autre version du provider quelques semaines plus tard.

Copier :

```powershell
Copy-Item `
  ".\environments\dev\.terraform.lock.hcl" `
  ".\environments\qual\.terraform.lock.hcl"

Copy-Item `
  ".\environments\dev\.terraform.lock.hcl" `
  ".\environments\prod\.terraform.lock.hcl"
```

---

# 64. Pourquoi copier le lock file ?

Nous voulons promouvoir :

```text
même code

mêmes modules

même provider
```

entre les environnements.

La différence doit venir principalement de la configuration environnementale.

Pas d'une mise à jour accidentelle d'AzureRM entre DEV et PROD.

---

# 65. Initialiser QUALIF

```powershell
terraform `
  -chdir="environments/qual" `
  init `
  -backend-config="backend.azurerm.tfbackend"
```

Puis :

```powershell
terraform `
  -chdir="environments/qual" `
  validate
```

---

# 66. Plan QUALIF

```powershell
terraform `
  -chdir="environments/qual" `
  plan `
  -out="qual.tfplan"
```

Vérifier :

```text
rg-mla-qual-fce-001

vnet-mla-qual-fce-001

10.30.0.0/16
```

---

# 67. Appliquer QUALIF

```powershell
terraform `
  -chdir="environments/qual" `
  apply `
  "qual.tfplan"
```

Puis :

```powershell
terraform `
  -chdir="environments/qual" `
  plan
```

Résultat attendu :

```text
No changes.
```

---

# 68. Initialiser PROD

```powershell
terraform `
  -chdir="environments/prod" `
  init `
  -backend-config="backend.azurerm.tfbackend"
```

Puis :

```powershell
terraform `
  -chdir="environments/prod" `
  validate
```

---

# 69. Plan PROD

```powershell
terraform `
  -chdir="environments/prod" `
  plan `
  -out="prod.tfplan"
```

Avant toute application PROD, vérifier notamment :

```text
environment = prod

rg-mla-prod-fce-001

vnet-mla-prod-fce-001

10.40.0.0/16
```

---

# 70. PROD doit demander davantage de contrôle

Même dans ce Lab, nous pouvons déjà appliquer un principe :

```text
DEV
plan
↓
apply
```

```text
QUALIF
plan
↓
validation
↓
apply
```

```text
PROD
plan
↓
review
↓
approbation
↓
apply
```

Les pipelines futures automatiseront cette différence.

---

# 71. Appliquer PROD

Après vérification :

```powershell
terraform `
  -chdir="environments/prod" `
  apply `
  "prod.tfplan"
```

Puis :

```powershell
terraform `
  -chdir="environments/prod" `
  plan
```

Résultat attendu :

```text
No changes.
```

---

# 72. Vérifier les trois states Azure

Récupérer le Storage Account si nécessaire :

```powershell
$backendStorageAccount = terraform `
  -chdir="..\lab-06\bootstrap" `
  output -raw storage_account_name
```

Puis :

```powershell
az storage blob list `
  --account-name $backendStorageAccount `
  --container-name "tfstate" `
  --auth-mode login `
  --query "[].name" `
  --output table
```

Nous devons retrouver :

```text
terraform-azure-platform/dev/platform.tfstate

terraform-azure-platform/qual/platform.tfstate

terraform-azure-platform/prod/platform.tfstate
```

---

# 73. Nous avons réellement trois infrastructures Terraform

L'organisation n'est plus simplement :

```text
une configuration
+
trois fichiers tfvars
```

Nous avons :

```text
Root DEV
│
└── State DEV

Root QUALIF
│
└── State QUALIF

Root PROD
│
└── State PROD
```

---

# 74. Vérifier le state DEV

```powershell
terraform `
  -chdir="environments/dev" `
  state list
```

Nous devons retrouver :

```text
azurerm_resource_group.main

module.network.azurerm_virtual_network.this

module.network.azurerm_subnet.app

module.network.azurerm_subnet.private_endpoints

module.network_security.azurerm_network_security_group.this

module.network_security.azurerm_subnet_network_security_group_association.app
```

---

# 75. Même adresse Terraform dans plusieurs states

QUALIF possède également :

```text
azurerm_resource_group.main
```

et PROD également.

Ce n'est pas un problème.

Chaque adresse appartient à un state différent.

Nous avons conceptuellement :

```text
STATE DEV

azurerm_resource_group.main
       │
       ▼
rg-mla-dev-fce-001
```

```text
STATE QUALIF

azurerm_resource_group.main
       │
       ▼
rg-mla-qual-fce-001
```

```text
STATE PROD

azurerm_resource_group.main
       │
       ▼
rg-mla-prod-fce-001
```

---

# 76. Le state donne le contexte à l'adresse

L'adresse :

```text
azurerm_resource_group.main
```

n'est donc pas globalement unique dans toute l'entreprise.

Elle doit être comprise dans le contexte de son Root Module et de son state.

---

# 77. Tester l'isolation des environnements

Modifier uniquement :

```text
environments/dev/terraform.tfvars
```

Ajouter :

```hcl
additional_tags = {
  workload     = "terraform-platform"
  criticality  = "low"
  isolation_test = "dev-only"
}
```

Puis :

```powershell
terraform `
  -chdir="environments/dev" `
  plan
```

Terraform doit proposer une modification uniquement sur DEV.

---

# 78. Tester QUALIF

Sans appliquer DEV, exécuter :

```powershell
terraform `
  -chdir="environments/qual" `
  plan
```

Résultat attendu :

```text
No changes.
```

La configuration DEV n'affecte pas QUALIF.

---

# 79. Tester PROD

```powershell
terraform `
  -chdir="environments/prod" `
  plan
```

Résultat attendu :

```text
No changes.
```

Nous venons de vérifier l'isolation opérationnelle des Root Modules.

---

# 80. Annuler le test DEV

Remettre :

```hcl
additional_tags = {
  workload    = "terraform-platform"
  criticality = "low"
}
```

Puis :

```powershell
terraform `
  -chdir="environments/dev" `
  plan
```

Résultat :

```text
No changes.
```

---

# 81. Promotion ne signifie pas copier le state

Une mauvaise approche serait :

```text
DEV validé
    │
    ▼
copier dev.tfstate
    │
    ▼
PROD
```

Nous ne faisons jamais cela pour promouvoir un environnement.

La promotion concerne :

```text
Code
+
version des modules
+
configuration validée
```

Pas le state.

---

# 82. Chaque environnement conserve son historique

Nous avons :

```text
DEV State
│
└── historique DEV
```

```text
QUALIF State
│
└── historique QUALIF
```

```text
PROD State
│
└── historique PROD
```

Ils ne doivent pas être fusionnés lors d'une promotion.

---

# 83. Modèle de promotion

Nous voulons progressivement construire :

```text
Code Terraform
      │
      ▼
      DEV
      │
      ▼
Tests
      │
      ▼
    QUALIF
      │
      ▼
Validation
      │
      ▼
Approval
      │
      ▼
     PROD
```

Le même changement passe progressivement entre les environnements.

---

# 84. Git branches ≠ environnements

Nous éviterons également de construire :

```text
branch dev

branch qual

branch prod
```

comme mécanisme principal de séparation permanente.

Cela peut provoquer des divergences importantes entre branches.

Notre cible sera plutôt :

```text
main
 │
 └── source de vérité
```

avec des pipelines qui déploient successivement vers :

```text
DEV
QUALIF
PROD
```

selon les validations obtenues.

---

# 85. Configuration spécifique à chaque environnement

Il est normal que certains paramètres changent.

Par exemple :

```text
DEV

10.20.0.0/16
criticality = low
```

```text
QUALIF

10.30.0.0/16
criticality = medium
```

```text
PROD

10.40.0.0/16
criticality = high
```

Mais la structure technique principale reste partagée via les modules.

---

# 86. Ce qui ne devrait pas diverger sans raison

Éviter :

```text
DEV
module network v1

QUALIF
code complètement différent

PROD
module réseau écrit à la main
```

Si une divergence existe, elle doit répondre à un besoin réel.

L'objectif est de tester en DEV et QUALIF ce qui sera réellement promu en PROD.

---

# 87. Environment parity

Nous recherchons une certaine **parité entre environnements**.

Cela ne signifie pas :

```text
DEV = PROD en taille et en coût
```

mais plutôt :

```text
même architecture logique

mêmes modules

mêmes mécanismes de sécurité

même processus de déploiement
```

avec éventuellement des dimensions ou paramètres différents.

---

# 88. Exemple futur

Nous pourrions avoir plus tard :

```text
DEV
Container App
0.5 CPU
1 replica

QUALIF
Container App
1 CPU
1 replica

PROD
Container App
2 CPU
3 replicas
```

Le module reste le même.

Les paramètres changent.

---

# 89. Pourquoi PROD mérite une identité différente

Dans une architecture professionnelle :

```text
Terraform DEV identity
```

ne devrait pas nécessairement posséder les droits pour modifier PROD.

Nous préférerons :

```text
Identity DEV
      │
      └── DEV Subscription

Identity QUALIF
      │
      └── QUALIF Subscription

Identity PROD
      │
      └── PROD Subscription
```

avec le principe du moindre privilège.

---

# 90. Une erreur DEV ne doit pas devenir une erreur PROD

Même si l'ingénieur possède des droits élevés, la plateforme doit ajouter des barrières :

```text
Environment Root Module

Remote State

RBAC

Pipeline

Approvals

Branch protection

Policy

Concurrency
```

La sécurité ne repose pas sur :

> l'ingénieur fera attention.

---

# 91. Mauvais scénario — backend key copiée par erreur

Imaginons que PROD possède :

```text
key = terraform-azure-platform/dev/platform.tfstate
```

au lieu de :

```text
key = terraform-azure-platform/prod/platform.tfstate
```

Nous pourrions exécuter du code PROD contre le state DEV.

C'est une erreur potentiellement grave.

---

# 92. Toujours vérifier la backend key

Lors d'une revue de code ou de pipeline, nous devons considérer la backend key comme une information critique.

```text
DEV
/dev/

QUALIF
/qual/

PROD
/prod/
```

La convention doit être simple et prévisible.

---

# 93. Backend et identity sont deux protections différentes

Séparer les states protège la représentation Terraform.

Séparer les identités protège les autorisations Azure.

Nous voulons idéalement :

```text
State DEV
+
Identity DEV
+
Subscription DEV
```

et :

```text
State PROD
+
Identity PROD
+
Subscription PROD
```

La combinaison fournit une isolation beaucoup plus solide.

---

# 94. Les `.tfvars` peuvent-ils être versionnés ?

Notre série continue d'ignorer :

```text
terraform.tfvars
```

par prudence.

Nous versionnons :

```text
terraform.tfvars.example
```

Cela dit, un fichier `.tfvars` contenant uniquement de la configuration non sensible peut être versionné si la stratégie de l'organisation le prévoit.

L'extension `.tfvars` ne signifie pas automatiquement :

```text
secret
```

---

# 95. Exemple de données non sensibles

```hcl
environment = "prod"
location    = "francecentral"
```

ne sont pas des secrets.

Même chose pour :

```text
Subscription ID
```

qui constitue un identifiant et non un credential d'authentification.

---

# 96. Exemple à ne pas versionner

À éviter :

```hcl
database_password = "..."
client_secret      = "..."
access_key         = "..."
```

Nous utiliserons des identités fédérées et des gestionnaires de secrets lorsque ces besoins apparaîtront.

---

# 97. `.gitignore`

Notre `.gitignore` doit notamment contenir :

```gitignore
# Terraform working directories
**/.terraform/*

# Terraform state
*.tfstate
*.tfstate.*

# Terraform plan files
*.tfplan

# Local variables
*.tfvars
*.tfvars.json

# Local backend configuration
*.tfbackend

# Crash logs
crash.log
crash.*.log

# Override files
override.tf
override.tf.json
*_override.tf
*_override.tf.json
```

Ne pas ignorer :

```text
.terraform.lock.hcl
```

---

# 98. Lock files des environnements

Chaque Root Module peut avoir :

```text
environments/dev/.terraform.lock.hcl

environments/qual/.terraform.lock.hcl

environments/prod/.terraform.lock.hcl
```

Nous les versionnons.

Pour maintenir la parité, nous devons nous assurer que les mêmes versions de providers sont promues entre les environnements.

---

# 99. Une mise à jour provider est elle-même un changement

Nous ne voulons pas :

```text
DEV
AzureRM 5.7

QUALIF
AzureRM 5.9

PROD
AzureRM 5.3
```

sans décision explicite.

La version des providers fait partie du changement testé.

---

# 100. Procédure de mise à jour future

Une montée de version pourrait suivre :

```text
Mettre à jour DEV
       │
       ▼
terraform init -upgrade
       │
       ▼
plan / tests
       │
       ▼
QUALIF
       │
       ▼
tests
       │
       ▼
PROD
```

Nous ne lancerons pas automatiquement :

```powershell
terraform init -upgrade
```

en PROD sans validation préalable.

---

# 101. Vérifier les trois plans

Depuis :

```text
lab-08/
```

nous pouvons exécuter :

```powershell
terraform `
  -chdir="environments/dev" `
  plan
```

Puis :

```powershell
terraform `
  -chdir="environments/qual" `
  plan
```

Puis :

```powershell
terraform `
  -chdir="environments/prod" `
  plan
```

Résultat attendu dans les trois cas :

```text
No changes.
```

---

# 102. Future automatisation

Plus tard, notre Continuous Integration / Continuous Delivery (CI/CD) pourra exécuter :

```text
Pull Request
     │
     ├── terraform fmt
     ├── terraform validate
     ├── lint
     ├── security scan
     └── terraform plan DEV
```

Puis :

```text
Merge
  │
  ▼
DEV Apply
  │
  ▼
Tests
  │
  ▼
QUALIF Apply
  │
  ▼
Validation
  │
  ▼
Approval
  │
  ▼
PROD Apply
```

---

# 103. Une pipeline par environnement

Notre structure se prête naturellement à :

```text
terraform-dev.yml

terraform-qual.yml

terraform-prod.yml
```

ou à une pipeline réutilisable appelée avec :

```text
environment = dev

environment = qual

environment = prod
```

Nous comparerons ces approches lorsque nous introduirons GitHub Actions.

---

# 104. Protection PROD

La pipeline PROD pourra exiger :

```text
Protected Environment

Manual Approval

Dedicated OIDC Identity

PROD RBAC

PROD Remote State

Concurrency protection
```

La différence DEV / PROD sera donc également opérationnelle.

---

# 105. Nettoyage — attention aux trois states

Nous avons maintenant trois environnements.

Nous devons les supprimer séparément.

Commencer par DEV :

```powershell
terraform `
  -chdir="environments/dev" `
  plan `
  -destroy `
  -out="destroy.tfplan"
```

Puis :

```powershell
terraform `
  -chdir="environments/dev" `
  apply `
  "destroy.tfplan"
```

---

# 106. Détruire QUALIF

```powershell
terraform `
  -chdir="environments/qual" `
  plan `
  -destroy `
  -out="destroy.tfplan"
```

Puis :

```powershell
terraform `
  -chdir="environments/qual" `
  apply `
  "destroy.tfplan"
```

---

# 107. Détruire PROD

Vérifier particulièrement le plan :

```powershell
terraform `
  -chdir="environments/prod" `
  plan `
  -destroy `
  -out="destroy.tfplan"
```

Lire :

```powershell
terraform `
  -chdir="environments/prod" `
  show `
  "destroy.tfplan"
```

Puis seulement :

```powershell
terraform `
  -chdir="environments/prod" `
  apply `
  "destroy.tfplan"
```

---

# 108. Les states restent indépendants après destruction

Après destruction :

```text
DEV state
0 resources

QUALIF state
0 resources

PROD state
0 resources
```

Les blobs peuvent rester présents dans Azure Storage.

Le backend du LAB-06 reste conservé.

---

# 109. Versionner LAB-08

Depuis la racine du repository :

```powershell
Set-Location "C:\Labs\MonLabAzure\terraform-azure-platform"
```

Vérifier :

```powershell
git status
```

Puis :

```powershell
git add .
```

Contrôler :

```powershell
git diff --cached --name-only
```

Nous devons versionner :

```text
lab-08/modules/

lab-08/environments/dev/*.tf

lab-08/environments/qual/*.tf

lab-08/environments/prod/*.tf

terraform.tfvars.example

.terraform.lock.hcl
```

---

# 110. Ne pas versionner

Vérifier l'absence de :

```text
terraform.tfvars

backend.azurerm.tfbackend

*.tfplan

terraform.tfstate

.terraform/
```

Puis :

```powershell
git commit -m "feat(terraform): add multi-environment architecture"
```

---

# 111. Architecture finale

Nous avons maintenant :

```text
terraform-azure-platform/
│
└── lab-08/
    │
    ├── modules/
    │   │
    │   ├── network/
    │   │
    │   └── network-security/
    │   │
    │   └──── code partagé
    │
    └── environments/
        │
        ├── dev/
        │   └── Root Module DEV
        │       └── State DEV
        │
        ├── qual/
        │   └── Root Module QUALIF
        │       └── State QUALIF
        │
        └── prod/
            └── Root Module PROD
                └── State PROD
```

---

# 112. Modèle mental

Il faut maintenant distinguer trois couches.

```text
MODULES
│
│ Définissent COMMENT construire
│
▼
REUTILISABLE
```

```text
ROOT MODULE ENVIRONMENT
│
│ Définit QUOI instancier
│ et avec quels paramètres
│
▼
DEV / QUAL / PROD
```

```text
STATE
│
│ Mémorise CE QUE Terraform gère
│ pour cette instance
│
▼
Remote Backend
```

---

# 113. Modules et environnements

Les modules répondent à :

> Comment construit-on notre réseau ?

Les environnements répondent à :

> Quelle instance de ce réseau voulons-nous ici ?

Exemple :

```text
module.network
       │
       ├── DEV → 10.20.0.0/16
       │
       ├── QUAL → 10.30.0.0/16
       │
       └── PROD → 10.40.0.0/16
```

---

# 114. Une seule base de code

Nous évitons :

```text
Terraform code DEV

copie Terraform QUALIF

copie Terraform PROD
```

Les implémentations principales sont partagées dans :

```text
modules/
```

Les Root Modules environnementaux restent volontairement légers.

---

# 115. Question d'entretien — comment séparez-vous DEV, QUALIF et PROD ?

Réponse possible :

> Je privilégie des Root Modules distincts par environnement, chacun avec son propre Remote State et sa propre backend key. Les environnements consomment les mêmes modules réutilisables mais possèdent leurs paramètres, leurs identités et éventuellement leurs subscriptions propres. Cela limite le blast radius et permet d'avoir des pipelines et des approbations différentes entre DEV et PROD.

---

# 116. Question d'entretien — utilisez-vous les Terraform CLI Workspaces ?

Réponse :

> Pas comme mécanisme principal d'isolation pour DEV, QUALIF et PROD lorsque j'ai besoin de credentials, droits ou backends distincts. Les CLI Workspaces séparent les states mais utilisent la même configuration et le même backend. Pour des environnements avec de vraies frontières de sécurité, je préfère des Root Modules et des states séparés.

---

# 117. Question d'entretien — pourquoi pas seulement trois `.tfvars` ?

Réponse :

> Les `.tfvars` changent les inputs mais ne créent pas une frontière de state. Si plusieurs fichiers `.tfvars` utilisent le même backend et la même key, je peux accidentellement exécuter une configuration PROD contre le state DEV. Je combine donc les paramètres environnementaux avec des states explicitement séparés.

---

# 118. Question d'entretien — un repository par environnement ?

Réponse :

> C'est possible et parfois pertinent pour une isolation organisationnelle très forte, mais cela augmente le risque de duplication et de divergence. Je préfère généralement conserver la configuration des environnements dans un même repository et partager les modules, puis utiliser des states, identités et pipelines séparés. Je sépare les repositories lorsqu'il existe une vraie frontière de responsabilité ou de sécurité.

---

# 119. Question d'entretien — comment promouvez-vous en production ?

Réponse :

> Je ne copie jamais le state DEV vers PROD. Je promeus le même changement de code et les mêmes versions de modules et providers à travers DEV, puis QUALIF, puis PROD. Chaque environnement conserve son propre state. PROD dispose ensuite de contrôles supplémentaires comme des approvals et une identité dédiée.

---

# 120. Validation du LAB-08

Le Lab est terminé lorsque les points suivants sont compris et testés :

```text
[ ] Comprendre pourquoi tfvars ne sépare pas le state

[ ] Comprendre Terraform CLI Workspaces

[ ] Comprendre la différence avec HCP Terraform Workspaces

[ ] Comprendre les limites des CLI Workspaces

[ ] Comprendre la stratégie dossiers par environnement

[ ] Comprendre la stratégie repositories séparés

[ ] Choisir 1 Root Module par environnement

[ ] Créer environments/dev

[ ] Créer environments/qual

[ ] Créer environments/prod

[ ] Partager modules/network

[ ] Partager modules/network-security

[ ] Configurer subscription_id explicitement

[ ] Créer une backend key DEV

[ ] Créer une backend key QUALIF

[ ] Créer une backend key PROD

[ ] Initialiser DEV

[ ] Déployer DEV

[ ] Initialiser QUALIF

[ ] Déployer QUALIF

[ ] Initialiser PROD

[ ] Vérifier le plan PROD avant apply

[ ] Déployer PROD

[ ] Vérifier trois blobs de state

[ ] Comprendre qu'une même resource address peut exister dans plusieurs states

[ ] Vérifier l'isolation DEV / QUALIF / PROD

[ ] Tester une modification DEV sans impact QUALIF

[ ] Tester une modification DEV sans impact PROD

[ ] Comprendre environment parity

[ ] Comprendre le blast radius

[ ] Comprendre pourquoi le state n'est jamais promu

[ ] Comprendre pourquoi les branches Git ne représentent pas les environnements

[ ] Partager les lock files providers

[ ] Comprendre la future séparation des identités

[ ] Préparer la future CI/CD
```

---

# 121. Progression

Nous avons maintenant :

```text
LAB-00
Environnement
     │
     ▼
LAB-01
Premier déploiement
     │
     ▼
LAB-02
HCL
     │
     ▼
LAB-03
Variables et types
     │
     ▼
LAB-04
Graphe
     │
     ▼
LAB-05
State
     │
     ▼
LAB-06
Remote State
     │
     ▼
LAB-07
Modules
     │
     ▼
LAB-08
DEV / QUALIF / PROD
     │
     ├── Root Modules
     ├── States séparés
     ├── Configurations séparées
     ├── Modules partagés
     └── Promotion contrôlée
```

Notre plateforme Terraform possède maintenant des frontières suffisamment propres pour commencer à travailler sérieusement sur l'identité et la sécurité.

---

# 122. Suite

Jusqu'ici, notre authentification locale repose principalement sur :

```text
Azure CLI
+
identité utilisateur
```

La prochaine étape consiste à comprendre ce que Terraform doit réellement avoir le droit de faire dans Azure.

Nous allons travailler sur :

```text
Microsoft Entra ID
       │
       ▼
Identity Terraform
       │
       ▼
Azure RBAC
       │
       ▼
Subscription / Resource Group
       │
       ▼
Least Privilege
```

Nous introduirons notamment :

- identité humaine vs identité workload ;
- Service Principal ;
- Managed Identity ;
- Workload Identity Federation ;
- RBAC ;
- scope ;
- rôle ;
- principal ;
- permissions management plane ;
- séparation DEV / QUALIF / PROD ;
- préparation à GitHub Actions OIDC.

Le prochain article sera :

**LAB-09 — Identité Terraform, Microsoft Entra ID et RBAC Azure**

---

## Références officielles

Les notions étudiées dans ce Lab sont documentées notamment dans :

- HashiCorp Developer — Terraform Workspaces ;
- HashiCorp Developer — CLI Workspaces ;
- HashiCorp Developer — Terraform Style Guide ;
- HashiCorp Developer — Files and Configuration Structure ;
- HashiCorp Developer — Modules ;
- HashiCorp Developer — AzureRM Backend ;
- Terraform Registry — HashiCorp AzureRM Provider ;
- Microsoft Learn — Terraform sur Microsoft Azure.

Pour Terraform CLI avec un backend externe, la séparation par répertoires et states distincts permet de définir des frontières d'environnement explicites. Les CLI Workspaces restent utiles dans certains scénarios, mais ne constituent pas une isolation adaptée lorsque les environnements nécessitent des credentials et contrôles d'accès réellement distincts.