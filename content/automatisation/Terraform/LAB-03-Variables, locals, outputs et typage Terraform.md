---
title: "LAB-03 — Variables, locals, outputs et typage Terraform"
date: 2026-10-06
draft: false
description: "Maîtriser les variables, locals, outputs, types, validations, null et données sensibles dans Terraform à travers une configuration Microsoft Azure structurée."
summary: "Construire un contrat de données Terraform propre avec variables typées, objets, listes, maps, validations, locals et outputs."
slug: "terraform-azure-lab-03-variables-locals-outputs-typage"
categories:
  - "Automatisation"
tags:
  - "Terraform"
  - "Azure"
  - "HCL"
  - "Variables"
  - "Locals"
  - "Outputs"
  - "Infrastructure as Code"
  - "AzureRM"
series:
  - "Terraform Azure — De zéro à production"
weight: 4
ShowToc: false
TocOpen: false
---

# LAB-03 — Variables, locals, outputs et typage Terraform

Dans le LAB-02, nous avons étudié le HashiCorp Configuration Language (HCL), les expressions, les références et le modèle déclaratif de Terraform.

Nous allons maintenant nous concentrer sur la circulation des données dans une configuration Terraform.

Nous avons déjà utilisé :

```text
variable
local
output
```

mais ces trois mécanismes ne jouent pas le même rôle.

Une bonne architecture Terraform doit clairement séparer :

```text
Ce que l'utilisateur fournit
          │
          ▼
       variables

Ce que Terraform calcule
          │
          ▼
        locals

Ce que Terraform expose
          │
          ▼
        outputs
```

Cette distinction paraît simple.

Elle devient pourtant essentielle lorsque nous commençons à construire :

- des modules ;
- plusieurs environnements ;
- des pipelines CI/CD ;
- des conventions de nommage ;
- des règles de gouvernance ;
- des configurations complexes.

Dans ce Lab, nous allons conserver la même infrastructure :

```text
Microsoft Azure
│
└── Resource Group
     │
     └── Virtual Network
```

Mais nous allons restructurer sa configuration afin qu'elle ressemble davantage à une véritable interface Terraform.

---

# 1. Objectifs du Lab

Nous allons apprendre à utiliser correctement :

- `variable` ;
- `default` ;
- `type` ;
- `description` ;
- `validation` ;
- `nullable` ;
- `sensitive` ;
- `null` ;
- `terraform.tfvars` ;
- `*.auto.tfvars` ;
- `TF_VAR_*` ;
- `-var` ;
- `-var-file` ;
- `locals` ;
- `output` ;
- `string` ;
- `number` ;
- `bool` ;
- `list` ;
- `set` ;
- `map` ;
- `object` ;
- `tuple` ;
- `optional()`.

Nous allons surtout répondre à une question importante :

> Qu'est-ce qui doit réellement être configurable dans une infrastructure Terraform ?

---

# 2. Architecture du Lab

L'infrastructure reste volontairement simple :

```text
Subscription Azure
│
└── rg-mla-dev-fce-001
     │
     └── vnet-mla-dev-fce-001
          │
          └── 10.20.0.0/16
```

Mais notre configuration va devenir plus structurée.

Flux de données :

```text
terraform.tfvars
       │
       ▼
   variables.tf
       │
       ▼
    var.*
       │
       ▼
    locals.tf
       │
       ▼
   local.*
       │
       ▼
    main.tf
       │
       ▼
Microsoft Azure
       │
       ▼
   outputs.tf
       │
       ▼
 terraform output
```

C'est ce flux que nous allons étudier.

---

# 3. Prérequis

Les LAB-00, LAB-01 et LAB-02 doivent avoir été réalisés.

Vérifier :

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

Puis :

```powershell
az account show `
  --query "{Subscription:name,SubscriptionId:id,TenantId:tenantId}" `
  --output table
```

Préparer le contexte Terraform :

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

# 4. Préparer LAB-03

Revenir à la racine du projet :

```powershell
Set-Location "C:\Labs\MonLabAzure\terraform-azure-platform"
```

Créer :

```powershell
New-Item -ItemType Directory -Name "lab-03"
```

Puis :

```powershell
Set-Location ".\lab-03"
code .
```

Arborescence cible :

```text
lab-03/
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

# 5. Le rôle d'une variable Terraform

Une variable représente une entrée du module.

Exemple :

```hcl
variable "environment" {
  type = string
}
```

Terraform permet ensuite d'utiliser :

```hcl
var.environment
```

Le flux est :

```text
Valeur fournie
      │
      ▼
variable "environment"
      │
      ▼
var.environment
```

Une variable doit représenter une donnée qu'il est raisonnable de fournir depuis l'extérieur du module.

Par exemple :

```text
environment
location
address space
tags métier
configuration réseau
```

---

# 6. Une variable n'est pas une variable mutable

Le mot `variable` peut être trompeur pour quelqu'un venant d'un langage de programmation.

Dans un programme classique, nous pourrions imaginer :

```text
x = 10
x = 20
```

Terraform ne fonctionne pas ainsi.

Une input variable reçoit une valeur pour l'exécution.

Nous ne faisons pas ensuite :

```text
var.environment = "prod"
```

à l'intérieur de Terraform.

Les variables représentent des **entrées déclaratives**.

---

# 7. Le rôle d'un local

Un local sert à nommer une expression interne au module.

Exemple :

```hcl
locals {
  environment = lower(var.environment)
}
```

Nous pouvons ensuite utiliser :

```hcl
local.environment
```

Le flux devient :

```text
var.environment
      │
      ▼
   lower()
      │
      ▼
local.environment
```

Le local permet notamment de :

- transformer une valeur ;
- centraliser une convention ;
- éviter une répétition ;
- rendre une expression complexe plus lisible ;
- construire des noms ;
- fusionner des structures.

---

# 8. Le rôle d'un output

Un output fait l'inverse d'une variable.

Une variable fait entrer une donnée :

```text
extérieur
   │
   ▼
Terraform
```

Un output expose une donnée :

```text
Terraform
   │
   ▼
extérieur
```

Exemple :

```hcl
output "virtual_network_id" {
  value = azurerm_virtual_network.main.id
}
```

Les outputs deviendront particulièrement importants lorsque nous utiliserons des modules.

---

# 9. Notre contrat de données

Nous allons structurer notre configuration autour des entrées suivantes :

```text
project_name
environment
location
resource_instance
owner
cost_center
network
additional_tags
enable_training_tag
```

Chaque variable aura une responsabilité précise.

---

# 10. `versions.tf`

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

# 11. `providers.tf`

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

# 12. `variables.tf`

Créer :

```text
variables.tf
```

avec la configuration complète suivante :

```hcl
variable "project_name" {
  description = "Nom court du projet utilisé dans les conventions de nommage."
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

  validation {
    condition     = length(trimspace(var.owner)) > 0
    error_message = "owner ne peut pas être vide."
  }
}

variable "cost_center" {
  description = "Centre de coûts associé aux ressources. Facultatif."
  type        = string
  default     = null
  nullable    = true

  validation {
    condition = (
      var.cost_center == null ||
      length(trimspace(var.cost_center)) > 0
    )

    error_message = "cost_center doit être null ou contenir une valeur non vide."
  }
}

variable "network" {
  description = "Configuration du Virtual Network."

  type = object({
    address_spaces = list(string)
    dns_servers    = optional(list(string), [])
  })

  default = {
    address_spaces = [
      "10.20.0.0/16"
    ]

    dns_servers = []
  }

  nullable = false

  validation {
    condition     = length(var.network.address_spaces) > 0
    error_message = "network.address_spaces doit contenir au moins une plage réseau."
  }
}

variable "additional_tags" {
  description = "Tags supplémentaires ajoutés aux tags standards."
  type        = map(string)
  default     = {}
  nullable    = false
}

variable "enable_training_tag" {
  description = "Ajoute le tag learning_path utilisé pour cette série de Labs."
  type        = bool
  default     = true
  nullable    = false
}
```

Nous allons maintenant analyser cette interface.

---

# 13. Anatomie d'une variable

Prenons :

```hcl
variable "location" {
  description = "Région Azure cible."
  type        = string
  default     = "francecentral"
  nullable    = false
}
```

Nous avons :

```text
variable
│
├── nom
├── description
├── type
├── default
└── nullable
```

Ces attributs ne répondent pas au même besoin.

---

# 14. `description`

```hcl
description = "Région Azure cible."
```

Une description n'a pas d'effet sur Azure.

Mais elle constitue une partie importante du contrat du module.

Une variable professionnelle devrait généralement expliquer :

- ce qu'elle représente ;
- son unité lorsqu'elle en possède une ;
- son format ;
- éventuellement ses contraintes.

Une variable nommée correctement mais non documentée peut devenir ambiguë plusieurs mois plus tard.

---

# 15. `type`

Nous avons :

```hcl
type = string
```

Le type contraint les valeurs acceptées.

Terraform peut ainsi détecter certaines incohérences avant de contacter Azure.

C'est également une forme de documentation.

Comparer :

```hcl
variable "network" {
}
```

avec :

```hcl
variable "network" {
  type = object({
    address_spaces = list(string)
    dns_servers    = optional(list(string), [])
  })
}
```

La seconde version définit beaucoup plus clairement le contrat attendu.

---

# 16. `default`

Avec :

```hcl
default = "francecentral"
```

la variable devient facultative pour l'appelant.

Si aucune valeur n'est fournie :

```text
location
   │
   ▼
francecentral
```

Sans `default`, Terraform doit obtenir une valeur depuis une autre source.

---

# 17. Faut-il mettre un `default` partout ?

Non.

Prenons :

```hcl
variable "environment" {
  type = string
}
```

Nous avons volontairement omis :

```hcl
default = "dev"
```

Pourquoi ?

Parce que l'environnement est une information suffisamment importante pour que nous préférions qu'elle soit explicitement fournie.

Un défaut pratique peut parfois devenir dangereux.

Imaginez une pipeline PROD qui oublie de fournir l'environnement et récupère automatiquement :

```text
dev
```

La configuration pourrait alors dériver vers des noms ou paramètres inattendus.

Le choix d'un `default` doit donc être réfléchi.

---

# 18. `nullable`

Nous avons :

```hcl
nullable = false
```

pour plusieurs variables.

Cela signifie que :

```hcl
location = null
```

n'est pas accepté.

Pour :

```hcl
variable "cost_center"
```

nous avons au contraire :

```hcl
nullable = true
default  = null
```

Le centre de coûts peut donc volontairement être absent.

---

# 19. Comprendre `null`

`null` représente l'absence de valeur.

Ce n'est pas la même chose que :

```text
""
```

une chaîne vide.

Ce n'est pas non plus :

```text
0
```

ni :

```text
false
```

Comparer :

```text
null
""
0
false
```

Ces valeurs n'ont pas la même signification.

---

# 20. Cas concret avec `cost_center`

Nous voulons permettre :

```hcl
cost_center = "CC-1001"
```

mais également :

```hcl
cost_center = null
```

Nous ne voulons pas créer un tag Azure comme :

```text
cost_center = ""
```

car il donnerait l'impression qu'une information existe alors qu'elle est en réalité absente.

Nous allons gérer cette logique dans les locals.

---

# 21. Type `string`

Exemples :

```hcl
project_name = "mla"
environment  = "dev"
location     = "francecentral"
```

Le type est :

```hcl
string
```

Une chaîne représente une valeur textuelle.

---

# 22. Type `number`

Nous utilisons :

```hcl
variable "resource_instance" {
  type = number
}
```

Exemple :

```hcl
resource_instance = 1
```

Nous allons ensuite transformer :

```text
1
```

en :

```text
001
```

pour notre convention de nommage.

---

# 23. Type `bool`

Nous utilisons :

```hcl
variable "enable_training_tag" {
  type = bool
}
```

Valeurs possibles :

```hcl
true
false
```

Cela permet d'exprimer clairement une décision binaire.

---

# 24. Type `list`

Dans :

```hcl
address_spaces = list(string)
```

nous attendons une liste ordonnée de chaînes.

Exemple :

```hcl
[
  "10.20.0.0/16",
  "10.21.0.0/16"
]
```

Les éléments sont accessibles par index :

```text
0
1
2
...
```

Dans `terraform console` :

```hcl
var.network.address_spaces[0]
```

---

# 25. Type `set`

Un `set` représente également une collection d'éléments du même type.

Exemple conceptuel :

```hcl
set(string)
```

Contrairement à une liste, un set :

- ne doit pas être utilisé lorsque l'ordre métier des éléments est important ;
- élimine les doublons ;
- n'est pas manipulé comme une séquence indexée classique.

Exemple :

```hcl
toset([
  "dev",
  "prod",
  "dev"
])
```

produit conceptuellement :

```text
dev
prod
```

Le deuxième `dev` ne représente pas un élément distinct.

---

# 26. `list` ou `set` ?

Choisir :

```text
list
```

lorsque :

- l'ordre a un sens ;
- l'indexation est pertinente ;
- les doublons peuvent avoir une signification.

Choisir :

```text
set
```

lorsque :

- l'unicité est importante ;
- l'ordre ne doit pas porter de signification.

Le choix du type fait partie de l'interface du module.

---

# 27. Type `map`

Nous utilisons :

```hcl
variable "additional_tags" {
  type = map(string)
}
```

Exemple :

```hcl
additional_tags = {
  application = "platform"
  criticality = "medium"
}
```

Chaque élément est identifié par une clé :

```text
application -> platform

criticality -> medium
```

C'est particulièrement adapté aux tags Azure.

---

# 28. Type `object`

Notre variable la plus intéressante est :

```hcl
variable "network" {
  type = object({
    address_spaces = list(string)
    dns_servers    = optional(list(string), [])
  })
}
```

Nous déclarons une structure.

Elle doit contenir :

```text
address_spaces
```

et peut contenir :

```text
dns_servers
```

Nous pouvons donc écrire :

```hcl
network = {
  address_spaces = [
    "10.20.0.0/16"
  ]
}
```

Terraform appliquera automatiquement la valeur par défaut :

```hcl
dns_servers = []
```

---

# 29. Pourquoi utiliser un `object` ?

Sans objet, nous pourrions créer :

```text
vnet_address_spaces
vnet_dns_servers
vnet_enable_ddos
vnet_x
vnet_y
vnet_z
```

À mesure que la configuration grandit, l'interface devient difficile à comprendre.

Avec :

```hcl
network = {
  ...
}
```

nous regroupons les paramètres appartenant au même concept.

Cela deviendra particulièrement intéressant avec les modules.

---

# 30. `optional()`

Dans :

```hcl
dns_servers = optional(list(string), [])
```

le champ `dns_servers` n'est pas obligatoire.

S'il est absent :

```hcl
network = {
  address_spaces = [
    "10.20.0.0/16"
  ]
}
```

Terraform utilise :

```hcl
[]
```

Cette approche permet de définir une interface structurée tout en conservant des valeurs par défaut raisonnables.

---

# 31. Type `tuple`

Terraform supporte également les tuples.

Exemple :

```hcl
tuple([
  string,
  number,
  bool
])
```

Une valeur compatible pourrait être :

```hcl
[
  "dev",
  2,
  true
]
```

Chaque position possède son propre type.

Pour les interfaces d'infrastructure, les objets sont souvent plus lisibles car :

```hcl
{
  environment = "dev"
  replicas    = 2
  enabled     = true
}
```

est plus explicite que :

```hcl
[
  "dev",
  2,
  true
]
```

Nous utiliserons donc beaucoup plus souvent `object` que `tuple` dans cette série.

---

# 32. Éviter `any` sans raison

Terraform permet :

```hcl
type = any
```

ou certaines formes comme :

```hcl
list(any)
```

Cela paraît flexible.

Mais cette flexibilité peut rendre l'interface moins explicite.

Pour du code d'infrastructure maintenable, nous préférerons généralement :

```hcl
list(string)
```

à :

```hcl
list(any)
```

lorsque le type attendu est connu.

Un contrat strict facilite :

- la compréhension ;
- les messages d'erreur ;
- les modules ;
- les revues de code ;
- les évolutions futures.

---

# 33. Les validations

Nous avons utilisé :

```hcl
validation {
  condition = contains(
    ["dev", "qual", "prod"],
    lower(var.environment)
  )

  error_message = "environment doit être dev, qual ou prod."
}
```

Une validation permet d'exprimer une contrainte métier ou technique plus précise que le type.

Le type nous dit :

```text
environment est une string
```

La validation ajoute :

```text
environment doit appartenir à :

dev
qual
prod
```

---

# 34. Type et validation ne remplacent pas Azure Policy

Une validation Terraform est utile.

Mais elle ne constitue pas une politique de gouvernance Azure globale.

Un utilisateur pourrait :

- utiliser un autre repository ;
- utiliser Azure CLI ;
- utiliser Bicep ;
- utiliser le portail Azure ;
- modifier Terraform.

Pour des règles d'organisation imposées à l'échelle Azure, nous utiliserons plus tard des mécanismes comme Azure Policy.

Terraform validation et Azure Policy ne répondent donc pas exactement au même besoin.

---

# 35. `locals.tf`

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
    resource_group = join(
      "-",
      [
        "rg",
        local.project_name,
        local.environment,
        local.location_code,
        local.instance_suffix
      ]
    )

    virtual_network = join(
      "-",
      [
        "vnet",
        local.project_name,
        local.environment,
        local.location_code,
        local.instance_suffix
      ]
    )
  }

  mandatory_tags = {
    environment = local.environment
    project     = local.project_name
    managed_by  = "terraform"
    owner       = trimspace(var.owner)
    lab         = "lab-03"
  }

  cost_center_tag = var.cost_center != null ? {
    cost_center = trimspace(var.cost_center)
  } : {}

  training_tag = var.enable_training_tag ? {
    learning_path = "terraform-azure"
  } : {}

  common_tags = merge(
    local.mandatory_tags,
    local.cost_center_tag,
    local.training_tag,
    var.additional_tags
  )
}
```

Nous pouvons maintenant observer la différence entre entrée et transformation.

---

# 36. Normaliser les entrées

Nous recevons :

```hcl
var.project_name
```

mais utilisons :

```hcl
lower(trimspace(var.project_name))
```

Nous évitons ainsi de propager inutilement :

```text
" MLA "
```

au lieu de :

```text
mla
```

Le local joue ici un rôle de normalisation.

---

# 37. Formater un nombre

Nous avons :

```hcl
instance_suffix = format(
  "%03d",
  var.resource_instance
)
```

Avec :

```hcl
resource_instance = 1
```

Terraform produit :

```text
001
```

Avec :

```hcl
resource_instance = 12
```

Terraform produit :

```text
012
```

Cette logique permet de séparer :

```text
valeur métier
1
```

de sa représentation dans notre convention :

```text
001
```

---

# 38. Construire un nom avec `join()`

Nous utilisons :

```hcl
join(
  "-",
  [
    "rg",
    local.project_name,
    local.environment,
    local.location_code,
    local.instance_suffix
  ]
)
```

Terraform produit :

```text
rg-mla-dev-fce-001
```

Cette écriture rend les différents composants du nom clairement visibles.

---

# 39. Utiliser `null` pour créer un tag conditionnel

Nous avons :

```hcl
cost_center_tag = var.cost_center != null ? {
  cost_center = trimspace(var.cost_center)
} : {}
```

Si :

```hcl
cost_center = "CC-1001"
```

nous obtenons :

```hcl
{
  cost_center = "CC-1001"
}
```

Si :

```hcl
cost_center = null
```

nous obtenons :

```hcl
{}
```

Nous n'ajoutons donc pas de tag vide.

---

# 40. Priorité dans `merge()`

Nous avons :

```hcl
common_tags = merge(
  local.mandatory_tags,
  local.cost_center_tag,
  local.training_tag,
  var.additional_tags
)
```

Lorsqu'une même clé apparaît plusieurs fois, les maps placées plus tard peuvent remplacer les précédentes.

Ici :

```hcl
var.additional_tags
```

est fourni en dernier.

Il peut donc modifier un tag existant.

Est-ce toujours souhaitable ?

Pas forcément.

---

# 41. Protection des tags obligatoires

Dans une plateforme de production, nous pourrions préférer :

```hcl
common_tags = merge(
  var.additional_tags,
  local.cost_center_tag,
  local.training_tag,
  local.mandatory_tags
)
```

Ainsi les tags standards sont appliqués en dernier.

Un utilisateur ne pourrait pas remplacer facilement :

```text
managed_by = terraform
```

par :

```text
managed_by = manual
```

via `additional_tags`.

Ce choix dépend de la gouvernance souhaitée.

Pour ce Lab, nous conserverons :

```hcl
local.mandatory_tags
local.cost_center_tag
local.training_tag
var.additional_tags
```

afin d'observer explicitement le comportement de `merge()`.

Dans une plateforme d'entreprise, cette décision devrait être documentée.

---

# 42. `main.tf`

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

  address_space = var.network.address_spaces
  dns_servers   = var.network.dns_servers

  tags = local.common_tags
}
```

Le code des ressources devient relativement simple.

Une grande partie de la logique est externalisée dans :

```text
variables.tf
locals.tf
```

C'est volontaire.

---

# 43. Une ressource doit-elle être totalement paramétrable ?

Non.

Une erreur fréquente consiste à transformer chaque argument Terraform en variable.

Exemple excessif :

```hcl
variable "resource_group_name" {}
variable "resource_group_location" {}
variable "resource_group_tag_1" {}
variable "resource_group_tag_2" {}
variable "resource_group_tag_3" {}
```

Nous finirions par déplacer la complexité sans réellement créer une bonne abstraction.

Dans notre cas :

```text
resource_group_name
```

est dérivé d'une convention.

Nous préférons donc le calculer avec des locals.

---

# 44. Règle pratique

Demander :

> Est-ce que l'appelant doit réellement décider de cette valeur ?

Si oui :

```text
variable
```

Si la valeur découle d'une règle interne :

```text
local
```

Si nous devons exposer le résultat :

```text
output
```

Ce n'est pas une règle absolue, mais c'est un bon point de départ.

---

# 45. `outputs.tf`

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

output "resource_group_id" {
  type        = string
  description = "Identifiant Azure du Resource Group."
  value       = azurerm_resource_group.main.id
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

output "virtual_network_address_spaces" {
  type        = list(string)
  description = "Address spaces du Virtual Network."
  value       = tolist(azurerm_virtual_network.main.address_space)
}

output "common_tags" {
  type        = map(string)
  description = "Tags appliqués aux ressources."
  value       = local.common_tags
}

output "deployment_summary" {
  type = object({
    project_name         = string
    environment          = string
    location             = string
    resource_group_name  = string
    virtual_network_name = string
    address_spaces       = list(string)
  })

  description = "Résumé structuré du déploiement."

  value = {
    project_name         = local.project_name
    environment          = local.environment
    location             = local.location
    resource_group_name  = azurerm_resource_group.main.name
    virtual_network_name = azurerm_virtual_network.main.name
    address_spaces       = tolist(azurerm_virtual_network.main.address_space)
  }
}
```

Nous typions maintenant également nos outputs.

---

# 46. Pourquoi typer les outputs ?

Un output représente lui aussi une interface.

Aujourd'hui :

```text
LAB-03
```

les outputs sont principalement utilisés par un humain.

Plus tard :

```text
module réseau
       │
       ▼
output vnet_id
       │
       ▼
module application
```

L'output deviendra un contrat entre plusieurs parties de l'architecture Terraform.

Le typage rend ce contrat plus explicite.

---

# 47. Output simple

Exemple :

```hcl
output "resource_group_name" {
  type  = string
  value = azurerm_resource_group.main.name
}
```

Résultat :

```text
rg-mla-dev-fce-001
```

---

# 48. Output structuré

Nous avons :

```hcl
output "deployment_summary" {
  type = object({
    ...
  })

  value = {
    ...
  }
}
```

Cela permet d'exposer plusieurs informations sous une seule structure cohérente.

---

# 49. `terraform.tfvars.example`

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

owner       = "cloud-platform"
cost_center = "CC-TRAINING"

network = {
  address_spaces = [
    "10.20.0.0/16"
  ]

  dns_servers = []
}

additional_tags = {
  workload    = "terraform-learning"
  criticality = "low"
}

enable_training_tag = true
```

Créer le fichier local :

```powershell
Copy-Item `
  ".\terraform.tfvars.example" `
  ".\terraform.tfvars"
```

---

# 50. Arborescence finale

Nous avons :

```text
lab-03/
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

Après initialisation :

```text
lab-03/
│
├── .terraform/
├── .terraform.lock.hcl
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

# 51. Les différentes sources de variables

Une variable Terraform peut recevoir sa valeur de plusieurs manières.

Exemples :

```text
default
TF_VAR_*
terraform.tfvars
*.auto.tfvars
-var-file
-var
```

Elles ne possèdent pas toutes la même priorité.

---

# 52. Valeur `default`

Dans :

```hcl
variable "owner" {
  default = "cloud-platform"
}
```

la valeur :

```text
cloud-platform
```

sera utilisée si aucune source prioritaire ne fournit une autre valeur.

Le `default` représente le niveau de priorité le plus faible parmi les méthodes utilisées ici.

---

# 53. Variable d'environnement `TF_VAR_*`

Terraform sait lire des variables d'environnement suivant la convention :

```text
TF_VAR_<nom_variable>
```

Exemple PowerShell :

```powershell
$env:TF_VAR_owner = "platform-team"
```

Terraform peut alors utiliser :

```text
platform-team
```

pour :

```hcl
var.owner
```

---

# 54. Vérifier avec `terraform console`

Initialiser :

```powershell
terraform init
```

Puis :

```powershell
terraform console
```

Tester :

```hcl
var.owner
```

Attention : si :

```text
terraform.tfvars
```

contient déjà :

```hcl
owner = "cloud-platform"
```

la valeur de ce fichier possède une priorité supérieure à `TF_VAR_owner`.

Nous devons donc comprendre la priorité des sources plutôt que supposer que la dernière valeur définie dans PowerShell gagne automatiquement.

Quitter :

```text
exit
```

---

# 55. `terraform.tfvars`

Terraform charge automatiquement :

```text
terraform.tfvars
```

s'il existe dans le répertoire courant.

C'est ce que nous utilisons dans ce Lab.

---

# 56. `*.auto.tfvars`

Terraform charge également automatiquement les fichiers :

```text
*.auto.tfvars
```

Exemples :

```text
10-common.auto.tfvars
20-dev.auto.tfvars
```

Lorsque plusieurs fichiers automatiques existent, leur ordre lexical intervient dans la priorité entre ces fichiers.

Cette possibilité est utile, mais elle peut aussi rendre une configuration difficile à comprendre si nous empilons trop de fichiers implicites.

---

# 57. `-var-file`

Nous pouvons utiliser explicitement :

```powershell
terraform plan `
  -var-file="dev.tfvars"
```

Cette approche deviendra intéressante lorsque nous construirons :

```text
DEV
QUALIF
PROD
```

Exemple futur :

```text
environments/
├── dev.tfvars
├── qual.tfvars
└── prod.tfvars
```

Mais nous n'allons pas encore construire cette architecture dans LAB-03.

---

# 58. `-var`

Nous pouvons également remplacer directement une variable :

```powershell
terraform plan `
  -var="owner=temporary-owner"
```

C'est pratique pour un test ponctuel.

Ce n'est généralement pas la meilleure manière de gérer une configuration complexe et durable.

---

# 59. Priorité des valeurs

Dans notre workflow CLI, il faut retenir cette hiérarchie générale, de la priorité la plus forte vers la plus faible :

```text
-var / -var-file
       │
       ▼
*.auto.tfvars
       │
       ▼
terraform.tfvars.json
       │
       ▼
terraform.tfvars
       │
       ▼
TF_VAR_*
       │
       ▼
default
```

Cette notion devient importante en CI/CD.

Sinon, nous pouvons croire qu'une valeur provient d'un fichier alors qu'elle a été remplacée ailleurs.

---

# 60. Tester la priorité

Notre `terraform.tfvars` contient :

```hcl
owner = "cloud-platform"
```

Exécuter :

```powershell
terraform console
```

Puis :

```hcl
var.owner
```

Résultat attendu :

```text
"cloud-platform"
```

Quitter.

Puis :

```powershell
terraform console `
  -var="owner=temporary-owner"
```

Tester :

```hcl
var.owner
```

Résultat :

```text
"temporary-owner"
```

Le `-var` a remplacé la valeur de `terraform.tfvars`.

---

# 61. Nettoyer les variables PowerShell de test

Si nous avons utilisé :

```powershell
$env:TF_VAR_owner
```

supprimer la variable de test :

```powershell
Remove-Item Env:TF_VAR_owner -ErrorAction SilentlyContinue
```

C'est une bonne habitude lorsque nous expérimentons avec plusieurs sources.

---

# 62. Initialiser et valider

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

Résultat attendu :

```text
Success! The configuration is valid.
```

---

# 63. Explorer les variables avec `terraform console`

Lancer :

```powershell
terraform console
```

Tester :

```hcl
var.project_name
```

Puis :

```hcl
var.resource_instance
```

Puis :

```hcl
var.network
```

Puis :

```hcl
var.network.address_spaces
```

Puis :

```hcl
var.network.dns_servers
```

Puis :

```hcl
var.additional_tags
```

---

# 64. Explorer les locals

Dans la console :

```hcl
local.project_name
```

Puis :

```hcl
local.instance_suffix
```

Résultat :

```text
"001"
```

Puis :

```hcl
local.resource_names
```

Puis :

```hcl
local.common_tags
```

Nous pouvons ainsi inspecter toute la transformation avant même de créer une ressource.

---

# 65. Tester `null`

Dans :

```text
terraform.tfvars
```

modifier temporairement :

```hcl
cost_center = null
```

Puis relancer :

```powershell
terraform console
```

Tester :

```hcl
var.cost_center
```

Résultat :

```text
null
```

Puis :

```hcl
local.cost_center_tag
```

Résultat :

```hcl
{}
```

Puis :

```hcl
local.common_tags
```

Le tag :

```text
cost_center
```

doit être absent.

---

# 66. Remettre le centre de coûts

Remettre :

```hcl
cost_center = "CC-TRAINING"
```

Puis :

```powershell
terraform console
```

Tester :

```hcl
local.cost_center_tag
```

Terraform doit maintenant retourner :

```hcl
{
  "cost_center" = "CC-TRAINING"
}
```

Quitter la console.

---

# 67. Tester une validation

Modifier temporairement :

```hcl
environment = "production"
```

Puis :

```powershell
terraform plan
```

Terraform doit refuser la valeur avec le message défini dans :

```hcl
error_message
```

Notre erreur est détectée avant d'essayer de créer le Resource Group.

Remettre :

```hcl
environment = "dev"
```

---

# 68. Tester le type `number`

Modifier temporairement :

```hcl
resource_instance = 12
```

Puis :

```powershell
terraform console
```

Tester :

```hcl
local.instance_suffix
```

Résultat attendu :

```text
"012"
```

Puis remettre :

```hcl
resource_instance = 1
```

---

# 69. Générer le plan

Exécuter :

```powershell
terraform plan -out="lab-03.tfplan"
```

Résultat attendu :

```text
Plan: 2 to add, 0 to change, 0 to destroy.
```

Nous devons créer :

```text
azurerm_resource_group.main
azurerm_virtual_network.main
```

---

# 70. Lire le plan

Exécuter :

```powershell
terraform show ".\lab-03.tfplan"
```

Vérifier notamment :

```text
Resource Group
rg-mla-dev-fce-001
```

et :

```text
Virtual Network
vnet-mla-dev-fce-001
```

avec :

```text
10.20.0.0/16
```

ainsi que les tags.

---

# 71. Appliquer

Une fois le plan vérifié :

```powershell
terraform apply ".\lab-03.tfplan"
```

Résultat attendu :

```text
Apply complete! Resources: 2 added, 0 changed, 0 destroyed.
```

---

# 72. Observer les outputs

Exécuter :

```powershell
terraform output
```

Nous devons retrouver :

```text
resource_group_name
resource_group_id
virtual_network_name
virtual_network_id
virtual_network_address_spaces
common_tags
deployment_summary
```

---

# 73. Récupérer un output simple

Exécuter :

```powershell
terraform output -raw resource_group_name
```

Résultat :

```text
rg-mla-dev-fce-001
```

Même principe :

```powershell
terraform output -raw virtual_network_name
```

---

# 74. Récupérer les outputs en JSON

Exécuter :

```powershell
terraform output -json
```

Cette représentation sera particulièrement utile pour :

- les scripts ;
- les pipelines ;
- l'automatisation ;
- l'intégration avec d'autres outils.

Il faut cependant être prudent avec les outputs sensibles : certaines formes d'extraction explicite peuvent rendre leur valeur visible.

---

# 75. Observer l'output structuré

Exécuter :

```powershell
terraform output deployment_summary
```

Résultat proche de :

```text
{
  "address_spaces" = tolist([
    "10.20.0.0/16",
  ])
  "environment" = "dev"
  "location" = "francecentral"
  "project_name" = "mla"
  "resource_group_name" = "rg-mla-dev-fce-001"
  "virtual_network_name" = "vnet-mla-dev-fce-001"
}
```

---

# 76. Vérifier dans Azure

Récupérer :

```powershell
$rgName   = terraform output -raw resource_group_name
$vnetName = terraform output -raw virtual_network_name
```

Puis :

```powershell
az group show `
  --name $rgName `
  --output table
```

Et :

```powershell
az network vnet show `
  --resource-group $rgName `
  --name $vnetName `
  --output table
```

Vérifier les tags :

```powershell
az group show `
  --name $rgName `
  --query tags `
  --output json
```

---

# 77. `sensitive = true`

Terraform permet de déclarer :

```hcl
variable "database_password" {
  description = "Mot de passe de la base de données."
  type        = string
  sensitive   = true
}
```

Terraform masque alors cette valeur dans plusieurs affichages CLI.

Mais :

```text
sensitive = true
```

ne signifie pas :

```text
chiffré
```

et ne signifie pas non plus automatiquement :

```text
absent du state
```

Cette distinction est critique.

---

# 78. Exemple à ne pas utiliser dans ce Lab

Ne pas ajouter réellement ceci :

```hcl
variable "database_password" {
  type      = string
  sensitive = true
  default   = "Password123!"
}
```

Le fait d'ajouter :

```hcl
sensitive = true
```

ne rendrait pas acceptable le stockage d'un secret directement dans le code source.

---

# 79. Valeurs sensibles et Terraform State

Lorsqu'une valeur sensible est utilisée d'une manière qui doit être persistée par Terraform, elle peut se retrouver dans le state.

Nous devons donc considérer le Terraform State comme une donnée à protéger.

C'est une des raisons pour lesquelles nous consacrerons plusieurs Labs à :

```text
remote state
backend Azure
RBAC
chiffrement
locking
contrôle d'accès
```

---

# 80. Valeurs éphémères

Les versions modernes de Terraform permettent également de travailler avec certaines valeurs dites éphémères.

Une variable peut par exemple être déclarée avec :

```hcl
variable "temporary_token" {
  type      = string
  sensitive = true
  ephemeral = true
}
```

L'objectif est différent de `sensitive`.

Conceptuellement :

```text
sensitive
   │
   └── masquer l'affichage

ephemeral
   │
   └── éviter la persistance dans plan/state
       dans les contextes supportés
```

Les valeurs éphémères possèdent toutefois des restrictions importantes sur les endroits où elles peuvent être utilisées.

Nous ne les utiliserons pas encore dans notre infrastructure.

Elles seront reprises lorsque nous travaillerons sur les secrets, les providers et l'automatisation.

---

# 81. `sensitive` n'est donc pas une solution complète

Pour une vraie architecture :

```text
Secret
  │
  ├── stockage
  ├── transport
  ├── exposition CLI
  ├── state
  ├── plan
  ├── logs
  └── rotation
```

doivent être analysés séparément.

Ajouter simplement :

```hcl
sensitive = true
```

ne couvre pas tous ces sujets.

---

# 82. Les locals peuvent-ils contenir des secrets ?

Techniquement, un local peut contenir une valeur sensible.

Mais écrire :

```hcl
locals {
  password = "SuperSecret!"
}
```

dans un fichier Terraform reste une mauvaise approche.

Le problème n'est pas seulement Terraform.

Le secret est maintenant présent dans :

```text
code source
Git
historique
poste local
éventuelles copies
```

Les secrets ne doivent pas être traités comme de simples constantes HCL.

---

# 83. Variables et secrets dans GitHub Actions

Plus tard, nous aurons :

```text
GitHub Actions
       │
       ▼
Terraform
       │
       ▼
Azure
```

Toutes les données ne seront pas fournies de la même manière.

Nous distinguerons :

```text
configuration non sensible
```

de :

```text
credentials
tokens
secrets
```

Et autant que possible, nous utiliserons des mécanismes d'identité comme OpenID Connect (OIDC) plutôt que des credentials permanents.

---

# 84. Test important — modifier un `local` sans changer Azure

Supposons que nous modifions uniquement la manière de calculer une valeur sans modifier son résultat.

Par exemple, cette expression :

```hcl
resource_group = join(
  "-",
  [
    "rg",
    local.project_name,
    local.environment,
    local.location_code,
    local.instance_suffix
  ]
)
```

pourrait être réécrite :

```hcl
resource_group = "rg-${local.project_name}-${local.environment}-${local.location_code}-${local.instance_suffix}"
```

Les deux produisent :

```text
rg-mla-dev-fce-001
```

Après la modification :

```powershell
terraform plan
```

doit retourner :

```text
No changes.
```

C'est un point essentiel.

Modifier le code Terraform ne signifie pas nécessairement modifier l'infrastructure.

Terraform s'intéresse au résultat déclaré.

---

# 85. Refactoring Terraform

Un refactoring consiste à améliorer la structure du code sans nécessairement modifier le comportement attendu.

Exemple :

```text
avant

expression répétée
expression répétée
expression répétée
```

devient :

```text
local
 │
 ├── utilisation 1
 ├── utilisation 2
 └── utilisation 3
```

Si les valeurs obtenues restent identiques :

```powershell
terraform plan
```

devrait idéalement confirmer :

```text
No changes.
```

Le plan devient ainsi un outil de validation du refactoring.

---

# 86. Quand utiliser un local ?

Bon candidat :

```hcl
local.environment
```

car nous normalisons :

```hcl
lower(trimspace(var.environment))
```

Bon candidat :

```hcl
local.common_tags
```

car plusieurs ressources utilisent la même structure.

Bon candidat :

```hcl
local.resource_names
```

car le naming suit une convention interne.

---

# 87. Quand ne pas utiliser un local ?

Créer :

```hcl
locals {
  france_region = "francecentral"
}
```

simplement pour remplacer :

```hcl
location = "francecentral"
```

n'apporte pas nécessairement de valeur.

Il faut éviter d'ajouter des niveaux d'indirection sans raison.

Chaque abstraction doit résoudre un problème réel.

---

# 88. Quand créer une variable ?

Une variable est pertinente lorsque l'appelant doit pouvoir décider.

Exemples :

```text
environment
location
network.address_spaces
cost_center
additional_tags
```

---

# 89. Quand ne pas créer une variable ?

Éviter une variable comme :

```hcl
variable "managed_by" {
  default = "terraform"
}
```

si l'organisation impose systématiquement :

```text
managed_by = terraform
```

Dans ce cas :

```hcl
managed_by = "terraform"
```

dans un local standard peut être plus approprié.

Sinon, nous permettrions inutilement à chaque utilisateur de modifier une convention que nous voulons justement imposer.

---

# 90. Une bonne interface Terraform réduit les choix inutiles

Une configuration flexible n'est pas forcément une bonne configuration.

Comparer :

```text
Module A

40 variables
toutes les propriétés configurables
```

avec :

```text
Module B

10 variables
choix utiles uniquement
conventions intégrées
```

Dans une plateforme d'entreprise, le deuxième modèle peut être beaucoup plus sûr et maintenable.

L'objectif n'est pas :

> permettre de tout configurer.

L'objectif est :

> permettre de configurer ce qui doit réellement varier.

---

# 91. Tester l'idempotence

Après le déploiement :

```powershell
terraform plan
```

Résultat attendu :

```text
No changes.
Your infrastructure matches the configuration.
```

---

# 92. Tester une modification réelle

Modifier :

```hcl
additional_tags = {
  workload    = "terraform-learning"
  criticality = "medium"
}
```

au lieu de :

```hcl
criticality = "low"
```

Exécuter :

```powershell
terraform plan
```

Terraform doit proposer une mise à jour des tags.

Nous avons maintenant modifié une valeur d'entrée qui change réellement l'état désiré.

---

# 93. Ne pas appliquer le changement immédiatement

Observer :

```text
~ update in-place
```

Nous devons nous attendre à une modification en place et non à une recréation complète du VNet.

Lire le plan avant tout `apply`.

---

# 94. Remettre la valeur initiale

Remettre :

```hcl
criticality = "low"
```

Si le précédent changement n'a pas été appliqué :

```powershell
terraform plan
```

doit à nouveau retourner :

```text
No changes.
```

---

# 95. Versionner LAB-03

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

Puis :

```powershell
git diff --cached --name-only
```

Vérifier que nous ne versionnons pas :

```text
terraform.tfstate
terraform.tfstate.backup
terraform.tfvars
*.tfplan
.terraform/
```

Nous devons en revanche versionner :

```text
terraform.tfvars.example
.terraform.lock.hcl
```

Puis :

```powershell
git commit -m "feat(terraform): add lab 03 variables and types"
```

---

# 96. À propos des fichiers `.tfvars` dans Git

Dans cette série, notre `.gitignore` ignore :

```text
*.tfvars
```

C'est une approche volontairement conservatrice.

Mais il faut comprendre qu'un fichier `.tfvars` n'est pas automatiquement secret.

Un fichier contenant uniquement :

```hcl
environment = "dev"
location    = "francecentral"
```

peut parfaitement être versionné selon la stratégie de l'organisation.

En revanche :

```hcl
database_password = "..."
```

ne doit pas être publié simplement parce qu'il se trouve dans un fichier `.tfvars`.

La décision doit dépendre du contenu et de la stratégie de configuration.

---

# 97. Exemple futur multi-environnements

Nous pourrions plus tard arriver à :

```text
environments/
│
├── dev.tfvars
├── qual.tfvars
└── prod.tfvars
```

avec :

```powershell
terraform plan `
  -var-file="environments\prod.tfvars"
```

Mais cette stratégie possède des implications sur :

- les states ;
- les subscriptions ;
- les droits ;
- les pipelines ;
- les approbations ;
- les risques d'erreur humaine.

Elle sera étudiée dans le Lab consacré aux environnements.

---

# 98. Nettoyage

Revenir :

```powershell
Set-Location "C:\Labs\MonLabAzure\terraform-azure-platform\lab-03"
```

Créer le plan de destruction :

```powershell
terraform plan `
  -destroy `
  -out="lab-03-destroy.tfplan"
```

Inspecter :

```powershell
terraform show ".\lab-03-destroy.tfplan"
```

Résultat attendu :

```text
Plan: 0 to add, 0 to change, 2 to destroy.
```

---

# 99. Appliquer la destruction

Après vérification :

```powershell
terraform apply ".\lab-03-destroy.tfplan"
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

# 100. Ce qu'il faut retenir

Le modèle principal de ce Lab peut être représenté ainsi :

```text
INPUT
 │
 ▼
variable
 │
 ▼
var.*
 │
 ▼
TRANSFORMATION
 │
 ▼
locals
 │
 ▼
local.*
 │
 ▼
RESOURCE
 │
 ▼
Azure
 │
 ▼
OUTPUT
 │
 ▼
terraform output
```

---

## Variable

Une variable représente une entrée :

```text
extérieur
   │
   ▼
Terraform
```

---

## Local

Un local transforme ou centralise une valeur à l'intérieur du module :

```text
variables/resources
       │
       ▼
expression
       │
       ▼
local
```

---

## Output

Un output expose une information :

```text
Terraform
   │
   ▼
extérieur
```

---

## Type

Un type définit le contrat attendu :

```text
string
number
bool
list
set
map
object
tuple
```

---

## Validation

Une validation ajoute une règle métier ou technique au type.

---

## `null`

`null` représente l'absence de valeur.

Il ne doit pas être confondu avec :

```text
""
0
false
```

---

## `sensitive`

`sensitive = true` masque une valeur dans plusieurs affichages.

Il ne signifie pas automatiquement :

```text
la donnée n'existe pas dans le state
```

---

# 101. Ce qu'un ingénieur Terraform doit pouvoir expliquer

Prenons :

```hcl
variable "network" {
  type = object({
    address_spaces = list(string)
    dns_servers    = optional(list(string), [])
  })
}
```

Une explication correcte serait :

> La variable `network` représente l'interface d'entrée de la configuration réseau. Elle est fortement typée avec un objet afin de regrouper les paramètres appartenant au même domaine. `address_spaces` est obligatoire et doit être une liste de chaînes, tandis que `dns_servers` est facultatif et reçoit une liste vide lorsqu'il n'est pas fourni. Le typage permet de rendre l'interface explicite et de détecter certaines erreurs avant le déploiement.

Pour :

```hcl
local.instance_suffix = format("%03d", var.resource_instance)
```

il faut pouvoir expliquer :

> `resource_instance` reste une valeur métier de type number. Le local transforme uniquement sa représentation pour la convention de nommage. L'appelant fournit donc `1` et non la chaîne `"001"`.

Et pour :

```hcl
cost_center_tag = var.cost_center != null ? {
  cost_center = trimspace(var.cost_center)
} : {}
```

il faut comprendre :

> `null` représente l'absence de centre de coûts. Dans ce cas, nous construisons une map vide afin de ne pas créer un tag Azure vide.

---

# 102. Validation du LAB-03

Le Lab est terminé lorsque les points suivants sont compris et testés :

```text
[ ] Comprendre le rôle d'une variable

[ ] Comprendre le rôle d'un local

[ ] Comprendre le rôle d'un output

[ ] Utiliser string

[ ] Utiliser number

[ ] Utiliser bool

[ ] Utiliser list(string)

[ ] Comprendre set(string)

[ ] Utiliser map(string)

[ ] Utiliser object(...)

[ ] Comprendre tuple(...)

[ ] Utiliser optional(...)

[ ] Comprendre null

[ ] Utiliser nullable

[ ] Écrire une validation

[ ] Comprendre default

[ ] Utiliser terraform.tfvars

[ ] Comprendre *.auto.tfvars

[ ] Comprendre TF_VAR_*

[ ] Utiliser -var

[ ] Comprendre -var-file

[ ] Comprendre la priorité des sources de variables

[ ] Utiliser terraform console

[ ] Utiliser format()

[ ] Utiliser join()

[ ] Utiliser merge()

[ ] Construire des tags conditionnels

[ ] Comprendre sensitive = true

[ ] Comprendre que sensitive ne supprime pas automatiquement la donnée du state

[ ] Comprendre le principe des valeurs éphémères

[ ] Créer le Resource Group

[ ] Créer le Virtual Network

[ ] Lire les outputs simples

[ ] Lire un output structuré

[ ] Tester l'idempotence

[ ] Effectuer un refactoring sans modification Azure

[ ] Détruire proprement l'infrastructure
```

---

# 103. Progression

Nous avons maintenant couvert :

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
Variables
Locals
Outputs
Types
        │
        ▼
Contrat de données Terraform
```

Nous sommes maintenant capables de structurer correctement les données d'une configuration.

Il manque encore un élément essentiel :

> comprendre précisément comment Terraform organise les relations entre les ressources.

---

# 104. Suite

Dans le prochain Lab, nous allons enrichir le réseau Azure avec :

```text
Virtual Network
      │
      ├── Subnet application
      ├── Subnet private endpoints
      │
      └── Network Security Group
```

Cela nous permettra d'étudier concrètement :

- les dépendances implicites ;
- les dépendances explicites ;
- le graphe Terraform ;
- `depends_on` ;
- les références entre ressources ;
- les cycles de dépendances ;
- l'ordre de création ;
- l'ordre de destruction ;
- le parallélisme.

Le prochain article sera :

**LAB-04 — Dépendances et graphe Terraform**

---

## Références officielles

Les notions étudiées dans ce Lab sont documentées notamment dans :

- HashiCorp Developer — Input Variables ;
- HashiCorp Developer — Variable Block Reference ;
- HashiCorp Developer — Type Constraints ;
- HashiCorp Developer — Local Values ;
- HashiCorp Developer — Output Values ;
- HashiCorp Developer — Manage Sensitive Data ;
- HashiCorp Developer — Terraform CLI Environment Variables ;
- HashiCorp Developer — Terraform Style Guide ;
- Terraform Registry — HashiCorp AzureRM Provider ;
- Microsoft Learn — Terraform sur Microsoft Azure ;
- Microsoft Learn — Azure Virtual Network.

Les mécanismes liés aux variables, outputs, valeurs sensibles et fonctionnalités récentes peuvent évoluer avec les versions de Terraform. Les versions réellement utilisées par le projet doivent toujours être vérifiées avant une utilisation en production.