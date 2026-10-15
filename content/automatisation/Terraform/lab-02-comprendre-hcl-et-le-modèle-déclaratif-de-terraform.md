---

title: "LAB-02 - Comprendre HCL et le modèle déclaratif de Terraform"
date: 2026-10-06
draft: false
description: "Comprendre le langage Terraform, HCL, les blocs, arguments, expressions, types, références et le modèle déclaratif à travers un déploiement Azure."
summary: "Manipuler HCL et comprendre comment Terraform transforme une configuration déclarative en graphe de ressources et en actions sur Microsoft Azure."
slug: "terraform-azure-lab-02-hcl-modele-declaratif"
ccategories:
  - "Automatisation"
tags:
  - "Terraform"
  - "Azure"
  - "Infrastructure as Code"
  - "AzureRM"
  - "HCL"
  - "Virtual Network"
series:
  - "Terraform Azure - De zéro à production"
weight: 3
ShowToc: false
TocOpen: false
---


Dans le LAB-01, nous avons créé notre première ressource Microsoft Azure avec Terraform.

Nous avons déjà utilisé plusieurs éléments sans encore étudier précisément leur fonctionnement :

```text
resource
variable
locals
output
provider
expression
reference
```

Ce Lab va revenir sur ces fondamentaux.

L'objectif n'est pas simplement de savoir écrire du Terraform valide.

Il faut comprendre comment Terraform interprète la configuration et surtout pourquoi l'ordre dans lequel nous écrivons les ressources n'est généralement pas l'ordre dans lequel elles seront créées.

Nous allons également créer notre premier **Virtual Network (VNet)** Azure.

À la fin du Lab, l'infrastructure sera :

```text
Terraform
    │
    ▼
AzureRM Provider
    │
    ▼
Microsoft Azure
    │
    └── Resource Group
          │
          └── Virtual Network
```

Avec :

```text
rg-mla-dev-fce-001
        │
        └── vnet-mla-dev-fce-001
                │
                └── 10.20.0.0/16
```

Le réseau reste volontairement simple.

Les subnets, Network Security Groups (NSG), routes et dépendances plus complexes viendront dans les Labs suivants.

---

# 1. Objectifs du Lab

Nous allons comprendre et manipuler :

- HashiCorp Configuration Language (HCL) ;
- le langage Terraform ;
- les blocs ;
- les labels ;
- les arguments ;
- les expressions ;
- les valeurs littérales ;
- les références ;
- les fonctions ;
- les types primitifs ;
- les collections ;
- les maps ;
- les objets ;
- les expressions conditionnelles ;
- les variables ;
- les locals ;
- les outputs ;
- les dépendances implicites ;
- le modèle déclaratif ;
- `terraform console`.

Nous allons également vérifier concrètement que :

> l'ordre d'écriture des ressources dans les fichiers Terraform ne détermine pas nécessairement leur ordre de création.

Terraform construit ses dépendances à partir des relations exprimées dans la configuration.

---

# 2. Architecture du Lab

Nous allons déployer :

```text
Subscription Azure
│
└── rg-mla-dev-fce-001
     │
     └── vnet-mla-dev-fce-001
          │
          └── Address Space
               10.20.0.0/16
```

Le VNet dépend du Resource Group.

Mais nous n'allons pas écrire :

```hcl
depends_on = [
  azurerm_resource_group.main
]
```

Terraform sera capable de détecter cette relation grâce aux références présentes dans le code.

Nous reviendrons sur ce point.

---

# 3. Prérequis

Le LAB-01 doit avoir été réalisé.

Les commandes suivantes doivent fonctionner :

```powershell
terraform version
az version
git --version
code --version
```

Vérifier l'authentification Azure :

```powershell
az account show
```

Puis la cible :

```powershell
az account show `
  --query "{Subscription:name,SubscriptionId:id,TenantId:tenantId}" `
  --output table
```

Définir le contexte de la session :

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

# 4. Préparer LAB-02

Revenir à la racine du projet :

```powershell
Set-Location "C:\Labs\MonLabAzure\terraform-azure-platform"
```

Créer :

```powershell
New-Item -ItemType Directory -Name "lab-02"
```

Puis :

```powershell
Set-Location ".\lab-02"
code .
```

Nous allons créer :

```text
lab-02/
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

# 5. HCL n'est pas exactement Terraform

Une distinction est utile dès maintenant.

**HashiCorp Configuration Language (HCL)** est un langage de configuration développé par HashiCorp.

Terraform utilise une syntaxe basée sur HCL.

Mais :

```text
HCL
```

et :

```text
Terraform
```

ne sont pas exactement synonymes.

HCL définit notamment une syntaxe permettant d'écrire :

- des blocs ;
- des arguments ;
- des expressions ;
- des valeurs.

Terraform ajoute sa propre sémantique.

Par exemple :

```hcl
resource "azurerm_resource_group" "main" {
  name     = "rg-mla-dev-fce-001"
  location = "francecentral"
}
```

HCL permet de comprendre la structure syntaxique.

Terraform donne une signification particulière au bloc :

```text
resource
```

et le provider AzureRM donne une signification au type :

```text
azurerm_resource_group
```

On peut simplifier ainsi :

```text
HCL
 │
 │ syntaxe
 ▼
Terraform Language
 │
 │ concepts Terraform
 ▼
Provider AzureRM
 │
 │ ressources Azure
 ▼
Microsoft Azure
```

---

# 6. Les trois éléments fondamentaux

La syntaxe Terraform repose principalement sur :

```text
BLOCKS
ARGUMENTS
EXPRESSIONS
```

Prenons :

```hcl
resource "azurerm_resource_group" "main" {
  name     = local.resource_group_name
  location = var.location
}
```

Nous pouvons le décomposer.

---

# 7. Le bloc

Le bloc complet est :

```hcl
resource "azurerm_resource_group" "main" {
  ...
}
```

Sa structure générale est :

```text
<BLOCK TYPE> "<LABEL 1>" "<LABEL 2>" {
    ...
}
```

Ici :

```text
BLOCK TYPE
resource
```

Premier label :

```text
azurerm_resource_group
```

Deuxième label :

```text
main
```

Nous obtenons :

```text
resource
   │
   ├── azurerm_resource_group
   │
   └── main
```

Terraform pourra identifier la ressource avec :

```text
azurerm_resource_group.main
```

---

# 8. Les arguments

À l'intérieur :

```hcl
name     = local.resource_group_name
location = var.location
```

nous avons deux arguments.

Structure :

```text
nom = expression
```

Par exemple :

```hcl
location = var.location
```

À gauche :

```text
location
```

est le nom de l'argument.

À droite :

```text
var.location
```

est une expression.

---

# 9. Les expressions

Une expression produit une valeur.

La forme la plus simple est une valeur littérale :

```hcl
location = "francecentral"
```

Ici :

```text
"francecentral"
```

est directement une chaîne de caractères.

Mais nous pouvons aussi utiliser :

```hcl
location = var.location
```

ou :

```hcl
name = "rg-${var.project_name}-${var.environment}"
```

ou :

```hcl
name = lower(var.project_name)
```

Les expressions rendent la configuration dynamique.

---

# 10. Valeur littérale ou expression

Comparer :

```hcl
environment = "dev"
```

et :

```hcl
environment = var.environment
```

Dans le premier cas :

```text
"dev"
```

est codé directement dans la configuration.

Dans le second :

```text
var.environment
```

référence une valeur fournie ailleurs.

Les deux peuvent produire :

```text
dev
```

mais la manière d'obtenir cette valeur est différente.

---

# 11. Premier fichier - `versions.tf`

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

Nous retrouvons ici plusieurs blocs imbriqués :

```text
terraform
│
└── required_providers
     │
     └── azurerm
```

Puis des arguments :

```text
source
version
required_version
```

---

# 12. `providers.tf`

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

La configuration reste volontairement minimale.

Nous utilisons toujours l'authentification Azure locale préparée avec Azure CLI.

---

# 13. `variables.tf`

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

  validation {
    condition     = can(regex("^[a-z0-9-]+$", var.project_name))
    error_message = "project_name doit contenir uniquement des lettres minuscules, chiffres et tirets."
  }
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
  description = "Région Azure cible."
  type        = string
  default     = "francecentral"

  validation {
    condition = contains(
      ["francecentral", "westeurope"],
      lower(var.location)
    )

    error_message = "location doit être francecentral ou westeurope."
  }
}

variable "owner" {
  description = "Équipe responsable des ressources."
  type        = string
  default     = "cloud-platform"
}

variable "vnet_address_space" {
  description = "Liste des plages réseau affectées au Virtual Network."
  type        = list(string)

  default = [
    "10.20.0.0/16"
  ]
}

variable "enable_training_tag" {
  description = "Ajoute ou non un tag indiquant que l'infrastructure appartient au parcours de formation."
  type        = bool
  default     = true
}
```

Ce fichier permet d'introduire plusieurs types.

---

# 14. Types primitifs

Terraform utilise notamment trois types primitifs.

## String

```hcl
type = string
```

Exemple :

```hcl
"francecentral"
```

---

## Number

Exemple :

```hcl
42
```

ou :

```hcl
3.14
```

Nous ne l'utilisons pas directement dans cette configuration.

---

## Bool

```hcl
type = bool
```

Valeurs possibles :

```hcl
true
false
```

Nous avons :

```hcl
variable "enable_training_tag" {
  type    = bool
  default = true
}
```

---

# 15. Collections

Nous avons également :

```hcl
variable "vnet_address_space" {
  type = list(string)
}
```

Sa valeur :

```hcl
[
  "10.20.0.0/16"
]
```

est une liste de chaînes de caractères.

Une liste peut contenir plusieurs éléments :

```hcl
[
  "10.20.0.0/16",
  "10.21.0.0/16"
]
```

Nous conserverons une seule plage dans ce Lab.

---

# 16. Pourquoi `list(string)` ?

Le provider AzureRM attend une collection pour l'address space d'un VNet.

Nous déclarons donc explicitement :

```hcl
list(string)
```

au lieu d'utiliser une simple chaîne :

```hcl
"10.20.0.0/16"
```

Cela permet aussi à Terraform de détecter une erreur de type plus tôt.

---

# 17. `locals.tf`

Créer :

```text
locals.tf
```

avec :

```hcl
locals {
  environment = lower(var.environment)
  location    = lower(var.location)

  location_codes = {
    francecentral = "fce"
    westeurope    = "weu"
  }

  location_code = local.location_codes[local.location]

  resource_names = {
    resource_group  = "rg-${var.project_name}-${local.environment}-${local.location_code}-001"
    virtual_network = "vnet-${var.project_name}-${local.environment}-${local.location_code}-001"
  }

  mandatory_tags = {
    environment = local.environment
    project     = var.project_name
    managed_by  = "terraform"
    owner       = var.owner
    lab         = "lab-02"
  }

  optional_tags = var.enable_training_tag ? {
    learning_path = "terraform-azure"
  } : {}

  common_tags = merge(
    local.mandatory_tags,
    local.optional_tags
  )
}
```

Ce fichier introduit plusieurs fonctionnalités HCL importantes.

---

# 18. Appeler une fonction

Nous utilisons :

```hcl
lower(var.environment)
```

`lower()` est une fonction Terraform.

Avec :

```text
DEV
```

elle produit :

```text
dev
```

Même principe :

```hcl
lower(var.location)
```

---

# 19. Utiliser une map

Nous avons :

```hcl
location_codes = {
  francecentral = "fce"
  westeurope    = "weu"
}
```

Conceptuellement :

```text
clé              valeur

francecentral -> fce
westeurope    -> weu
```

Nous pouvons ensuite accéder à une valeur :

```hcl
local.location_codes[local.location]
```

Si :

```text
local.location = francecentral
```

Terraform obtient :

```text
fce
```

---

# 20. Utiliser un objet logique

Nous centralisons les noms :

```hcl
resource_names = {
  resource_group  = "rg-${var.project_name}-${local.environment}-${local.location_code}-001"
  virtual_network = "vnet-${var.project_name}-${local.environment}-${local.location_code}-001"
}
```

Nous pourrons ensuite utiliser :

```hcl
local.resource_names.resource_group
```

et :

```hcl
local.resource_names.virtual_network
```

Cette organisation devient intéressante lorsque le nombre de ressources augmente.

---

# 21. Interpolation de chaîne

Cette expression :

```hcl
"vnet-${var.project_name}-${local.environment}-${local.location_code}-001"
```

combine du texte et plusieurs expressions.

Avec :

```text
project_name  = mla
environment   = dev
location_code = fce
```

Terraform produit :

```text
vnet-mla-dev-fce-001
```

---

# 22. Expression conditionnelle

Nous utilisons :

```hcl
optional_tags = var.enable_training_tag ? {
  learning_path = "terraform-azure"
} : {}
```

La structure est :

```text
condition ? valeur_si_vrai : valeur_si_faux
```

Si :

```hcl
enable_training_tag = true
```

nous obtenons :

```hcl
{
  learning_path = "terraform-azure"
}
```

Sinon :

```hcl
{}
```

Une map vide.

---

# 23. Fusionner plusieurs maps

Nous utilisons :

```hcl
common_tags = merge(
  local.mandatory_tags,
  local.optional_tags
)
```

La fonction :

```text
merge()
```

combine plusieurs maps.

Nous obtenons ainsi une seule structure :

```hcl
{
  environment   = "dev"
  project       = "mla"
  managed_by    = "terraform"
  owner         = "cloud-platform"
  lab           = "lab-02"
  learning_path = "terraform-azure"
}
```

Cette approche évite de dupliquer les tags dans chaque ressource.

---

# 24. `main.tf`

Créer :

```text
main.tf
```

avec :

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
  address_space       = var.vnet_address_space

  tags = local.common_tags
}
```

Nous avons maintenant deux ressources.

---

# 25. Lire la première ressource

```hcl
resource "azurerm_resource_group" "main" {
  name     = local.resource_names.resource_group
  location = local.location

  tags = local.common_tags
}
```

Terraform devra créer :

```text
rg-mla-dev-fce-001
```

dans :

```text
francecentral
```

---

# 26. Lire la deuxième ressource

```hcl
resource "azurerm_virtual_network" "main" {
  name                = local.resource_names.virtual_network
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  address_space       = var.vnet_address_space

  tags = local.common_tags
}
```

Le point important n'est pas encore le VNet.

Ce qui nous intéresse est :

```hcl
azurerm_resource_group.main.location
```

et :

```hcl
azurerm_resource_group.main.name
```

Nous faisons référence à une autre ressource Terraform.

---

# 27. Référence à une ressource

La structure :

```text
azurerm_resource_group.main.name
```

peut être lue :

```text
azurerm_resource_group
          │
          ▼
         main
          │
          ▼
         name
```

Autrement dit :

> récupérer l'attribut `name` de la ressource `azurerm_resource_group.main`.

Même principe :

```hcl
azurerm_resource_group.main.location
```

---

# 28. Pourquoi ne pas utiliser directement une variable ?

Nous aurions pu écrire :

```hcl
resource_group_name = local.resource_names.resource_group
```

Mais nous préférons ici :

```hcl
resource_group_name = azurerm_resource_group.main.name
```

Pourquoi ?

Parce que cette référence exprime une relation réelle :

```text
Virtual Network
      │
      │ utilise
      ▼
Resource Group
```

Terraform peut exploiter cette relation pour construire son graphe de dépendances.

---

# 29. Dépendance implicite

Terraform détecte :

```hcl
azurerm_resource_group.main.name
```

dans la configuration du VNet.

Il comprend donc que :

```text
azurerm_virtual_network.main
```

dépend de :

```text
azurerm_resource_group.main
```

Conceptuellement :

```text
azurerm_resource_group.main
            │
            │
            ▼
azurerm_virtual_network.main
```

Terraform sait alors que le Resource Group doit être disponible avant de créer le VNet.

C'est une **dépendance implicite**.

---

# 30. Pourquoi pas `depends_on` ?

Nous pourrions être tentés d'écrire :

```hcl
depends_on = [
  azurerm_resource_group.main
]
```

Mais ce serait inutile ici.

La dépendance existe déjà grâce à :

```hcl
resource_group_name = azurerm_resource_group.main.name
```

Une bonne règle est :

> préférer les références naturelles entre ressources lorsque la dépendance peut être exprimée par les données utilisées.

`depends_on` reste utile lorsque Terraform ne peut pas déduire lui-même une dépendance nécessaire.

Nous l'étudierons plus en détail dans LAB-04.

---

# 31. L'ordre des blocs n'est pas l'ordre de création

Nous avons écrit :

```text
Resource Group
puis
Virtual Network
```

On pourrait donc croire que Terraform exécute les fichiers de haut en bas.

Ce n'est pas le modèle.

Terraform construit les relations entre objets.

Si nous écrivions le VNet avant le Resource Group :

```hcl
resource "azurerm_virtual_network" "main" {
  name                = local.resource_names.virtual_network
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  address_space       = var.vnet_address_space

  tags = local.common_tags
}

resource "azurerm_resource_group" "main" {
  name     = local.resource_names.resource_group
  location = local.location

  tags = local.common_tags
}
```

Terraform comprendrait toujours :

```text
Resource Group
      │
      ▼
Virtual Network
```

grâce aux références.

---

# 32. Même principe entre les fichiers

Terraform ne raisonne pas comme ceci :

```text
01-resource-group.tf
puis
02-network.tf
puis
03-output.tf
```

Les noms de fichiers sont principalement une convention d'organisation pour les humains.

Les fichiers `.tf` d'un même module sont considérés ensemble comme une configuration.

Ainsi :

```text
main.tf
variables.tf
locals.tf
outputs.tf
```

ne représentent pas une séquence d'exécution.

---

# 33. Le modèle impératif

Pour comprendre Terraform, comparons rapidement deux approches.

Une approche impérative pourrait ressembler conceptuellement à :

```text
1. Créer le Resource Group
2. Attendre
3. Créer le VNet
4. Ajouter les tags
5. Vérifier
```

Nous décrivons les étapes.

---

# 34. Le modèle déclaratif

Avec Terraform, nous déclarons :

```text
Je veux :

un Resource Group

et

un Virtual Network dans ce Resource Group.
```

Terraform doit déterminer :

```text
ce qui existe déjà
       │
       +
ce que nous demandons
       │
       +
les dépendances
       │
       ▼
les opérations nécessaires
```

C'est un changement important de modèle mental.

---

# 35. `outputs.tf`

Créer :

```text
outputs.tf
```

avec :

```hcl
output "resource_group_name" {
  description = "Nom du Resource Group."
  value       = azurerm_resource_group.main.name
}

output "resource_group_id" {
  description = "Identifiant Azure du Resource Group."
  value       = azurerm_resource_group.main.id
}

output "virtual_network_name" {
  description = "Nom du Virtual Network."
  value       = azurerm_virtual_network.main.name
}

output "virtual_network_id" {
  description = "Identifiant Azure du Virtual Network."
  value       = azurerm_virtual_network.main.id
}

output "virtual_network_address_space" {
  description = "Address spaces configurés sur le Virtual Network."
  value       = azurerm_virtual_network.main.address_space
}

output "deployment_summary" {
  description = "Résumé du déploiement."
  value = {
    environment            = local.environment
    location               = local.location
    resource_group_name    = azurerm_resource_group.main.name
    virtual_network_name   = azurerm_virtual_network.main.name
    virtual_network_cidrs  = azurerm_virtual_network.main.address_space
    terraform_managed      = true
  }
}
```

Le dernier output est volontairement plus riche.

Il expose un objet structuré.

---

# 36. `terraform.tfvars.example`

Créer :

```text
terraform.tfvars.example
```

avec :

```hcl
project_name = "mla"
environment  = "dev"
location     = "francecentral"
owner        = "cloud-platform"

vnet_address_space = [
  "10.20.0.0/16"
]

enable_training_tag = true
```

Créer ensuite :

```powershell
Copy-Item `
  ".\terraform.tfvars.example" `
  ".\terraform.tfvars"
```

---

# 37. Configuration complète

Notre dossier contient maintenant :

```text
lab-02/
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

# 38. Formater

Exécuter :

```powershell
terraform fmt -recursive
```

Terraform harmonise le format HCL.

Vérifier ensuite :

```powershell
terraform fmt -check -recursive
```

Si tout est correctement formaté, la commande ne doit normalement pas signaler de fichier à reformater.

Cette variante sera particulièrement utile plus tard dans la Continuous Integration / Continuous Delivery (CI/CD).

---

# 39. Initialiser

Exécuter :

```powershell
terraform init
```

Terraform va notamment :

- initialiser le répertoire ;
- installer AzureRM ;
- générer `.terraform.lock.hcl`.

Puis :

```powershell
terraform providers
```

Nous devons retrouver :

```text
registry.terraform.io/hashicorp/azurerm
```

---

# 40. Valider

Exécuter :

```powershell
terraform validate
```

Résultat attendu :

```text
Success! The configuration is valid.
```

Nous allons maintenant étudier HCL sans déployer quoi que ce soit.

---

# 41. Découvrir `terraform console`

Terraform fournit une console permettant d'évaluer des expressions.

Lancer :

```powershell
terraform console
```

Nous obtenons une invite :

```text
>
```

Nous pouvons maintenant tester notre configuration.

---

# 42. Lire une variable

Dans la console :

```hcl
var.environment
```

Résultat attendu :

```text
"dev"
```

Puis :

```hcl
var.project_name
```

Résultat :

```text
"mla"
```

---

# 43. Tester une fonction

Exécuter :

```hcl
upper(var.environment)
```

Résultat :

```text
"DEV"
```

Puis :

```hcl
lower("PROD")
```

Résultat :

```text
"prod"
```

---

# 44. Lire un local

Exécuter :

```hcl
local.resource_names
```

Terraform doit afficher quelque chose proche de :

```hcl
{
  "resource_group" = "rg-mla-dev-fce-001"
  "virtual_network" = "vnet-mla-dev-fce-001"
}
```

Puis :

```hcl
local.resource_names.virtual_network
```

Résultat :

```text
"vnet-mla-dev-fce-001"
```

---

# 45. Manipuler une liste

Tester :

```hcl
var.vnet_address_space
```

Résultat :

```hcl
[
  "10.20.0.0/16",
]
```

Premier élément :

```hcl
var.vnet_address_space[0]
```

Résultat :

```text
"10.20.0.0/16"
```

Compter les éléments :

```hcl
length(var.vnet_address_space)
```

Résultat :

```text
1
```

---

# 46. Tester une expression booléenne

Exécuter :

```hcl
var.environment == "dev"
```

Résultat :

```text
true
```

Puis :

```hcl
var.environment == "prod"
```

Résultat :

```text
false
```

---

# 47. Tester une expression conditionnelle

Exécuter :

```hcl
var.environment == "prod" ? "production" : "non-production"
```

Résultat :

```text
"non-production"
```

Nous retrouvons la forme :

```text
condition ? valeur_vraie : valeur_fausse
```

---

# 48. Tester `contains`

Exécuter :

```hcl
contains(["dev", "qual", "prod"], var.environment)
```

Résultat :

```text
true
```

Cette logique ressemble à celle utilisée dans notre validation de variable.

---

# 49. Tester une map

Exécuter :

```hcl
local.location_codes
```

Puis :

```hcl
local.location_codes["francecentral"]
```

Résultat :

```text
"fce"
```

Enfin :

```hcl
local.location_codes[local.location]
```

Terraform utilise alors dynamiquement la région courante.

---

# 50. Quitter la console

Exécuter :

```text
exit
```

ou :

```text
Ctrl+C
```

`terraform console` est particulièrement utile pour :

- tester une expression ;
- vérifier une fonction ;
- inspecter un local ;
- comprendre une collection ;
- tester une condition ;
- déboguer une transformation de données.

Il évite de lancer un déploiement simplement pour savoir ce qu'une expression retourne.

---

# 51. Générer le plan

Exécuter :

```powershell
terraform plan -out="lab-02.tfplan"
```

Résultat attendu :

```text
Plan: 2 to add, 0 to change, 0 to destroy.
```

Terraform doit proposer :

```text
+ azurerm_resource_group.main

+ azurerm_virtual_network.main
```

---

# 52. Pourquoi deux ressources mais un ordre déterminé ?

Notre configuration contient deux ressources :

```text
Resource Group

Virtual Network
```

Terraform analyse :

```hcl
resource_group_name = azurerm_resource_group.main.name
```

et :

```hcl
location = azurerm_resource_group.main.location
```

Il construit conceptuellement :

```text
azurerm_resource_group.main
            │
            │ dépendance
            ▼
azurerm_virtual_network.main
```

C'est le début du graphe de dépendances Terraform.

Nous approfondirons ce mécanisme dans LAB-04.

---

# 53. Appliquer

Lire d'abord le plan :

```powershell
terraform show ".\lab-02.tfplan"
```

Puis :

```powershell
terraform apply ".\lab-02.tfplan"
```

Résultat attendu :

```text
Apply complete! Resources: 2 added, 0 changed, 0 destroyed.
```

---

# 54. Inspecter le state

Exécuter :

```powershell
terraform state list
```

Résultat :

```text
azurerm_resource_group.main
azurerm_virtual_network.main
```

Inspecter le VNet :

```powershell
terraform state show azurerm_virtual_network.main
```

Nous pouvons désormais observer la différence entre :

```text
arguments fournis
```

et :

```text
attributs calculés ou retournés par Azure
```

---

# 55. Vérification Azure

Récupérer les outputs :

```powershell
$rgName   = terraform output -raw resource_group_name
$vnetName = terraform output -raw virtual_network_name
```

Vérifier :

```powershell
az network vnet show `
  --resource-group $rgName `
  --name $vnetName `
  --output table
```

Vérifier l'address space :

```powershell
az network vnet show `
  --resource-group $rgName `
  --name $vnetName `
  --query "addressSpace.addressPrefixes" `
  --output json
```

Résultat attendu :

```json
[
  "10.20.0.0/16"
]
```

Un VNet représente une brique fondamentale du réseau privé Azure et possède un ou plusieurs address spaces. Dans ce Lab, nous utilisons une plage privée simple :

```text
10.20.0.0/16
```

Nous introduirons les subnets plus tard.

---

# 56. Tester l'idempotence

Sans modifier le code :

```powershell
terraform plan
```

Résultat attendu :

```text
No changes.
Your infrastructure matches the configuration.
```

Terraform ne rejoue pas aveuglément :

```text
CREATE Resource Group

CREATE VNet
```

à chaque exécution.

Il cherche à atteindre l'état déclaré.

---

# 57. Test important - changer l'ordre des blocs

Ouvrir :

```text
main.tf
```

Placer volontairement le bloc VNet avant le Resource Group :

```hcl
resource "azurerm_virtual_network" "main" {
  name                = local.resource_names.virtual_network
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  address_space       = var.vnet_address_space

  tags = local.common_tags
}

resource "azurerm_resource_group" "main" {
  name     = local.resource_names.resource_group
  location = local.location

  tags = local.common_tags
}
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
```

Nous venons de démontrer quelque chose d'essentiel :

> l'ordre physique des blocs n'est pas ce qui définit la dépendance entre ces deux ressources.

La référence :

```hcl
azurerm_resource_group.main.name
```

continue d'exprimer la relation.

---

# 58. Pourquoi cette propriété est importante ?

Dans une infrastructure réelle, nous pouvons avoir :

```text
Resource Groups
VNets
Subnets
Route Tables
NSGs
Private Endpoints
Private DNS Zones
Managed Identities
Key Vaults
Applications
```

Il serait très fragile de demander à l'ingénieur de maintenir manuellement :

```text
étape 1
étape 2
étape 3
...
étape 150
```

Terraform construit un graphe permettant de raisonner sur les dépendances.

---

# 59. Terraform peut paralléliser

Si deux ressources n'ont aucune relation entre elles, Terraform peut généralement les traiter en parallèle lorsque cela est possible.

Exemple conceptuel :

```text
                  ┌── Resource A
Root configuration│
                  └── Resource B
```

Si :

```text
A ne dépend pas de B
```

et :

```text
B ne dépend pas de A
```

Terraform n'a pas forcément besoin de les traiter séquentiellement.

Avec une dépendance :

```text
A
│
▼
B
```

l'ordre devient contraint.

C'est une différence importante avec un script séquentiel classique.

---

# 60. Test du modèle déclaratif

Modifier temporairement dans :

```text
terraform.tfvars
```

la valeur :

```hcl
enable_training_tag = false
```

Puis :

```powershell
terraform plan
```

Terraform doit détecter la suppression du tag :

```text
learning_path
```

Nous n'avons pas écrit :

```text
supprimer le tag learning_path
```

Nous avons déclaré que le tag ne faisait plus partie de l'état souhaité.

Terraform calcule le changement à effectuer.

---

# 61. Remettre la valeur initiale

Avant de continuer :

```hcl
enable_training_tag = true
```

Puis :

```powershell
terraform plan
```

Si le précédent changement n'a pas été appliqué :

```text
No changes.
```

C'est une distinction essentielle :

```text
modifier le code
```

ne modifie pas immédiatement :

```text
Azure
```

Le changement ne sera effectué que lorsqu'un `apply` correspondant sera exécuté.

---

# 62. Configuration et exécution sont séparées

Terraform sépare :

```text
CONFIGURATION
      │
      ▼
PLAN
      │
      ▼
APPLY
```

Modifier :

```text
main.tf
```

ne modifie pas Azure.

Exécuter :

```powershell
terraform plan
```

ne modifie pas non plus l'infrastructure dans le cadre normal de ce workflow.

L'opération réelle intervient lors de l'application.

Cette séparation est fondamentale pour les futures Pull Requests.

---

# 63. Une expression peut devenir trop complexe

HCL est suffisamment puissant pour écrire des expressions complexes.

Ce n'est pas une raison pour le faire systématiquement.

Un code comme :

```hcl
name = lower(
  replace(
    "${var.project_name}-${var.environment}-${var.location}",
    "_",
    "-"
  )
)
```

peut être acceptable.

Mais si les transformations deviennent difficiles à lire, mieux vaut souvent les décomposer avec des locals.

Exemple :

```hcl
locals {
  project     = lower(var.project_name)
  environment = lower(var.environment)
  location    = lower(var.location)

  resource_name = "${local.project}-${local.environment}-${local.location}"
}
```

La lisibilité est une caractéristique importante du code d'infrastructure.

---

# 64. Ne pas transformer Terraform en langage de programmation généraliste

Terraform supporte :

- conditions ;
- boucles ;
- fonctions ;
- collections ;
- transformations ;
- expressions complexes.

Mais l'objectif reste de **décrire une infrastructure**.

Un excès de logique peut rendre la configuration :

- difficile à comprendre ;
- difficile à tester ;
- difficile à maintenir ;
- imprévisible pour les autres ingénieurs.

Une configuration ingénieuse n'est pas nécessairement une bonne configuration.

En production, la lisibilité est souvent préférable à une abstraction excessive.

---

# 65. Arguments et attributs

Dans :

```hcl
resource "azurerm_virtual_network" "main" {
  name          = local.resource_names.virtual_network
  address_space = var.vnet_address_space
}
```

`name` et `address_space` sont des arguments que nous fournissons.

Après création, la ressource expose également différents attributs.

Par exemple :

```hcl
azurerm_virtual_network.main.id
```

L'identifiant :

```text
id
```

est obtenu à partir de la ressource créée.

Nous pouvons donc distinguer conceptuellement :

```text
Argument
    │
    └── valeur fournie à la ressource

Attribut
    │
    └── information disponible sur la ressource
```

La frontière précise dépend du schéma défini par le provider.

---

# 66. Les fichiers `.tf` ne sont pas des scripts

Il est important de sortir de ce modèle :

```text
Terraform lit main.tf
puis variables.tf
puis outputs.tf
```

Ce n'est pas le bon raisonnement.

Il vaut mieux penser :

```text
Tous les fichiers .tf du module
          │
          ▼
Configuration Terraform
          │
          ▼
Analyse des références
          │
          ▼
Graphe
          │
          ▼
Plan
```

Les noms de fichiers améliorent l'organisation pour les humains.

---

# 67. Organisation recommandée à ce stade

Pour nos Labs :

```text
versions.tf
```

versions Terraform/providers.

```text
providers.tf
```

configuration des providers.

```text
variables.tf
```

entrées.

```text
locals.tf
```

calculs et conventions internes.

```text
main.tf
```

ressources principales.

```text
outputs.tf
```

sorties.

Cette organisation n'est pas imposée par Terraform.

C'est une convention de lisibilité.

---

# 68. Sécurité - HCL n'est pas un coffre-fort

Une valeur écrite dans HCL n'est pas protégée simplement parce qu'elle se trouve dans une variable.

À éviter :

```hcl
variable "admin_password" {
  default = "P@ssword123!"
}
```

À éviter également :

```hcl
locals {
  sql_password = "P@ssword123!"
}
```

Terraform n'est pas un gestionnaire de secrets.

Nous utiliserons plus tard :

```text
Microsoft Entra ID
Managed Identity
Azure Key Vault
OIDC
```

et étudierons également les implications du state Terraform.

---

# 69. Attention aux outputs

Un output peut exposer une information.

Exemple à ne pas reproduire :

```hcl
output "password" {
  value = var.password
}
```

Terraform permet de marquer certaines valeurs comme sensibles, mais cela ne doit pas être confondu avec du chiffrement ou avec la suppression de la donnée du state.

Nous approfondirons ce sujet dans le Lab consacré aux secrets.

---

# 70. Erreurs HCL fréquentes

## Guillemets manquants

Incorrect :

```hcl
environment = dev
```

Terraform interprète alors `dev` comme une référence potentielle et non comme la chaîne :

```text
"dev"
```

Correct :

```hcl
environment = "dev"
```

---

## Mauvais type

Variable :

```hcl
type = list(string)
```

Incorrect :

```hcl
vnet_address_space = "10.20.0.0/16"
```

Correct :

```hcl
vnet_address_space = [
  "10.20.0.0/16"
]
```

---

## Mauvaise référence

Incorrect :

```hcl
azurerm_resource_group.rg.name
```

si la ressource est déclarée :

```hcl
resource "azurerm_resource_group" "main"
```

Correct :

```hcl
azurerm_resource_group.main.name
```

---

## Référence à une clé inexistante

Avec :

```hcl
location_codes = {
  francecentral = "fce"
  westeurope    = "weu"
}
```

une tentative :

```hcl
local.location_codes["northeurope"]
```

échouera puisque la clé n'existe pas.

Notre validation de variable permet justement de limiter ce type de situation.

---

# 71. Tester avant de déployer

Lorsque nous modifions des expressions :

```powershell
terraform fmt
terraform validate
terraform console
terraform plan
```

Cette séquence est très utile.

```text
fmt
 │
 └── format

validate
 │
 └── structure

console
 │
 └── expressions

plan
 │
 └── impact réel prévu
```

Ces contrôles n'ont pas le même rôle.

---

# 72. Observer les outputs

Exécuter :

```powershell
terraform output deployment_summary
```

Nous devons obtenir un objet proche de :

```text
{
  "environment" = "dev"
  "location" = "francecentral"
  "resource_group_name" = "rg-mla-dev-fce-001"
  "terraform_managed" = true
  "virtual_network_cidrs" = tolist([
    "10.20.0.0/16",
  ])
  "virtual_network_name" = "vnet-mla-dev-fce-001"
}
```

Nous manipulons désormais des données structurées, et non uniquement des chaînes.

---

# 73. Pourquoi les types deviennent importants en production

Avec quelques ressources, les erreurs restent faciles à comprendre.

Avec des modules et des dizaines de paramètres, un contrat clair devient indispensable.

Nous préférerons progressivement :

```hcl
type = object({
  ...
})
```

ou :

```hcl
type = map(...)
```

à des structures ambiguës.

Le typage devient particulièrement utile pour les interfaces de modules.

---

# 74. Ce que nous ne faisons pas encore

Nous n'introduisons volontairement pas encore :

```text
count
for_each
dynamic blocks
data sources
depends_on avancé
lifecycle
modules
preconditions
postconditions
```

Ces fonctionnalités seraient faciles à ajouter maintenant.

Mais accumuler les fonctionnalités sans comprendre le modèle sous-jacent irait à l'encontre de l'objectif de la série.

---

# 75. Versionner LAB-02

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

Contrôler :

```powershell
git diff --cached --name-only
```

Vérifier notamment que nous ne publions pas :

```text
terraform.tfstate
terraform.tfstate.backup
terraform.tfvars
*.tfplan
.terraform/
```

Puis :

```powershell
git commit -m "feat(terraform): add lab 02 hcl fundamentals"
```

---

# 76. Nettoyage

Revenir :

```powershell
Set-Location "C:\Labs\MonLabAzure\terraform-azure-platform\lab-02"
```

Créer le plan :

```powershell
terraform plan `
  -destroy `
  -out="lab-02-destroy.tfplan"
```

Lire :

```powershell
terraform show ".\lab-02-destroy.tfplan"
```

Résultat attendu :

```text
Plan: 0 to add, 0 to change, 2 to destroy.
```

Nous devons retrouver :

```text
azurerm_virtual_network.main

azurerm_resource_group.main
```

---

# 77. Ordre de destruction

Nous avons appris :

```text
Resource Group
      │
      ▼
Virtual Network
```

Lors de la destruction, Terraform doit prendre en compte les dépendances.

Conceptuellement :

```text
Création

Resource Group
      │
      ▼
Virtual Network
```

mais :

```text
Destruction

Virtual Network
      │
      ▼
Resource Group
```

C'est une autre raison pour laquelle Terraform maintient son graphe de dépendances.

---

# 78. Appliquer la destruction

Après vérification :

```powershell
terraform apply ".\lab-02-destroy.tfplan"
```

Résultat attendu :

```text
Apply complete! Resources: 0 added, 0 changed, 2 destroyed.
```

Vérifier :

```powershell
az group exists `
  --name "rg-mla-dev-fce-001"
```

Résultat :

```text
false
```

---

# 79. Ce qu'il faut retenir

Le point principal du Lab n'est pas :

```text
comment créer un VNet
```

Nous apprendrons le networking Azure plus profondément plus tard.

Le point principal est le modèle Terraform.

---

## HCL fournit la syntaxe

```text
blocks
arguments
expressions
values
```

---

## Terraform donne une signification aux constructions

```text
resource
variable
locals
output
provider
```

---

## Le provider définit les ressources Azure

```text
azurerm_resource_group
azurerm_virtual_network
```

---

## Les références relient les ressources

```text
azurerm_resource_group.main.name
```

crée une relation exploitable par Terraform.

---

## Terraform est déclaratif

Nous décrivons :

```text
ce que nous voulons obtenir
```

plutôt qu'une longue série d'ordres impératifs.

---

## Terraform construit un graphe

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

## L'ordre des fichiers n'est pas l'ordre des opérations

```text
main.tf
variables.tf
outputs.tf
```

sont avant tout des conventions d'organisation.

---

# 80. Ce qu'un ingénieur Terraform doit pouvoir expliquer

Après ce Lab, il faut pouvoir expliquer cette configuration :

```hcl
resource "azurerm_virtual_network" "main" {
  name                = local.resource_names.virtual_network
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  address_space       = var.vnet_address_space

  tags = local.common_tags
}
```

non pas comme :

> ce code crée un VNet.

mais comme :

> Le bloc `resource` déclare une ressource gérée par Terraform. `azurerm_virtual_network` est le type fourni par le provider AzureRM et `main` est son label local. Les valeurs `location` et `resource_group_name` référencent le Resource Group géré par Terraform, ce qui crée également une dépendance implicite. `address_space` reçoit une liste de chaînes depuis une variable typée, tandis que les tags sont centralisés dans des locals.

C'est cette capacité d'explication que nous allons développer pendant toute la série.

---

# 81. Validation du LAB-02

Le Lab est terminé lorsque les étapes suivantes sont validées :

```text
[ ] Comprendre la différence entre HCL et Terraform

[ ] Identifier un block

[ ] Identifier ses labels

[ ] Identifier un argument

[ ] Identifier une expression

[ ] Manipuler string

[ ] Manipuler bool

[ ] Manipuler list(string)

[ ] Manipuler une map

[ ] Utiliser lower()

[ ] Utiliser merge()

[ ] Utiliser une condition ? :

[ ] Utiliser terraform console

[ ] Lire var.environment

[ ] Lire local.resource_names

[ ] Indexer une liste

[ ] Indexer une map

[ ] Resource Group créé

[ ] VNet créé

[ ] terraform state list contient deux ressources

[ ] terraform plan retourne No changes

[ ] Inversion physique des blocs sans changement Terraform

[ ] Compréhension de la dépendance implicite

[ ] Infrastructure détruite proprement
```

---

# 82. Ce que nous avons maintenant

Notre progression est :

```text
LAB-00
Préparation du poste
       │
       ▼
LAB-01
Premier déploiement Terraform
       │
       ▼
LAB-02
HCL + modèle déclaratif
       │
       ▼
Configuration
       │
       ▼
Expressions
       │
       ▼
Références
       │
       ▼
Dépendances
       │
       ▼
Graphe Terraform
```

Les fondations commencent maintenant à être suffisamment solides pour rendre notre configuration plus configurable.

---

# 83. Suite

Dans le prochain Lab, nous allons nous concentrer spécifiquement sur la manière de transmettre, transformer et exposer les données dans Terraform.

Nous travaillerons notamment avec :

```text
variables
tfvars
locals
outputs
types
objects
maps
lists
validations
sensitive
null
```

Nous commencerons aussi à réfléchir à une question importante :

> qu'est-ce qui doit être configurable et qu'est-ce qui ne devrait pas l'être ?

Le prochain article sera :

**LAB-03 - Variables, locals, outputs et typage Terraform**

---

## Références officielles

Les mécanismes étudiés dans ce Lab sont documentés principalement dans :

- HashiCorp Developer - Terraform Language Overview ;
- HashiCorp Developer - Configuration Syntax ;
- HashiCorp Developer - Expressions ;
- HashiCorp Developer - Types and Values ;
- HashiCorp Developer - References to Values ;
- HashiCorp Developer - Resource Configuration ;
- HashiCorp Developer - Dependency Graph ;
- Terraform Registry - HashiCorp AzureRM Provider ;
- Terraform Registry - `azurerm_virtual_network` ;
- Microsoft Learn - Azure Virtual Network concepts and best practices ;
- Microsoft Learn - Azure Virtual Networks and subnets.

Les comportements de Terraform et les schémas du provider AzureRM peuvent évoluer. Pour une utilisation en production, les versions et la documentation correspondant aux versions réellement verrouillées dans le projet doivent toujours être vérifiées.
