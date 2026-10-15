---
title: "LAB-06 - Remote State Azure, backend et locking"
date: 2026-10-08
draft: false
description: "Migrer un Terraform State local vers un backend Azure Storage sécurisé avec Microsoft Entra ID, RBAC, versioning et state locking."
summary: "Construire un backend Terraform Azure, migrer le state local, sécuriser l'accès avec Microsoft Entra ID et comprendre le locking et les accès concurrents."
slug: "terraform-azure-lab-06-remote-state-backend-locking"
categories:
  - "Automatisation"
tags:
  - "Terraform"
  - "Azure"
  - "Remote State"
  - "Azure Storage"
  - "Backend"
  - "State Locking"
  - "Microsoft Entra ID"
  - "RBAC"
  - "AzureRM"
series:
  - "Terraform Azure - De zéro à production"
weight: 7
ShowToc: false
TocOpen: false
---


Dans le LAB-05, nous avons vu pourquoi le Terraform State est un composant critique.

Notre architecture possède pourtant encore une faiblesse importante :

```text
Poste ingénieur
│
├── configuration Terraform
│
└── terraform.tfstate
```

Cette approche fonctionne pour apprendre Terraform seul sur son poste.

Elle ne constitue pas une architecture satisfaisante pour une équipe.

Si le poste est perdu, si deux ingénieurs possèdent chacun une copie différente du state ou si plusieurs pipelines travaillent simultanément, nous pouvons rapidement créer des incohérences.

Nous allons donc passer à :

```text
Terraform
    │
    ▼
Backend azurerm
    │
    ▼
Microsoft Entra ID
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

Le state ne sera plus attaché au poste d'un ingénieur.

Il deviendra une ressource d'infrastructure partagée et sécurisée.

---

# 1. Objectifs du Lab

Nous allons apprendre à :

- comprendre ce qu'est un backend Terraform ;
- distinguer backend et provider ;
- comprendre le problème du bootstrap ;
- créer un backend Azure avec Terraform ;
- utiliser Azure Storage comme backend ;
- utiliser Microsoft Entra ID plutôt qu'une Storage Account Key ;
- utiliser Role-Based Access Control (RBAC) ;
- désactiver Shared Key sur le Storage Account ;
- activer le versioning des blobs ;
- activer le soft delete ;
- protéger le backend contre une suppression accidentelle ;
- utiliser une configuration backend partielle ;
- migrer un state local vers Azure ;
- vérifier le state distant ;
- comprendre les state keys ;
- comprendre le state locking ;
- provoquer volontairement une concurrence ;
- comprendre `-lock-timeout` ;
- comprendre pourquoi `-lock=false` est dangereux ;
- comprendre `terraform force-unlock` ;
- préparer la séparation future DEV / QUALIF / PROD.

---

# 2. Architecture finale

Nous allons construire deux ensembles distincts.

## Infrastructure backend

```text
Subscription Azure
│
└── rg-mla-tfstate-fce-001
     │
     └── Storage Account
          │
          └── Container : tfstate
               │
               └── terraform-azure-platform/
                    └── dev/
                         └── platform.tfstate
```

## Infrastructure gérée

```text
Subscription Azure
│
└── rg-mla-dev-fce-001
     │
     ├── vnet-mla-dev-fce-001
     │    │
     │    ├── snet-app-dev-fce-001
     │    └── snet-pe-dev-fce-001
     │
     └── nsg-app-dev-fce-001
```

La relation devient :

```text
Terraform Configuration
         │
         ├───────────────┐
         │               │
         ▼               ▼
Azure Infrastructure   Azure Storage
                         │
                         ▼
                   Terraform State
```

---

# 3. Pourquoi un backend distant ?

Avec le backend local :

```text
Ingénieur A
│
└── terraform.tfstate A

Ingénieur B
│
└── terraform.tfstate B
```

Nous avons deux visions différentes.

Avec un backend distant :

```text
Ingénieur A ──────┐
                  │
                  ▼
             Remote State
                  ▲
                  │
Ingénieur B ──────┤
                  │
Pipeline CI/CD ───┘
```

Tous les acteurs utilisent le même state.

---

# 4. Le backend n'est pas le provider

Cette distinction est fondamentale.

Le provider :

```text
Terraform
    │
    ▼
AzureRM Provider
    │
    ▼
Azure Resources
```

sert à gérer les ressources Azure.

Le backend :

```text
Terraform
    │
    ▼
azurerm Backend
    │
    ▼
Terraform State
```

sert principalement à stocker le state et à fournir les mécanismes associés au backend, notamment le locking lorsqu'il est supporté.

Nous utilisons donc le mot `azurerm` dans deux contextes différents :

```text
provider "azurerm"

backend "azurerm"
```

Ils ne représentent pas la même fonctionnalité.

---

# 5. Le problème du bootstrap

Pour utiliser :

```hcl
backend "azurerm" {
}
```

le Storage Account doit déjà exister.

Mais nous voulons justement créer le Storage Account avec Terraform.

Nous avons donc un problème circulaire :

```text
Terraform
   │
   ▼
Backend Azure nécessaire
   │
   ▼
Storage Account
   │
   └── doit être créé par Terraform
```

Nous ne pouvons pas utiliser une ressource qui n'existe pas encore comme backend.

---

# 6. Solution : séparer le bootstrap

Nous allons créer :

```text
lab-06/
│
├── bootstrap/
│
└── workload/
```

Le rôle de :

```text
bootstrap/
```

sera uniquement de créer l'infrastructure destinée au backend.

Le rôle de :

```text
workload/
```

sera de gérer notre plateforme Azure.

---

# 7. Architecture du bootstrap

```text
bootstrap/
│
│ State local temporaire
│
▼
Terraform
│
▼
Azure
│
└── rg-mla-tfstate-fce-001
     │
     └── Storage Account
          │
          └── tfstate
```

Une fois le backend disponible :

```text
workload/
│
│ State local
│
▼
terraform init -migrate-state
│
▼
Azure Storage
│
└── Remote State
```

---

# 8. Préparer LAB-06

Depuis PowerShell :

```powershell
Set-Location "C:\Labs\MonLabAzure\terraform-azure-platform"
```

Créer :

```powershell
New-Item -ItemType Directory -Name "lab-06"
```

Puis :

```powershell
Set-Location ".\lab-06"

New-Item -ItemType Directory -Name "bootstrap"
New-Item -ItemType Directory -Name "workload"
```

Nous obtenons :

```text
lab-06/
│
├── bootstrap/
└── workload/
```

---

# 9. Vérifier Azure

Avant toute opération :

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

Vérifier :

```powershell
$env:ARM_SUBSCRIPTION_ID
$env:ARM_TENANT_ID
```

---

# 10. Permissions nécessaires pour le bootstrap

L'identité utilisée doit pouvoir créer :

- Resource Group ;
- Storage Account ;
- Storage Container ;
- Role Assignment.

Le dernier point est important.

Créer :

```text
azurerm_role_assignment
```

nécessite une permission Azure permettant de gérer les attributions RBAC.

Si l'identité peut créer un Storage Account mais pas créer de Role Assignment, le bootstrap échouera à cette étape.

---

# 11. Préparer `bootstrap/versions.tf`

Créer :

```text
bootstrap/versions.tf
```

avec :

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

Aucun backend distant n'est déclaré.

Le bootstrap utilise donc encore le backend local par défaut.

---

# 12. `bootstrap/providers.tf`

Créer :

```text
bootstrap/providers.tf
```

avec :

```hcl
provider "azurerm" {
  features {}

  storage_use_azuread = true
}
```

Nous demandons au provider AzureRM d'utiliser Microsoft Entra ID pour les opérations Azure Storage compatibles.

---

# 13. `bootstrap/main.tf`

Créer :

```text
bootstrap/main.tf
```

avec :

```hcl
data "azurerm_client_config" "current" {}

locals {
  location      = "francecentral"
  location_code = "fce"

  resource_group_name = "rg-mla-tfstate-${local.location_code}-001"

  storage_account_name = substr(
    "stmlatf${md5(data.azurerm_client_config.current.subscription_id)}",
    0,
    24
  )

  container_name = "tfstate"

  common_tags = {
    project     = "mla"
    component   = "terraform-backend"
    managed_by  = "terraform"
    environment = "shared"
    criticality = "high"
    lab         = "lab-06"
  }
}

resource "azurerm_resource_group" "tfstate" {
  name     = local.resource_group_name
  location = local.location

  tags = local.common_tags
}

resource "azurerm_role_assignment" "bootstrap_blob_data" {
  scope                = azurerm_resource_group.tfstate.id
  role_definition_name = "Storage Blob Data Contributor"
  principal_id         = data.azurerm_client_config.current.object_id
}

resource "azurerm_storage_account" "tfstate" {
  name                = local.storage_account_name
  resource_group_name = azurerm_resource_group.tfstate.name
  location            = azurerm_resource_group.tfstate.location

  account_kind             = "StorageV2"
  account_tier             = "Standard"
  account_replication_type = "LRS"

  https_traffic_only_enabled     = true
  min_tls_version                = "TLS1_2"
  allow_nested_items_to_be_public = false

  default_to_oauth_authentication = true
  shared_access_key_enabled       = false

  public_network_access = "Enabled"

  blob_properties {
    versioning_enabled = true

    delete_retention_policy {
      days = 14
    }

    container_delete_retention_policy {
      days = 14
    }
  }

  lifecycle {
    prevent_destroy = true
  }

  tags = local.common_tags

  depends_on = [
    azurerm_role_assignment.bootstrap_blob_data
  ]
}

resource "azurerm_storage_container" "tfstate" {
  name                  = local.container_name
  storage_account_id    = azurerm_storage_account.tfstate.id
  container_access_type = "private"

  lifecycle {
    prevent_destroy = true
  }
}
```

---

# 14. Pourquoi le nom du Storage Account est calculé ?

Un nom de Storage Account Azure doit être globalement unique.

Nous utilisons :

```hcl
md5(data.azurerm_client_config.current.subscription_id)
```

afin de produire un suffixe déterministe dépendant de notre subscription.

Puis :

```hcl
substr(...)
```

limite le nom à la longueur autorisée.

Le résultat ressemble à :

```text
stmlatf3f48d7a58a12c4d1
```

Le nom reste :

- déterministe ;
- composé de caractères compatibles ;
- fortement susceptible d'être unique ;
- reproductible pour la même subscription.

---

# 15. Pourquoi ne pas utiliser un nom fixe ?

Un nom comme :

```text
stmlatfstate001
```

peut déjà être utilisé par quelqu'un d'autre dans Azure.

Le Storage Account possède une portée globale pour son nom.

Notre mécanisme évite de demander à l'utilisateur d'inventer manuellement un suffixe.

---

# 16. Authentification Entra ID

Nous avons :

```hcl
shared_access_key_enabled = false
```

Nous ne voulons donc pas construire notre backend autour de :

```text
Storage Account Key
```

ou :

```text
SAS permanent
```

Nous utilisons :

```text
Microsoft Entra ID
       │
       ▼
Azure RBAC
       │
       ▼
Storage Blob Data Contributor
```

---

# 17. Pourquoi `Storage Blob Data Contributor` ?

Le backend doit pouvoir travailler sur le blob contenant le state.

Il doit notamment pouvoir :

- lire ;
- créer ;
- modifier ;
- verrouiller.

Le rôle de données adapté est :

```text
Storage Blob Data Contributor
```

Dans une architecture de production, on cherchera à limiter son scope au container nécessaire.

---

# 18. Pourquoi le Lab utilise-t-il le Resource Group comme scope ?

Le Lab utilise :

```hcl
scope = azurerm_resource_group.tfstate.id
```

pour une raison de bootstrap.

Le provider doit pouvoir configurer les propriétés Blob du Storage Account alors que le Storage Account est encore en cours de création.

Notre identité joue ici deux rôles :

```text
Bootstrap administrator
+
Terraform user
```

Dans une architecture d'entreprise, ces responsabilités peuvent être séparées.

---

# 19. Modèle de production plus strict

Nous pourrions avoir :

```text
Bootstrap Identity
       │
       └── permissions sur backend infrastructure

GitHub Actions Identity
       │
       └── Storage Blob Data Contributor
           uniquement sur container tfstate
```

C'est cette deuxième identité que nous construirons lorsque nous introduirons GitHub Actions et OpenID Connect (OIDC).

---

# 20. Pourquoi désactiver Shared Key ?

Avec :

```hcl
shared_access_key_enabled = false
```

nous voulons éviter qu'un utilisateur disposant simplement de la clé du Storage Account contourne notre modèle RBAC.

Nous privilégions :

```text
Microsoft Entra ID
+
RBAC
```

Ce choix améliore notamment :

- la traçabilité ;
- le moindre privilège ;
- la suppression des credentials statiques ;
- l'intégration future avec OIDC.

---

# 21. Pourquoi `default_to_oauth_authentication` ?

Nous configurons :

```hcl
default_to_oauth_authentication = true
```

afin de privilégier l'autorisation Microsoft Entra ID dans les expériences Azure compatibles.

Cela ne remplace pas les Role Assignments.

Il faut toujours disposer des permissions nécessaires.

---

# 22. Pourquoi le réseau public reste activé dans ce Lab ?

Nous avons :

```hcl
public_network_access = "Enabled"
```

Notre poste local doit pouvoir contacter le Blob endpoint.

Cela ne signifie pas que le container devient public.

Nous avons :

```hcl
container_access_type = "private"
```

et une authentification Entra ID.

La différence est importante :

```text
Public network reachable
```

ne signifie pas :

```text
Anonymous public access
```

---

# 23. Ce qui changera en production

Une cible plus stricte serait :

```text
Terraform Runner
      │
      ▼
Private Network
      │
      ▼
Private Endpoint
      │
      ▼
Storage Account Backend
```

avec :

- firewall Storage ;
- Private Endpoint ;
- Private DNS ;
- runners capables d'atteindre le réseau privé.

Nous n'introduisons pas encore cette complexité afin de rester concentrés sur le backend Terraform.

---

# 24. Versioning du state

Nous activons :

```hcl
versioning_enabled = true
```

Chaque nouvelle version du blob peut ainsi conserver un historique Azure Storage.

Cela ajoute une couche utile de récupération.

---

# 25. Soft delete

Nous ajoutons également :

```hcl
delete_retention_policy {
  days = 14
}
```

et :

```hcl
container_delete_retention_policy {
  days = 14
}
```

Nous améliorons ainsi la récupération en cas de suppression accidentelle.

---

# 26. Le versioning ne remplace pas une stratégie de sauvegarde

Disposer de versions du blob est utile.

Mais une vraie politique de résilience doit également analyser :

- droits de suppression ;
- rétention ;
- réplication ;
- erreurs humaines ;
- compromission d'identité ;
- disaster recovery ;
- tests de restauration.

Le backend mérite le même niveau de sérieux qu'une autre ressource critique.

---

# 27. Pourquoi `prevent_destroy` ?

Nous protégeons :

```text
Storage Account
Container
```

avec :

```hcl
lifecycle {
  prevent_destroy = true
}
```

Une commande :

```powershell
terraform destroy
```

sur le bootstrap ne doit pas pouvoir supprimer silencieusement notre backend critique.

Terraform bloquera la destruction tant que cette protection est présente.

---

# 28. `prevent_destroy` n'est pas une sauvegarde

`prevent_destroy` protège contre certaines destructions Terraform.

Il ne protège pas nécessairement contre :

- une suppression Azure Portal ;
- un autre code Terraform ;
- une identité avec des droits élevés ;
- une suppression du Resource Group hors Terraform ;
- une compromission.

La protection doit être multicouche.

---

# 29. `bootstrap/outputs.tf`

Créer :

```text
bootstrap/outputs.tf
```

avec :

```hcl
output "resource_group_name" {
  description = "Resource Group hébergeant le backend Terraform."
  value       = azurerm_resource_group.tfstate.name
}

output "storage_account_name" {
  description = "Storage Account utilisé par le backend Terraform."
  value       = azurerm_storage_account.tfstate.name
}

output "container_name" {
  description = "Container utilisé pour les Terraform States."
  value       = azurerm_storage_container.tfstate.name
}

output "tenant_id" {
  description = "Microsoft Entra tenant utilisé."
  value       = data.azurerm_client_config.current.tenant_id
}

output "subscription_id" {
  description = "Subscription Azure hébergeant le backend."
  value       = data.azurerm_client_config.current.subscription_id
}
```

---

# 30. Initialiser le bootstrap

Exécuter :

```powershell
Set-Location "C:\Labs\MonLabAzure\terraform-azure-platform\lab-06\bootstrap"

terraform fmt -recursive
terraform init
terraform validate
```

Puis :

```powershell
terraform plan -out="bootstrap.tfplan"
```

Lire :

```powershell
terraform show ".\bootstrap.tfplan"
```

---

# 31. Appliquer le bootstrap

Après vérification :

```powershell
terraform apply ".\bootstrap.tfplan"
```

Nous devons obtenir :

- Resource Group ;
- Role Assignment ;
- Storage Account ;
- Blob Container.

Azure RBAC peut nécessiter un court délai de propagation. Si une erreur `403` apparaît immédiatement après la création du rôle, vérifier le Role Assignment avant de conclure à une erreur Terraform.

---

# 32. Vérifier les outputs

```powershell
terraform output
```

Récupérer le Storage Account :

```powershell
$backendStorageAccount = terraform output -raw storage_account_name
```

Puis :

```powershell
$backendStorageAccount
```

---

# 33. Vérifier Azure

```powershell
$backendResourceGroup = terraform output -raw resource_group_name
```

Puis :

```powershell
az storage account show `
  --resource-group $backendResourceGroup `
  --name $backendStorageAccount `
  --output table
```

---

# 34. Vérifier le container

```powershell
az storage container show `
  --account-name $backendStorageAccount `
  --name "tfstate" `
  --auth-mode login `
  --output table
```

Nous utilisons :

```text
--auth-mode login
```

afin d'utiliser l'identité Microsoft Entra ID connectée.

Nous ne récupérons aucune Storage Account Key.

---

# 35. Limite actuelle : le bootstrap possède encore un state local

Nous avons désormais :

```text
bootstrap/
│
└── terraform.tfstate
```

Ce state gère uniquement l'infrastructure du backend.

Pour ce Lab, nous le conservons localement.

Ce n'est pas notre cible d'entreprise définitive.

---

# 36. Pourquoi ne pas stocker immédiatement le bootstrap dans le même backend ?

Nous pourrions techniquement imaginer :

```text
Storage Account
     │
     └── contient le state Terraform
           qui gère ce même Storage Account
```

Cette architecture crée une dépendance opérationnelle délicate.

Pour une vraie plateforme, l'infrastructure du backend possède généralement un cycle de vie distinct.

Elle peut être gérée :

- par une plateforme centrale ;
- par un bootstrap dédié ;
- par une Landing Zone ;
- depuis un autre backend ;
- par un processus d'initialisation contrôlé.

---

# 37. Préparer le workload

Nous allons maintenant créer une infrastructure Terraform utilisant d'abord un state local.

Puis nous migrerons ce state.

Se placer dans :

```powershell
Set-Location "C:\Labs\MonLabAzure\terraform-azure-platform\lab-06\workload"
```

---

# 38. `workload/versions.tf`

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

Pour le moment, ne pas encore ajouter le backend.

Nous voulons d'abord obtenir un state local à migrer.

---

# 39. `workload/providers.tf`

```hcl
provider "azurerm" {
  features {}
}
```

---

# 40. `workload/variables.tf`

```hcl
variable "project_name" {
  description = "Nom court du projet."
  type        = string
  default     = "mla"
}

variable "environment" {
  description = "Environnement cible."
  type        = string

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
}

variable "resource_instance" {
  description = "Numéro d'instance."
  type        = number
  default     = 1
}

variable "owner" {
  description = "Équipe responsable."
  type        = string
  default     = "cloud-platform"
}

variable "network" {
  description = "Configuration réseau."

  type = object({
    vnet_address_spaces = list(string)
    app_subnet_prefixes = list(string)
    pe_subnet_prefixes  = list(string)
  })
}

variable "additional_tags" {
  description = "Tags supplémentaires."
  type        = map(string)
  default     = {}
}
```

---

# 41. `workload/locals.tf`

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
    lab         = "lab-06"
  }

  common_tags = merge(
    var.additional_tags,
    local.mandatory_tags
  )
}
```

---

# 42. `workload/main.tf`

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

# 43. `workload/outputs.tf`

```hcl
output "resource_group_name" {
  description = "Nom du Resource Group."
  value       = azurerm_resource_group.main.name
}

output "virtual_network_name" {
  description = "Nom du Virtual Network."
  value       = azurerm_virtual_network.main.name
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

# 44. `workload/terraform.tfvars.example`

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

# 45. Créer d'abord le state local

Exécuter :

```powershell
terraform fmt -recursive
terraform init
terraform validate
```

Puis :

```powershell
terraform plan -out="local-state.tfplan"
terraform apply ".\local-state.tfplan"
```

Vérifier :

```powershell
terraform plan
```

Résultat attendu :

```text
No changes.
```

---

# 46. Vérifier le state local

```powershell
Get-ChildItem ".\terraform.tfstate"
```

Puis :

```powershell
terraform state list
```

Nous devons retrouver les six ressources.

À cet instant :

```text
workload/
│
└── terraform.tfstate
```

est encore le backend actif.

---

# 47. Ajouter le backend Azure

Créer maintenant :

```text
workload/backend.tf
```

avec :

```hcl
terraform {
  backend "azurerm" {}
}
```

Il s'agit d'une **partial backend configuration**.

Nous déclarons le type de backend :

```text
azurerm
```

mais pas encore ses paramètres.

---

# 48. Pourquoi ne pas utiliser des variables ?

Nous pourrions être tentés d'écrire :

```hcl
backend "azurerm" {
  storage_account_name = var.backend_storage_account_name
}
```

Ce modèle n'est pas valide.

La configuration du backend est nécessaire très tôt pendant :

```text
terraform init
```

avant l'évaluation normale des variables, resources et data sources.

Un backend ne peut donc pas utiliser :

```text
var.*
local.*
data.*
resource.*
```

comme une configuration Terraform classique.

---

# 49. Solution : backend partiel

Nous utilisons :

```hcl
backend "azurerm" {}
```

et fournissons le reste pendant :

```powershell
terraform init
```

C'est particulièrement adapté aux environnements et aux pipelines.

---

# 50. Préparer l'authentification du backend

Dans PowerShell :

```powershell
$env:ARM_USE_AZUREAD = "true"
$env:ARM_USE_CLI     = "true"

$env:ARM_TENANT_ID = az account show --query tenantId -o tsv
```

Le backend utilisera la session Azure CLI déjà authentifiée.

---

# 51. Générer automatiquement le fichier backend

Depuis :

```text
workload/
```

récupérer le nom du Storage Account :

```powershell
$backendStorageAccount = terraform `
  -chdir="..\bootstrap" `
  output -raw storage_account_name
```

Puis créer :

```powershell
@"
storage_account_name = "$backendStorageAccount"
container_name       = "tfstate"
key                  = "terraform-azure-platform/dev/platform.tfstate"

use_azuread_auth = true
use_cli          = true
"@ | Set-Content `
  -Encoding utf8 `
  ".\backend.azurerm.tfbackend"
```

---

# 52. Contenu obtenu

Le fichier ressemble à :

```hcl
storage_account_name = "stmlatfxxxxxxxxxxxxxxxx"
container_name       = "tfstate"
key                  = "terraform-azure-platform/dev/platform.tfstate"

use_azuread_auth = true
use_cli          = true
```

Aucun mot de passe.

Aucune Storage Account Key.

Aucun Client Secret.

---

# 53. Ne pas mettre des credentials dans le fichier backend

À éviter :

```hcl
client_secret = "..."
```

ou :

```hcl
access_key = "..."
```

Terraform peut conserver les paramètres backend dans :

```text
.terraform/
```

et dans certaines informations associées aux plans.

Les credentials doivent être fournis par des mécanismes adaptés, notamment des variables d'environnement ou une identité fédérée.

---

# 54. Ajouter les fichiers backend locaux au `.gitignore`

À la racine du repository, ajouter si ce n'est pas déjà fait :

```gitignore
# Terraform backend local configuration
*.tfbackend
```

Notre fichier :

```text
backend.azurerm.tfbackend
```

est spécifique à l'environnement.

Il ne sera pas commité dans ce Lab.

---

# 55. La `key`

Nous avons :

```text
terraform-azure-platform/dev/platform.tfstate
```

La `key` représente le nom du blob utilisé pour stocker le state.

Les `/` sont utilisés pour rendre l'organisation lisible.

Conceptuellement :

```text
tfstate
└── terraform-azure-platform
     └── dev
          └── platform.tfstate
```

Il s'agit toujours d'un blob dans le container.

---

# 56. Pourquoi ne pas utiliser simplement `terraform.tfstate` ?

Avec plusieurs plateformes et environnements :

```text
terraform.tfstate
```

devient ambigu.

Nous préférons une convention :

```text
<plateforme>/<environnement>/<stack>.tfstate
```

Par exemple :

```text
terraform-azure-platform/dev/platform.tfstate
terraform-azure-platform/qual/platform.tfstate
terraform-azure-platform/prod/platform.tfstate
```

---

# 57. Les environnements doivent avoir des states séparés

Nous éviterons :

```text
DEV
QUALIF
PROD
     │
     ▼
même state
```

Nous chercherons plutôt :

```text
DEV
└── dev/platform.tfstate

QUALIF
└── qual/platform.tfstate

PROD
└── prod/platform.tfstate
```

La stratégie complète sera étudiée dans le Lab consacré aux environnements.

---

# 58. Migrer le state

Nous avons maintenant :

```text
State local
      │
      ▼
terraform init -migrate-state
      │
      ▼
Azure Storage
```

Exécuter :

```powershell
terraform init `
  -migrate-state `
  -backend-config="backend.azurerm.tfbackend"
```

Terraform détecte le changement de backend.

Il propose de migrer le state existant.

Lire attentivement le message avant de confirmer.

---

# 59. Ce que fait `-migrate-state`

Terraform prend le state actuellement associé au backend local et tente de le copier vers le nouveau backend.

Nous voulons obtenir :

```text
AVANT

workload/
└── terraform.tfstate


APRÈS

Azure Storage
└── tfstate
     └── terraform-azure-platform/dev/platform.tfstate
```

Les Resource IDs et les bindings ne changent pas.

Seul l'emplacement du state change.

---

# 60. `-migrate-state` ou `-reconfigure` ?

Il faut distinguer les deux.

```text
-migrate-state
```

signifie :

> changer de backend et migrer le state existant.

```text
-reconfigure
```

signifie essentiellement :

> oublier la configuration backend mémorisée localement et initialiser à nouveau avec cette configuration.

`-reconfigure` n'est pas un remplacement automatique de la migration du state.

---

# 61. Vérifier après migration

Exécuter :

```powershell
terraform state list
```

Nous devons toujours retrouver les six ressources.

Puis :

```powershell
terraform plan
```

Résultat attendu :

```text
No changes.
Your infrastructure matches the configuration.
```

C'est le contrôle essentiel.

---

# 62. Vérifier le blob avec Azure CLI

Récupérer :

```powershell
$backendStorageAccount = terraform `
  -chdir="..\bootstrap" `
  output -raw storage_account_name
```

Puis :

```powershell
az storage blob list `
  --account-name $backendStorageAccount `
  --container-name "tfstate" `
  --auth-mode login `
  --output table
```

Nous devons retrouver le blob :

```text
terraform-azure-platform/dev/platform.tfstate
```

---

# 63. Inspecter le blob

```powershell
az storage blob show `
  --account-name $backendStorageAccount `
  --container-name "tfstate" `
  --name "terraform-azure-platform/dev/platform.tfstate" `
  --auth-mode login `
  --output json
```

Nous pouvons observer les métadonnées du blob sans télécharger le state.

---

# 64. `terraform state pull` fonctionne toujours

Même avec un backend distant :

```powershell
terraform state pull
```

fonctionne toujours.

Terraform contacte le backend distant.

Le fonctionnement utilisateur reste donc similaire :

```text
terraform state list
terraform state show
terraform state pull
terraform console
```

La différence est l'emplacement du state.

---

# 65. Le state n'est plus normalement persisté sur le disque local

Avec un backend distant, Terraform travaille avec le state fourni par le backend.

Le state actif n'est plus le fichier :

```text
workload/terraform.tfstate
```

du modèle local.

Terraform peut cependant écrire exceptionnellement un state local si une erreur irrécupérable empêche l'enregistrement du nouveau state dans le backend.

Ce mécanisme vise à éviter une perte de données.

---

# 66. Supprimer les anciennes copies locales après validation

Une fois que nous avons vérifié :

```text
Remote blob présent
+
terraform state list correct
+
terraform plan = No changes
```

les anciennes copies locales du state ne doivent pas rester indéfiniment sur le poste.

Avant suppression, vérifier précisément leur contenu et leur rôle.

Ne jamais supprimer un state local avant d'avoir validé que la migration distante est correcte.

---

# 67. Architecture obtenue

Nous avons maintenant :

```text
                  ┌──────── Engineer
                  │
                  ├──────── Engineer
                  │
Terraform Backend ◄──────── Pipeline futur
        │
        ▼
Microsoft Entra ID
        │
        ▼
Azure RBAC
        │
        ▼
Storage Account
        │
        ▼
Container tfstate
        │
        ▼
platform.tfstate
```

---

# 68. Qu'est-ce que le state locking ?

Imaginez deux ingénieurs.

```text
Ingénieur A
terraform apply

Ingénieur B
terraform apply
```

Tous deux utilisent le même state.

Sans coordination :

```text
State version N

A lit N
B lit N

A modifie Azure
B modifie Azure

A écrit N+1
B écrit N+1
```

Nous pouvons créer une corruption logique ou perdre des informations.

---

# 69. Le locking évite plusieurs writers simultanés

Avec un backend supportant le locking :

```text
Ingénieur A
      │
      ▼
acquiert LOCK
      │
      ▼
modifie state
      │
      ▼
libère LOCK


Ingénieur B
      │
      ▼
doit attendre
```

Le backend Azure Storage fournit ce mécanisme pour le blob de state.

---

# 70. Le locking est automatique

Nous n'avons pas besoin de créer :

```text
terraform.lock
```

ou une table SQL de verrouillage.

Lorsqu'une opération Terraform nécessitant le verrouillage du state est exécutée, Terraform demande automatiquement le lock au backend.

---

# 71. Ne pas confondre avec `.terraform.lock.hcl`

Deux concepts totalement différents existent.

## Dependency lock file

```text
.terraform.lock.hcl
```

sert notamment à verrouiller les versions des providers.

## State locking

```text
Remote State Lock
```

sert à empêcher des opérations Terraform concurrentes sur le même state.

La ressemblance des noms provoque souvent de la confusion.

---

# 72. Premier test de locking

Nous allons ouvrir deux terminaux PowerShell dans :

```text
lab-06/workload
```

Dans les deux terminaux, définir :

```powershell
$env:ARM_SUBSCRIPTION_ID = az account show --query id -o tsv
$env:ARM_TENANT_ID       = az account show --query tenantId -o tsv
$env:ARM_USE_AZUREAD     = "true"
$env:ARM_USE_CLI         = "true"
```

---

# 73. Créer une modification

Dans :

```text
terraform.tfvars
```

ajouter :

```hcl
additional_tags = {
  workload    = "terraform-learning"
  criticality = "low"
  lock_test   = "terminal-1"
}
```

---

# 74. Terminal 1

Exécuter :

```powershell
terraform apply
```

Terraform calcule son plan puis demande confirmation.

À ce moment, ne pas répondre immédiatement.

Laisser l'opération ouverte.

---

# 75. Terminal 2

Dans le second terminal :

```powershell
terraform plan `
  -lock-timeout=10s
```

Terraform essaie d'obtenir le lock.

Mais le premier processus le possède déjà.

Le second processus doit donc attendre puis échouer si le lock n'est pas libéré avant la fin du timeout.

---

# 76. Informations de lock

Le message Terraform peut contenir des informations permettant d'identifier le verrou :

```text
Lock Info

ID
Path
Operation
Who
Version
Created
```

Ces informations sont utiles pour le diagnostic.

---

# 77. Libérer normalement le lock

Revenir au Terminal 1.

Nous pouvons :

```text
yes
```

pour appliquer,

ou annuler proprement l'opération.

Terraform doit ensuite libérer le lock.

---

# 78. Retester

Dans le Terminal 2 :

```powershell
terraform plan `
  -lock-timeout=10s
```

Cette fois, Terraform doit réussir à acquérir le lock.

---

# 79. `-lock-timeout`

Par défaut, une opération peut échouer rapidement si le lock est déjà détenu.

Nous pouvons demander :

```powershell
terraform plan `
  -lock-timeout=30s
```

Cela signifie :

> essaie d'acquérir le lock pendant au maximum 30 secondes avant d'abandonner.

Cette option est utile dans les pipelines ou environnements où deux opérations peuvent se suivre de près.

---

# 80. Ne pas utiliser `-lock=false` comme solution

Terraform permet :

```powershell
terraform apply -lock=false
```

Mais cette option désactive précisément la protection que nous venons de mettre en place.

À éviter en environnement partagé.

Ne pas transformer :

```text
Error acquiring the state lock
```

en :

```text
je vais désactiver le lock
```

La bonne question est :

> Qui possède le lock et pourquoi ?

---

# 81. Procédure de diagnostic d'un lock

Lorsque Terraform indique qu'un state est verrouillé :

```text
1. Identifier le state concerné

2. Lire les informations du lock

3. Identifier l'utilisateur ou le pipeline

4. Vérifier si l'opération est toujours active

5. Ne pas lancer un deuxième apply concurrent

6. Attendre ou arrêter proprement l'opération active

7. N'utiliser force-unlock que si le lock est réellement orphelin
```

---

# 82. `terraform force-unlock`

Terraform fournit :

```powershell
terraform force-unlock <LOCK_ID>
```

Cette commande doit être utilisée avec beaucoup de prudence.

Elle ne doit pas être utilisée simplement parce qu'un autre ingénieur travaille trop longtemps.

---

# 83. Quand utiliser `force-unlock` ?

Exemple :

```text
Pipeline
   │
   ▼
acquiert le lock
   │
   ▼
processus crash
   │
   X
lock non libéré
```

Après vérification qu'aucune opération Terraform n'utilise réellement ce state, un `force-unlock` peut devenir nécessaire.

---

# 84. Quand ne pas utiliser `force-unlock` ?

Si :

```text
Ingénieur A
terraform apply
```

est toujours actif, ne pas lancer :

```powershell
terraform force-unlock
```

Sinon nous pouvons obtenir :

```text
Writer A
+
Writer B
+
même state
```

C'est exactement le scénario que le locking cherche à empêcher.

---

# 85. Le lock appartient au state, pas au repository

Deux branches Git différentes :

```text
feature/network
feature/security
```

peuvent cibler :

```text
la même backend key
```

Elles travaillent alors sur le même state.

Le repository et le state sont deux concepts différents.

---

# 86. Concurrency GitHub Actions

Lorsque nous arriverons à GitHub Actions, nous ajouterons également une protection au niveau pipeline :

```text
GitHub Actions concurrency
          +
Terraform state locking
```

Ces protections sont complémentaires.

---

# 87. State locking ne remplace pas les Pull Requests

Le lock empêche principalement les writers concurrents.

Il ne vérifie pas que :

- le changement est souhaité ;
- le plan est correct ;
- PROD a été approuvée ;
- la sécurité est correcte.

Nous aurons donc :

```text
Pull Request
     │
     ▼
Review
     │
     ▼
Plan
     │
     ▼
Approval
     │
     ▼
Pipeline concurrency
     │
     ▼
Terraform state lock
     │
     ▼
Apply
```

---

# 88. Chiffrement du state

Azure Storage chiffre les données stockées au repos.

Notre blob Terraform bénéficie donc du chiffrement Azure Storage.

Mais cela ne change pas un principe important :

> toute identité capable de lire le blob peut potentiellement lire les données présentes dans le Terraform State.

Le principal contrôle reste donc :

```text
Identity
+
RBAC
+
Network
+
Audit
```

---

# 89. State et moindre privilège

Le runner Terraform n'a pas besoin d'être :

```text
Owner du Storage Account
```

simplement pour utiliser son backend.

Pour l'accès direct au Blob State, nous chercherons en production à utiliser le rôle :

```text
Storage Blob Data Contributor
```

avec le scope le plus réduit adapté, idéalement le container concerné.

---

# 90. Ne pas utiliser Contributor comme raccourci

Le rôle Azure :

```text
Contributor
```

sur le Storage Account concerne principalement le management plane.

Il ne remplace pas nécessairement les permissions de données nécessaires pour lire et modifier les blobs.

Il faut distinguer :

```text
Management plane
```

de :

```text
Data plane
```

C'est une distinction Azure importante.

---

# 91. Management plane

Exemple :

```text
Créer Storage Account
Modifier configuration Storage
Configurer réseau
```

Ces opérations utilisent le plan de gestion Azure Resource Manager.

---

# 92. Data plane

Exemple :

```text
Lire le blob terraform.tfstate
Écrire le blob
Modifier le blob
```

Ces opérations utilisent le data plane Azure Storage.

Notre backend travaille directement avec les données Blob.

---

# 93. Tester un accès non autorisé

Un utilisateur possédant seulement :

```text
Reader
```

sur le Storage Account ne doit pas être considéré automatiquement comme capable de lire le contenu du state via Microsoft Entra ID.

Il faut un rôle de données approprié.

C'est précisément pour cette raison que nous utilisons :

```text
Storage Blob Data Contributor
```

---

# 94. Vérifier les versions du state

Azure Storage versioning étant activé, les nouvelles écritures du blob peuvent générer des versions.

Après plusieurs opérations Terraform, nous pouvons vérifier :

```powershell
az storage blob list `
  --account-name $backendStorageAccount `
  --container-name "tfstate" `
  --include v `
  --auth-mode login `
  --output table
```

Le versioning fournit une capacité supplémentaire de récupération.

---

# 95. Le backend doit être traité comme une infrastructure critique

Éviter une architecture dans laquelle :

```text
Backend Terraform
```

est considéré comme un simple Storage Account technique sans gouvernance.

Il contient potentiellement la connaissance permettant à Terraform de gérer :

```text
réseaux
identités
bases de données
applications
secrets
permissions
infrastructure critique
```

Sa compromission peut avoir des conséquences importantes.

---

# 96. Production : protections supplémentaires

Dans une plateforme réelle, nous étudierions notamment :

```text
Dedicated Resource Group

Private Endpoint

Private DNS

Public Network Access disabled

Azure RBAC

Dedicated identities

Blob versioning

Soft delete

Diagnostic logs

Resource Locks

Azure Policy

Backup / recovery procedure

Separate DEV / QUALIF / PROD states
```

Nous n'avons volontairement pas tout ajouté dans un seul Lab.

---

# 97. Resource Lock Azure

Une protection supplémentaire peut être ajoutée au niveau Azure :

```text
CanNotDelete
```

sur le Storage Account ou le Resource Group backend.

Cela protège également contre certaines suppressions en dehors de Terraform.

Il faut cependant intégrer correctement ce verrou dans le modèle d'exploitation afin de ne pas bloquer les opérations légitimes.

---

# 98. LRS ou ZRS ?

Notre Lab utilise :

```hcl
account_replication_type = "LRS"
```

C'est suffisant pour l'apprentissage.

Pour un backend d'entreprise, nous devons évaluer :

```text
LRS
ZRS
GRS
GZRS
```

selon :

- criticité ;
- région ;
- disponibilité ;
- stratégie de disaster recovery ;
- coût.

Le choix ne doit pas être automatique.

---

# 99. Backend et coûts

Un Terraform State représente généralement peu de volume.

Le coût brut du stockage est donc souvent faible.

La décision d'architecture doit être guidée principalement par :

- sécurité ;
- disponibilité ;
- récupération ;
- gouvernance ;

plutôt que par quelques centimes de stockage.

---

# 100. Vérifier l'état final

Depuis :

```text
workload/
```

exécuter :

```powershell
terraform plan
```

Résultat attendu :

```text
No changes.
```

Puis :

```powershell
terraform state list
```

Nous devons retrouver les six ressources.

Puis :

```powershell
az storage blob list `
  --account-name $backendStorageAccount `
  --container-name "tfstate" `
  --auth-mode login `
  --output table
```

Nous devons retrouver :

```text
terraform-azure-platform/dev/platform.tfstate
```

---

# 101. Le code reste séparé du state

Notre repository contient :

```text
Git
│
├── versions.tf
├── providers.tf
├── variables.tf
├── locals.tf
├── main.tf
├── outputs.tf
└── backend.tf
```

Azure contient :

```text
Remote Backend
│
└── platform.tfstate
```

C'est une séparation fondamentale.

---

# 102. Ce qui doit être versionné

Nous versionnons :

```text
*.tf

terraform.tfvars.example

.terraform.lock.hcl
```

Nous ne versionnons pas :

```text
terraform.tfstate
terraform.tfstate.*
.terraform/
*.tfplan
terraform.tfvars
*.tfbackend
```

dans la stratégie utilisée par cette série.

---

# 103. Versionner LAB-06

Depuis la racine :

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

Vérifier qu'aucun state local ou fichier backend local n'est présent.

Puis :

```powershell
git commit -m "feat(terraform): add lab 06 azure remote state"
```

---

# 104. Nettoyage du workload

Le backend doit être conservé pour les prochains Labs.

Nous pouvons en revanche supprimer les ressources du workload :

```powershell
Set-Location "C:\Labs\MonLabAzure\terraform-azure-platform\lab-06\workload"
```

Créer :

```powershell
terraform plan `
  -destroy `
  -out="lab-06-destroy.tfplan"
```

Lire :

```powershell
terraform show ".\lab-06-destroy.tfplan"
```

Puis :

```powershell
terraform apply ".\lab-06-destroy.tfplan"
```

---

# 105. Que devient le remote state après destroy ?

Après :

```text
Resources: 0
```

le blob du backend peut toujours exister.

Le state représente alors simplement une configuration ne contenant plus les ressources précédemment détruites.

Nous ne supprimons pas automatiquement le backend.

---

# 106. Ne pas détruire le bootstrap

Ne pas lancer simplement :

```powershell
terraform destroy
```

dans :

```text
bootstrap/
```

Notre Storage Account contient désormais le backend utilisé par Terraform.

De plus :

```hcl
prevent_destroy = true
```

bloque volontairement sa destruction via cette configuration.

---

# 107. Procédure de suppression d'un backend

La suppression d'un backend doit être une opération planifiée.

Il faut d'abord vérifier :

```text
Quels states sont présents ?

Quelles plateformes utilisent le backend ?

Existe-t-il un backup ?

Les states ont-ils été migrés ?

Les versions sont-elles récupérables ?

Les pipelines utilisent-ils encore ce backend ?
```

Supprimer le backend n'est pas une simple opération de nettoyage de Lab.

---

# 108. Ce qu'il faut retenir

Nous sommes passés de :

```text
PC
│
└── terraform.tfstate
```

à :

```text
Terraform
    │
    ▼
Microsoft Entra ID
    │
    ▼
Azure RBAC
    │
    ▼
Azure Storage
    │
    ▼
Remote Terraform State
```

---

## Le backend

Détermine principalement où Terraform stocke le state et comment il y accède.

---

## Le provider

Permet à Terraform de gérer les ressources Azure.

Backend et provider sont deux responsabilités différentes.

---

## Le bootstrap

Le backend doit exister avant de pouvoir être utilisé.

Nous devons donc traiter sa création séparément.

---

## Remote State

Permet à plusieurs ingénieurs ou pipelines de travailler avec la même représentation Terraform.

---

## Locking

Empêche plusieurs opérations concurrentes incompatibles d'écrire simultanément dans le même state.

---

## Microsoft Entra ID

Nous permet d'éviter une dépendance aux Storage Account Keys pour accéder au backend.

---

## RBAC

Permet de donner uniquement les permissions nécessaires aux identités Terraform.

---

## Versioning

Ajoute une capacité de récupération en conservant plusieurs versions du blob.

---

## Soft delete

Ajoute une protection supplémentaire contre certaines suppressions accidentelles.

---

# 109. Ce qu'un ingénieur Terraform doit savoir expliquer

> Pourquoi utiliser un remote backend ?

Une réponse solide serait :

> Un remote backend permet de centraliser le Terraform State afin que les ingénieurs et les pipelines travaillent sur le même ensemble de bindings. Un backend adapté apporte également un mécanisme de locking qui protège contre certaines écritures concurrentes. Sur Azure, j'utilise typiquement Azure Blob Storage avec Microsoft Entra ID et RBAC plutôt qu'une Storage Account Key permanente.

---

# 110. Où stockez-vous le state ?

Réponse possible :

> Je stocke le state dans un backend Azure Storage dédié, séparé de l'infrastructure applicative. Les states sont organisés par plateforme et environnement, par exemple `terraform-azure-platform/prod/platform.tfstate`. L'accès au data plane est contrôlé avec Microsoft Entra ID et Storage Blob Data Contributor.

---

# 111. Comment empêchez-vous deux applies simultanés ?

Réponse :

> J'utilise d'abord le locking fourni par le backend Terraform. Avec Azure Storage, le blob de state est automatiquement verrouillé pour les opérations qui doivent écrire le state. Dans une pipeline, j'ajoute également une gestion de concurrence au niveau CI/CD afin d'éviter que plusieurs déploiements du même environnement soient lancés simultanément.

---

# 112. Utilisez-vous `-lock=false` ?

Réponse :

> Pas dans un workflow normal. Cette option désactive une protection essentielle contre les opérations concurrentes. Si un lock pose problème, je commence par identifier l'opération qui le détient. `force-unlock` ne doit être utilisé que lorsque le lock est réellement orphelin.

---

# 113. Backend et provider, quelle différence ?

Réponse :

> Le provider AzureRM permet de gérer les ressources Azure alors que le backend azurerm détermine où Terraform stocke et verrouille son state. Les deux utilisent Azure mais interviennent à des moments et pour des responsabilités différentes.

---

# 114. Pourquoi ne pas mettre le backend dans une variable ?

Réponse :

> Le backend est initialisé très tôt pendant `terraform init`, avant l'évaluation normale de la configuration. Il ne peut donc pas utiliser les input variables, locals ou attributs de ressources classiques. J'utilise une partial backend configuration et je fournis les paramètres pendant `terraform init`.

---

# 115. Validation du LAB-06

Le Lab est terminé lorsque les points suivants sont compris et testés :

```text
[ ] Comprendre backend vs provider

[ ] Comprendre le problème du bootstrap

[ ] Créer le Resource Group backend

[ ] Créer le Storage Account backend

[ ] Créer le container tfstate

[ ] Activer HTTPS uniquement

[ ] Utiliser TLS 1.2

[ ] Interdire le public blob access

[ ] Désactiver Shared Key

[ ] Utiliser Microsoft Entra ID

[ ] Utiliser Storage Blob Data Contributor

[ ] Activer le blob versioning

[ ] Activer le soft delete

[ ] Ajouter prevent_destroy

[ ] Comprendre la limite du public network access du Lab

[ ] Créer l'infrastructure workload avec state local

[ ] Vérifier terraform.tfstate local

[ ] Ajouter backend "azurerm"

[ ] Comprendre une partial backend configuration

[ ] Comprendre pourquoi var.* ne fonctionne pas dans backend

[ ] Générer backend.azurerm.tfbackend

[ ] Migrer avec terraform init -migrate-state

[ ] Vérifier le state distant

[ ] Vérifier le blob avec Azure CLI

[ ] Obtenir terraform plan = No changes

[ ] Comprendre la backend key

[ ] Comprendre la séparation future DEV / QUALIF / PROD

[ ] Comprendre le state locking

[ ] Provoquer une concurrence avec deux terminaux

[ ] Utiliser -lock-timeout

[ ] Comprendre pourquoi -lock=false est dangereux

[ ] Comprendre force-unlock

[ ] Comprendre le rôle du versioning

[ ] Comprendre le rôle du soft delete

[ ] Comprendre management plane vs data plane

[ ] Comprendre le principe du moindre privilège

[ ] Conserver le backend pour les prochains Labs
```

---

# 116. Progression

Nous avons maintenant parcouru :

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
     ▼
LAB-06
Remote State
     │
     ├── Azure Storage
     ├── Entra ID
     ├── RBAC
     ├── migration
     ├── locking
     ├── versioning
     └── recovery
```

Pour la première fois, notre architecture commence à correspondre à un véritable environnement Terraform collaboratif.

---

# 117. Suite

Notre code possède maintenant une autre limite.

Toutes les ressources sont toujours directement écrites dans le root module :

```text
root module
│
├── Resource Group
├── Virtual Network
├── Subnets
├── NSG
└── associations
```

À mesure que l'infrastructure grandit, ce modèle devient difficile à maintenir et à réutiliser.

Le prochain Lab introduira donc les modules Terraform :

```text
Root Module
│
├── module resource-group
│
├── module network
│
└── module security
```

Nous étudierons notamment :

- root module ;
- child module ;
- interface d'un module ;
- inputs ;
- outputs ;
- encapsulation ;
- composition ;
- responsabilités ;
- modules trop génériques ;
- modules trop spécifiques ;
- versionnement ;
- dépendances entre modules ;
- refactoring sans destruction de ressources.

Le prochain article sera :

**LAB-07 - Concevoir des modules Terraform professionnels**

---

## Références officielles

Les mécanismes étudiés dans ce Lab sont documentés principalement dans :

- HashiCorp Developer - Backend Configuration ;
- HashiCorp Developer - AzureRM Backend ;
- HashiCorp Developer - State Storage and Locking ;
- HashiCorp Developer - State Locking ;
- HashiCorp Developer - `terraform init` ;
- HashiCorp Developer - Backend Migration ;
- Microsoft Learn - Store Terraform State in Azure Storage ;
- Microsoft Learn - Azure RBAC for Blob Data ;
- Microsoft Learn - Azure Storage Security ;
- Terraform Registry - `azurerm_storage_account` ;
- Terraform Registry - `azurerm_storage_container` ;
- Terraform Registry - `azurerm_role_assignment`.

Le backend Terraform doit être considéré comme une infrastructure critique. Son réseau, ses identités, ses permissions et ses mécanismes de récupération doivent être conçus avant d'être utilisés pour des environnements de production.