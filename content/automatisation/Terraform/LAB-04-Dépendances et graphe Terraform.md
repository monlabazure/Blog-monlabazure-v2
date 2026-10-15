---
title: "LAB-04 - Dépendances et graphe Terraform"
date: 2026-10-06
draft: false
description: "Comprendre comment Terraform construit son graphe de dépendances, ordonne les ressources, parallélise les opérations et utilise les dépendances implicites et depends_on."
summary: "Construire un réseau Azure avec VNet, subnets et NSG pour comprendre concrètement le graphe de dépendances Terraform."
slug: "terraform-azure-lab-04-dependances-graphe"
categories:
  - "Automatisation"
tags:
  - "Terraform"
  - "Azure"
  - "Dependency Graph"
  - "depends_on"
  - "Virtual Network"
  - "Subnet"
  - "Network Security Group"
  - "AzureRM"
series:
  - "Terraform Azure - De zéro à production"
weight: 5
ShowToc: false
TocOpen: false
---


Dans les Labs précédents, nous avons déjà rencontré plusieurs dépendances Terraform sans réellement étudier leur fonctionnement interne.

Par exemple :

```hcl
resource_group_name = azurerm_resource_group.main.name
```

Cette référence ne sert pas uniquement à récupérer le nom du Resource Group.

Elle indique également à Terraform qu'une relation existe entre deux ressources.

Terraform exploite ces relations pour construire un **graphe de dépendances**.

Dans ce Lab, nous allons enrichir notre plateforme Azure avec :

```text
Resource Group
│
├── Virtual Network
│    │
│    ├── Subnet application
│    │
│    └── Subnet Private Endpoints
│
└── Network Security Group
      │
      └── Association avec le Subnet application
```

L'objectif principal n'est pas encore d'approfondir le réseau Azure.

Nous allons utiliser cette architecture pour comprendre :

```text
références
    ↓
dépendances implicites
    ↓
dépendances explicites
    ↓
graphe Terraform
    ↓
ordre d'exécution
    ↓
parallélisme
    ↓
ordre de destruction
```

---

# 1. Objectifs du Lab

À la fin de ce Lab, nous devons être capables de comprendre :

- ce qu'est une dépendance Terraform ;
- comment Terraform détecte une dépendance implicite ;
- pourquoi une référence crée une relation entre ressources ;
- ce qu'est le graphe de dépendances ;
- pourquoi l'ordre physique des blocs ne définit pas l'ordre de création ;
- comment Terraform peut paralléliser certaines opérations ;
- quand utiliser `depends_on` ;
- quand ne pas utiliser `depends_on` ;
- comment une dépendance artificielle peut ralentir ou compliquer un déploiement ;
- comment Terraform détecte un cycle ;
- comment lire un graphe avec `terraform graph` ;
- comment visualiser le graphe avec Graphviz ;
- comment les dépendances influencent la destruction.

---

# 2. Architecture cible

Nous allons construire :

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

Adressage :

```text
VNet
10.20.0.0/16

├── Application
│   10.20.1.0/24
│
└── Private Endpoints
    10.20.2.0/24
```

---

# 3. Graphe logique attendu

Avant même d'écrire le code, nous pouvons représenter les dépendances :

```text
                  Resource Group
                  /            \
                 /              \
                ▼                ▼
             VNet              NSG
            /    \               │
           /      \              │
          ▼        ▼             │
   Subnet APP   Subnet PE        │
          │                       │
          └──────────┬────────────┘
                     ▼
             NSG Association
```

Plusieurs informations sont déjà visibles.

Le VNet dépend du Resource Group.

Le NSG dépend également du Resource Group.

En revanche :

```text
VNet
```

et :

```text
NSG
```

ne dépendent pas directement l'un de l'autre.

Terraform pourra donc potentiellement travailler sur ces deux branches en parallèle.

---

# 4. Prérequis

Les LAB-00 à LAB-03 doivent avoir été réalisés.

Vérifier :

```powershell
terraform version
az version
git --version
code --version
```

Vérifier Azure :

```powershell
az account show `
  --query "{Subscription:name,SubscriptionId:id,TenantId:tenantId}" `
  --output table
```

Définir le contexte :

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

# 5. Créer LAB-04

Revenir à la racine :

```powershell
Set-Location "C:\Labs\MonLabAzure\terraform-azure-platform"
```

Créer :

```powershell
New-Item -ItemType Directory -Name "lab-04"
```

Puis :

```powershell
Set-Location ".\lab-04"
code .
```

---

# 6. Arborescence

Nous utiliserons :

```text
lab-04/
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

La séparation reste volontairement simple.

Nous n'utilisons pas encore de modules.

---

# 7. `versions.tf`

Créer :

```text
versions.tf
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

---

# 8. `providers.tf`

Créer :

```text
providers.tf
```

avec :

```hcl
provider "azurerm" {
  features {}
}
```

---

# 9. `variables.tf`

Créer :

```text
variables.tf
```

avec :

```hcl
variable "project_name" {
  description = "Nom court du projet."
  type        = string
  default     = "mla"
  nullable    = false

  validation {
    condition = (
      length(trimspace(var.project_name)) >= 2 &&
      length(trimspace(var.project_name)) <= 10 &&
      can(regex("^[a-z0-9-]+$", var.project_name))
    )

    error_message = "project_name doit contenir entre 2 et 10 caractères : lettres minuscules, chiffres et tirets uniquement."
  }
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

  validation {
    condition = contains(
      ["francecentral", "westeurope"],
      lower(var.location)
    )

    error_message = "location doit être francecentral ou westeurope."
  }
}

variable "resource_instance" {
  description = "Numéro d'instance utilisé dans les conventions de nommage."
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
  description = "Équipe responsable des ressources."
  type        = string
  default     = "cloud-platform"
  nullable    = false
}

variable "network" {
  description = "Configuration réseau utilisée par le Lab."

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

  validation {
    condition = (
      length(var.network.vnet_address_spaces) > 0 &&
      length(var.network.app_subnet_prefixes) > 0 &&
      length(var.network.pe_subnet_prefixes) > 0
    )

    error_message = "Chaque collection réseau doit contenir au moins un préfixe."
  }
}

variable "additional_tags" {
  description = "Tags Azure supplémentaires."
  type        = map(string)
  default     = {}
  nullable    = false
}
```

---

# 10. `locals.tf`

Créer :

```text
locals.tf
```

avec :

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
    lab         = "lab-04"
  }

  common_tags = merge(
    var.additional_tags,
    local.mandatory_tags
  )
}
```

Les tags obligatoires sont placés en dernier dans `merge()`.

Ainsi, un utilisateur ne peut pas remplacer via `additional_tags` :

```text
managed_by
environment
project
owner
lab
```

---

# 11. `main.tf`

Créer :

```text
main.tf
```

avec le code complet suivant :

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

Nous avons maintenant suffisamment de ressources pour observer un graphe intéressant.

---

# 12. Ne pas lire `main.tf` comme un script

Une erreur classique serait de lire :

```text
1. Resource Group
2. VNet
3. Subnet APP
4. Subnet PE
5. NSG
6. Association
```

et d'en conclure que Terraform va suivre cet ordre strict.

Ce n'est pas le bon modèle mental.

Terraform analyse les références.

---

# 13. Première dépendance implicite

Dans :

```hcl
resource "azurerm_virtual_network" "main" {
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
}
```

le VNet référence :

```text
azurerm_resource_group.main
```

Terraform peut donc déduire :

```text
Resource Group
      │
      ▼
     VNet
```

Nous parlons de **dépendance implicite**.

---

# 14. Dépendance du Subnet

Le Subnet application contient :

```hcl
resource_group_name  = azurerm_resource_group.main.name
virtual_network_name = azurerm_virtual_network.main.name
```

Terraform voit donc des références vers :

```text
Resource Group
VNet
```

Le Subnet ne peut pas être créé avant que le VNet nécessaire soit disponible.

---

# 15. Deuxième branche du graphe

Le NSG contient :

```hcl
location            = azurerm_resource_group.main.location
resource_group_name = azurerm_resource_group.main.name
```

Le NSG dépend donc du Resource Group :

```text
Resource Group
      │
      ▼
     NSG
```

Mais le NSG ne référence pas le VNet.

Le VNet ne référence pas non plus le NSG.

---

# 16. Première possibilité de parallélisme

Après la création du Resource Group :

```text
            Resource Group
            /            \
           ▼              ▼
         VNet            NSG
```

Terraform n'a pas besoin d'attendre que le VNet soit terminé pour commencer nécessairement le NSG.

Ces deux branches peuvent être traitées indépendamment lorsque le provider et les API le permettent.

---

# 17. L'association crée deux dépendances

Nous avons :

```hcl
resource "azurerm_subnet_network_security_group_association" "app" {
  subnet_id = azurerm_subnet.app.id

  network_security_group_id = azurerm_network_security_group.app.id
}
```

Terraform détecte deux références :

```text
azurerm_subnet.app.id
```

et :

```text
azurerm_network_security_group.app.id
```

L'association dépend donc des deux ressources.

---

# 18. Graphe complet

Conceptuellement :

```text
                       Resource Group
                      /              \
                     /                \
                    ▼                  ▼
                  VNet                NSG
                 /    \                │
                /      \               │
               ▼        ▼              │
        Subnet APP   Subnet PE         │
               │                       │
               └───────────┬───────────┘
                           ▼
                    NSG Association
```

Ce graphe représente les relations importantes entre nos ressources.

---

# 19. `outputs.tf`

Créer :

```text
outputs.tf
```

avec :

```hcl
output "resource_group_name" {
  type        = string
  description = "Nom du Resource Group."
  value       = azurerm_resource_group.main.name
}

output "virtual_network_name" {
  type        = string
  description = "Nom du Virtual Network."
  value       = azurerm_virtual_network.main.name
}

output "virtual_network_id" {
  type        = string
  description = "Identifiant Azure du Virtual Network."
  value       = azurerm_virtual_network.main.id
}

output "app_subnet_id" {
  type        = string
  description = "Identifiant du Subnet application."
  value       = azurerm_subnet.app.id
}

output "private_endpoints_subnet_id" {
  type        = string
  description = "Identifiant du Subnet réservé aux Private Endpoints."
  value       = azurerm_subnet.private_endpoints.id
}

output "app_nsg_id" {
  type        = string
  description = "Identifiant du Network Security Group du Subnet application."
  value       = azurerm_network_security_group.app.id
}

output "network_summary" {
  description = "Résumé de l'infrastructure réseau créée dans le Lab."

  type = object({
    resource_group = string
    virtual_network = object({
      name           = string
      address_spaces = list(string)
    })
    app_subnet = object({
      name     = string
      prefixes = list(string)
    })
    private_endpoints_subnet = object({
      name     = string
      prefixes = list(string)
    })
    app_nsg = string
  })

  value = {
    resource_group = azurerm_resource_group.main.name

    virtual_network = {
      name           = azurerm_virtual_network.main.name
      address_spaces = tolist(azurerm_virtual_network.main.address_space)
    }

    app_subnet = {
      name     = azurerm_subnet.app.name
      prefixes = tolist(azurerm_subnet.app.address_prefixes)
    }

    private_endpoints_subnet = {
      name     = azurerm_subnet.private_endpoints.name
      prefixes = tolist(azurerm_subnet.private_endpoints.address_prefixes)
    }

    app_nsg = azurerm_network_security_group.app.name
  }
}
```

---

# 20. `terraform.tfvars.example`

Créer :

```text
terraform.tfvars.example
```

avec :

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

Créer le fichier local :

```powershell
Copy-Item `
  ".\terraform.tfvars.example" `
  ".\terraform.tfvars"
```

---

# 21. Initialiser

Exécuter :

```powershell
terraform fmt -recursive
terraform init
terraform validate
```

Puis :

```powershell
terraform fmt -check -recursive
```

Résultat attendu pour la validation :

```text
Success! The configuration is valid.
```

---

# 22. Générer le premier graphe Terraform

Terraform possède une commande dédiée :

```powershell
terraform graph
```

La sortie n'est pas destinée principalement à être lue directement par un humain.

Terraform produit une représentation au format DOT.

Exécuter :

```powershell
terraform graph > dependency-graph.dot
```

Nous obtenons :

```text
dependency-graph.dot
```

---

# 23. Pourquoi le format DOT ?

Terraform ne cherche pas à produire lui-même une belle image.

Il produit une description du graphe que des outils spécialisés peuvent ensuite afficher.

Nous allons utiliser Graphviz.

---

# 24. Installer Graphviz sous Windows

Si Graphviz n'est pas déjà installé :

```powershell
winget install --exact --id Graphviz.Graphviz
```

Fermer puis rouvrir PowerShell si nécessaire.

Vérifier :

```powershell
dot -V
```

---

# 25. Générer le visuel du LAB-04

Utiliser directement :

```powershell
terraform graph |
  dot -Tpng -o terraform-lab-04-dependency-graph.png
```

Le fichier produit porte directement son nom final :

```text
terraform-lab-04-dependency-graph.png
```

Il pourra être placé dans :

```text
static/images/terraform/
```

du site Hugo.

Dans l'article :

```markdown
![Graphe de dépendances Terraform du LAB-04](/images/terraform/terraform-lab-04-dependency-graph.png)
```

![Graphe de dépendances Terraform du LAB-04](/images/terraform/terraform-lab-04-dependency-graph.png)

Le graphe généré automatiquement par Terraform peut contenir davantage de détails que le schéma conceptuel présenté au début du Lab.

C'est normal.

---

# 26. Ce que montre `terraform graph`

Par défaut, le graphe simplifié permet principalement d'observer les relations entre les ressources et data sources de la configuration.

Nous devons retrouver les relations principales :

```text
Resource Group
      ↓
VNet
      ↓
Subnets
```

ainsi que :

```text
Resource Group
      ↓
NSG
```

et :

```text
Subnet APP ───────┐
                  ▼
             Association
                  ▲
NSG APP ──────────┘
```

---

# 27. Le graphe ne représente pas uniquement un ordre

Il est tentant de penser :

```text
A
↓
B
↓
C
```

comme une simple liste d'étapes.

Mais Terraform utilise réellement les dépendances pour déterminer quelles opérations peuvent devenir disponibles lorsque leurs prérequis sont satisfaits.

Le graphe influence donc :

- le planning ;
- l'exécution ;
- la parallélisation ;
- la destruction.

---

# 28. Générer le plan

Exécuter :

```powershell
terraform plan -out="lab-04.tfplan"
```

Résultat attendu :

```text
Plan: 6 to add, 0 to change, 0 to destroy.
```

Les six ressources sont :

```text
azurerm_resource_group.main

azurerm_virtual_network.main

azurerm_subnet.app

azurerm_subnet.private_endpoints

azurerm_network_security_group.app

azurerm_subnet_network_security_group_association.app
```

---

# 29. Lire le plan

Exécuter :

```powershell
terraform show ".\lab-04.tfplan"
```

La présence de six ressources ne signifie pas que Terraform va simplement exécuter six commandes dans l'ordre du fichier.

Terraform utilise le graphe construit à partir des dépendances.

---

# 30. Graphe associé à un plan

Terraform peut également produire un graphe plus détaillé lié à une opération de plan.

Par exemple :

```powershell
terraform graph -type=plan > plan-graph.dot
```

Ce graphe contient davantage d'éléments internes que la représentation simplifiée.

Il est surtout utile pour du troubleshooting avancé.

Pour l'apprentissage courant, le graphe par défaut reste plus facile à lire.

---

# 31. Appliquer le plan

Après vérification :

```powershell
terraform apply ".\lab-04.tfplan"
```

Résultat attendu :

```text
Apply complete! Resources: 6 added, 0 changed, 0 destroyed.
```

Pendant l'exécution, observer attentivement les lignes :

```text
Creating...
Creation complete...
```

Nous pouvons parfois constater que plusieurs ressources sont en cours de création simultanément.

---

# 32. Pourquoi Terraform parallélise ?

Prenons :

```text
VNet
```

et :

```text
NSG
```

Les deux dépendent du Resource Group.

Mais aucun ne dépend directement de l'autre.

Une fois le Resource Group disponible :

```text
                    ┌────► VNet
Resource Group ─────┤
                    └────► NSG
```

Terraform peut potentiellement traiter les deux branches parallèlement.

---

# 33. Le parallélisme n'est pas garanti pour tout

Le fait que deux ressources soient indépendantes dans le graphe signifie qu'elles peuvent être éligibles à une exécution parallèle.

Cela ne signifie pas nécessairement qu'Azure terminera physiquement les deux opérations exactement au même moment.

Les API, providers, limites du service et temps de traitement peuvent intervenir.

Le point important est que Terraform n'impose pas une dépendance qui n'existe pas.

---

# 34. Inspecter le state

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

L'ordre affiché par :

```powershell
terraform state list
```

n'est pas l'ordre de création de l'infrastructure.

Le state n'est pas un script d'exécution.

---

# 35. Vérifier Azure

Récupérer :

```powershell
$rgName   = terraform output -raw resource_group_name
$vnetName = terraform output -raw virtual_network_name
```

Vérifier le VNet :

```powershell
az network vnet show `
  --resource-group $rgName `
  --name $vnetName `
  --output table
```

Lister les subnets :

```powershell
az network vnet subnet list `
  --resource-group $rgName `
  --vnet-name $vnetName `
  --output table
```

---

# 36. Vérifier l'association NSG

Récupérer le Subnet application :

```powershell
az network vnet subnet show `
  --resource-group $rgName `
  --vnet-name $vnetName `
  --name "snet-app-dev-fce-001" `
  --query "{Subnet:name,NSG:networkSecurityGroup.id}" `
  --output json
```

Nous devons retrouver l'identifiant du NSG associé.

---

# 37. Dépendance implicite ou explicite ?

Jusqu'ici, nous n'avons utilisé aucun :

```hcl
depends_on
```

Et pourtant Terraform connaît toutes les relations nécessaires.

Pourquoi ?

Grâce aux références.

---

# 38. Exemple de dépendance implicite

```hcl
resource "azurerm_virtual_network" "main" {
  resource_group_name = azurerm_resource_group.main.name
}
```

La relation est visible dans la donnée utilisée.

Terraform n'a besoin d'aucune instruction supplémentaire.

---

# 39. Mauvaise utilisation de `depends_on`

Nous pourrions écrire :

```hcl
resource "azurerm_virtual_network" "main" {
  name                = local.resource_names.virtual_network
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  address_space       = var.network.vnet_address_spaces

  depends_on = [
    azurerm_resource_group.main
  ]

  tags = local.common_tags
}
```

Cela serait redondant.

Terraform connaît déjà cette dépendance grâce à :

```hcl
azurerm_resource_group.main.location
```

et :

```hcl
azurerm_resource_group.main.name
```

Le `depends_on` n'apporte rien ici.

---

# 40. Règle importante

Préférer :

```hcl
resource_group_name = azurerm_resource_group.main.name
```

à une dépendance artificielle.

Autrement dit :

> si la relation peut être exprimée naturellement par une référence de données, utiliser cette référence.

`depends_on` doit principalement servir aux dépendances que Terraform ne peut pas déduire lui-même.

---

# 41. Qu'est-ce qu'une dépendance cachée ?

Imaginons :

```text
Resource A
```

qui provoque un comportement ou un effet nécessaire à :

```text
Resource B
```

mais B n'utilise aucune propriété de A dans sa configuration.

Terraform ne voit alors aucune référence.

Conceptuellement :

```text
A
│
│ dépendance réelle
▼
B
```

mais dans le code :

```text
A       B

aucune référence
```

Terraform pourrait les considérer comme indépendantes.

C'est ce genre de situation qui peut justifier :

```hcl
depends_on = [
  resource_a.example
]
```

---

# 42. Expérience volontaire avec `depends_on`

Nous allons maintenant créer volontairement une dépendance inutile.

Dans :

```hcl
resource "azurerm_subnet" "private_endpoints"
```

ajouter temporairement :

```hcl
depends_on = [
  azurerm_network_security_group.app
]
```

Le bloc devient :

```hcl
resource "azurerm_subnet" "private_endpoints" {
  name = local.resource_names.subnet_private_endpoints

  resource_group_name  = azurerm_resource_group.main.name
  virtual_network_name = azurerm_virtual_network.main.name

  address_prefixes = var.network.pe_subnet_prefixes

  depends_on = [
    azurerm_network_security_group.app
  ]
}
```

---

# 43. Générer le nouveau graphe

Exécuter :

```powershell
terraform graph |
  dot -Tpng -o terraform-lab-04-dependency-graph-explicit.png
```

Comparer avec :

```text
terraform-lab-04-dependency-graph.png
```

Nous avons ajouté une relation :

```text
NSG
 │
 ▼
Subnet PE
```

qui n'existait pas réellement dans notre architecture.

---

# 44. Pourquoi cette dépendance est mauvaise ?

Le Subnet Private Endpoints n'a pas besoin du NSG application.

Nous avons donc artificiellement ajouté :

```text
NSG
 ↓
Subnet PE
```

Terraform doit maintenant respecter cette relation.

Cela réduit inutilement ses possibilités d'exécution parallèle.

Sur six ressources, l'impact est négligeable.

Sur une plateforme avec plusieurs centaines de ressources, une accumulation de dépendances artificielles peut compliquer fortement le graphe.

---

# 45. Retirer la dépendance artificielle

Supprimer :

```hcl
depends_on = [
  azurerm_network_security_group.app
]
```

Puis :

```powershell
terraform fmt
terraform validate
terraform plan
```

Résultat attendu :

```text
No changes.
Your infrastructure matches the configuration.
```

Nous avons changé le graphe Terraform sans changer l'état Azure souhaité.

---

# 46. `depends_on` est un méta-argument

`depends_on` ne fait pas partie du schéma propre à :

```text
azurerm_subnet
```

ou :

```text
azurerm_virtual_network
```

Il appartient au langage Terraform.

Nous parlons de **méta-argument**.

D'autres méta-arguments seront étudiés plus tard, notamment :

```text
count
for_each
lifecycle
provider
```

---

# 47. Pourquoi éviter `depends_on` systématique ?

Une configuration comme :

```hcl
resource "example_b" "main" {
  depends_on = [
    example_a.main
  ]
}
```

indique uniquement :

> B doit attendre A.

Elle n'indique pas :

> quelle information de A est réellement utilisée par B.

Une référence directe transmet davantage d'information à Terraform.

---

# 48. Dépendance transitive

Notre architecture contient :

```text
Resource Group
      │
      ▼
     VNet
      │
      ▼
  Subnet APP
      │
      ▼
 Association
```

L'association ne référence pas directement le VNet.

Elle référence :

```text
Subnet APP
```

qui lui-même dépend du VNet.

Terraform peut donc respecter la chaîne complète.

Nous parlons ici d'une dépendance transitive dans le graphe.

---

# 49. Le graphe est acyclique

Pour pouvoir déterminer un ordre cohérent, Terraform doit disposer d'un graphe sans cycle de dépendances.

Un cycle serait :

```text
A
│
▼
B
│
▼
C
│
└────► A
```

Il devient impossible de déterminer quelle ressource doit être disponible en premier.

---

# 50. Créer volontairement un cycle

Nous allons provoquer une erreur pédagogique.

Dans :

```hcl
resource "azurerm_resource_group" "main"
```

ajouter temporairement :

```hcl
depends_on = [
  azurerm_virtual_network.main
]
```

Le bloc devient :

```hcl
resource "azurerm_resource_group" "main" {
  name     = local.resource_names.resource_group
  location = local.location

  depends_on = [
    azurerm_virtual_network.main
  ]

  tags = local.common_tags
}
```

---

# 51. Observer le problème

Nous avons maintenant :

```text
Resource Group
      │
      │ dépend explicitement de
      ▼
     VNet
      │
      │ dépend implicitement de
      ▼
Resource Group
```

Donc :

```text
Resource Group
      ↓
     VNet
      ↓
Resource Group
```

C'est un cycle.

---

# 52. Détecter le cycle

Exécuter :

```powershell
terraform validate
```

Terraform doit signaler une erreur de type :

```text
Cycle
```

avec les ressources concernées.

Terraform ne peut pas résoudre correctement cette configuration.

---

# 53. Pourquoi Terraform bloque ?

Terraform ne peut pas :

```text
créer le Resource Group
```

avant le VNet puisque nous avons demandé que le Resource Group attende le VNet.

Mais le VNet ne peut pas être créé avant son Resource Group.

Nous avons créé une contradiction logique.

---

# 54. Corriger le cycle

Supprimer immédiatement :

```hcl
depends_on = [
  azurerm_virtual_network.main
]
```

Puis :

```powershell
terraform fmt
terraform validate
terraform plan
```

Résultat attendu :

```text
Success! The configuration is valid.
```

puis :

```text
No changes.
```

---

# 55. Troubleshooting d'un cycle

Lorsque Terraform retourne :

```text
Cycle
```

ne pas commencer par :

- supprimer le state ;
- relancer `apply` plusieurs fois ;
- ajouter davantage de `depends_on` ;
- modifier Azure manuellement.

Commencer par analyser les références entre les ressources concernées.

Le problème est généralement dans la logique du graphe.

---

# 56. Méthode de diagnostic d'un cycle

Rechercher :

```text
A référence B ?

B référence A ?

Existe-t-il un depends_on ?

Existe-t-il une troisième ressource reliant les deux ?

Une relation pourrait-elle être supprimée ?

La responsabilité des ressources est-elle correctement séparée ?
```

Un cycle est souvent le symptôme d'une mauvaise relation entre composants.

---

# 57. Visualiser un graphe plus détaillé

Pour du troubleshooting avancé :

```powershell
terraform graph -type=plan |
  dot -Tpng -o terraform-lab-04-plan-graph.png
```

Le résultat est généralement beaucoup plus chargé.

C'est normal.

Terraform y expose davantage de détails liés à l'opération.

---

# 58. `-draw-cycles`

Terraform propose également une option destinée à aider au diagnostic :

```text
-draw-cycles
```

Elle peut être utilisée avec certains graphes d'opérations détaillés pour mettre en évidence des relations impliquées dans un cycle.

Cette fonctionnalité est surtout destinée au troubleshooting et non à la documentation courante d'une architecture.

---

# 59. Parallélisme Terraform

Le graphe permet à Terraform de traiter plusieurs nœuds lorsque leurs dépendances sont satisfaites.

Conceptuellement :

```text
                  ┌────► VNet ────► Subnet APP
                  │          \
Resource Group ───┤           └───► Subnet PE
                  │
                  └────► NSG
```

Le VNet et le NSG peuvent progresser indépendamment.

Les deux subnets attendent le VNet.

L'association attend :

```text
Subnet APP
+
NSG
```

---

# 60. Pourquoi le parallélisme est important ?

Une infrastructure Azure réelle peut contenir :

```text
100
200
500
```

ressources ou davantage.

Si toutes les opérations devaient être exécutées strictement les unes après les autres, les déploiements pourraient devenir inutilement longs.

Le graphe permet à Terraform de préserver les dépendances tout en exploitant les branches indépendantes.

---

# 61. `-parallelism`

Terraform permet de contrôler le niveau maximal de parallélisme avec une option comme :

```powershell
terraform apply `
  -parallelism=5
```

Mais modifier ce paramètre n'est pas une optimisation à appliquer systématiquement.

Le comportement par défaut convient généralement.

Un changement de parallélisme doit répondre à un problème identifié et mesuré.

---

# 62. Réduire le parallélisme pour corriger une architecture ?

Non.

Si Terraform échoue parce que notre graphe ne représente pas correctement les dépendances, réduire :

```text
parallelism
```

peut parfois masquer temporairement le problème.

Mais cela ne corrige pas la modélisation.

La bonne question reste :

> la dépendance nécessaire est-elle correctement exprimée ?

---

# 63. Dépendances et destruction

Lors de la création :

```text
Resource Group
      ↓
     VNet
      ↓
   Subnet
```

Terraform doit respecter cette direction logique.

Lors de la destruction, il doit éviter de supprimer un parent nécessaire avant ses dépendants.

Conceptuellement :

```text
Subnet
   ↓
 VNet
   ↓
Resource Group
```

Le graphe est donc également essentiel lors d'un `destroy`.

---

# 64. Préparer le plan de destruction sans l'appliquer

Exécuter :

```powershell
terraform plan `
  -destroy `
  -out="lab-04-destroy.tfplan"
```

Résultat attendu :

```text
Plan: 0 to add, 0 to change, 6 to destroy.
```

Ne pas l'appliquer immédiatement.

---

# 65. Lire la destruction

Exécuter :

```powershell
terraform show ".\lab-04-destroy.tfplan"
```

Nous devons retrouver :

```text
NSG Association
Subnet APP
Subnet PE
NSG
VNet
Resource Group
```

Le plan exprime les ressources à supprimer.

Terraform utilisera le graphe pour déterminer les dépendances pendant l'exécution.

---

# 66. Ne pas confondre affichage et séquence exacte

L'ordre dans lequel les ressources apparaissent visuellement dans :

```text
terraform plan
```

ou :

```text
terraform state list
```

ne doit pas être utilisé comme preuve de leur ordre exact d'exécution.

Le graphe de dépendances reste la référence conceptuelle.

---

# 67. Dépendance et state

Le state conserve les informations sur les ressources gérées.

Mais le graphe n'est pas simplement une liste stockée dans :

```text
terraform.tfstate
```

Terraform reconstruit les relations en fonction notamment de la configuration, des références, des dépendances explicites et des informations disponibles lors de l'opération.

Nous étudierons beaucoup plus profondément ce sujet dans le Lab consacré au state.

---

# 68. Supprimer une référence peut changer le graphe

Considérons :

```hcl
resource_group_name = azurerm_resource_group.main.name
```

Si nous remplaçons cette expression par :

```hcl
resource_group_name = local.resource_names.resource_group
```

la valeur finale peut être exactement la même :

```text
rg-mla-dev-fce-001
```

Mais nous avons supprimé une référence directe à :

```text
azurerm_resource_group.main
```

Le résultat textuel est identique.

Le graphe exprimé par le code peut être différent.

---

# 69. Valeur identique ne signifie pas relation identique

Comparer :

```hcl
resource_group_name = local.resource_names.resource_group
```

avec :

```hcl
resource_group_name = azurerm_resource_group.main.name
```

Les deux peuvent produire :

```text
rg-mla-dev-fce-001
```

Mais la seconde expression communique davantage à Terraform :

```text
cette ressource consomme une information
provenant du Resource Group géré ici
```

C'est une raison importante pour privilégier les références naturelles.

---

# 70. Dépendances et refactoring

Un refactoring Terraform peut modifier le graphe sans modifier immédiatement les valeurs visibles.

Après un changement structurel :

```powershell
terraform validate
terraform graph
terraform plan
```

sont donc complémentaires.

Le plan nous indique l'impact infrastructure.

Le graphe peut nous aider à comprendre les relations.

---

# 71. Vérifier l'idempotence

Nous avons retiré toutes les modifications expérimentales.

Exécuter :

```powershell
terraform plan
```

Résultat attendu :

```text
No changes.
Your infrastructure matches the configuration.
```

C'est notre état propre avant versionnement.

---

# 72. Code final à conserver

Le `main.tf` final doit être exactement :

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

Aucun `depends_on` n'est nécessaire dans cette configuration.

Toutes les dépendances nécessaires sont exprimées naturellement par les références.

---

# 73. Versionner LAB-04

Revenir à la racine :

```powershell
Set-Location "C:\Labs\MonLabAzure\terraform-azure-platform"
```

Vérifier :

```powershell
git status
```

Ajouter :

```powershell
git add .
```

Inspecter :

```powershell
git diff --cached --name-only
```

Nous ne devons pas versionner :

```text
terraform.tfstate
terraform.tfstate.backup
terraform.tfvars
*.tfplan
.terraform/
```

Le fichier :

```text
terraform-lab-04-dependency-graph.png
```

peut être utilisé pour l'article MonLabAzure, mais il n'a pas besoin d'être conservé dans le repository Terraform si les sources du blog sont gérées séparément.

---

# 74. Commit

Créer :

```powershell
git commit -m "feat(terraform): add lab 04 dependency graph"
```

Puis :

```powershell
git status
```

---

# 75. Nettoyage Azure

Revenir :

```powershell
Set-Location "C:\Labs\MonLabAzure\terraform-azure-platform\lab-04"
```

Si le plan de destruction précédent est ancien, le régénérer.

Ne pas réutiliser automatiquement un ancien plan après avoir modifié la configuration ou l'infrastructure.

Exécuter :

```powershell
terraform plan `
  -destroy `
  -out="lab-04-destroy.tfplan"
```

Lire :

```powershell
terraform show ".\lab-04-destroy.tfplan"
```

Vérifier :

```text
Plan: 0 to add, 0 to change, 6 to destroy.
```

---

# 76. Appliquer la destruction

Après vérification :

```powershell
terraform apply ".\lab-04-destroy.tfplan"
```

Résultat attendu :

```text
Apply complete! Resources: 0 added, 0 changed, 6 destroyed.
```

---

# 77. Vérifier Azure

Exécuter :

```powershell
az group exists `
  --name "rg-mla-dev-fce-001"
```

Résultat attendu :

```text
false
```

Puis :

```powershell
terraform state list
```

Aucune des six ressources ne doit rester dans le state.

---

# 78. Ce qu'il faut retenir

Terraform ne lit pas notre configuration comme un script séquentiel.

Le modèle est :

```text
Configuration
      │
      ▼
Références
      │
      ▼
Dépendances
      │
      ▼
Graphe
      │
      ▼
Plan
      │
      ▼
Exécution
```

---

## Dépendance implicite

Une référence comme :

```hcl
resource_group_name = azurerm_resource_group.main.name
```

exprime naturellement une dépendance.

C'est l'approche à privilégier.

---

## Dépendance explicite

```hcl
depends_on = [
  ...
]
```

sert lorsque la dépendance réelle ne peut pas être déduite à partir des données utilisées.

Il ne doit pas être ajouté systématiquement.

---

## Parallélisme

Deux branches indépendantes du graphe peuvent progresser sans attendre artificiellement l'une après l'autre.

---

## Cycle

Une relation comme :

```text
A → B → A
```

est impossible à résoudre correctement.

Terraform détecte ce type de contradiction.

---

## Destruction

Le graphe permet également de supprimer les ressources dans un ordre compatible avec leurs relations.

---

# 79. Ce qu'un ingénieur Terraform doit savoir expliquer

> Comment Terraform sait-il qu'un Subnet doit être créé après le VNet ?

Une réponse solide serait :

> Terraform construit un graphe de dépendances à partir de la configuration. Dans mon code, le Subnet utilise `azurerm_virtual_network.main.name`. Cette référence crée une dépendance implicite entre le Subnet et le VNet. Terraform sait donc que le VNet doit être disponible avant de traiter le Subnet. Je n'ajoute pas de `depends_on` parce que la dépendance est déjà exprimée naturellement par la référence.

---

# 80. Question fréquente : Terraform exécute-t-il les fichiers dans l'ordre ?

Non.

Un fichier nommé :

```text
01-network.tf
```

n'est pas automatiquement exécuté avant :

```text
02-application.tf
```

Les noms de fichiers organisent surtout le code pour les humains.

Terraform analyse la configuration du module et construit son graphe de dépendances.

---

# 81. Question fréquente : faut-il ajouter `depends_on` partout ?

Non.

Une utilisation excessive :

```text
resource A
↓
depends_on
↓
resource B
↓
depends_on
↓
resource C
```

peut artificiellement sérialiser une architecture qui pourrait autrement être traitée en parallèle.

Elle rend aussi les relations moins naturelles et plus difficiles à maintenir.

---

# 82. Question fréquente : réduire `parallelism` résout-il les problèmes de dépendances ?

Pas réellement.

Si une dépendance nécessaire manque, la priorité est de corriger :

```text
le graphe
```

et non de forcer Terraform à exécuter presque toutes les ressources une par une.

---

# 83. Validation du LAB-04

Le Lab est terminé lorsque les points suivants sont compris et testés :

```text
[ ] Resource Group créé

[ ] Virtual Network créé

[ ] Deux Subnets créés

[ ] Network Security Group créé

[ ] NSG associé au Subnet application

[ ] terraform state list affiche six ressources

[ ] Comprendre une dépendance implicite

[ ] Comprendre depends_on

[ ] Savoir quand ne pas utiliser depends_on

[ ] Identifier les branches parallèles du graphe

[ ] Générer terraform graph

[ ] Installer Graphviz

[ ] Générer terraform-lab-04-dependency-graph.png

[ ] Observer l'impact d'un depends_on artificiel

[ ] Retirer la dépendance artificielle

[ ] Créer volontairement un cycle

[ ] Observer l'erreur Cycle

[ ] Corriger le cycle

[ ] Obtenir terraform validate sans erreur

[ ] Obtenir terraform plan sans changement

[ ] Comprendre l'ordre de destruction

[ ] Détruire proprement les six ressources
```

---

# 84. Progression

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
HCL et modèle déclaratif
     │
     ▼
LAB-03
Variables, locals, outputs et types
     │
     ▼
LAB-04
Dépendances
     │
     ├── références
     ├── graphe
     ├── depends_on
     ├── parallélisme
     └── cycles
```

Nous commençons maintenant à comprendre non seulement **ce que Terraform fait**, mais aussi **comment Terraform raisonne sur les relations entre les ressources**.

---

# 85. Suite

Jusqu'ici, nous avons utilisé Terraform State sans l'étudier en profondeur.

Nous avons simplement observé :

```text
terraform.tfstate
```

et exécuté quelques commandes :

```powershell
terraform state list
terraform state show
```

Le prochain Lab va entrer dans l'un des sujets les plus importants de Terraform.

Nous allons étudier :

```text
configuration
      │
      ▼
resource address
      │
      ▼
Terraform State
      │
      ▼
Azure Resource ID
      │
      ▼
ressource réelle
```

Nous provoquerons également plusieurs situations :

```text
drift
suppression manuelle
modification Azure
state différent du réel
refresh
state backup
perte de state
```

Le prochain article sera :

**LAB-05 - Comprendre profondément le Terraform State**

---

## Références officielles

Les mécanismes étudiés dans ce Lab sont documentés principalement dans :

- HashiCorp Developer - Dependency Graph ;
- HashiCorp Developer - `depends_on` Meta-Argument ;
- HashiCorp Developer - Terraform Graph Command ;
- HashiCorp Developer - Resource References ;
- HashiCorp Developer - Terraform Plan ;
- HashiCorp Developer - Troubleshooting Terraform ;
- Terraform Registry - HashiCorp AzureRM Provider ;
- Terraform Registry - `azurerm_subnet` ;
- Terraform Registry - `azurerm_network_security_group` ;
- Terraform Registry - `azurerm_subnet_network_security_group_association` ;
- Microsoft Learn - Azure Virtual Network ;
- Microsoft Learn - Network Security Groups.

Les comportements présentés doivent être vérifiés avec les versions de Terraform et du provider AzureRM réellement verrouillées par le projet avant toute utilisation en production.