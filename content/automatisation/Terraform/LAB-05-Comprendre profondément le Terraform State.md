---
title: "LAB-05 - Comprendre profondément le Terraform State"
date: 2026-10-06
draft: false
description: "Comprendre en profondeur le Terraform State : mapping entre configuration et Azure, drift, refresh-only, perte de state, import, state rm, moved blocks, sécurité et récupération."
summary: "Manipuler et diagnostiquer le Terraform State à travers plusieurs incidents réalistes sur Microsoft Azure."
slug: "terraform-azure-lab-05-terraform-state"
categories:
  - "Automatisation"
tags:
  - "Terraform"
  - "Azure"
  - "Terraform State"
  - "Drift"
  - "Import"
  - "Infrastructure as Code"
  - "AzureRM"
series:
  - "Terraform Azure - De zéro à production"
weight: 6
ShowToc: false
TocOpen: false
---


Le Terraform State est probablement l'un des concepts les plus importants à maîtriser pour travailler sérieusement avec Terraform.

Jusqu'ici, nous avons surtout observé :

```text
terraform.tfstate
```

et utilisé quelques commandes :

```powershell
terraform state list
terraform state show
```

Mais cela ne suffit pas pour comprendre réellement ce que Terraform fait.

Dans ce Lab, nous allons provoquer volontairement plusieurs situations réalistes :

```text
modification manuelle dans Azure
        │
        ▼
drift

suppression manuelle d'une ressource
        │
        ▼
drift

disparition du fichier terraform.tfstate
        │
        ▼
Terraform perd ses associations

suppression d'une ressource du state
        │
        ▼
Terraform oublie la ressource

import
        │
        ▼
Terraform rattache une ressource existante

renommage d'une adresse Terraform
        │
        ▼
moved block
```

L'objectif est de comprendre précisément la relation entre :

```text
Configuration Terraform
        │
        ▼
Terraform State
        │
        ▼
Provider AzureRM
        │
        ▼
Microsoft Azure
```

---

# 1. Objectifs du Lab

Nous allons apprendre à comprendre et manipuler :

- `terraform.tfstate` ;
- `terraform.tfstate.backup` ;
- resource address ;
- Azure Resource ID ;
- mapping entre configuration et ressource réelle ;
- cache d'attributs ;
- state local ;
- `terraform state list` ;
- `terraform state show` ;
- `terraform state pull` ;
- `terraform show` ;
- drift ;
- refresh implicite ;
- `terraform plan -refresh-only` ;
- `terraform apply -refresh-only` ;
- pourquoi `terraform refresh` n'est plus recommandé ;
- suppression manuelle Azure ;
- perte de state ;
- `terraform state rm` ;
- `terraform import` ;
- `moved` blocks ;
- `terraform state mv` ;
- sécurité du state ;
- informations sensibles ;
- backups ;
- risques du state local ;
- principes de récupération après incident.

---

# 2. Architecture utilisée

Nous repartons de l'architecture du LAB-04 :

```text
Subscription Azure
│
└── rg-mla-dev-fce-001
     │
     ├── vnet-mla-dev-fce-001
     │    │
     │    ├── snet-app-dev-fce-001
     │    │     │
     │    │     └─────────────┐
     │    │                   │
     │    └── snet-pe-dev-fce-001
     │                        │
     └── nsg-app-dev-fce-001 │
              │              │
              └──────────────┘
                     │
                     ▼
              Association
              Subnet ↔ NSG
```

Soit six objets gérés par Terraform :

```text
azurerm_resource_group.main

azurerm_virtual_network.main

azurerm_subnet.app

azurerm_subnet.private_endpoints

azurerm_network_security_group.app

azurerm_subnet_network_security_group_association.app
```

---

# 3. Le problème que résout le state

Prenons cette configuration :

```hcl
resource "azurerm_resource_group" "main" {
  name     = "rg-mla-dev-fce-001"
  location = "francecentral"
}
```

Pour Terraform, l'adresse logique est :

```text
azurerm_resource_group.main
```

Mais Azure connaît la ressource avec un identifiant de ce type :

```text
/subscriptions/<subscription-id>
/resourceGroups/rg-mla-dev-fce-001
```

Terraform doit donc mémoriser la relation :

```text
Adresse Terraform
azurerm_resource_group.main
          │
          │ correspond à
          ▼
Azure Resource ID
/subscriptions/.../resourceGroups/rg-mla-dev-fce-001
```

Cette association est l'un des rôles fondamentaux du state.

---

# 4. Le state n'est pas simplement un cache

Il est fréquent d'entendre :

> Le state est juste un cache de l'infrastructure Azure.

C'est incomplet.

Le state sert principalement à maintenir les **bindings**, c'est-à-dire les associations entre les instances déclarées dans Terraform et les objets réels gérés par le provider.

Il conserve également :

- des attributs ;
- des métadonnées ;
- des informations nécessaires au provider ;
- des outputs ;
- des dépendances et informations techniques utiles à Terraform.

Une représentation simplifiée serait :

```text
Configuration
│
│  azurerm_virtual_network.main
│
▼
Terraform State
│
│  Resource ID Azure
│  attributs connus
│  provider
│  métadonnées
│
▼
Azure
│
└── vnet-mla-dev-fce-001
```

---

# 5. Pourquoi Terraform ne découvre-t-il pas simplement tout Azure ?

Imaginons une subscription contenant :

```text
2 000 ressources
```

Terraform pourrait théoriquement demander à Azure :

> Parmi toutes ces ressources, laquelle correspond à `azurerm_virtual_network.main` ?

Mais rien ne garantit qu'un nom Azure permette à lui seul de reconstruire l'association correcte.

Terraform doit également gérer :

- plusieurs ressources du même type ;
- plusieurs subscriptions ;
- plusieurs providers ;
- plusieurs instances `count` ;
- plusieurs instances `for_each` ;
- plusieurs modules ;
- changements de noms ;
- refactorings.

Le state constitue donc une base de données de correspondance propre à Terraform.

---

# 6. Trois réalités à ne jamais confondre

Il faut maintenant raisonner avec trois éléments distincts.

```text
┌──────────────────────────┐
│      CONFIGURATION       │
│                          │
│ Fichiers .tf             │
│ Ce que nous voulons      │
└────────────┬─────────────┘
             │
             ▼
┌──────────────────────────┐
│          STATE           │
│                          │
│ Ce que Terraform suit    │
│ et associe               │
└────────────┬─────────────┘
             │
             ▼
┌──────────────────────────┐
│          AZURE           │
│                          │
│ Ce qui existe réellement │
└──────────────────────────┘
```

Ces trois éléments peuvent diverger.

C'est précisément ce que nous allons provoquer.

---

# 7. Préparer LAB-05

Depuis PowerShell :

```powershell
Set-Location "C:\Labs\MonLabAzure\terraform-azure-platform"
```

Créer :

```powershell
New-Item -ItemType Directory -Name "lab-05"
```

Puis :

```powershell
Set-Location ".\lab-05"
code .
```

---

# 8. Arborescence

Nous utiliserons :

```text
lab-05/
│
├── versions.tf
├── providers.tf
├── variables.tf
├── locals.tf
├── main.tf
├── outputs.tf
├── terraform.tfvars.example
└── terraform.tfvars
```

---

# 9. `versions.tf`

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

# 10. `providers.tf`

```hcl
provider "azurerm" {
  features {}
}
```

---

# 11. `variables.tf`

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
  description = "Région Azure cible."
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
  description = "Configuration réseau du Lab."

  type = object({
    vnet_address_spaces = list(string)
    app_subnet_prefixes = list(string)
    pe_subnet_prefixes  = list(string)
  })

  default = {
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

  nullable = false
}

variable "additional_tags" {
  description = "Tags Azure supplémentaires."
  type        = map(string)
  default     = {}
}
```

---

# 12. `locals.tf`

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
    lab         = "lab-05"
  }

  common_tags = merge(
    var.additional_tags,
    local.mandatory_tags
  )
}
```

---

# 13. `main.tf`

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

resource "azurerm_network_security_group" "app" {
  name                = local.resource_names.network_security_group_app
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name

  tags = local.common_tags
}

resource "azurerm_subnet_network_security_group_association" "app" {
  subnet_id = azurerm_subnet.app.id

  network_security_group_id = azurerm_network_security_group.app.id
}
```

---

# 14. `outputs.tf`

```hcl
output "resource_group_name" {
  description = "Nom du Resource Group."
  value       = azurerm_resource_group.main.name
}

output "resource_group_id" {
  description = "Resource ID Azure du Resource Group."
  value       = azurerm_resource_group.main.id
}

output "virtual_network_name" {
  description = "Nom du Virtual Network."
  value       = azurerm_virtual_network.main.name
}

output "virtual_network_id" {
  description = "Resource ID Azure du Virtual Network."
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
  value       = azurerm_network_security_group.app.id
}
```

---

# 15. `terraform.tfvars.example`

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

# 16. Préparer Azure

Vérifier :

```powershell
az account show `
  --query "{Subscription:name,SubscriptionId:id,TenantId:tenantId}" `
  --output table
```

Puis :

```powershell
$env:ARM_SUBSCRIPTION_ID = az account show --query id -o tsv
$env:ARM_TENANT_ID       = az account show --query tenantId -o tsv
```

---

# 17. Initialiser

```powershell
terraform fmt -recursive
terraform init
terraform validate
```

Résultat attendu :

```text
Success! The configuration is valid.
```

---

# 18. Créer l'infrastructure

```powershell
terraform plan -out="lab-05.tfplan"
```

Résultat attendu :

```text
Plan: 6 to add, 0 to change, 0 to destroy.
```

Appliquer :

```powershell
terraform apply ".\lab-05.tfplan"
```

Puis :

```powershell
terraform plan
```

Résultat :

```text
No changes.
Your infrastructure matches the configuration.
```

Nous disposons maintenant d'un état de référence propre.

---

# 19. Observer les fichiers locaux

Lister les fichiers :

```powershell
Get-ChildItem -Force
```

Nous devons notamment retrouver :

```text
terraform.tfstate
```

Avec le backend local par défaut, Terraform écrit le state dans ce fichier.

---

# 20. Ne pas éditer directement `terraform.tfstate`

Le fichier est au format JSON.

Il est techniquement possible de l'ouvrir.

Mais il ne faut pas en déduire :

> Je peux corriger Terraform en modifiant directement ce JSON.

HashiCorp fournit des commandes spécifiques pour manipuler le state :

```text
terraform state list
terraform state show
terraform state pull
terraform state mv
terraform state rm
terraform state replace-provider
```

Le format interne du state ne doit pas devenir notre interface de gestion quotidienne.

---

# 21. Lister les bindings

Exécuter :

```powershell
terraform state list
```

Résultat attendu :

```text
azurerm_network_security_group.app
azurerm_resource_group.main
azurerm_subnet.app
azurerm_subnet.private_endpoints
azurerm_subnet_network_security_group_association.app
azurerm_virtual_network.main
```

Chaque ligne représente une adresse Terraform actuellement suivie dans ce state.

---

# 22. Inspecter une ressource

Exécuter :

```powershell
terraform state show azurerm_virtual_network.main
```

Nous obtenons notamment :

```text
id
name
location
resource_group_name
address_space
tags
```

Le state contient donc bien plus que le simple nom du VNet.

---

# 23. Resource address vs Azure Resource ID

Terraform utilise :

```text
azurerm_virtual_network.main
```

Azure utilise quelque chose comme :

```text
/subscriptions/...
/resourceGroups/rg-mla-dev-fce-001
/providers/Microsoft.Network
/virtualNetworks/vnet-mla-dev-fce-001
```

Le state relie ces deux identités.

```text
Terraform address
azurerm_virtual_network.main
          │
          ▼
Terraform State
          │
          ▼
Azure Resource ID
/subscriptions/.../virtualNetworks/vnet-mla-dev-fce-001
```

---

# 24. Inspecter le state avec `terraform state pull`

Exécuter :

```powershell
terraform state pull
```

La sortie représente le state actuel.

Pour l'étudier localement :

```powershell
terraform state pull |
  Set-Content -Encoding utf8 ".\state-inspection.json"
```

Ouvrir :

```powershell
code ".\state-inspection.json"
```

Ce fichier est créé uniquement pour le Lab.

Il ne doit pas être versionné ni partagé.

---

# 25. Supprimer la copie après inspection

Après observation :

```powershell
Remove-Item ".\state-inspection.json"
```

Pourquoi ?

Parce qu'un state peut contenir des informations sensibles.

Créer des copies inutiles multiplie les emplacements à protéger.

---

# 26. State et informations sensibles

Un state peut contenir :

- Resource IDs ;
- configurations réseau ;
- attributs de ressources ;
- outputs ;
- secrets utilisés par certaines ressources ;
- tokens ou données sensibles selon les providers.

Même lorsqu'une variable est déclarée :

```hcl
sensitive = true
```

sa valeur peut toujours être présente dans le state.

`sensitive` masque principalement certains affichages.

Ce n'est pas un mécanisme de chiffrement du state.

---

# 27. `terraform show`

Nous pouvons également inspecter le state avec :

```powershell
terraform show
```

Pour obtenir une représentation JSON exploitable par une machine :

```powershell
terraform show -json
```

Attention :

> les sorties JSON peuvent exposer des valeurs marquées sensibles.

Elles ne doivent donc pas être copiées automatiquement dans des logs CI/CD publics.

---

# 28. Premier incident - création d'un drift

Nous allons maintenant modifier Azure **en dehors de Terraform**.

Récupérer :

```powershell
$rgName = terraform output -raw resource_group_name
```

Puis ajouter manuellement un tag :

```powershell
az group update `
  --name $rgName `
  --set tags.drift_source="manual"
```

Vérifier :

```powershell
az group show `
  --name $rgName `
  --query tags `
  --output json
```

Nous avons maintenant volontairement créé un drift.

---

# 29. Qu'est-ce qu'un drift ?

Avant :

```text
Configuration
drift_source absent

State
drift_source absent

Azure
drift_source absent
```

Après la modification manuelle :

```text
Configuration
drift_source absent

State
drift_source absent

Azure
drift_source = manual
```

Les trois réalités ne correspondent plus.

---

# 30. Lancer un plan normal

Exécuter :

```powershell
terraform plan
```

Terraform contacte Azure via le provider.

Il détecte le changement effectué en dehors de Terraform.

Le plan doit montrer que le tag :

```text
drift_source
```

devrait être supprimé pour que la réalité revienne à la configuration déclarée.

---

# 31. Le refresh implicite

Lors d'un `terraform plan` normal, Terraform effectue un refresh en mémoire des ressources qu'il suit.

Conceptuellement :

```text
State enregistré
      │
      ▼
Provider AzureRM
      │
      ▼
Azure
      │
      ▼
État réel actuel
      │
      ▼
Comparaison avec configuration
      │
      ▼
Plan
```

Terraform peut ainsi détecter les changements réalisés en dehors de son workflow.

---

# 32. Le plan normal ne signifie pas accepter le drift

La configuration dit toujours :

```text
drift_source absent
```

Azure dit :

```text
drift_source = manual
```

Terraform propose donc :

```text
Azure
drift_source = manual

        ↓ apply

Azure
drift_source absent
```

Le workflow normal considère la configuration comme l'état désiré.

---

# 33. Deux décisions possibles face à un drift

Lorsqu'un drift est détecté, il faut d'abord déterminer son origine.

### Changement non autorisé

Nous voulons revenir à Terraform :

```text
Azure manuel
     │
     ▼
Terraform apply
     │
     ▼
Configuration restaurée
```

### Changement volontaire et validé

Nous devons généralement intégrer ce changement au code :

```text
Changement Azure validé
        │
        ▼
Modification Terraform
        │
        ▼
Plan
        │
        ▼
Convergence
```

Ne pas adopter automatiquement toute modification manuelle.

---

# 34. Découvrir `-refresh-only`

Exécuter :

```powershell
terraform plan -refresh-only
```

Cette fois, l'objectif n'est pas de remettre Azure en conformité avec la configuration.

Terraform propose uniquement de mettre à jour son state pour refléter ce qu'il observe à distance.

Conceptuellement :

```text
Azure
drift_source = manual
       │
       ▼
-refresh-only
       │
       ▼
State
drift_source = manual
```

---

# 35. `plan -refresh-only` ne modifie pas encore le state

La commande :

```powershell
terraform plan -refresh-only
```

permet d'abord de revoir les changements.

Elle ne valide pas encore l'écriture du nouveau snapshot de state.

C'est précisément l'intérêt de ce mode.

---

# 36. Appliquer le refresh-only

Pour l'expérience :

```powershell
terraform apply -refresh-only
```

Lire attentivement le plan proposé.

Puis approuver.

Terraform met à jour son state pour refléter le changement distant.

Il ne modifie pas Azure dans ce mode.

---

# 37. Vérifier le state

Exécuter :

```powershell
terraform state show azurerm_resource_group.main
```

Le tag :

```text
drift_source = manual
```

doit maintenant être visible dans le state.

Nous avons :

```text
Configuration
drift_source absent

State
drift_source = manual

Azure
drift_source = manual
```

---

# 38. Le problème n'est pas résolu pour autant

Exécuter maintenant :

```powershell
terraform plan
```

Terraform compare :

```text
Configuration
drift_source absent
```

avec :

```text
Azure
drift_source = manual
```

Il propose toujours de supprimer le tag.

C'est important :

> `refresh-only` synchronise le state avec la réalité distante. Il ne transforme pas automatiquement la configuration Terraform.

---

# 39. Corriger le drift

Dans notre cas, le changement manuel n'est pas souhaité.

Appliquer :

```powershell
terraform plan -out="fix-drift.tfplan"
terraform apply ".\fix-drift.tfplan"
```

Puis :

```powershell
terraform plan
```

Résultat attendu :

```text
No changes.
```

Les trois réalités correspondent de nouveau.

---

# 40. Pourquoi ne pas utiliser `terraform refresh` ?

Historiquement, Terraform proposait :

```powershell
terraform refresh
```

Cette commande existe encore mais elle est dépréciée.

Elle revient essentiellement à réaliser un refresh-only avec application automatique.

Le problème est qu'elle peut modifier immédiatement le state sans nous permettre de revoir correctement les changements avant écriture.

Nous préférons donc :

```powershell
terraform plan -refresh-only
```

puis, lorsque la mise à jour du state est réellement souhaitée :

```powershell
terraform apply -refresh-only
```

---

# 41. Deuxième incident - suppression manuelle Azure

Nous allons supprimer volontairement le Subnet Private Endpoints.

Récupérer :

```powershell
$vnetName = terraform output -raw virtual_network_name
```

Puis :

```powershell
az network vnet subnet delete `
  --resource-group $rgName `
  --vnet-name $vnetName `
  --name "snet-pe-dev-fce-001"
```

Vérifier :

```powershell
az network vnet subnet list `
  --resource-group $rgName `
  --vnet-name $vnetName `
  --output table
```

Le Subnet Private Endpoints doit avoir disparu.

---

# 42. Situation actuelle

Terraform pense encore avoir un binding :

```text
azurerm_subnet.private_endpoints
       │
       ▼
Subnet Azure
```

mais Azure répond :

```text
objet introuvable
```

Nous avons créé un autre type de drift.

---

# 43. Lancer le plan

Exécuter :

```powershell
terraform plan
```

Terraform doit détecter que la ressource distante n'existe plus.

La configuration demande toujours :

```hcl
resource "azurerm_subnet" "private_endpoints"
```

Terraform propose donc de la recréer.

Résultat attendu approximatif :

```text
Plan: 1 to add, 0 to change, 0 to destroy.
```

---

# 44. Terraform ne recrée pas parce que le state est vide

Le mécanisme est plus subtil.

Terraform commence avec :

```text
State
Subnet connu
```

puis interroge Azure :

```text
Azure
Subnet absent
```

Il actualise sa compréhension pendant le plan.

Puis il compare avec :

```text
Configuration
Subnet demandé
```

Conclusion :

```text
Resource missing
        │
        ▼
Create
```

---

# 45. Recréer le Subnet

Créer un plan sauvegardé :

```powershell
terraform plan -out="restore-subnet.tfplan"
```

Puis :

```powershell
terraform apply ".\restore-subnet.tfplan"
```

Vérifier :

```powershell
terraform plan
```

Résultat :

```text
No changes.
```

---

# 46. Troisième incident - perte du state

Nous allons maintenant simuler un incident beaucoup plus sérieux.

Ne jamais faire cette manipulation directement sur un environnement important sans sauvegarde et procédure de récupération.

Créer d'abord une copie :

```powershell
Copy-Item `
  ".\terraform.tfstate" `
  ".\terraform.tfstate.lab-backup"
```

Puis déplacer le state actif :

```powershell
Move-Item `
  ".\terraform.tfstate" `
  ".\terraform.tfstate.simulated-loss"
```

Azure n'est pas modifié.

---

# 47. Que possède maintenant Terraform ?

Configuration :

```text
présente
```

Azure :

```text
six ressources présentes
```

State :

```text
absent
```

Terraform a perdu sa base de correspondance.

---

# 48. Vérifier

Exécuter :

```powershell
terraform state list
```

Terraform ne retrouve plus les bindings précédents.

Puis :

```powershell
terraform plan
```

Terraform peut maintenant proposer de créer les ressources comme s'il ne les gérait pas encore.

Conceptuellement :

```text
Configuration
6 ressources

State
0 ressource

Azure
6 ressources
```

Terraform ne fait pas automatiquement :

> Ces noms ressemblent à mes ressources, je vais les adopter.

Ce serait dangereux.

---

# 49. Ne surtout pas appliquer

Si le plan propose :

```text
6 to add
```

ne pas exécuter :

```powershell
terraform apply
```

Azure possède déjà les objets.

Selon la ressource, l'API pourrait retourner :

```text
already exists
```

ou provoquer d'autres comportements indésirables.

Le bon diagnostic est :

> le state n'est plus disponible.

---

# 50. Restaurer le state

Supprimer éventuellement le nouveau fichier vide créé pendant les tests s'il existe :

```powershell
Remove-Item ".\terraform.tfstate" -ErrorAction SilentlyContinue
```

Puis restaurer :

```powershell
Move-Item `
  ".\terraform.tfstate.simulated-loss" `
  ".\terraform.tfstate"
```

Vérifier :

```powershell
terraform state list
```

Nous devons retrouver les six ressources.

Puis :

```powershell
terraform plan
```

Résultat attendu :

```text
No changes.
```

---

# 51. Ce que démontre cet incident

La configuration seule ne suffit pas.

Nous avions :

```text
100 % du code Terraform
```

mais sans state, Terraform avait perdu les associations avec les ressources Azure existantes.

C'est une raison majeure pour laquelle la perte du state en production est un incident sérieux.

---

# 52. Quatrième incident - Terraform oublie une seule ressource

Nous allons maintenant utiliser :

```powershell
terraform state rm
```

Cette commande ne détruit pas la ressource distante.

Elle supprime uniquement le binding dans le state.

Nous utiliserons le Resource Group.

Récupérer son identifiant avant l'opération :

```powershell
$rgId = terraform output -raw resource_group_id
```

Afficher :

```powershell
$rgId
```

---

# 53. Supprimer le binding

Exécuter :

```powershell
terraform state rm azurerm_resource_group.main
```

Terraform indique que l'objet a été retiré du state.

Vérifier :

```powershell
terraform state list
```

L'entrée :

```text
azurerm_resource_group.main
```

a disparu.

---

# 54. Azure n'a pas changé

Vérifier :

```powershell
az group exists `
  --name "rg-mla-dev-fce-001"
```

Résultat :

```text
true
```

Nous avons donc :

```text
Configuration
Resource Group présent

State
Resource Group absent

Azure
Resource Group présent
```

---

# 55. Pourquoi `state rm` est dangereux ?

Lancer :

```powershell
terraform plan
```

Terraform voit une ressource déclarée mais sans binding.

Il peut donc proposer :

```text
+ azurerm_resource_group.main
```

Pour Terraform, il s'agit d'un nouvel objet à créer.

Mais Azure possède déjà :

```text
rg-mla-dev-fce-001
```

Un `apply` pourrait échouer avec un message indiquant que la ressource existe déjà et doit être importée.

---

# 56. `state rm` ne signifie pas destroy

C'est une distinction fondamentale :

```text
terraform destroy
```

vise à supprimer l'objet distant et son binding.

```text
terraform state rm
```

vise à supprimer uniquement le binding.

Conceptuellement :

```text
state rm

Terraform ─X─ Resource binding

Azure Resource
     │
     └── continue d'exister
```

---

# 57. Backup automatique des commandes de modification du state

Les commandes Terraform qui modifient directement le state créent des fichiers de sauvegarde.

Après :

```powershell
terraform state rm
```

observer :

```powershell
Get-ChildItem *state*
```

Terraform protège ainsi l'opération en conservant un snapshot précédent.

Ces backups doivent eux-mêmes être considérés comme sensibles.

---

# 58. Réparer avec `terraform import`

Nous avons :

```text
Configuration
azurerm_resource_group.main

Azure
rg-mla-dev-fce-001

State
binding absent
```

Nous voulons reconstruire :

```text
azurerm_resource_group.main
           │
           ▼
Azure Resource Group existant
```

Utiliser :

```powershell
terraform import `
  azurerm_resource_group.main `
  $rgId
```

Terraform rattache la ressource existante à l'adresse Terraform.

---

# 59. Vérifier l'import

Exécuter :

```powershell
terraform state list
```

Puis :

```powershell
terraform state show azurerm_resource_group.main
```

Enfin :

```powershell
terraform plan
```

Si configuration et ressource sont alignées :

```text
No changes.
```

Le binding a été reconstruit.

---

# 60. Ce que fait réellement `terraform import`

L'import ne signifie pas :

```text
copier la ressource Azure
```

Il signifie essentiellement :

```text
Objet Azure existant
        │
        ▼
Associer
        │
        ▼
Adresse Terraform
```

Avec la commande CLI classique, l'import met à jour le state.

Il ne génère pas automatiquement toute la configuration Terraform correspondante.

---

# 61. Import moderne avec un bloc `import`

Terraform permet également une approche déclarative :

```hcl
import {
  to = azurerm_resource_group.main
  id = "/subscriptions/.../resourceGroups/rg-mla-dev-fce-001"
}
```

Cette approche présente un avantage important :

```text
configuration
      │
      ▼
terraform plan
      │
      ▼
review
      │
      ▼
terraform apply
      │
      ▼
import
```

L'import peut alors être revu dans le workflow plan/apply.

Cette approche est particulièrement intéressante en CI/CD.

---

# 62. Import CLI ou bloc `import` ?

Pour un dépannage ponctuel :

```text
terraform import
```

reste pratique.

Pour une opération :

- reproductible ;
- relue en Pull Request ;
- auditable ;
- intégrée au CI/CD ;

le bloc :

```hcl
import {
}
```

est généralement préférable.

---

# 63. Une règle fondamentale de Terraform

Terraform attend normalement une relation :

```text
1 adresse Terraform
        │
        ▼
1 objet distant
```

Éviter d'importer le même objet Azure vers plusieurs adresses Terraform.

Exemple dangereux :

```text
azurerm_resource_group.main ──┐
                              ├── même RG Azure
azurerm_resource_group.other ─┘
```

Cela peut produire des comportements incohérents.

---

# 64. Cinquième incident - renommer une adresse Terraform

Notre NSG est actuellement déclaré :

```hcl
resource "azurerm_network_security_group" "app" {
```

Son adresse est :

```text
azurerm_network_security_group.app
```

Supposons que nous voulions améliorer le nom logique :

```text
app
```

vers :

```text
application
```

---

# 65. Pourquoi un simple renommage est risqué ?

Si nous remplaçons simplement :

```hcl
resource "azurerm_network_security_group" "app"
```

par :

```hcl
resource "azurerm_network_security_group" "application"
```

Terraform voit conceptuellement :

```text
ancienne adresse disparue
azurerm_network_security_group.app

nouvelle adresse apparue
azurerm_network_security_group.application
```

Il ne peut pas supposer automatiquement que nous voulions simplement renommer le binding.

---

# 66. Utiliser un `moved` block

Modifier le bloc NSG :

```hcl
resource "azurerm_network_security_group" "application" {
  name                = local.resource_names.network_security_group_app
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name

  tags = local.common_tags
}
```

Modifier l'association :

```hcl
resource "azurerm_subnet_network_security_group_association" "app" {
  subnet_id = azurerm_subnet.app.id

  network_security_group_id = azurerm_network_security_group.application.id
}
```

Modifier également l'output :

```hcl
output "app_nsg_id" {
  description = "Resource ID du NSG application."
  value       = azurerm_network_security_group.application.id
}
```

---

# 67. Créer `moved.tf`

Créer :

```text
moved.tf
```

avec :

```hcl
moved {
  from = azurerm_network_security_group.app
  to   = azurerm_network_security_group.application
}
```

---

# 68. Ce que dit le `moved` block

Nous expliquons explicitement à Terraform :

```text
ancienne adresse
azurerm_network_security_group.app

        │
        │ devient
        ▼

nouvelle adresse
azurerm_network_security_group.application
```

Mais l'objet Azure doit rester le même :

```text
nsg-app-dev-fce-001
```

---

# 69. Générer le plan

Exécuter :

```powershell
terraform fmt
terraform validate
terraform plan
```

Terraform doit indiquer que la ressource a été déplacée vers la nouvelle adresse.

Le résumé doit rester :

```text
Plan: 0 to add, 0 to change, 0 to destroy.
```

Nous modifions le binding Terraform, pas le NSG Azure.

---

# 70. Appliquer le changement d'adresse

Exécuter :

```powershell
terraform apply
```

Puis :

```powershell
terraform state list
```

Nous devons maintenant retrouver :

```text
azurerm_network_security_group.application
```

et non plus :

```text
azurerm_network_security_group.app
```

Le NSG Azure n'a pas été détruit.

---

# 71. `moved` block ou `terraform state mv` ?

Nous aurions pu exécuter :

```powershell
terraform state mv `
  azurerm_network_security_group.app `
  azurerm_network_security_group.application
```

Mais le `moved` block possède plusieurs avantages.

Il est :

- déclaré dans le code ;
- versionné ;
- visible en Pull Request ;
- réutilisable par d'autres environnements ;
- vérifiable avec `terraform plan`.

Pour les refactorings de configuration, c'est généralement l'approche à privilégier.

---

# 72. Pourquoi conserver le `moved` block ?

Le fichier :

```text
moved.tf
```

constitue également une trace de l'évolution de notre configuration.

Dans une plateforme partagée ou un module versionné, retirer trop tôt un `moved` block peut casser le chemin de migration d'un environnement n'ayant pas encore appliqué la nouvelle version.

Nous conservons donc le bloc dans ce Lab.

---

# 73. Code final du NSG

Notre code final utilise désormais :

```hcl
resource "azurerm_network_security_group" "application" {
  name                = local.resource_names.network_security_group_app
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name

  tags = local.common_tags
}
```

L'association finale :

```hcl
resource "azurerm_subnet_network_security_group_association" "app" {
  subnet_id = azurerm_subnet.app.id

  network_security_group_id = azurerm_network_security_group.application.id
}
```

Et :

```hcl
moved {
  from = azurerm_network_security_group.app
  to   = azurerm_network_security_group.application
}
```

---

# 74. State corruption

Une véritable corruption du state est différente d'un simple drift.

Le drift signifie :

```text
State valide

mais

Azure a changé
```

Une corruption signifie que le snapshot lui-même est endommagé, incohérent ou inutilisable.

Causes possibles :

- manipulation manuelle ;
- écriture interrompue ;
- erreur de stockage ;
- mauvaise opération de state ;
- restauration du mauvais fichier ;
- écrasement accidentel.

---

# 75. Ne pas simuler une corruption en éditant le JSON

Nous pourrions volontairement casser :

```text
terraform.tfstate
```

avec un éditeur.

Mais cela n'apporte pas beaucoup de valeur pédagogique et encourage une mauvaise habitude.

Nous avons déjà simulé deux incidents plus réalistes :

```text
perte du state

et

perte d'un binding
```

Ce sont de meilleurs exercices pour comprendre le fonctionnement.

---

# 76. Que faire face à un state potentiellement corrompu ?

Ne pas commencer par :

```text
supprimer le state
```

ou :

```text
terraform apply
```

Procédure de diagnostic :

```text
1. Stopper les modifications

2. Identifier le backend exact

3. Vérifier qu'aucun autre pipeline n'écrit

4. Sauvegarder le state actuel

5. Vérifier les backups / versions

6. Utiliser terraform state pull

7. Comparer avec Azure

8. Identifier le dernier snapshot valide

9. Évaluer import / restauration / reconstruction

10. Tester terraform plan avant toute action
```

---

# 77. `terraform state push`

Terraform permet :

```powershell
terraform state push
```

Cette commande peut écrire manuellement un snapshot dans le backend.

Elle est extrêmement sensible.

Elle ne doit pas devenir une commande habituelle.

Un mauvais state poussé peut modifier la manière dont Terraform comprend toute une infrastructure.

Nous ne l'utiliserons pas dans ce Lab.

---

# 78. `lineage`

Un state possède notamment une notion de :

```text
lineage
```

Elle permet d'identifier une famille de snapshots appartenant au même historique logique.

Deux states créés indépendamment peuvent posséder des lineages différents.

Terraform utilise cette information pour limiter certains écrasements dangereux lors de manipulations manuelles.

---

# 79. `serial`

Un state possède également un numéro :

```text
serial
```

Il évolue avec les nouvelles versions du snapshot.

Conceptuellement :

```text
state
serial 10

      ↓ nouvelle écriture

state
serial 11

      ↓ nouvelle écriture

state
serial 12
```

Terraform peut ainsi détecter certains scénarios où nous essayons de pousser un state plus ancien sur un state plus récent.

---

# 80. State local et collaboration

Notre architecture actuelle utilise :

```text
PC ingénieur
      │
      └── terraform.tfstate
```

Imaginons deux ingénieurs :

```text
Ingénieur A
terraform.tfstate A

Ingénieur B
terraform.tfstate B
```

Très rapidement, nous pouvons obtenir :

```text
A crée une ressource

B ne possède pas le nouveau state

B lance Terraform

les deux visions divergent
```

C'est une architecture fragile.

---

# 81. Autre problème : opérations simultanées

Imaginez :

```text
Ingénieur A
terraform apply

au même moment

Ingénieur B
terraform apply
```

Sans coordination correcte autour d'un state partagé, les deux opérations peuvent entrer en concurrence.

C'est une raison supplémentaire d'utiliser :

```text
remote state
+
locking
```

en environnement d'équipe.

---

# 82. Le state ne doit pas être stocké dans Git

À éviter :

```text
repository/
├── main.tf
├── variables.tf
└── terraform.tfstate
```

Git :

- n'est pas un backend Terraform ;
- n'assure pas le locking Terraform ;
- duplique le state ;
- conserve l'historique ;
- peut exposer des secrets.

Notre `.gitignore` doit continuer d'exclure :

```gitignore
*.tfstate
*.tfstate.*
```

---

# 83. Vérifier avant commit

Depuis la racine :

```powershell
Set-Location "C:\Labs\MonLabAzure\terraform-azure-platform"
```

Puis :

```powershell
git status
```

Contrôler :

```powershell
git diff --cached --name-only
```

Ne jamais publier :

```text
terraform.tfstate
terraform.tfstate.backup
terraform.tfstate.lab-backup
*.tfplan
.terraform/
```

---

# 84. Supprimer notre backup pédagogique

Une fois les exercices de récupération terminés :

```powershell
Set-Location ".\lab-05"
```

Puis :

```powershell
Remove-Item `
  ".\terraform.tfstate.lab-backup" `
  -ErrorAction SilentlyContinue
```

Ne pas accumuler des copies de state sur le poste.

---

# 85. Versionner le code

Depuis la racine :

```powershell
Set-Location "C:\Labs\MonLabAzure\terraform-azure-platform"
```

Puis :

```powershell
git add .
```

Vérifier :

```powershell
git diff --cached --name-only
```

Puis :

```powershell
git commit -m "feat(terraform): add lab 05 state fundamentals"
```

---

# 86. Nettoyage final

Revenir :

```powershell
Set-Location "C:\Labs\MonLabAzure\terraform-azure-platform\lab-05"
```

Vérifier d'abord :

```powershell
terraform plan
```

Résultat attendu :

```text
No changes.
```

Puis :

```powershell
terraform plan `
  -destroy `
  -out="lab-05-destroy.tfplan"
```

Lire :

```powershell
terraform show ".\lab-05-destroy.tfplan"
```

Nous devons obtenir :

```text
Plan: 0 to add, 0 to change, 6 to destroy.
```

---

# 87. Appliquer la destruction

Après vérification :

```powershell
terraform apply ".\lab-05-destroy.tfplan"
```

Puis :

```powershell
az group exists `
  --name "rg-mla-dev-fce-001"
```

Résultat :

```text
false
```

Enfin :

```powershell
terraform state list
```

Le state ne doit plus contenir les six ressources.

---

# 88. Modèle mental définitif

À ce stade, il faut arrêter de considérer Terraform comme :

```text
fichiers .tf
      │
      ▼
Azure
```

Le modèle réel est davantage :

```text
              CONFIGURATION
                    │
                    │ état désiré
                    ▼
                 TERRAFORM
                /         \
               /           \
              ▼             ▼
           STATE         PROVIDER
              │             │
              │             ▼
              │           AZURE
              │             │
              └──────┬──────┘
                     ▼
                 COMPARAISON
                     │
                     ▼
                    PLAN
```

---

# 89. Configuration

La configuration répond à la question :

> Que voulons-nous ?

Exemple :

```hcl
resource "azurerm_virtual_network" "main" {
  ...
}
```

---

# 90. State

Le state répond notamment à la question :

> Quel objet distant Terraform associe-t-il à cette adresse ?

Exemple :

```text
azurerm_virtual_network.main
      │
      ▼
/subscriptions/.../virtualNetworks/vnet-mla-dev-fce-001
```

---

# 91. Azure

Azure représente :

> Ce qui existe réellement maintenant.

Cette réalité peut avoir changé :

- manuellement ;
- via un autre outil ;
- à cause d'un incident ;
- à cause d'un autre pipeline ;
- à cause d'une suppression.

---

# 92. Plan

Le plan réconcilie ces informations pour déterminer :

```text
create
update
replace
destroy
no-op
```

C'est pourquoi un plan Terraform n'est pas une simple lecture statique du fichier `.tf`.

---

# 93. Le state est critique mais n'est pas l'état désiré

Il faut éviter un autre raccourci :

> Le state est la source de vérité Terraform.

C'est trop simplificateur.

Le state contient la connaissance opérationnelle de Terraform et les bindings nécessaires.

Mais l'état désiré reste défini par la configuration.

La réalité distante est fournie par Azure.

Terraform raisonne entre ces éléments.

---

# 94. Résumé des scénarios du Lab

### Situation normale

```text
Configuration = State = Azure
```

Résultat :

```text
No changes.
```

### Drift manuel

```text
Configuration ≠ Azure
```

Terraform détecte la différence.

### Suppression Azure

```text
Configuration demande ressource
State connaît ressource
Azure ne possède plus ressource
```

Terraform propose de recréer.

### State perdu

```text
Configuration demande ressources
State ne connaît rien
Azure possède déjà ressources
```

Terraform peut proposer des créations qui entreront en conflit avec l'existant.

### `state rm`

```text
Terraform oublie un binding
Azure ne change pas
```

### Import

```text
Terraform rattache un objet existant à une adresse
```

### `moved`

```text
Terraform change l'adresse logique
sans recréer l'objet Azure
```

---

# 95. Commandes à connaître

Inspection :

```powershell
terraform state list
terraform state show <ADDRESS>
terraform state pull
terraform show
```

Diagnostic du drift :

```powershell
terraform plan
terraform plan -refresh-only
```

Synchronisation volontaire du state :

```powershell
terraform apply -refresh-only
```

Modification avancée :

```powershell
terraform state rm <ADDRESS>
terraform state mv <SOURCE> <DESTINATION>
```

Reconstruction d'un binding :

```powershell
terraform import <ADDRESS> <RESOURCE_ID>
```

Ces dernières commandes doivent être utilisées avec davantage de prudence.

---

# 96. Ce qu'un ingénieur Terraform doit pouvoir expliquer

Si on demande :

> À quoi sert réellement le Terraform State ?

Une réponse solide serait :

> Le state permet principalement à Terraform de maintenir le binding entre une instance de ressource dans la configuration, par exemple `azurerm_virtual_network.main`, et l'objet distant correspondant dans Azure. Il conserve également des attributs et des métadonnées utilisés pendant les opérations. Lors d'un plan normal, Terraform utilise le provider pour actualiser sa compréhension des objets suivis puis compare la réalité avec la configuration afin de calculer les changements nécessaires.

---

# 97. Question d'entretien - quelqu'un modifie Azure manuellement

Réponse :

> Lors du prochain plan, Terraform rafraîchit sa connaissance des ressources distantes et détecte normalement le drift. Je vérifie d'abord si le changement manuel est autorisé. S'il ne l'est pas, j'utilise le workflow Terraform pour revenir à la configuration attendue. S'il est légitime, je mets généralement la configuration à jour avant d'appliquer afin que le code redevienne la référence de l'état souhaité.

---

# 98. Question d'entretien - que se passe-t-il si le state est perdu ?

Réponse :

> Terraform perd les bindings entre ses resource addresses et les objets distants. Le code seul ne lui permet pas forcément de redécouvrir automatiquement les ressources existantes. Un plan peut donc proposer de recréer des objets déjà présents. Je n'applique rien. Je cherche d'abord à restaurer un snapshot valide du backend ou de ses versions. Si nécessaire, je reconstruis les bindings avec des imports contrôlés.

---

# 99. Question d'entretien - peut-on modifier le state manuellement ?

Réponse :

> J'évite d'éditer directement le JSON. Terraform fournit des commandes de state et des mécanismes déclaratifs comme `moved`, `removed` et `import`. Une modification directe du fichier doit rester une procédure exceptionnelle de récupération, avec backup, revue et validation très rigoureuse.

---

# 100. Question d'entretien - `sensitive = true` protège-t-il le state ?

Réponse :

> Non. `sensitive = true` masque principalement certaines valeurs dans l'affichage CLI et les plans. Si la valeur doit être persistée par Terraform, elle peut toujours être présente dans le state. L'accès au backend doit donc être considéré comme un accès à des données potentiellement sensibles.

---

# 101. Validation du LAB-05

Le Lab est terminé lorsque les points suivants sont compris et testés :

```text
[ ] Comprendre pourquoi Terraform a besoin d'un state

[ ] Comprendre le binding resource address ↔ Azure Resource ID

[ ] Comprendre configuration / state / Azure

[ ] Utiliser terraform state list

[ ] Utiliser terraform state show

[ ] Utiliser terraform state pull

[ ] Comprendre le contenu sensible du state

[ ] Comprendre le refresh implicite

[ ] Créer volontairement un drift

[ ] Détecter le drift avec terraform plan

[ ] Utiliser terraform plan -refresh-only

[ ] Comprendre terraform apply -refresh-only

[ ] Comprendre pourquoi terraform refresh est déprécié

[ ] Supprimer manuellement une ressource Azure

[ ] Observer sa recréation proposée

[ ] Restaurer la ressource avec Terraform

[ ] Simuler une perte du state

[ ] Comprendre pourquoi Terraform propose alors de recréer

[ ] Restaurer le state

[ ] Utiliser terraform state rm

[ ] Vérifier que la ressource Azure reste présente

[ ] Utiliser terraform import

[ ] Restaurer un binding

[ ] Renommer une adresse Terraform

[ ] Utiliser un moved block

[ ] Vérifier 0 add / 0 change / 0 destroy

[ ] Comprendre terraform state mv

[ ] Comprendre lineage et serial

[ ] Comprendre les backups du state

[ ] Comprendre les risques du state local

[ ] Comprendre pourquoi le state ne doit pas être dans Git

[ ] Détruire proprement l'infrastructure
```

---

# 102. Progression

Nous avons désormais couvert :

```text
LAB-00
Environnement de travail
        │
        ▼
LAB-01
Premier déploiement
        │
        ▼
LAB-02
HCL et modèle déclaratif
        │
        ▼
LAB-03
Variables, locals, outputs et types
        │
        ▼
LAB-04
Dépendances et graphe
        │
        ▼
LAB-05
Terraform State
        │
        ├── binding
        ├── drift
        ├── refresh
        ├── recovery
        ├── import
        └── moved
```

Nous savons maintenant pourquoi le state est critique.

Mais notre architecture possède toujours un problème évident :

```text
terraform.tfstate
       │
       ▼
poste local
```

Ce modèle ne convient pas à une véritable équipe.

---

# 103. Suite

Dans le prochain Lab, nous allons sortir le state du poste de l'ingénieur.

Nous construirons un backend Azure dédié :

```text
Terraform
    │
    ▼
Azure Storage Account
    │
    ▼
Blob Container
    │
    ▼
Terraform State
```

Nous étudierons :

- backend `azurerm` ;
- bootstrap du backend ;
- migration du state local ;
- state distant ;
- authentification ;
- Microsoft Entra ID ;
- Role-Based Access Control (RBAC) ;
- verrouillage ;
- concurrence ;
- sécurité du Storage Account ;
- versioning ;
- récupération ;
- séparation du state et de l'infrastructure qu'il gère ;
- stratégie de nommage des state keys ;
- préparation aux environnements DEV / QUALIF / PROD.

Le prochain article sera :

**LAB-06 - Remote State Azure, backend et locking**

---

## Références officielles

Les mécanismes étudiés dans ce Lab sont documentés principalement dans :

- HashiCorp Developer - Terraform State ;
- HashiCorp Developer - Purpose of Terraform State ;
- HashiCorp Developer - State Commands ;
- HashiCorp Developer - `terraform state list` ;
- HashiCorp Developer - `terraform state show` ;
- HashiCorp Developer - `terraform state pull` ;
- HashiCorp Developer - Refresh-Only Mode ;
- HashiCorp Developer - Import Existing Resources ;
- HashiCorp Developer - Import Block ;
- HashiCorp Developer - `terraform state rm` ;
- HashiCorp Developer - `moved` Block ;
- HashiCorp Developer - Refactoring Terraform ;
- HashiCorp Developer - Manage Sensitive Data ;
- Terraform Registry - HashiCorp AzureRM Provider.

Le state contient potentiellement des informations sensibles et doit être traité comme un composant critique de l'architecture Terraform. Le backend distant et sa sécurisation seront mis en place dans le LAB-06.