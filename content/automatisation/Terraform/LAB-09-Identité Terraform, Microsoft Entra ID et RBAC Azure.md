---
title: "LAB-09 — Identité Terraform, Microsoft Entra ID et RBAC Azure"
date: 2026-10-08
draft: false
description: "Concevoir une identité Terraform dédiée avec Microsoft Entra ID, User Assigned Managed Identity et Azure RBAC en appliquant le principe du moindre privilège."
summary: "Séparer identité humaine et identité Terraform, créer une Managed Identity par environnement et limiter les permissions Azure avec RBAC."
slug: "terraform-azure-lab-09-identite-entra-id-rbac"
categories:
  - "Automatisation"
tags:
  - "Terraform"
  - "Azure"
  - "Microsoft Entra ID"
  - "RBAC"
  - "Managed Identity"
  - "Identity"
  - "Least Privilege"
  - "AzureRM"
series:
  - "Terraform Azure — De zéro à production"
weight: 10
ShowToc: true
TocOpen: true
---

# LAB-09 — Identité Terraform, Microsoft Entra ID et RBAC Azure

Jusqu'ici, nous avons principalement exécuté Terraform depuis notre poste de travail après :

```powershell id="24arqp"
az login
```

Terraform utilise alors l'identité Microsoft Entra ID de l'ingénieur connecté.

Cette approche est adaptée à l'apprentissage et au travail interactif.

Elle n'est pas notre cible pour une chaîne CI/CD.

Nous ne voulons pas qu'une pipeline de production dépende :

- du compte personnel d'un ingénieur ;
- de sa session Azure CLI ;
- de ses permissions ;
- de son départ de l'entreprise ;
- d'un mot de passe ou d'un Client Secret permanent.

Nous allons donc préparer une architecture différente :

```text id="ybmkzt"
Ingénieur
   │
   │ bootstrap uniquement
   ▼
Microsoft Entra ID
   │
   ▼
Création des identités Terraform
   │
   ├── Terraform DEV
   ├── Terraform QUALIF
   └── Terraform PROD
            │
            ▼
         Azure RBAC
            │
            ▼
   Scope propre à l'environnement
```

L'objectif est d'arriver progressivement à :

```text id="6bcb3k"
GitHub Actions
      │
      ▼
OIDC
      │
      ▼
Managed Identity
      │
      ▼
Azure RBAC
      │
      ▼
Terraform
```

Aucun secret permanent ne sera nécessaire.

La fédération GitHub sera mise en place dans un Lab ultérieur.

---

# 1. Objectifs du Lab

Nous allons comprendre :

- authentification vs autorisation ;
- identité humaine vs workload identity ;
- Microsoft Entra ID ;
- Service Principal ;
- Managed Identity ;
- System Assigned Managed Identity ;
- User Assigned Managed Identity ;
- Workload Identity Federation ;
- OpenID Connect (OIDC) ;
- Azure Role-Based Access Control (RBAC) ;
- principal ;
- role definition ;
- role assignment ;
- scope ;
- héritage RBAC ;
- Contributor ;
- Owner ;
- Role Based Access Control Administrator ;
- principe du moindre privilège ;
- Management Plane ;
- Data Plane ;
- séparation DEV / QUALIF / PROD ;
- bootstrap d'identité ;
- préparation de GitHub Actions.

---

# 2. Architecture cible

Nous allons créer :

```text id="t0haay"
Subscription Azure
│
├── rg-mla-identity-fce-001
│    │
│    ├── id-mla-tf-dev-fce-001
│    │
│    ├── id-mla-tf-qual-fce-001
│    │
│    └── id-mla-tf-prod-fce-001
│
├── rg-mla-dev-fce-001
│    ▲
│    │ Contributor
│    │
│    └── id-mla-tf-dev-fce-001
│
├── rg-mla-qual-fce-001
│    ▲
│    │ Contributor
│    │
│    └── id-mla-tf-qual-fce-001
│
└── rg-mla-prod-fce-001
     ▲
     │ Contributor
     │
     └── id-mla-tf-prod-fce-001
```

Chaque environnement possède sa propre identité Terraform.

---

# 3. Pourquoi trois identités ?

Nous pourrions créer :

```text id="pnfm5w"
id-mla-terraform
```

et lui donner accès à :

```text id="eeivqv"
DEV
QUALIF
PROD
```

Ce serait plus simple.

Mais une compromission de cette identité donnerait accès aux trois environnements.

Nous préférons :

```text id="o5zwqi"
Identity DEV
   │
   └── DEV uniquement

Identity QUALIF
   │
   └── QUALIF uniquement

Identity PROD
   │
   └── PROD uniquement
```

Nous limitons ainsi le blast radius.

---

# 4. Authentification et autorisation

Ces deux notions sont souvent mélangées.

## Authentification

Répond à :

> Qui es-tu ?

Exemple :

```text id="fnn1jp"
Microsoft Entra ID
        │
        ▼
Identity
```

---

## Autorisation

Répond à :

> Qu'as-tu le droit de faire ?

Exemple :

```text id="xqow75"
Identity
   │
   ▼
Azure RBAC
   │
   ▼
Contributor sur rg-mla-dev-fce-001
```

Une identité peut être correctement authentifiée mais ne posséder aucune permission sur la ressource ciblée.

---

# 5. Microsoft Entra ID

Microsoft Entra ID gère les identités.

Nous pouvons notamment y rencontrer :

```text id="ktlf19"
Users

Groups

Applications

Service Principals

Managed Identities

Workload Identities
```

Azure RBAC utilise ces identités comme principals auxquels des permissions peuvent être attribuées.

---

# 6. Security Principal

Un principal peut être :

```text id="6o2h2l"
User

Group

Service Principal

Managed Identity
```

Une role assignment associe ce principal à :

```text id="h9py8d"
Role
+
Scope
```

---

# 7. Formule RBAC

Un Role Assignment peut être résumé par :

```text id="c8m6da"
WHO
+
WHAT
+
WHERE
```

Soit :

```text id="w7dcf0"
Principal
+
Role Definition
+
Scope
```

Exemple :

```text id="x4r8in"
id-mla-tf-dev-fce-001
+
Contributor
+
rg-mla-dev-fce-001
```

---

# 8. Les scopes Azure RBAC

Les principaux niveaux sont :

```text id="sfv0e6"
Management Group
       │
       ▼
Subscription
       │
       ▼
Resource Group
       │
       ▼
Resource
```

Une permission attribuée en haut de la hiérarchie est héritée par les scopes descendants.

---

# 9. Exemple de scope trop large

Nous pourrions donner :

```text id="1bja74"
Contributor
```

à :

```text id="cwm6om"
id-mla-tf-dev-fce-001
```

au niveau :

```text id="hsjtw9"
Subscription
```

L'identité DEV pourrait alors modifier quasiment toutes les ressources de cette subscription.

Nous voulons éviter cela.

---

# 10. Scope retenu

Nous utiliserons :

```text id="49i2dr"
DEV identity
→ DEV Resource Group

QUAL identity
→ QUAL Resource Group

PROD identity
→ PROD Resource Group
```

C'est une application simple du principe du moindre privilège.

---

# 11. Pourquoi `Contributor` ?

Le rôle :

```text id="lhf4id"
Contributor
```

permet de gérer les ressources Azure dans le scope attribué.

Il ne permet pas de gérer les Role Assignments Azure RBAC.

Autrement dit :

```text id="6v9bv0"
Terraform workload identity
```

peut gérer :

```text id="ub24d5"
VNet
Subnets
NSG
VM
App Service
Key Vault resource
etc.
```

mais ne peut pas automatiquement s'octroyer des permissions supplémentaires.

---

# 12. Contributor n'est pas toujours Least Privilege parfait

Le rôle Contributor reste large.

Dans une plateforme très mature, nous pourrions construire un Custom Role limité à :

```text id="d8zwn6"
Microsoft.Network/*

Microsoft.KeyVault/vaults/*

Microsoft.ManagedIdentity/*
...
```

selon le besoin.

Mais cela demande une maintenance plus importante.

Pour notre Lab, nous retenons :

```text id="m46fg3"
Contributor
```

au scope :

```text id="zy1xf8"
Resource Group
```

Le scope réduit limite déjà fortement l'exposition.

---

# 13. Pourquoi ne pas donner `Owner` ?

`Owner` permet :

```text id="clu5ap"
gérer les ressources
+
gérer les accès RBAC
```

Terraform n'a pas besoin de pouvoir attribuer librement des permissions dans le workload normal.

Nous séparons :

```text id="4r34x3"
Bootstrap / IAM
```

et :

```text id="irgpns"
Workload deployment
```

---

# 14. Bootstrap privilégié

L'identité humaine utilisée pour ce Lab doit pouvoir créer :

```text id="xt7pm6"
Managed Identities

Resource Groups

Role Assignments
```

La création de Role Assignments nécessite notamment une permission :

```text id="1n3dne"
Microsoft.Authorization/roleAssignments/write
```

Un simple `Contributor` ne suffit donc pas.

---

# 15. Exemple de rôles permettant le bootstrap

Selon la gouvernance Azure, l'opérateur peut disposer par exemple de :

```text id="wuu7rb"
Role Based Access Control Administrator
```

ou :

```text id="exfgfp"
User Access Administrator
```

avec les autres permissions nécessaires à la création des ressources.

Dans un vrai environnement, ces rôles privilégiés doivent être étroitement contrôlés.

---

# 16. Identity humaine vs Terraform Identity

Nous avons :

```text id="gm3m63"
Human Identity
```

utilisée pour :

```text id="rkxfzm"
bootstrap
administration
break-glass
diagnostic
```

et :

```text id="sgxjhr"
Workload Identity
```

utilisée par :

```text id="dxsj3c"
Terraform automation
```

Les deux responsabilités ne doivent pas être confondues.

---

# 17. Service Principal

Un Service Principal représente une identité applicative dans Microsoft Entra ID.

Une application peut s'authentifier via :

```text id="amvi80"
Client Secret

Certificate

OIDC / Federated Credential
```

Historiquement, beaucoup de pipelines Terraform utilisaient :

```text id="z4q9sz"
Client ID
Client Secret
Tenant ID
Subscription ID
```

---

# 18. Problème du Client Secret

Un Client Secret doit être :

```text id="hvvuho"
créé

stocké

protégé

rotaté

renouvelé avant expiration
```

S'il fuite :

```text id="z4q9l5"
attaquant
    │
    ▼
Client Secret
    │
    ▼
Service Principal
    │
    ▼
Azure
```

Nous voulons donc progressivement supprimer les credentials statiques.

---

# 19. Managed Identity

Une Managed Identity est une identité Microsoft Entra gérée par Azure.

Azure gère les credentials associés.

Nous n'avons pas accès à :

```text id="1q6z0v"
password
client secret
private key
```

C'est précisément l'intérêt du mécanisme.

---

# 20. Deux types de Managed Identity

## System Assigned Managed Identity

Attachée à une ressource Azure.

Exemple :

```text id="mjtqje"
Azure VM
│
└── Managed Identity
```

Si la VM est supprimée, l'identité disparaît avec elle.

---

## User Assigned Managed Identity

Ressource Azure indépendante :

```text id="x3lt0n"
User Assigned Managed Identity
        │
        ├── VM
        ├── App Service
        ├── Automation
        └── External federation
```

Son cycle de vie est indépendant des workloads qui l'utilisent.

---

# 21. Pourquoi choisir une User Assigned Managed Identity ?

Pour notre plateforme Terraform :

```text id="fztkne"
Identity
```

doit avoir un cycle de vie indépendant :

```text id="8zqqhp"
du runner

de l'application

du pipeline
```

Nous choisissons donc :

```text id="ozlh2s"
User Assigned Managed Identity
```

---

# 22. Autre avantage important

Une User Assigned Managed Identity peut être utilisée avec la Workload Identity Federation.

Nous pourrons donc plus tard établir :

```text id="3zcdxh"
GitHub
   │
   │ OIDC token
   ▼
Microsoft Entra ID
   │
   ▼
User Assigned Managed Identity
   │
   ▼
Azure
```

Sans Client Secret.

---

# 23. Important : nous ne configurons pas encore OIDC

Dans ce Lab nous allons créer :

```text id="uu86iw"
Identity
+
RBAC
```

Nous ne créons pas encore :

```text id="qjtb8i"
GitHub federated credential
```

car ce mécanisme dépendra :

- du repository GitHub ;
- des branches ;
- des environments GitHub ;
- du workflow ;
- des claims OIDC.

Il sera traité avec GitHub Actions.

---

# 24. Important : la Managed Identity ne fonctionne pas directement depuis notre PC

Notre poste Windows local ne possède pas un endpoint Managed Identity Azure.

Nous ne pouvons donc pas simplement créer une UAMI puis exécuter :

```powershell id="n5b7qw"
terraform plan
```

localement en espérant que Terraform devienne cette identité.

---

# 25. Managed Identity authentication

L'authentification native Managed Identity est adaptée lorsque Terraform s'exécute sur une ressource Azure disposant de cette identité.

Par exemple :

```text id="7y6ckh"
Azure VM
    │
    └── Managed Identity
          │
          ▼
       Terraform
```

Terraform peut alors obtenir un token via l'infrastructure Azure.

---

# 26. Notre modèle actuel

Dans LAB-09 :

```text id="yn2lhb"
Poste local
   │
   ▼
az login
   │
   ▼
Human Identity
   │
   ▼
Terraform
```

Nous utilisons cette identité uniquement pour créer l'architecture IAM.

---

# 27. Notre modèle futur

Plus tard :

```text id="4kkj20"
GitHub Actions
      │
      ▼
OIDC token GitHub
      │
      ▼
Microsoft Entra ID
      │
      ▼
UAMI Terraform
      │
      ▼
Azure RBAC
```

Aucune identité humaine n'intervient dans le déploiement automatique.

---

# 28. Préparer LAB-09

Depuis :

```powershell id="iycsbq"
Set-Location "C:\Labs\MonLabAzure\terraform-azure-platform"
```

Créer :

```powershell id="ujpdm8"
New-Item `
  -ItemType Directory `
  -Name "lab-09"

Set-Location ".\lab-09"
```

Créer :

```powershell id="u5d0qh"
New-Item `
  -ItemType Directory `
  -Name "identity-bootstrap"
```

Puis :

```powershell id="3mrik9"
New-Item `
  -ItemType Directory `
  -Path ".\modules" `
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

# 29. Arborescence

Nous obtenons :

```text id="ej9gpv"
lab-09/
│
├── identity-bootstrap/
│
├── modules/
│
└── environments/
    ├── dev/
    ├── qual/
    └── prod/
```

---

# 30. Prérequis important sur LAB-08

Les Resource Groups :

```text id="l53dxe"
rg-mla-dev-fce-001

rg-mla-qual-fce-001

rg-mla-prod-fce-001
```

doivent être absents avant de lancer le bootstrap de LAB-09.

Si les ressources du LAB-08 existent encore, détruire d'abord LAB-08 proprement.

Ne pas tenter de recréer les mêmes Resource Groups dans deux states Terraform différents.

---

# 31. Nouvelle responsabilité du bootstrap

À partir de ce Lab :

```text id="dafpq7"
identity-bootstrap
```

va gérer :

```text id="tq4uof"
Resource Groups de déploiement

Managed Identities

Role Assignments
```

Les Root Modules DEV / QUALIF / PROD vont gérer uniquement leur contenu.

---

# 32. Pourquoi déplacer la création des Resource Groups ?

Si l'identité DEV doit recevoir :

```text id="q00pg3"
Contributor
```

au scope :

```text id="uee0rf"
rg-mla-dev-fce-001
```

ce Resource Group doit déjà exister.

Nous séparons donc :

```text id="y7m4ea"
Bootstrap
   │
   ├── créer RG
   ├── créer identity
   └── affecter RBAC
```

de :

```text id="232bmp"
Workload Terraform
   │
   └── créer ressources dans le RG
```

Cette séparation permet un vrai moindre privilège.

---

# 33. `identity-bootstrap/backend.tf`

Créer :

```hcl id="tp3qub"
terraform {
  backend "azurerm" {}
}
```

Le bootstrap possède lui aussi son propre state.

---

# 34. `identity-bootstrap/versions.tf`

```hcl id="379abj"
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

# 35. `identity-bootstrap/providers.tf`

```hcl id="48x7hp"
provider "azurerm" {
  features {}

  subscription_id = var.subscription_id
}
```

Le bootstrap utilise toujours l'identité Azure CLI actuellement connectée.

---

# 36. `identity-bootstrap/variables.tf`

```hcl id="4gbnjf"
variable "subscription_id" {
  description = "Subscription Azure contenant la plateforme."
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

variable "location" {
  description = "Région Azure."
  type        = string
  default     = "francecentral"
  nullable    = false
}

variable "owner" {
  description = "Équipe responsable."
  type        = string
  default     = "cloud-platform"
  nullable    = false
}
```

---

# 37. `identity-bootstrap/locals.tf`

```hcl id="otb68d"
locals {
  project_name = lower(trimspace(var.project_name))
  location     = lower(trimspace(var.location))

  location_codes = {
    francecentral = "fce"
    westeurope    = "weu"
  }

  location_code = local.location_codes[local.location]

  environments = {
    dev = {
      criticality = "low"
    }

    qual = {
      criticality = "medium"
    }

    prod = {
      criticality = "high"
    }
  }

  identity_resource_group_name = "rg-${local.project_name}-identity-${local.location_code}-001"

  workload_resource_group_names = {
    for environment, config in local.environments :
    environment => "rg-${local.project_name}-${environment}-${local.location_code}-001"
  }

  identity_names = {
    for environment, config in local.environments :
    environment => "id-${local.project_name}-tf-${environment}-${local.location_code}-001"
  }

  common_tags = {
    project    = local.project_name
    managed_by = "terraform"
    owner      = trimspace(var.owner)
    lab        = "lab-09"
  }
}
```

---

# 38. `identity-bootstrap/main.tf`

```hcl id="50kcl9"
data "azurerm_client_config" "current" {}

resource "azurerm_resource_group" "identity" {
  name     = local.identity_resource_group_name
  location = local.location

  tags = merge(
    local.common_tags,
    {
      component = "identity"
    }
  )
}

resource "azurerm_resource_group" "workload" {
  for_each = local.environments

  name     = local.workload_resource_group_names[each.key]
  location = local.location

  tags = merge(
    local.common_tags,
    {
      environment = each.key
      criticality = each.value.criticality
      component   = "workload"
    }
  )
}

resource "azurerm_user_assigned_identity" "terraform" {
  for_each = local.environments

  name                = local.identity_names[each.key]
  location            = azurerm_resource_group.identity.location
  resource_group_name = azurerm_resource_group.identity.name

  tags = merge(
    local.common_tags,
    {
      environment = each.key
      component   = "terraform-identity"
    }
  )
}

resource "azurerm_role_assignment" "terraform_contributor" {
  for_each = local.environments

  scope                = azurerm_resource_group.workload[each.key].id
  role_definition_name = "Contributor"

  principal_id   = azurerm_user_assigned_identity.terraform[each.key].principal_id
  principal_type = "ServicePrincipal"

  description = "Terraform ${upper(each.key)} workload deployment identity."

  skip_service_principal_aad_check = true
}
```

---

# 39. Pourquoi `for_each` ?

Nous avons trois environnements :

```text id="tzoji3"
dev
qual
prod
```

Plutôt que copier trois fois :

```hcl id="mxjico"
resource "azurerm_user_assigned_identity"
```

nous utilisons :

```hcl id="719tjn"
for_each = local.environments
```

Terraform crée trois instances.

---

# 40. Adresses Terraform obtenues

Nous aurons :

```text id="opgx20"
azurerm_user_assigned_identity.terraform["dev"]

azurerm_user_assigned_identity.terraform["qual"]

azurerm_user_assigned_identity.terraform["prod"]
```

Même principe pour les Resource Groups et Role Assignments.

---

# 41. Pourquoi `principal_id` et non `client_id` ?

La Role Assignment Azure RBAC attend l'identité du principal Microsoft Entra.

Nous utilisons :

```hcl id="uufmds"
principal_id
```

qui représente l'Object ID du Service Principal associé à la Managed Identity.

Le :

```hcl id="1b43dv"
client_id
```

sera utilisé plus tard pour l'authentification.

---

# 42. Différence importante

Pour une User Assigned Managed Identity :

```text id="opk8jg"
client_id
```

identifie l'application/identity pour l'authentification.

```text id="tzu5ys"
principal_id
```

identifie son Service Principal pour les Role Assignments Azure RBAC.

Ne pas confondre les deux.

---

# 43. Pourquoi `principal_type = "ServicePrincipal"` ?

Une Managed Identity est représentée dans Microsoft Entra ID par un Service Principal.

Nous indiquons donc explicitement :

```hcl id="jlph5x"
principal_type = "ServicePrincipal"
```

Cela rend l'intention claire et évite certains problèmes de résolution lors des Role Assignments.

---

# 44. Réplication Entra ID

Une Managed Identity nouvellement créée peut nécessiter un court délai avant d'être visible partout dans Microsoft Entra ID.

Nous utilisons :

```hcl id="fr54u9"
skip_service_principal_aad_check = true
```

dans cette situation précise afin d'éviter que le provider échoue uniquement à cause d'un délai de réplication du nouveau Service Principal.

Cela ne contourne pas Azure RBAC.

---

# 45. `identity-bootstrap/outputs.tf`

```hcl id="kimp6f"
output "operator_object_id" {
  description = "Object ID de l'identité ayant exécuté le bootstrap."
  value       = data.azurerm_client_config.current.object_id
}

output "tenant_id" {
  description = "Microsoft Entra Tenant ID."
  value       = data.azurerm_client_config.current.tenant_id
}

output "subscription_id" {
  description = "Subscription Azure."
  value       = data.azurerm_client_config.current.subscription_id
}

output "identity_resource_group_name" {
  description = "Resource Group contenant les Managed Identities."
  value       = azurerm_resource_group.identity.name
}

output "workload_resource_groups" {
  description = "Resource Groups dédiés aux environnements."

  value = {
    for environment, resource_group in azurerm_resource_group.workload :
    environment => {
      name = resource_group.name
      id   = resource_group.id
    }
  }
}

output "terraform_identities" {
  description = "Managed Identities créées pour Terraform."

  value = {
    for environment, identity in azurerm_user_assigned_identity.terraform :
    environment => {
      name         = identity.name
      resource_id  = identity.id
      client_id    = identity.client_id
      principal_id = identity.principal_id
      tenant_id    = identity.tenant_id
    }
  }
}
```

---

# 46. `identity-bootstrap/terraform.tfvars.example`

```hcl id="huq3pz"
subscription_id = "<SUBSCRIPTION_ID>"

project_name = "mla"
location     = "francecentral"
owner        = "cloud-platform"
```

Créer le fichier local :

```powershell id="f4hm9p"
$subscriptionId = az account show --query id -o tsv

(Get-Content `
  ".\identity-bootstrap\terraform.tfvars.example" `
  -Raw
).Replace(
  "<SUBSCRIPTION_ID>",
  $subscriptionId
) | Set-Content `
  -Encoding utf8 `
  ".\identity-bootstrap\terraform.tfvars"
```

---

# 47. Préparer le Remote State du bootstrap

Récupérer le Storage Account du LAB-06 :

```powershell id="76c01x"
$backendStorageAccount = terraform `
  -chdir="..\lab-06\bootstrap" `
  output -raw storage_account_name
```

Créer :

```powershell id="qyz31u"
@"
storage_account_name = "$backendStorageAccount"
container_name       = "tfstate"
key                  = "terraform-azure-platform/bootstrap/identity-rbac.tfstate"

use_azuread_auth = true
use_cli          = true
"@ | Set-Content `
  -Encoding utf8 `
  ".\identity-bootstrap\backend.azurerm.tfbackend"
```

---

# 48. Vérifier les permissions de l'opérateur

Avant de lancer Terraform :

```powershell id="ldn8x9"
az account show `
  --query "{Subscription:name,SubscriptionId:id,TenantId:tenantId,User:user.name}" `
  --output table
```

Il faut également disposer des permissions nécessaires pour créer les Role Assignments.

Si Terraform retourne :

```text id="r70x98"
AuthorizationFailed
```

sur :

```text id="ws645m"
Microsoft.Authorization/roleAssignments/write
```

le problème n'est pas dans le code Terraform.

L'identité exécutant le bootstrap n'a simplement pas les droits IAM nécessaires.

---

# 49. Initialiser le bootstrap

```powershell id="k4h5z3"
terraform `
  -chdir="identity-bootstrap" `
  init `
  -backend-config="backend.azurerm.tfbackend"
```

Puis :

```powershell id="uw0a90"
terraform `
  -chdir="identity-bootstrap" `
  fmt `
  -recursive
```

Puis :

```powershell id="5wumf5"
terraform `
  -chdir="identity-bootstrap" `
  validate
```

---

# 50. Plan du bootstrap

```powershell id="5yk0u7"
terraform `
  -chdir="identity-bootstrap" `
  plan `
  -out="identity-bootstrap.tfplan"
```

Nous devons notamment retrouver :

```text id="rk753l"
1 Resource Group identité

3 Resource Groups workload

3 User Assigned Managed Identities

3 Role Assignments
```

---

# 51. Appliquer

Après lecture du plan :

```powershell id="j8f2uk"
terraform `
  -chdir="identity-bootstrap" `
  apply `
  "identity-bootstrap.tfplan"
```

---

# 52. Vérifier les identités

```powershell id="tqxbjn"
terraform `
  -chdir="identity-bootstrap" `
  output terraform_identities
```

Nous devons retrouver trois identités.

---

# 53. Vérifier DEV

Récupérer les outputs en JSON :

```powershell id="qy48wr"
$identitiesJson = terraform `
  -chdir="identity-bootstrap" `
  output `
  -json `
  terraform_identities

$identities = $identitiesJson | ConvertFrom-Json
```

Puis :

```powershell id="wwrjt9"
$identities.dev
```

Nous devons retrouver :

```text id="dx3gm9"
name

client_id

principal_id

tenant_id

resource_id
```

---

# 54. Vérifier Azure

```powershell id="o5hoxk"
az identity list `
  --resource-group "rg-mla-identity-fce-001" `
  --output table
```

Nous devons retrouver :

```text id="lfhjno"
id-mla-tf-dev-fce-001

id-mla-tf-qual-fce-001

id-mla-tf-prod-fce-001
```

---

# 55. Vérifier RBAC DEV

```powershell id="8oc0ua"
$devPrincipalId = $identities.dev.principal_id
```

Puis :

```powershell id="wuq5bg"
$devRgId = az group show `
  --name "rg-mla-dev-fce-001" `
  --query id `
  -o tsv
```

Puis :

```powershell id="tcxvwp"
az role assignment list `
  --assignee-object-id $devPrincipalId `
  --scope $devRgId `
  --include-inherited `
  --output table
```

Nous devons retrouver :

```text id="6x87cn"
Contributor
```

---

# 56. Vérifier PROD

```powershell id="ku847w"
$prodPrincipalId = $identities.prod.principal_id

$prodRgId = az group show `
  --name "rg-mla-prod-fce-001" `
  --query id `
  -o tsv

az role assignment list `
  --assignee-object-id $prodPrincipalId `
  --scope $prodRgId `
  --include-inherited `
  --output table
```

La Managed Identity PROD ne doit pas avoir reçu volontairement `Contributor` sur DEV ou QUALIF.

---

# 57. Vérifier le moindre privilège

Nous pouvons vérifier toutes les affectations de DEV :

```powershell id="dljykx"
az role assignment list `
  --assignee-object-id $devPrincipalId `
  --all `
  --output table
```

Notre objectif est de retrouver uniquement les Role Assignments nécessaires.

---

# 58. Refactorer les Root Modules

Dans LAB-08, chaque environnement créait :

```hcl id="ax7q9y"
resource "azurerm_resource_group" "main"
```

Ce n'est plus nécessaire.

Le Resource Group est désormais créé par :

```text id="fnylhj"
identity-bootstrap
```

Le Root Module workload va le lire.

---

# 59. Copier les modules du LAB-08

Depuis :

```text id="nzcvry"
lab-09/
```

exécuter :

```powershell id="q4nfky"
Copy-Item `
  "..\lab-08\modules\*" `
  ".\modules" `
  -Recurse `
  -Force
```

Les modules :

```text id="wxdre5"
network

network-security
```

restent inchangés.

---

# 60. Copier les Root Modules

```powershell id="4fyzzo"
Copy-Item `
  "..\lab-08\environments\dev\*.tf" `
  ".\environments\dev" `
  -Force

Copy-Item `
  "..\lab-08\environments\qual\*.tf" `
  ".\environments\qual" `
  -Force

Copy-Item `
  "..\lab-08\environments\prod\*.tf" `
  ".\environments\prod" `
  -Force
```

Copier également les fichiers :

```text id="z5glp3"
terraform.tfvars.example
```

puis créer les fichiers locaux comme dans LAB-08.

---

# 61. Modifier `main.tf` dans chaque environnement

Supprimer :

```hcl id="k7v0m7"
resource "azurerm_resource_group" "main" {
  ...
}
```

et ajouter :

```hcl id="mtpk4f"
data "azurerm_resource_group" "main" {
  name = local.resource_names.resource_group
}
```

---

# 62. `main.tf` final

Le fichier final de chaque environnement doit être :

```hcl id="brbkh5"
data "azurerm_resource_group" "main" {
  name = local.resource_names.resource_group
}

module "network" {
  source = "../../modules/network"

  resource_group_name = data.azurerm_resource_group.main.name
  location            = data.azurerm_resource_group.main.location

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

  resource_group_name = data.azurerm_resource_group.main.name
  location            = data.azurerm_resource_group.main.location

  network_security_group_name = local.resource_names.network_security_group_app

  app_subnet_id = module.network.subnet_ids["app"]

  tags = local.common_tags
}
```

---

# 63. Adapter `outputs.tf`

Remplacer :

```hcl id="ecuwfb"
azurerm_resource_group.main.name
```

par :

```hcl id="pnvzls"
data.azurerm_resource_group.main.name
```

Exemple :

```hcl id="lm9i0p"
output "resource_group_name" {
  description = "Nom du Resource Group."
  value       = data.azurerm_resource_group.main.name
}
```

Les autres outputs restent basés sur les modules.

---

# 64. Le state workload ne possède plus le Resource Group

Notre workload state contiendra :

```text id="w5oqk3"
module.network...

module.network_security...
```

mais pas :

```text id="c7mbqu"
azurerm_resource_group.main
```

Le Resource Group appartient désormais au state :

```text id="uk0n02"
identity-bootstrap
```

---

# 65. Pourquoi cette séparation est utile ?

La Managed Identity DEV possède :

```text id="5bg4di"
Contributor
```

sur :

```text id="gp41ls"
rg-mla-dev-fce-001
```

Elle n'a pas besoin de créer ou supprimer ce Resource Group.

Le boundary existe déjà.

---

# 66. Architecture obtenue

```text id="9hb44n"
Human / IAM Bootstrap
          │
          ▼
identity-bootstrap state
          │
          ├── Resource Groups
          ├── Managed Identities
          └── RBAC
                 │
                 ▼
       Terraform identities
                 │
       ┌─────────┼─────────┐
       ▼         ▼         ▼
      DEV       QUAL      PROD
       │         │         │
       ▼         ▼         ▼
 Workload RG Workload RG Workload RG
```

---

# 67. Initialiser DEV

Le backend DEV reste celui utilisé dans LAB-08.

Créer le backend config si nécessaire :

```powershell id="fs3nvs"
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

Puis :

```powershell id="slmxx1"
terraform `
  -chdir="environments/dev" `
  init `
  -backend-config="backend.azurerm.tfbackend"
```

---

# 68. Dans ce Lab, qui exécute réellement Terraform DEV ?

Toujours notre utilisateur connecté via :

```powershell id="p6gtd4"
az login
```

Nous n'utilisons pas encore :

```text id="6hkpwf"
id-mla-tf-dev-fce-001
```

pour l'authentification du provider.

Nous avons uniquement préparé l'identité et ses permissions.

---

# 69. Pourquoi ?

Une User Assigned Managed Identity ne peut pas être utilisée directement depuis un ordinateur Windows classique comme une identité interactive.

Il faut soit :

```text id="xmx1sp"
Terraform exécuté sur Azure avec Managed Identity
```

soit :

```text id="7yiemj"
Terraform exécuté depuis GitHub avec Workload Identity Federation
```

Notre cible sera la seconde option.

---

# 70. Plan DEV

Exécuter :

```powershell id="6ve4pa"
terraform `
  -chdir="environments/dev" `
  validate
```

Puis :

```powershell id="uj5mjs"
terraform `
  -chdir="environments/dev" `
  plan `
  -out="dev.tfplan"
```

Nous devons maintenant créer :

```text id="bzjnwv"
Virtual Network

2 Subnets

NSG

NSG Association
```

Le Resource Group existe déjà.

---

# 71. Résultat attendu

Le plan doit donc être proche de :

```text id="n34ij6"
5 to add
```

et non :

```text id="mk20te"
6 to add
```

comme dans LAB-08.

---

# 72. Appliquer DEV

```powershell id="m55xwn"
terraform `
  -chdir="environments/dev" `
  apply `
  "dev.tfplan"
```

Puis :

```powershell id="nygbnp"
terraform `
  -chdir="environments/dev" `
  plan
```

Résultat :

```text id="0t17q5"
No changes.
```

---

# 73. Répéter pour QUALIF et PROD

Les mêmes principes s'appliquent.

QUALIF utilise :

```text id="lrqz0n"
rg-mla-qual-fce-001
```

et PROD :

```text id="v8mv8r"
rg-mla-prod-fce-001
```

Les Root Modules restent indépendants.

---

# 74. Peut-on réellement tester la Managed Identity maintenant ?

Pas directement depuis le poste local.

Nous pouvons toutefois vérifier :

```text id="akzq4y"
Identity existe
```

et :

```text id="4qt6gq"
Role Assignment existe
```

ce qui constitue la préparation nécessaire avant l'ajout de la fédération.

---

# 75. Authentification Managed Identity depuis Azure

Sur un runner hébergé dans Azure avec une Managed Identity, Terraform peut être configuré conceptuellement avec :

```powershell id="z8hkc2"
$env:ARM_USE_MSI = "true"

$env:ARM_CLIENT_ID = "<MANAGED_IDENTITY_CLIENT_ID>"

$env:ARM_SUBSCRIPTION_ID = "<SUBSCRIPTION_ID>"

$env:ARM_TENANT_ID = "<TENANT_ID>"
```

Le runner Azure récupère alors un token via le mécanisme Managed Identity.

Ce mécanisme ne fonctionne pas de la même façon depuis notre poste local.

---

# 76. Authentification OIDC future

Avec GitHub Actions, nous utiliserons plutôt :

```text id="abzhko"
ARM_USE_OIDC
ARM_CLIENT_ID
ARM_TENANT_ID
ARM_SUBSCRIPTION_ID
```

avec un token OIDC fourni dynamiquement par GitHub.

Aucun :

```text id="a13ftu"
ARM_CLIENT_SECRET
```

ne sera nécessaire.

---

# 77. Architecture OIDC future

```text id="rotbm2"
GitHub Workflow
      │
      │ id-token
      ▼
GitHub OIDC Provider
      │
      ▼
Federated Identity Credential
      │
      ▼
User Assigned Managed Identity
      │
      ▼
Azure RBAC
      │
      ▼
Azure Resources
```

---

# 78. Federated Identity Credential

Le Federated Identity Credential définit notamment la confiance sur :

```text id="4smkst"
Issuer

Subject

Audience
```

Exemple conceptuel :

```text id="zckzwb"
Issuer
https://token.actions.githubusercontent.com

Subject
repo:organisation/repository:environment:prod

Audience
api://AzureADTokenExchange
```

Ces valeurs seront configurées lorsque nous créerons les workflows GitHub.

---

# 79. Pourquoi OIDC est préférable à un Client Secret ?

Client Secret :

```text id="rk6hyd"
Secret permanent
      │
      ▼
GitHub Secret
      │
      ▼
Azure
```

OIDC :

```text id="qo8vjw"
GitHub
   │
   ▼
Token court
   │
   ▼
Trust validation
   │
   ▼
Azure access token
```

Il n'existe pas de credential Azure permanent à voler dans le repository ou le gestionnaire de secrets GitHub.

---

# 80. RBAC Azure et Microsoft Entra Roles ne sont pas identiques

Azure RBAC :

```text id="656jcz"
Contributor
Reader
Owner
Network Contributor
```

contrôle l'accès aux ressources Azure.

Microsoft Entra roles :

```text id="uxodfr"
Global Administrator

Application Administrator

User Administrator
```

contrôlent les objets du tenant Microsoft Entra.

Ce sont deux systèmes d'autorisation distincts.

---

# 81. Terraform AzureRM vs AzureAD

Le provider :

```text id="xx7odf"
azurerm
```

gère principalement :

```text id="1mv78q"
Azure Resource Manager resources
```

Le provider :

```text id="lrmujh"
azuread
```

gère les objets Microsoft Entra via Microsoft Graph.

Dans ce Lab, nous créons des User Assigned Managed Identities comme ressources Azure.

Nous n'avons donc pas besoin d'introduire immédiatement le provider `azuread`.

---

# 82. Management Plane

Le Management Plane concerne la gestion des ressources Azure.

Exemples :

```text id="6fn8w8"
Créer un VNet

Créer un Key Vault

Modifier un NSG

Créer un Storage Account
```

Notre rôle :

```text id="wzopku"
Contributor
```

agit principalement sur ce plan.

---

# 83. Data Plane

Le Data Plane concerne le contenu d'une ressource.

Exemples :

```text id="d2g1h3"
Lire un Blob

Lire un Secret Key Vault

Publier un message dans une Queue

Accéder à des données SQL
```

Posséder Contributor sur un Resource Group ne signifie pas automatiquement avoir accès aux données de toutes les ressources.

---

# 84. Exemple Key Vault

Terraform pourrait avoir :

```text id="koxh4z"
Contributor
```

pour créer :

```text id="wwnqlp"
Azure Key Vault
```

sans avoir automatiquement le droit de lire :

```text id="6uqgur"
les secrets
```

à l'intérieur.

Cette distinction sera importante dans le LAB-10.

---

# 85. Exemple Storage

Même principe :

```text id="mfq7hb"
Contributor
```

peut gérer le Storage Account.

Mais lire les blobs avec Microsoft Entra ID nécessite un rôle de Data Plane comme :

```text id="7h0kcq"
Storage Blob Data Reader
```

ou :

```text id="6gf4t9"
Storage Blob Data Contributor
```

selon le besoin.

---

# 86. Backend Terraform et identité workload

Pour accéder au Remote State du LAB-06, notre future identité GitHub devra également recevoir un rôle Data Plane approprié sur Azure Storage.

Nous ne l'ajoutons pas encore dans ce Lab.

Pourquoi ?

Parce que le LAB-08 utilise actuellement :

```text id="e1nzeo"
un container partagé
```

pour :

```text id="vc2an7"
DEV
QUALIF
PROD
```

Donner `Storage Blob Data Contributor` au niveau du container à l'identité DEV pourrait également lui donner accès aux states QUALIF et PROD.

Nous corrigerons cette séparation lorsque nous mettrons en place la fédération CI/CD.

---

# 87. Point d'architecture important

Séparer uniquement les backend keys :

```text id="rg9w48"
dev/platform.tfstate

qual/platform.tfstate

prod/platform.tfstate
```

sépare bien les states du point de vue Terraform.

Cela ne crée pas nécessairement une frontière RBAC Azure suffisamment fine entre les identités.

Nous devons donc distinguer :

```text id="c04og1"
Terraform state isolation
```

de :

```text id="p31a7i"
Azure authorization isolation
```

---

# 88. Cible future du backend

Pour une séparation plus forte, nous pourrons utiliser :

```text id="y61n82"
Container tfstate-dev

Container tfstate-qual

Container tfstate-prod
```

voire :

```text id="p5txqf"
Storage Account backend DEV

Storage Account backend QUALIF

Storage Account backend PROD
```

selon les exigences de sécurité.

---

# 89. Least Privilege est un processus

Le moindre privilège ne signifie pas uniquement :

```text id="3bxqf8"
ne pas utiliser Owner
```

Il faut examiner :

```text id="j5fn50"
Identity

Role

Scope

Management Plane

Data Plane

Duration

Environment

Network access
```

L'ensemble constitue le modèle d'autorisation.

---

# 90. Custom Role Terraform

Dans une organisation mature, un rôle personnalisé pourrait être construit avec uniquement les permissions nécessaires.

Conceptuellement :

```text id="5op5ma"
Terraform Network Operator

Actions:
Microsoft.Network/*
Microsoft.Resources/*/read
...
```

Mais un Custom Role doit être maintenu à mesure que la plateforme évolue.

---

# 91. Risque du rôle trop restrictif

Un rôle trop strict peut provoquer des échecs permanents lorsqu'un nouveau service Azure est ajouté.

Exemple :

```text id="wq8rnj"
Terraform ajoute Key Vault
```

mais le Custom Role permet uniquement :

```text id="qdxsfr"
Microsoft.Network/*
```

Le pipeline échoue.

---

# 92. Risque du rôle trop large

À l'inverse :

```text id="bqrrjb"
Owner au niveau subscription
```

pour toutes les pipelines Terraform élimine pratiquement toute séparation de privilège.

Nous cherchons donc un compromis contrôlé.

---

# 93. Notre modèle de progression

Pour MonLabAzure :

```text id="ovg2wx"
Étape 1
Contributor sur Resource Group

Étape 2
séparer Management Plane / Data Plane

Étape 3
OIDC

Étape 4
policies

Étape 5
éventuels Custom Roles
```

Nous augmentons progressivement la maturité sans rendre le Lab artificiellement complexe.

---

# 94. Supprimer un Role Assignment

Si nous supprimons :

```text id="wsh7js"
azurerm_role_assignment.terraform_contributor["dev"]
```

l'identité existe toujours.

Elle perd simplement le droit :

```text id="h07mrz"
Contributor
```

sur :

```text id="4ux4bw"
rg-mla-dev-fce-001
```

L'identité et l'autorisation ont des cycles de vie distincts.

---

# 95. Supprimer une Managed Identity

Si nous supprimons :

```text id="sa5rbp"
id-mla-tf-dev-fce-001
```

son Service Principal Microsoft Entra est supprimé.

Les Role Assignments associés doivent également être nettoyés.

Il est recommandé de ne pas laisser des Role Assignments orphelins.

---

# 96. Vérifier une identity avec Azure CLI

DEV :

```powershell id="pdcc27"
az identity show `
  --resource-group "rg-mla-identity-fce-001" `
  --name "id-mla-tf-dev-fce-001" `
  --query "{Name:name,ClientId:clientId,PrincipalId:principalId,TenantId:tenantId}" `
  --output table
```

---

# 97. Vérifier les permissions

```powershell id="q1af6g"
az role assignment list `
  --assignee-object-id $devPrincipalId `
  --all `
  --query "[].{Role:roleDefinitionName,Scope:scope}" `
  --output table
```

Nous devons pouvoir expliquer chaque ligne.

Si une permission apparaît sans justification claire, elle mérite une revue.

---

# 98. Ne jamais raisonner uniquement avec le nom du rôle

Le scope est aussi important que le rôle.

Comparer :

```text id="d14o9l"
Contributor
sur
rg-mla-dev-fce-001
```

et :

```text id="w6gf9u"
Contributor
sur
Subscription
```

Le nom du rôle est identique.

Le niveau d'accès est très différent.

---

# 99. Héritage RBAC

Une permission attribuée à :

```text id="eq8rmm"
Subscription
```

est héritée par :

```text id="g4fsy9"
Resource Groups
Resources
```

Une permission attribuée uniquement à :

```text id="q375pl"
rg-mla-dev-fce-001
```

ne s'étend pas automatiquement à :

```text id="o6xvcs"
rg-mla-prod-fce-001
```

C'est précisément la frontière que nous recherchons.

---

# 100. Plan final

Nous avons maintenant :

```text id="qdbrka"
ADMINISTRATION PLANE

Human Identity
      │
      ▼
Identity Bootstrap
      │
      ├── Workload Resource Groups
      ├── Managed Identities
      └── Role Assignments


WORKLOAD PLANE

Terraform Identity DEV
      │
      ▼
DEV Resource Group

Terraform Identity QUAL
      │
      ▼
QUAL Resource Group

Terraform Identity PROD
      │
      ▼
PROD Resource Group
```

---

# 101. Versionner LAB-09

Depuis :

```powershell id="xx3rne"
Set-Location "C:\Labs\MonLabAzure\terraform-azure-platform"
```

Vérifier :

```powershell id="4ik3vw"
git status
```

Puis :

```powershell id="od3jg0"
git add .
```

Contrôler :

```powershell id="8pktnx"
git diff --cached --name-only
```

Ne pas versionner :

```text id="7ed1nu"
terraform.tfvars

*.tfbackend

*.tfplan

terraform.tfstate*

.terraform/
```

Puis :

```powershell id="6l1lzh"
git commit -m "feat(terraform): add identity and RBAC architecture"
```

---

# 102. Nettoyage — ordre critique

Nous avons maintenant deux catégories de states :

```text id="gzom29"
workload states
```

et :

```text id="7pvi8k"
identity-bootstrap state
```

Ne pas détruire le bootstrap en premier.

---

# 103. Mauvais ordre

À éviter :

```text id="hpu5vo"
Destroy identity-bootstrap
      │
      ▼
Resource Groups supprimés
      │
      ▼
Workload resources supprimées hors de leurs states
```

Les states DEV / QUALIF / PROD se retrouveraient avec des ressources disparues.

---

# 104. Bon ordre

Toujours :

```text id="202e07"
1. Destroy PROD workload

2. Destroy QUALIF workload

3. Destroy DEV workload

4. Vérifier les states

5. Destroy identity-bootstrap
```

---

# 105. Détruire DEV

```powershell id="2ep93m"
terraform `
  -chdir="environments/dev" `
  plan `
  -destroy `
  -out="destroy.tfplan"
```

Puis :

```powershell id="ofz93p"
terraform `
  -chdir="environments/dev" `
  apply `
  "destroy.tfplan"
```

Le Resource Group doit rester présent.

---

# 106. Pourquoi le Resource Group reste présent ?

Il n'appartient pas au state DEV.

Il appartient à :

```text id="438vom"
identity-bootstrap
```

Le workload détruit seulement ce qu'il gère.

---

# 107. Détruire QUALIF et PROD

Effectuer la même opération pour :

```text id="kmodnt"
environments/qual
```

et :

```text id="j221bh"
environments/prod
```

Puis vérifier que les trois Resource Groups sont vides.

---

# 108. Détruire le bootstrap

Une fois les workloads supprimés :

```powershell id="1l0uak"
terraform `
  -chdir="identity-bootstrap" `
  plan `
  -destroy `
  -out="destroy.tfplan"
```

Lire le plan.

Il doit notamment supprimer :

```text id="p7j28y"
Role Assignments

Managed Identities

Workload Resource Groups

Identity Resource Group
```

Puis :

```powershell id="k8okhs"
terraform `
  -chdir="identity-bootstrap" `
  apply `
  "destroy.tfplan"
```

---

# 109. Ce qu'il faut retenir

Une plateforme Terraform professionnelle ne doit pas dépendre du compte personnel d'un ingénieur.

Nous distinguons désormais :

```text id="kc58q2"
Human Identity
```

et :

```text id="0drjbm"
Terraform Workload Identity
```

---

## Authentication

Détermine :

```text id="lx4omd"
qui est l'identité
```

---

## Authorization

Détermine :

```text id="jz3rxf"
ce qu'elle peut faire
```

---

## Managed Identity

Permet d'utiliser une identité Microsoft Entra gérée par Azure sans stocker ses credentials.

---

## Azure RBAC

Associe :

```text id="et3px0"
Principal
+
Role
+
Scope
```

---

## Scope

Doit être aussi réduit que raisonnablement possible.

---

## Contributor

Permet de gérer les ressources mais pas de gérer librement les Role Assignments Azure RBAC.

---

## OIDC

Nous permettra plus tard d'utiliser les Managed Identities depuis GitHub Actions sans Client Secret permanent.

---

# 110. comment authentifiez-vous Terraform localement ?

Réponse :

> Pour le travail interactif local, j'utilise Azure CLI avec mon identité utilisateur. Je vérifie explicitement le tenant et la subscription. Je n'utilise pas cette identité personnelle pour les pipelines automatisées.

---

# 111. comment authentifiez-vous Terraform en CI/CD ?

Réponse cible :

> J'utilise une workload identity dédiée par environnement et Workload Identity Federation avec OIDC. La pipeline obtient un token court depuis son Identity Provider, par exemple GitHub, qui est échangé auprès de Microsoft Entra ID. Je n'ai donc pas besoin de stocker de Client Secret Azure permanent dans GitHub.

---

# 112. pourquoi une identité par environnement ?

Réponse :

> Pour réduire le blast radius et appliquer le moindre privilège. L'identité DEV n'a pas besoin de permissions PROD. Chaque identité possède donc ses Role Assignments uniquement sur le scope correspondant à son environnement.

---

# 113. pourquoi Contributor plutôt qu'Owner ?

Réponse :

> Le workload Terraform doit pouvoir gérer les ressources mais n'a généralement pas besoin de gérer les droits RBAC. Contributor permet de gérer les ressources sans autoriser les role assignments. Les opérations IAM restent dans un bootstrap ou un pipeline privilégié séparé.

---

# 114. Service Principal ou Managed Identity ?

Réponse :

> Les deux représentent des workload identities. J'utilise une Managed Identity lorsqu'elle correspond au runtime ou lorsque je peux utiliser une User Assigned Managed Identity avec Workload Identity Federation. Pour une workload externe, une application/service principal avec OIDC peut également convenir. Dans tous les cas, je privilégie la fédération ou les identités managées plutôt qu'un Client Secret permanent.

---

# 115. Management Plane et Data Plane ?

Réponse :

> Le Management Plane sert à gérer la ressource elle-même via Azure Resource Manager, par exemple créer un Storage Account ou un Key Vault. Le Data Plane concerne les données contenues dans cette ressource, par exemple lire un blob ou un secret. Un rôle Contributor sur le Resource Group ne signifie donc pas automatiquement que Terraform peut lire les données de tous les services.

---

# 116. comment gérez-vous le moindre privilège ?

Réponse :

> Je raisonne sur trois éléments : principal, rôle et scope. Je sépare les identités par environnement, j'évite les rôles privilégiés comme Owner dans les workloads, je limite le scope au Resource Group ou à la ressource nécessaire et j'ajoute séparément les rôles Data Plane lorsque le besoin existe.

---

# 117. Validation du LAB-09

Le Lab est terminé lorsque les points suivants sont compris et testés :

```text id="h0h4j7"
[ ] Comprendre authentification vs autorisation

[ ] Comprendre Microsoft Entra ID

[ ] Comprendre un Security Principal

[ ] Comprendre Service Principal

[ ] Comprendre Managed Identity

[ ] Comprendre System Assigned Managed Identity

[ ] Comprendre User Assigned Managed Identity

[ ] Comprendre pourquoi nous choisissons une UAMI

[ ] Comprendre Azure RBAC

[ ] Comprendre Principal + Role + Scope

[ ] Comprendre les scopes Azure

[ ] Comprendre l'héritage RBAC

[ ] Comprendre Contributor

[ ] Comprendre pourquoi Contributor ne gère pas les role assignments

[ ] Comprendre pourquoi nous évitons Owner

[ ] Comprendre le rôle privilégié du bootstrap

[ ] Créer rg-mla-identity-fce-001

[ ] Créer les trois workload Resource Groups

[ ] Créer une UAMI DEV

[ ] Créer une UAMI QUALIF

[ ] Créer une UAMI PROD

[ ] Assigner Contributor à DEV sur son Resource Group

[ ] Assigner Contributor à QUALIF sur son Resource Group

[ ] Assigner Contributor à PROD sur son Resource Group

[ ] Vérifier les Role Assignments avec Azure CLI

[ ] Comprendre client_id vs principal_id

[ ] Comprendre principal_type

[ ] Comprendre la réplication Entra ID

[ ] Séparer Resource Group bootstrap et workload

[ ] Modifier les Root Modules pour utiliser un data source Resource Group

[ ] Comprendre pourquoi le Resource Group n'appartient plus au workload state

[ ] Comprendre pourquoi une UAMI ne s'utilise pas directement depuis le poste local

[ ] Comprendre ARM_USE_MSI

[ ] Comprendre le principe d'ARM_USE_OIDC

[ ] Comprendre Workload Identity Federation

[ ] Comprendre le futur Federated Identity Credential

[ ] Comprendre Management Plane

[ ] Comprendre Data Plane

[ ] Comprendre que Contributor ne donne pas automatiquement accès aux secrets et blobs

[ ] Comprendre les limites actuelles de l'isolation du backend

[ ] Comprendre l'ordre de destruction des states
```

---

# 118. Progression

Nous avons maintenant :

```text id="1nq43b"
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
     ▼
LAB-09
Identity & RBAC
     │
     ├── Human Identity
     ├── Managed Identity
     ├── RBAC
     ├── Scope
     ├── Least Privilege
     └── préparation OIDC
```

Notre architecture commence maintenant à séparer correctement :

```text id="4y76v5"
Code

State

Environment

Identity

Authorization
```

C'est une étape importante avant d'introduire des ressources contenant des données sensibles.

---

# 119. Suite

Le prochain Lab va introduire Azure Key Vault.

Nous allons étudier une différence fondamentale :

```text id="9dxzhw"
Terraform peut créer un Key Vault
```

sans nécessairement :

```text id="ptq2f8"
avoir le droit de lire ses secrets
```

Nous travaillerons sur :

- Azure Key Vault ;
- RBAC permission model ;
- Key Vault Contributor ;
- Key Vault Secrets Officer ;
- Key Vault Secrets User ;
- Managed Identity ;
- Management Plane ;
- Data Plane ;
- secrets Terraform ;
- `sensitive` ;
- Terraform State ;
- références Key Vault ;
- pourquoi éviter de gérer la valeur d'un secret dans Terraform lorsque ce n'est pas nécessaire.

Le prochain article sera :

**LAB-10 — Azure Key Vault, secrets et Terraform**

---

## Références officielles

Les concepts étudiés dans ce Lab sont documentés principalement dans :

- Microsoft Learn — Azure Role-Based Access Control ;
- Microsoft Learn — Azure Role Assignments ;
- Microsoft Learn — Azure Built-in Roles ;
- Microsoft Learn — Managed Identities for Azure Resources ;
- Microsoft Learn — Workload Identity Federation ;
- Microsoft Learn — GitHub Actions avec OpenID Connect ;
- HashiCorp Terraform Registry — AzureRM Authentication ;
- HashiCorp Terraform Registry — Managed Identity Authentication ;
- HashiCorp Terraform Registry — OpenID Connect Authentication ;
- Terraform Registry — `azurerm_user_assigned_identity` ;
- Terraform Registry — `azurerm_role_assignment`.

La stratégie retenue dans ce Lab sépare volontairement l'administration IAM du déploiement des workloads. Les permissions réellement nécessaires doivent toujours être évaluées en fonction des ressources gérées par chaque environnement.