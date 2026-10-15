---
title: "LAB-07 - Concevoir des modules Terraform professionnels"
date: 2026-10-08
draft: false
description: "Refactorer une infrastructure Azure existante en modules Terraform professionnels sans recréer les ressources grâce aux interfaces inputs/outputs et aux moved blocks."
summary: "Construire des modules Terraform maintenables, composer réseau et sécurité, gérer les providers, les versions et refactorer le state sans destruction."
slug: "terraform-azure-lab-07-modules-professionnels"
categories:
  - "Automatisation"
tags:
  - "Terraform"
  - "Azure"
  - "Modules"
  - "Infrastructure as Code"
  - "Refactoring"
  - "AzureRM"
  - "Moved Block"
series:
  - "Terraform Azure - De zéro à production"
weight: 8
ShowToc: false
TocOpen: false
---


Dans les Labs précédents, toute notre infrastructure était déclarée directement dans le répertoire depuis lequel Terraform était exécuté.

Nous avions donc un seul module :

```text
Root Module
│
├── Resource Group
├── Virtual Network
├── Subnet application
├── Subnet Private Endpoints
├── Network Security Group
└── Association Subnet / NSG
```

Cette organisation fonctionne parfaitement pour une petite infrastructure.

Mais à mesure que la plateforme grandit, nous allons avoir :

```text
Virtual Networks
Private DNS
Key Vault
Managed Identities
Container Registry
Applications
Application Gateway
WAF
Monitoring
RBAC
Private Endpoints
```

Tout conserver dans le même `main.tf` deviendrait rapidement difficile à maintenir.

L'objectif de ce Lab n'est toutefois pas de transformer chaque ressource en module.

Nous allons plutôt apprendre à définir des **frontières cohérentes**.

Notre architecture finale sera :

```text
Root Module
│
├── Resource Group
│
├── module.network
│   │
│   ├── Virtual Network
│   ├── Subnet application
│   └── Subnet Private Endpoints
│
└── module.network_security
    │
    ├── Network Security Group
    └── Association NSG / Subnet
```

Le Root Module va orchestrer les composants.

Les Child Modules vont encapsuler les responsabilités techniques.

---

# 1. Objectifs du Lab

Nous allons comprendre :

- ce qu'est réellement un module Terraform ;
- Root Module ;
- Child Module ;
- interface d'un module ;
- inputs ;
- outputs ;
- encapsulation ;
- composition ;
- module local ;
- module distant ;
- `source` ;
- versionnement ;
- provider requirements ;
- provider configuration ;
- dépendances entre modules ;
- module composition ;
- anti-pattern `1 ressource = 1 module` ;
- modules trop génériques ;
- modules trop spécifiques ;
- refactoring vers des modules ;
- déplacement d'une ressource vers un module ;
- `moved` blocks ;
- modification des adresses dans le state sans recréer Azure.

---

# 2. Qu'est-ce qu'un module Terraform ?

Un module Terraform est simplement un ensemble de fichiers Terraform regroupés dans un même répertoire.

Lorsque nous exécutons :

```powershell
terraform plan
```

dans :

```text
lab-07/
```

les fichiers `.tf` présents dans ce répertoire constituent le :

```text
Root Module
```

Si ce Root Module appelle :

```hcl
module "network" {
  source = "./modules/network"
}
```

alors :

```text
./modules/network
```

constitue un :

```text
Child Module
```

Architecture :

```text
Root Module
│
├── variables.tf
├── locals.tf
├── main.tf
│
└── module "network"
     │
     ▼
modules/network/
│
├── versions.tf
├── variables.tf
├── main.tf
└── outputs.tf
```

---

# 3. Un module n'est pas une ressource

Un module peut contenir :

```text
1 ressource
```

mais aussi :

```text
5
20
50
```

ressources.

Le bon découpage ne consiste donc pas automatiquement à créer :

```text
module-resource-group

module-vnet

module-subnet

module-nsg
```

pour chaque ressource Azure.

Cela peut produire une abstraction sans réelle valeur.

---

# 4. Mauvaise modularisation

Architecture possible mais inutilement fragmentée :

```text
Root
│
├── module.rg
│
├── module.vnet
│
├── module.subnet_app
│
├── module.subnet_pe
│
├── module.nsg
│
└── module.nsg_association
```

Nous aurions remplacé six ressources simples par six modules.

Le résultat pourrait être plus difficile à comprendre que le code initial.

---

# 5. Modulariser par responsabilité

Nous allons plutôt créer :

```text
module.network
```

responsable de :

```text
Virtual Network
Subnets
```

et :

```text
module.network_security
```

responsable de :

```text
Network Security Group
Association NSG / Subnet
```

Le Resource Group reste dans le Root Module.

---

# 6. Pourquoi laisser le Resource Group dans le Root Module ?

Nous pourrions créer :

```text
module.resource_group
```

mais notre Resource Group contient actuellement une seule ressource très simple.

Le placer dans un module n'apporterait pratiquement aucune abstraction.

En revanche, il joue parfaitement le rôle de ressource structurante du Root Module.

Nous allons donc conserver :

```hcl
resource "azurerm_resource_group" "main"
```

directement dans le Root Module.

---

# 7. Principe de responsabilité

Une frontière de module devrait idéalement correspondre à quelque chose que nous pouvons expliquer simplement.

Par exemple :

```text
network
```

signifie :

> Je crée et expose le réseau utilisé par la plateforme.

Et :

```text
network-security
```

signifie :

> Je crée les contrôles de sécurité réseau et les rattache aux réseaux qui me sont fournis.

C'est plus utile que :

```text
module-subnet-01
```

---

# 8. Infrastructure avant refactoring

Avant :

```text
Root Module
│
├── azurerm_resource_group.main
├── azurerm_virtual_network.main
├── azurerm_subnet.app
├── azurerm_subnet.private_endpoints
├── azurerm_network_security_group.application
└── azurerm_subnet_network_security_group_association.app
```

Adresses Terraform :

```text
azurerm_resource_group.main

azurerm_virtual_network.main

azurerm_subnet.app

azurerm_subnet.private_endpoints

azurerm_network_security_group.application

azurerm_subnet_network_security_group_association.app
```

---

# 9. Infrastructure après refactoring

Après :

```text
Root
│
├── azurerm_resource_group.main
│
├── module.network
│   ├── azurerm_virtual_network.this
│   ├── azurerm_subnet.app
│   └── azurerm_subnet.private_endpoints
│
└── module.network_security
    ├── azurerm_network_security_group.this
    └── azurerm_subnet_network_security_group_association.app
```

Les adresses deviennent :

```text
azurerm_resource_group.main

module.network.azurerm_virtual_network.this

module.network.azurerm_subnet.app

module.network.azurerm_subnet.private_endpoints

module.network_security.azurerm_network_security_group.this

module.network_security.azurerm_subnet_network_security_group_association.app
```

Azure ne doit pourtant recréer aucune ressource.

---

# 10. Pourquoi le refactoring est sensible ?

Terraform identifie une ressource avec son adresse.

Avant :

```text
azurerm_virtual_network.main
```

Après :

```text
module.network.azurerm_virtual_network.this
```

Sans indication supplémentaire, Terraform pourrait interpréter cela comme :

```text
ancienne adresse disparue
       │
       ▼
destroy

nouvelle adresse apparue
       │
       ▼
create
```

Ce n'est évidemment pas ce que nous voulons.

---

# 11. Objectif du refactoring

Nous voulons obtenir :

```text
Azure Resource
vnet-mla-dev-fce-001
       │
       │ reste identique
       ▼
ancienne adresse Terraform
       │
       ▼
nouvelle adresse Terraform
```

Nous utiliserons pour cela :

```hcl
moved {
}
```

---

# 12. Réutiliser le backend du LAB-06

Le backend Azure créé dans LAB-06 doit être conservé.

Nous allons utiliser une nouvelle state key :

```text
terraform-azure-platform/dev/lab-07.tfstate
```

Cela permet d'isoler ce Lab du state du LAB-06.

---

# 13. Créer LAB-07

Depuis :

```powershell
Set-Location "C:\Labs\MonLabAzure\terraform-azure-platform"
```

Créer :

```powershell
New-Item -ItemType Directory -Name "lab-07"
Set-Location ".\lab-07"
```

Puis :

```powershell
New-Item -ItemType Directory -Path ".\modules\network" -Force

New-Item -ItemType Directory -Path ".\modules\network-security" -Force
```

---

# 14. Arborescence finale

Nous allons obtenir :

```text
lab-07/
│
├── versions.tf
├── providers.tf
├── variables.tf
├── locals.tf
├── main.tf
├── outputs.tf
├── moved.tf
├── terraform.tfvars.example
│
└── modules/
    │
    ├── network/
    │   ├── versions.tf
    │   ├── variables.tf
    │   ├── main.tf
    │   └── outputs.tf
    │
    └── network-security/
        ├── versions.tf
        ├── variables.tf
        ├── main.tf
        └── outputs.tf
```

Le fichier :

```text
backend.azurerm.tfbackend
```

sera créé localement mais non versionné.

---

# 15. Première phase : déployer sans modules

Pour comprendre réellement le refactoring, nous allons commencer LAB-07 avec la structure plate utilisée jusqu'ici.

Ne pas créer les Child Modules immédiatement.

Nous allons d'abord déployer :

```text
Root Module
│
└── 6 ressources
```

puis les déplacer.

---

# 16. `versions.tf` initial

Créer :

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

---

# 17. `providers.tf`

```hcl
provider "azurerm" {
  features {}
}
```

Le provider est configuré dans le Root Module.

Nous reviendrons sur cette décision lorsque nous construirons les Child Modules.

---

# 18. `variables.tf`

```hcl
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
}

variable "resource_instance" {
  description = "Numéro d'instance."
  type        = number
  default     = 1
  nullable    = false
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

# 19. `locals.tf`

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
    lab         = "lab-07"
  }

  common_tags = merge(
    var.additional_tags,
    local.mandatory_tags
  )
}
```

---

# 20. `main.tf` initial

Avant modularisation :

```hcl
resource "azurerm_resource_group" "main" {
  name     = local.resource_names.resource_group
  location = local.location

  tags = local.common_tags
}

resource "azurerm_virtual_network" "main" {
  name                = local.resource_names.virtual_network
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  address_space       = var.network.vnet_address_spaces

  tags = local.common_tags
}

resource "azurerm_subnet" "app" {
  name = local.resource_names.subnet_app

  resource_group_name  = azurerm_resource_group.main.name
  virtual_network_name = azurerm_virtual_network.main.name

  address_prefixes = var.network.app_subnet_prefixes
}

resource "azurerm_subnet" "private_endpoints" {
  name = local.resource_names.subnet_private_endpoints

  resource_group_name  = azurerm_resource_group.main.name
  virtual_network_name = azurerm_virtual_network.main.name

  address_prefixes = var.network.pe_subnet_prefixes
}

resource "azurerm_network_security_group" "application" {
  name                = local.resource_names.network_security_group_app
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name

  tags = local.common_tags
}

resource "azurerm_subnet_network_security_group_association" "app" {
  subnet_id = azurerm_subnet.app.id

  network_security_group_id = azurerm_network_security_group.application.id
}
```

---

# 21. `outputs.tf` initial

```hcl
output "resource_group_name" {
  description = "Nom du Resource Group."
  value       = azurerm_resource_group.main.name
}

output "virtual_network_name" {
  description = "Nom du Virtual Network."
  value       = azurerm_virtual_network.main.name
}

output "virtual_network_id" {
  description = "Resource ID du Virtual Network."
  value       = azurerm_virtual_network.main.id
}

output "app_subnet_id" {
  description = "Resource ID du Subnet application."
  value       = azurerm_subnet.app.id
}

output "private_endpoints_subnet_id" {
  description = "Resource ID du Subnet Private Endpoints."
  value       = azurerm_subnet.private_endpoints.id
}

output "app_nsg_id" {
  description = "Resource ID du NSG application."
  value       = azurerm_network_security_group.application.id
}
```

---

# 22. `terraform.tfvars.example`

```hcl
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
  workload    = "terraform-learning"
  criticality = "low"
}
```

Créer :

```powershell
Copy-Item `
  ".\terraform.tfvars.example" `
  ".\terraform.tfvars"
```

---

# 23. Préparer le backend LAB-07

Définir :

```powershell
$env:ARM_USE_AZUREAD = "true"
$env:ARM_USE_CLI     = "true"

$env:ARM_SUBSCRIPTION_ID = az account show --query id -o tsv
$env:ARM_TENANT_ID       = az account show --query tenantId -o tsv
```

Récupérer le Storage Account créé dans LAB-06 :

```powershell
$backendStorageAccount = terraform `
  -chdir="..\lab-06\bootstrap" `
  output -raw storage_account_name
```

Créer :

```powershell
@"
storage_account_name = "$backendStorageAccount"
container_name       = "tfstate"
key                  = "terraform-azure-platform/dev/lab-07.tfstate"

use_azuread_auth = true
use_cli          = true
"@ | Set-Content `
  -Encoding utf8 `
  ".\backend.azurerm.tfbackend"
```

---

# 24. Initialiser et déployer la version plate

Exécuter :

```powershell
terraform fmt -recursive
```

Puis :

```powershell
terraform init `
  -backend-config="backend.azurerm.tfbackend"
```

Puis :

```powershell
terraform validate
```

Créer le plan :

```powershell
terraform plan `
  -out="lab-07-flat.tfplan"
```

Résultat attendu :

```text
Plan: 6 to add, 0 to change, 0 to destroy.
```

Appliquer :

```powershell
terraform apply ".\lab-07-flat.tfplan"
```

Puis :

```powershell
terraform plan
```

Résultat attendu :

```text
No changes.
```

---

# 25. Observer le state avant refactoring

Exécuter :

```powershell
terraform state list
```

Nous devons retrouver :

```text
azurerm_network_security_group.application

azurerm_resource_group.main

azurerm_subnet.app

azurerm_subnet.private_endpoints

azurerm_subnet_network_security_group_association.app

azurerm_virtual_network.main
```

C'est notre point de départ.

---

# 26. Concevoir l'interface du module `network`

Le Root Module doit fournir au module :

```text
Resource Group name
Location
VNet name
VNet address spaces
Subnet names
Subnet address prefixes
Tags
```

Le module doit retourner :

```text
VNet ID
VNet name
Subnet IDs
Subnet names
```

Nous créons donc un contrat clair.

---

# 27. `modules/network/versions.tf`

Créer :

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

Remarquer un point important :

```text
provider requirement
```

n'est pas :

```text
provider configuration
```

Le module indique qu'il nécessite AzureRM.

Il ne configure pas l'authentification Azure.

---

# 28. Aucun `provider` block dans le Child Module

Nous ne créons pas :

```hcl
provider "azurerm" {
  features {}
}
```

dans :

```text
modules/network/
```

La configuration du provider appartient au Root Module.

Le Child Module déclare uniquement son besoin :

```hcl
required_providers {
  azurerm = {
    source = "hashicorp/azurerm"
  }
}
```

---

# 29. `modules/network/variables.tf`

```hcl
variable "resource_group_name" {
  description = "Resource Group contenant les ressources réseau."
  type        = string
  nullable    = false
}

variable "location" {
  description = "Région Azure des ressources réseau."
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
    error_message = "address_spaces doit contenir au moins une plage réseau."
  }
}

variable "app_subnet_name" {
  description = "Nom du Subnet application."
  type        = string
  nullable    = false
}

variable "app_subnet_prefixes" {
  description = "Préfixes réseau du Subnet application."
  type        = list(string)
  nullable    = false

  validation {
    condition     = length(var.app_subnet_prefixes) > 0
    error_message = "app_subnet_prefixes doit contenir au moins un préfixe."
  }
}

variable "private_endpoints_subnet_name" {
  description = "Nom du Subnet Private Endpoints."
  type        = string
  nullable    = false
}

variable "private_endpoints_subnet_prefixes" {
  description = "Préfixes réseau du Subnet Private Endpoints."
  type        = list(string)
  nullable    = false

  validation {
    condition     = length(var.private_endpoints_subnet_prefixes) > 0
    error_message = "private_endpoints_subnet_prefixes doit contenir au moins un préfixe."
  }
}

variable "tags" {
  description = "Tags appliqués aux ressources supportant les tags."
  type        = map(string)
  default     = {}
  nullable    = false
}
```

---

# 30. Pourquoi le module reçoit-il `resource_group_name` ?

Le module réseau ne crée pas son propre Resource Group.

Il reçoit simplement :

```hcl
var.resource_group_name
```

Cette décision améliore sa réutilisabilité.

Nous pouvons ainsi placer plusieurs instances du module dans différents Resource Groups.

---

# 31. Éviter le module qui fait tout

Nous ne voulons pas un module réseau qui décide automatiquement :

```text
Resource Group
Subscription
Environment
Naming
RBAC
Monitoring
Application Gateway
Key Vault
```

Un module avec trop de responsabilités devient difficile à réutiliser.

---

# 32. `modules/network/main.tf`

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

# 33. Pourquoi nommer la ressource principale `this` ?

Dans le Root Module, nous avions :

```text
azurerm_virtual_network.main
```

Dans un module spécialisé :

```text
module.network
```

le contexte indique déjà la fonction.

L'adresse :

```text
module.network.azurerm_virtual_network.this
```

est donc lisible.

`this` est une convention courante pour désigner la ressource principale d'un module.

Ce n'est pas une obligation Terraform.

---

# 34. `modules/network/outputs.tf`

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
  description = "Resource IDs des Subnets créés par le module."

  value = {
    app               = azurerm_subnet.app.id
    private_endpoints = azurerm_subnet.private_endpoints.id
  }
}

output "subnet_names" {
  description = "Noms des Subnets créés par le module."

  value = {
    app               = azurerm_subnet.app.name
    private_endpoints = azurerm_subnet.private_endpoints.name
  }
}
```

---

# 35. L'output est l'interface de sortie du module

Le Root Module ne devrait pas avoir besoin de connaître tous les détails internes du module.

Il récupère :

```hcl
module.network.virtual_network_id
```

ou :

```hcl
module.network.subnet_ids["app"]
```

Conceptuellement :

```text
module.network
      │
      ├── internal resources
      │
      ▼
    outputs
      │
      ▼
Root Module
```

---

# 36. Concevoir `network-security`

Le second module reçoit :

```text
Resource Group
Location
NSG name
Subnet ID
Tags
```

et expose :

```text
NSG ID
NSG name
```

---

# 37. `modules/network-security/versions.tf`

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

# 38. `modules/network-security/variables.tf`

```hcl
variable "resource_group_name" {
  description = "Resource Group contenant le NSG."
  type        = string
  nullable    = false
}

variable "location" {
  description = "Région Azure du NSG."
  type        = string
  nullable    = false
}

variable "network_security_group_name" {
  description = "Nom du Network Security Group."
  type        = string
  nullable    = false
}

variable "app_subnet_id" {
  description = "Resource ID du Subnet application auquel le NSG sera associé."
  type        = string
  nullable    = false
}

variable "tags" {
  description = "Tags appliqués au Network Security Group."
  type        = map(string)
  default     = {}
  nullable    = false
}
```

---

# 39. `modules/network-security/main.tf`

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

# 40. `modules/network-security/outputs.tf`

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

# 41. Composer les modules depuis le Root Module

Nous allons maintenant remplacer le `main.tf` initial.

Le Root Module ne contiendra plus les ressources réseau directement.

---

# 42. `main.tf` final

Remplacer complètement `main.tf` par :

```hcl
resource "azurerm_resource_group" "main" {
  name     = local.resource_names.resource_group
  location = local.location

  tags = local.common_tags
}

module "network" {
  source = "./modules/network"

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
  source = "./modules/network-security"

  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location

  network_security_group_name = local.resource_names.network_security_group_app

  app_subnet_id = module.network.subnet_ids["app"]

  tags = local.common_tags
}
```

---

# 43. Le Root Module devient un orchestrateur

Avant :

```text
Root Module
│
├── toutes les ressources
├── tous les détails
└── toutes les dépendances
```

Maintenant :

```text
Root Module
│
├── Resource Group
│
├── Network
│
└── Network Security
```

Le Root Module exprime davantage l'architecture de la plateforme.

---

# 44. Dépendance entre modules

Observer :

```hcl
app_subnet_id = module.network.subnet_ids["app"]
```

Le module :

```text
network-security
```

a besoin d'une information produite par :

```text
network
```

Terraform peut donc déduire :

```text
module.network
      │
      ▼
module.network_security
```

Il s'agit d'une dépendance implicite.

Aucun :

```hcl
depends_on = [
  module.network
]
```

n'est nécessaire.

---

# 45. Éviter que les Child Modules s'appellent directement

Nous aurions pu essayer de faire en sorte que :

```text
modules/network-security
```

appelle directement :

```text
modules/network
```

Mais nous préférons :

```text
           Root Module
           /         \
          ▼           ▼
     network    network-security
          │           ▲
          └── output ─┘
```

Le Root Module compose les modules.

Cela garde l'arbre relativement plat et rend les composants plus indépendants.

---

# 46. Modifier `outputs.tf` du Root Module

Remplacer par :

```hcl
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

# 47. Ne pas lancer `apply` maintenant

Nous avons modifié les adresses Terraform.

Si nous exécutons immédiatement :

```powershell
terraform plan
```

sans préparer le refactoring, Terraform pourrait considérer que plusieurs ressources ont changé d'adresse.

Nous allons déclarer explicitement les déplacements.

---

# 48. Créer `moved.tf`

Créer :

```text
moved.tf
```

avec :

```hcl
moved {
  from = azurerm_virtual_network.main
  to   = module.network.azurerm_virtual_network.this
}

moved {
  from = azurerm_subnet.app
  to   = module.network.azurerm_subnet.app
}

moved {
  from = azurerm_subnet.private_endpoints
  to   = module.network.azurerm_subnet.private_endpoints
}

moved {
  from = azurerm_network_security_group.application
  to   = module.network_security.azurerm_network_security_group.this
}

moved {
  from = azurerm_subnet_network_security_group_association.app
  to   = module.network_security.azurerm_subnet_network_security_group_association.app
}
```

Le Resource Group n'a pas changé d'adresse.

Nous ne créons donc aucun `moved` block pour lui.

---

# 49. Ce que signifie un `moved` block

Prenons :

```hcl
moved {
  from = azurerm_virtual_network.main
  to   = module.network.azurerm_virtual_network.this
}
```

Terraform comprend :

```text
ancien binding
azurerm_virtual_network.main
           │
           ▼
nouveau binding
module.network.azurerm_virtual_network.this
```

L'objet Azure reste :

```text
vnet-mla-dev-fce-001
```

---

# 50. Initialiser les nouveaux modules

Les sources de modules ont changé.

Exécuter :

```powershell
terraform init `
  -backend-config="backend.azurerm.tfbackend"
```

Terraform affiche notamment :

```text
Initializing modules...
```

Les modules locaux sont maintenant disponibles dans la configuration.

---

# 51. Valider

Exécuter :

```powershell
terraform fmt -recursive
```

Puis :

```powershell
terraform validate
```

Résultat attendu :

```text
Success! The configuration is valid.
```

---

# 52. Vérifier les providers

Exécuter :

```powershell
terraform providers
```

Terraform doit montrer le provider AzureRM utilisé par :

```text
root
module.network
module.network_security
```

Nous n'avons pourtant défini qu'un seul :

```hcl
provider "azurerm"
```

dans le Root Module.

---

# 53. Générer le plan de refactoring

Exécuter :

```powershell
terraform plan `
  -out="lab-07-refactor.tfplan"
```

Observer attentivement le résultat.

Terraform doit indiquer les déplacements.

Par exemple conceptuellement :

```text
azurerm_virtual_network.main
has moved to
module.network.azurerm_virtual_network.this
```

---

# 54. Résultat attendu

Le point crucial est :

```text
Plan: 0 to add, 0 to change, 0 to destroy.
```

Nous voulons modifier :

```text
structure Terraform
```

sans modifier :

```text
infrastructure Azure
```

---

# 55. Si Terraform propose un destroy

Ne pas appliquer.

Si nous observons :

```text
-/+
```

ou :

```text
destroy
```

sur le VNet ou les Subnets, arrêter.

Vérifier :

- `moved.tf` ;
- les anciennes adresses ;
- les nouvelles adresses ;
- les noms des ressources dans les modules ;
- les arguments des ressources ;
- les provider configurations.

Un refactoring de structure ne doit pas provoquer accidentellement une reconstruction réseau.

---

# 56. Appliquer le déplacement

Lorsque le plan indique :

```text
0 add
0 change
0 destroy
```

appliquer :

```powershell
terraform apply ".\lab-07-refactor.tfplan"
```

Terraform met à jour les bindings dans le state.

Azure reste inchangé.

---

# 57. Observer le state après refactoring

Exécuter :

```powershell
terraform state list
```

Résultat attendu :

```text
azurerm_resource_group.main

module.network.azurerm_subnet.app

module.network.azurerm_subnet.private_endpoints

module.network.azurerm_virtual_network.this

module.network_security.azurerm_network_security_group.this

module.network_security.azurerm_subnet_network_security_group_association.app
```

Nous avons changé la structure du state.

Pas l'infrastructure Azure.

---

# 58. Vérifier Azure

Récupérer :

```powershell
$rgName = terraform output -raw resource_group_name

$vnetName = terraform output -raw virtual_network_name
```

Puis :

```powershell
az network vnet show `
  --resource-group $rgName `
  --name $vnetName `
  --output table
```

Les mêmes ressources Azure doivent être présentes.

---

# 59. Vérification finale

Exécuter :

```powershell
terraform plan
```

Résultat attendu :

```text
No changes.
Your infrastructure matches the configuration.
```

Nous avons réalisé notre premier refactoring Terraform sans modification Azure.

---

# 60. Interface du module

Un Child Module doit être vu comme une petite API.

Pour `module.network` :

```text
INPUTS
│
├── resource_group_name
├── location
├── virtual_network_name
├── address_spaces
├── subnet names
├── subnet prefixes
└── tags
       │
       ▼
   MODULE NETWORK
       │
       ▼
OUTPUTS
│
├── virtual_network_id
├── virtual_network_name
├── subnet_ids
└── subnet_names
```

L'intérieur du module peut évoluer tant que son contrat reste compatible.

---

# 61. Éviter d'exposer tous les attributs

Un module ne devrait pas automatiquement exposer :

```text
100 outputs
```

simplement parce que les ressources possèdent 100 attributs.

Exposer seulement ce qui est nécessaire à :

- l'appelant ;
- d'autres modules ;
- l'exploitation ;
- l'automatisation.

---

# 62. Éviter les variables inutiles

Même problème avec les inputs.

Un module qui possède :

```text
70 variables
```

pour exposer absolument chaque argument de toutes ses ressources devient parfois une simple couche de translation autour du provider.

Il faut trouver un équilibre entre :

```text
flexibilité
```

et :

```text
contrat compréhensible
```

---

# 63. Module trop spécifique

Exemple :

```text
module-network-for-my-app-dev-francecentral
```

avec :

```text
environment = dev
location = francecentral
VNet = 10.20.0.0/16
```

codés directement dans le module.

Ce module sera difficilement réutilisable ailleurs.

---

# 64. Module trop générique

À l'inverse :

```text
module-network-everything
```

avec :

```text
150 variables
40 booléens
20 dynamic blocks
15 modes de fonctionnement
```

peut devenir difficile à comprendre et tester.

La généricité excessive est également une forme de complexité.

---

# 65. Module professionnel

Un module professionnel cherche plutôt à fournir :

```text
Responsabilité claire

Interface claire

Defaults raisonnables

Types stricts

Validations utiles

Outputs nécessaires

Documentation

Tests

Versionnement

Upgrade path
```

---

# 66. Provider dans les modules

Règle importante :

```text
Root Module
│
└── provider configuration
```

Les Child Modules déclarent :

```text
required_providers
```

mais n'embarquent normalement pas leur propre bloc :

```hcl
provider "azurerm"
```

La configuration provider du Root Module peut être transmise aux Child Modules.

---

# 67. Pourquoi ?

L'identité Azure appartient à l'exécution de la plateforme.

Un module réseau ne devrait pas décider :

```text
quel tenant
quelle subscription
quelle identité
quel credential
```

Le consommateur du module contrôle cela depuis le Root Module.

---

# 68. Provider alias

Plus tard, nous pourrions avoir :

```text
Subscription Connectivity

Subscription Identity

Subscription Workload
```

avec plusieurs configurations AzureRM.

Nous pourrons alors passer explicitement une configuration provider à un module.

Exemple conceptuel :

```hcl
module "network" {
  source = "./modules/network"

  providers = {
    azurerm = azurerm.connectivity
  }
}
```

Nous traiterons ce scénario lorsque nous aborderons des architectures Azure multi-subscriptions.

---

# 69. Modules et state

Créer un Child Module ne crée pas automatiquement un nouveau Terraform State.

Notre architecture actuelle reste :

```text
Root Module
│
├── module.network
└── module.network_security
        │
        ▼
Même Remote State
```

Le backend appartient à la configuration Root.

---

# 70. Un module n'est donc pas une frontière de state

Nous devons distinguer :

```text
Module boundary
```

et :

```text
State boundary
```

Un module est une frontière de code et d'interface.

Un state représente une frontière opérationnelle différente.

Cette distinction deviendra essentielle avec DEV / QUALIF / PROD.

---

# 71. Source locale d'un module

Nous utilisons :

```hcl
source = "./modules/network"
```

Terraform charge le module depuis le repository local.

C'est pratique lorsque :

- Root Module et modules évoluent ensemble ;
- le module n'est pas encore partagé ;
- nous développons le composant.

---

# 72. Module Registry

Un module partagé peut être publié dans un registry.

Le consommateur pourrait alors utiliser :

```hcl
module "network" {
  source  = "organisation/network/azurerm"
  version = "1.2.0"

  # ...
}
```

Le numéro de version devient alors partie intégrante du contrat.

---

# 73. Local module et `version`

Avec :

```hcl
source = "./modules/network"
```

nous n'ajoutons pas :

```hcl
version = "1.0.0"
```

Le paramètre `version` s'applique aux modules provenant d'un registry.

Un module local utilise la version du code présent dans le repository.

---

# 74. Versionner les modules partagés

Lorsqu'un module devient partagé entre plusieurs plateformes :

```text
Platform A
Platform B
Platform C
       │
       ▼
terraform-azurerm-network
```

il devient risqué de suivre automatiquement :

```text
latest
```

Nous voulons pouvoir dire :

```text
Platform A → module 1.2.0

Platform B → module 1.1.4

Platform C → module 2.0.0
```

et contrôler les migrations.

---

# 75. Semantic Versioning

Une convention fréquente est :

```text
MAJOR.MINOR.PATCH
```

Exemple :

```text
1.4.2
```

Conceptuellement :

```text
PATCH
correction compatible

MINOR
nouvelle capacité compatible

MAJOR
breaking change
```

Cela nécessite toutefois que l'équipe respecte réellement cette politique.

---

# 76. Root Module et contraintes provider

Notre Root Module utilise :

```hcl
version = "~> 5.0"
```

Il contrôle l'ensemble de la plateforme.

Les Child Modules utilisent :

```hcl
version = ">= 5.0, < 6.0"
```

Ils expriment les versions AzureRM avec lesquelles leur code est prévu pour fonctionner.

Terraform doit trouver une version qui satisfait toutes les contraintes.

---

# 77. `.terraform.lock.hcl`

Nous ne créons pas :

```text
modules/network/.terraform.lock.hcl
```

pour chacun de nos Child Modules dans cette architecture.

Le Root Module initialise les providers et possède son Dependency Lock File :

```text
lab-07/.terraform.lock.hcl
```

Le module exprime des contraintes.

Le Root Module verrouille la sélection réellement utilisée par cette configuration.

---

# 78. Tester un module indépendamment

Un module professionnel devrait pouvoir être testé avec une configuration dédiée.

Exemple futur :

```text
modules/network/
│
└── ...

examples/
└── network-basic/
    ├── main.tf
    ├── versions.tf
    └── variables.tf
```

L'exemple consomme le module comme le ferait un utilisateur.

---

# 79. README d'un module

Un module destiné à être partagé devrait documenter au minimum :

```text
Purpose

Architecture

Requirements

Providers

Inputs

Outputs

Examples

Limitations

Versioning

Upgrade notes
```

L'interface doit pouvoir être comprise sans lire toutes les ressources internes.

---

# 80. Nommage des modules

Pour un module distribué dans son propre repository, une convention courante est :

```text
terraform-<provider>-<purpose>
```

Par exemple :

```text
terraform-azurerm-network

terraform-azurerm-private-endpoint

terraform-azurerm-application-platform
```

Le nom doit décrire la responsabilité du module.

---

# 81. Un repository par module ?

Dans notre Lab :

```text
terraform-azure-platform/
└── modules/
```

est parfaitement adapté.

Les modules évoluent avec la plateforme.

Lorsque plusieurs repositories et équipes commencent à consommer un module commun, un repository indépendant devient souvent plus pertinent :

```text
terraform-azurerm-network
```

Il peut alors bénéficier de :

- versioning indépendant ;
- pipeline de test ;
- changelog ;
- releases ;
- documentation ;
- registry privé.

---

# 82. Éviter les arbres de modules trop profonds

Architecture difficile à suivre :

```text
Root
└── module A
    └── module B
        └── module C
            └── module D
                └── Resource
```

Le diagnostic d'une erreur devient beaucoup plus difficile.

Nous préférons généralement une composition plus plate :

```text
             Root
        /      |      \
       ▼       ▼       ▼
   Network  Security  Application
```

---

# 83. `terraform graph` avec les modules

Exécuter :

```powershell
terraform graph |
  dot -Tpng -o terraform-lab-07-modules-graph.png
```

Le graphe permettra d'observer notamment la relation :

```text
Resource Group
      │
      ▼
module.network
      │
      ▼
module.network_security
```

Le fichier final pour MonLabAzure sera :

```text
terraform-lab-07-modules-graph.png
```

---

# 84. Module-to-module dependency

Notre relation :

```hcl
app_subnet_id = module.network.subnet_ids["app"]
```

est particulièrement importante.

Nous ne transmettons pas :

```text
nom calculé nous-mêmes
```

Nous transmettons :

```text
ID réellement produit par le module réseau
```

La relation est explicite dans les données.

---

# 85. Ne pas reconstruire un Resource ID

À éviter :

```hcl
app_subnet_id = "/subscriptions/${var.subscription_id}/resourceGroups/..."
```

si :

```hcl
module.network.subnet_ids["app"]
```

est disponible.

La seconde approche :

- évite de reconstruire l'identifiant ;
- exprime une vraie dépendance ;
- réduit les risques de divergence.

---

# 86. Abstraction Leak

Un module devient peu utile lorsque son consommateur doit connaître trop de détails internes.

Exemple :

```text
Pour utiliser module.network,
le Root doit connaître les noms internes exacts
de toutes les resources Terraform.
```

Ce serait une mauvaise encapsulation.

Les outputs doivent fournir les informations nécessaires sans obliger l'appelant à connaître l'organisation interne.

---

# 87. Ne pas utiliser `terraform_remote_state` pour faire communiquer deux modules du même Root

Nos deux modules appartiennent au même Root Module.

Nous utilisons simplement :

```hcl
module.network.subnet_ids["app"]
```

Nous n'avons aucune raison de lire notre propre backend avec :

```text
terraform_remote_state
```

pour transmettre cette information.

---

# 88. Module ou séparation de state ?

Supposons demain :

```text
Network Platform
```

et :

```text
Application Platform
```

avec :

- équipes différentes ;
- droits différents ;
- pipelines différents ;
- cycles de vie différents.

La bonne solution pourrait ne plus être :

```text
deux modules dans le même state
```

mais :

```text
deux Root Modules
+
deux states
```

Les modules ne remplacent donc pas une réflexion sur les frontières opérationnelles.

---

# 89. Azure Verified Modules

Avant de développer systématiquement un module interne pour chaque service Azure, il est pertinent d'évaluer les **Azure Verified Modules (AVM)** disponibles.

Ils fournissent des modules Azure conçus selon des standards Microsoft et publiés pour être réutilisés.

Cela ne signifie pas :

```text
toujours utiliser AVM
```

ni :

```text
toujours développer son propre module
```

La décision dépend notamment :

- des besoins internes ;
- des standards de sécurité ;
- de la couverture fonctionnelle ;
- du niveau d'abstraction recherché ;
- du modèle de support ;
- du cycle de versionnement.

---

# 90. Notre module n'est pas un AVM

Les modules développés dans ce Lab utilisent directement :

```text
AzureRM
```

pour apprendre les mécanismes Terraform.

Ils ne doivent pas être présentés comme des modules conformes au standard d'authoring Azure Verified Modules.

L'objectif du LAB-07 est :

```text
comprendre la conception et la composition Terraform
```

pas :

```text
implémenter la spécification complète AVM
```

---

# 91. Refactoring et `terraform state mv`

Nous aurions pu effectuer :

```powershell
terraform state mv `
  azurerm_virtual_network.main `
  module.network.azurerm_virtual_network.this
```

Cela fonctionne pour une migration ponctuelle du state.

Mais le changement n'aurait pas été exprimé dans le code.

---

# 92. Pourquoi préférer `moved` ?

Le fichier :

```text
moved.tf
```

est :

```text
versionné
relu en Pull Request
reproductible
auditable
applicable aux autres environnements
```

C'est particulièrement important lorsque :

```text
DEV
QUALIF
PROD
```

doivent tous passer par la même migration.

---

# 93. Conserver les `moved` blocks

Ne pas supprimer immédiatement :

```text
moved.tf
```

après avoir migré DEV.

QUALIF ou PROD peuvent encore posséder les anciennes adresses.

Si nous supprimons trop tôt les `moved` blocks, Terraform pourrait perdre le chemin de migration automatique.

---

# 94. Vérification avant production

Un refactoring modulaire devrait typiquement suivre :

```text
Code
 │
 ▼
terraform fmt
 │
 ▼
terraform validate
 │
 ▼
terraform init
 │
 ▼
terraform plan
 │
 ▼
Vérifier moved resources
 │
 ▼
0 add
0 change
0 destroy
 │
 ▼
Review
 │
 ▼
Apply
```

Le plan est la preuve principale que le refactoring structurel n'entraîne pas un changement infrastructure inattendu.

---

# 95. Tester une erreur volontaire

Supprimer temporairement ce `moved` block :

```hcl
moved {
  from = azurerm_virtual_network.main
  to   = module.network.azurerm_virtual_network.this
}
```

Si le state n'a pas encore été migré, exécuter :

```powershell
terraform plan
```

Terraform peut proposer :

```text
destroy ancien VNet

create nouveau VNet
```

Ne pas appliquer.

Remettre immédiatement le `moved` block.

Cet exercice montre pourquoi le state et les adresses Terraform sont importants lors d'un refactoring.

---

# 96. Bon module : question à poser

Pour chaque module, demander :

```text
Quelle responsabilité possède ce module ?

Qu'est-ce que son consommateur doit décider ?

Qu'est-ce que le module doit imposer ?

Quels outputs sont réellement nécessaires ?

Quel est son cycle de vie ?

Qui le maintient ?

Qui le consomme ?

Peut-il être testé indépendamment ?

Comment est-il versionné ?

Comment migrer une breaking change ?
```

C'est cette réflexion qui distingue un simple dossier Terraform d'un module réellement maintenable.

---

# 97. Versionner LAB-07

Depuis :

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

Ne pas versionner :

```text
terraform.tfstate
*.tfplan
terraform.tfvars
*.tfbackend
.terraform/
```

Versionner :

```text
*.tf
terraform.tfvars.example
.terraform.lock.hcl
moved.tf
modules/
```

---

# 98. Commit

Créer :

```powershell
git commit -m "refactor(terraform): modularize network platform"
```

---

# 99. Nettoyage du workload

Revenir :

```powershell
Set-Location "C:\Labs\MonLabAzure\terraform-azure-platform\lab-07"
```

Vérifier :

```powershell
terraform plan
```

Nous voulons d'abord :

```text
No changes.
```

Puis :

```powershell
terraform plan `
  -destroy `
  -out="lab-07-destroy.tfplan"
```

Lire :

```powershell
terraform show ".\lab-07-destroy.tfplan"
```

---

# 100. Appliquer la destruction

Après vérification :

```powershell
terraform apply ".\lab-07-destroy.tfplan"
```

Terraform utilise maintenant les adresses modulaires pour détruire les ressources.

Le backend Azure du LAB-06 reste conservé.

---

# 101. Code final du Root Module

À la fin du Lab, le Root Module doit principalement ressembler à :

```text
lab-07/
│
├── versions.tf
├── providers.tf
├── variables.tf
├── locals.tf
├── main.tf
├── outputs.tf
├── moved.tf
│
└── modules/
    ├── network/
    └── network-security/
```

Et :

```hcl
module "network" {
  source = "./modules/network"

  # ...
}

module "network_security" {
  source = "./modules/network-security"

  app_subnet_id = module.network.subnet_ids["app"]

  # ...
}
```

Le Root Module compose.

Les Child Modules implémentent.

---

# 102. Modèle mental

Nous sommes passés de :

```text
ROOT MODULE

Resource
Resource
Resource
Resource
Resource
Resource
```

à :

```text
ROOT MODULE
     │
     ├── Resource Group
     │
     ├── NETWORK MODULE
     │     ├── VNet
     │     ├── Subnet
     │     └── Subnet
     │
     └── SECURITY MODULE
           ├── NSG
           └── Association
```

---

# 103. Ce qu'il faut retenir

Un module Terraform n'est pas simplement :

```text
un dossier pour ranger le code
```

Il représente une frontière logique avec :

```text
inputs
      │
      ▼
module
      │
      ▼
outputs
```

---

## Root Module

Le Root Module :

- configure le provider ;
- configure le backend ;
- assemble les composants ;
- transmet les inputs ;
- consomme les outputs ;
- représente une unité de déploiement.

---

## Child Module

Un Child Module :

- possède une responsabilité claire ;
- déclare ses provider requirements ;
- reçoit ses paramètres via des variables ;
- expose les informations nécessaires via des outputs ;
- ne décide généralement pas de l'identité Azure utilisée par l'appelant.

---

## Composition

Les modules doivent idéalement être composés depuis le Root Module :

```text
Root
├── module A
├── module B
└── module C
```

plutôt que créer inutilement une arborescence profondément imbriquée.

---

## Refactoring

Déplacer une ressource dans un module change son adresse Terraform.

Les `moved` blocks permettent de préserver les bindings existants sans recréer la ressource distante.

---

# 104. Qu'est-ce qu'un module Terraform ?

Réponse possible :

> Un module Terraform est un ensemble de fichiers Terraform appartenant au même répertoire et représentant une unité de configuration. Le répertoire depuis lequel Terraform est exécuté est le Root Module. Celui-ci peut appeler des Child Modules avec des blocs `module`. J'utilise les modules pour encapsuler des responsabilités cohérentes, définir des interfaces via inputs et outputs et faciliter la réutilisation, sans tomber dans le pattern un module par ressource.

---

# 105. Provider dans Root ou Child Module ?

Réponse :

> Je configure les providers dans le Root Module. Un Child Module réutilisable déclare ses `required_providers`, mais il ne doit généralement pas embarquer sa propre configuration `provider`. Cela permet au consommateur de contrôler l'identité, la subscription et les éventuels aliases du provider.

---

# 106. Comment déplacer une ressource dans un module sans la recréer ?

Réponse :

> Le déplacement change l'adresse Terraform. J'ajoute donc un `moved` block entre l'ancienne adresse et la nouvelle adresse dans le Child Module. Je vérifie ensuite que le plan indique le déplacement avec zéro création, zéro modification et zéro destruction avant d'appliquer.

---

# 107. Module local ou registry ?

Réponse :

> J'utilise volontiers un module local lorsque le composant évolue avec la même plateforme. Lorsqu'il devient partagé entre plusieurs repositories ou équipes, je préfère le versionner indépendamment et le publier dans un registry privé ou un repository dédié. Cela permet aux consommateurs de maîtriser les upgrades.

---

# 108. Faut-il tout modulariser ?

Réponse :

> Non. Une abstraction doit résoudre un problème. Encapsuler une ressource triviale dans un module uniquement pour dire que tout est modulaire peut rendre le code plus complexe. Je crée plutôt des modules autour de responsabilités cohérentes, de conventions réutilisables ou de composants nécessitant une interface stable.

---

# 109. Validation du LAB-07

Le Lab est terminé lorsque les points suivants sont compris et testés :

```text
[ ] Comprendre Root Module

[ ] Comprendre Child Module

[ ] Comprendre module source

[ ] Comprendre inputs

[ ] Comprendre outputs

[ ] Comprendre l'encapsulation

[ ] Comprendre la composition

[ ] Comprendre pourquoi 1 ressource ≠ forcément 1 module

[ ] Déployer la version plate

[ ] Observer les adresses initiales dans le state

[ ] Créer modules/network

[ ] Créer modules/network-security

[ ] Déclarer required_providers dans les Child Modules

[ ] Garder provider "azurerm" dans le Root Module

[ ] Appeler module.network

[ ] Appeler module.network_security

[ ] Faire transiter le Subnet ID via un output

[ ] Comprendre la dépendance implicite entre modules

[ ] Créer moved.tf

[ ] Déplacer le VNet dans module.network

[ ] Déplacer les deux Subnets

[ ] Déplacer le NSG

[ ] Déplacer l'association NSG

[ ] Obtenir 0 add / 0 change / 0 destroy

[ ] Appliquer le refactoring

[ ] Observer les nouvelles adresses du state

[ ] Obtenir terraform plan = No changes

[ ] Comprendre modules vs state boundaries

[ ] Comprendre module local vs module registry

[ ] Comprendre le versionnement des modules

[ ] Comprendre les risques d'un module trop générique

[ ] Comprendre les risques d'un module trop spécifique

[ ] Comprendre l'intérêt des Azure Verified Modules

[ ] Détruire proprement le workload
```

---

# 110. Progression

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
     ├── Root Module
     ├── Child Modules
     ├── Inputs
     ├── Outputs
     ├── Composition
     ├── Provider inheritance
     ├── Versioning
     └── Refactoring
```

Notre code commence maintenant à ressembler à une vraie plateforme Terraform structurée.

---

# 111. Suite

Nous avons encore une infrastructure uniquement :

```text
DEV
```

La prochaine étape consiste à résoudre une question centrale des plateformes Terraform :

> Comment gérer proprement DEV, QUALIF et PROD sans copier trois fois tout le code ?

Nous allons comparer plusieurs stratégies :

```text
Workspaces

Folders

tfvars

Repositories séparés

States séparés

Modules partagés
```

Puis construire une architecture explicite :

```text
environments/
│
├── dev/
├── qual/
└── prod/

modules/
│
├── network/
└── network-security/
```

avec :

```text
1 environnement
=
1 Root Module
=
1 State
=
1 pipeline
=
1 frontière de déploiement
```

Le prochain article sera :

**LAB-08 - Structurer DEV, QUALIF et PROD avec Terraform**

---

## Références officielles

Pour approfondir les mécanismes étudiés :

- HashiCorp Developer - Modules Overview ;
- HashiCorp Developer - Creating Modules ;
- HashiCorp Developer - Module Block Reference ;
- HashiCorp Developer - Providers Within Modules ;
- HashiCorp Developer - Standard Module Structure ;
- HashiCorp Developer - Refactor Modules ;
- HashiCorp Developer - `moved` Block ;
- HashiCorp Developer - Module Version Constraints ;
- HashiCorp Developer - Terraform Style Guide ;
- Microsoft Azure - Azure Verified Modules.

Un module doit rester une abstraction utile, testable et compréhensible. Ajouter une couche de module qui ne réduit ni la duplication, ni la complexité, ni le risque opérationnel n'améliore pas automatiquement une architecture Terraform.