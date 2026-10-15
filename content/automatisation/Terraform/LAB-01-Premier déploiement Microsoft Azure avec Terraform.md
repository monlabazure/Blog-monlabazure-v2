---
title: "LAB-01 - Premier déploiement Microsoft Azure avec Terraform"
date: 2026-10-06
draft: false
description: "Créer une première ressource Microsoft Azure avec Terraform et comprendre le provider AzureRM, le plan, l'apply, le state et l'idempotence."
summary: "Premier déploiement Terraform sur Azure : Resource Group, variables, locals, outputs, plan, state, vérifications et nettoyage."
slug: "terraform-azure-lab-01-premier-deploiement"
categories:
  - "Automatisation"
tags:
  - "Terraform"
  - "Azure"
  - "Infrastructure as Code"
  - "AzureRM"
  - "HCL"
series:
  - "Terraform Azure - De zéro à production"
weight: 2
ShowToc: false
TocOpen: false
---


Dans le LAB-00, nous avons préparé le poste de travail avec Terraform, Azure CLI, Git et Visual Studio Code.

Nous allons maintenant utiliser Terraform pour créer notre première ressource dans Microsoft Azure.

L'infrastructure sera volontairement simple : un seul Resource Group.

L'objectif n'est pas de montrer qu'un Resource Group peut être créé en quelques lignes. L'objectif est de comprendre ce qui se passe entre notre configuration, Terraform, le provider AzureRM, le state et Azure.

À la fin de ce Lab, nous aurons parcouru pour la première fois le cycle complet :

```text
Configuration HCL
      │
      ▼
terraform init
      │
      ▼
terraform validate
      │
      ▼
terraform plan
      │
      ▼
terraform apply
      │
      ▼
Microsoft Azure
      │
      ▼
Terraform State
```

---

## 1. Objectifs du Lab

Nous allons apprendre à :

- écrire une première configuration Terraform ;
- comprendre le rôle du provider AzureRM ;
- déclarer une ressource Azure ;
- utiliser des variables ;
- utiliser des valeurs locales avec `locals` ;
- récupérer des informations avec des outputs ;
- initialiser un répertoire Terraform ;
- formater et valider une configuration ;
- générer un plan ;
- appliquer un plan sauvegardé ;
- observer le Terraform State ;
- vérifier la ressource directement dans Azure ;
- constater l'idempotence de Terraform ;
- effectuer un premier changement contrôlé ;
- détruire proprement l'infrastructure.

Nous commencerons également à distinguer trois éléments fondamentaux :

```text
CONFIGURATION
Ce que nous voulons

STATE
Ce que Terraform connaît et gère

AZURE
Ce qui existe réellement
```

Cette distinction accompagnera toute la série.

---

## 2. Architecture du Lab

Nous allons construire uniquement ceci :

```text
Poste Windows
│
├── Terraform CLI
│
├── Azure CLI
│
└── Configuration HCL
        │
        ▼
   Provider AzureRM
        │
        ▼
Microsoft Azure
        │
        ▼
Subscription
        │
        ▼
rg-mla-dev-fce-001
```

Le Resource Group sera créé dans la région :

```text
France Central
```

et portera le nom :

```text
rg-mla-dev-fce-001
```

La convention utilisée est :

```text
rg-<projet>-<environnement>-<region>-<instance>
```

Soit :

```text
rg
│
│   mla
│    │
│    │   dev
│    │    │
│    │    │   fce
│    │    │    │
│    │    │    │   001
▼    ▼    ▼    ▼    ▼
rg-mla-dev-fce-001
```

Cette convention sera enrichie au fur et à mesure de la série.

---

## 3. Prérequis

Le LAB-00 doit être terminé.

Les commandes suivantes doivent fonctionner :

```powershell
terraform version
az version
git --version
code --version
```

L'authentification Azure doit également être fonctionnelle :

```powershell
az account show
```

Vérifier la subscription sélectionnée :

```powershell
az account show `
  --query "{Subscription:name,SubscriptionId:id,TenantId:tenantId}" `
  --output table
```

Ne pas continuer tant que la subscription affichée n'est pas celle prévue pour le Lab.

---

## 4. Préparer le contexte Azure

Dans le LAB-00, nous avons utilisé Azure CLI pour nous authentifier :

```powershell
az login
```

Terraform doit également savoir dans quelle subscription travailler.

Définir explicitement :

```powershell
$env:ARM_SUBSCRIPTION_ID = az account show --query id -o tsv
```

Puis :

```powershell
$env:ARM_TENANT_ID = az account show --query tenantId -o tsv
```

Vérifier :

```powershell
$env:ARM_SUBSCRIPTION_ID
$env:ARM_TENANT_ID
```

Ces variables sont définies uniquement dans la session PowerShell actuelle.

Elles ne sont donc pas conservées après fermeture du terminal.

Dans ce Lab, c'est volontaire.

---

## 5. Revenir dans le projet

Le projet créé dans le LAB-00 est :

```text
C:\Labs\MonLabAzure\terraform-azure-platform
```

Ouvrir PowerShell :

```powershell
Set-Location "C:\Labs\MonLabAzure\terraform-azure-platform"
```

Puis :

```powershell
Set-Location ".\lab-01"
```

Ouvrir Visual Studio Code :

```powershell
code .
```

---

## 6. Arborescence cible

À la fin de la préparation, le dossier contiendra :

```text
lab-01/
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

Après `terraform init`, Terraform ajoutera également :

```text
.terraform/
.terraform.lock.hcl
```

Après le premier `apply`, nous verrons apparaître :

```text
terraform.tfstate
```

Nous reviendrons sur chacun de ces éléments.

---

# 7. `versions.tf` - Déclarer Terraform et AzureRM

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

Ce fichier répond à deux questions différentes.

### Quelle version de Terraform peut exécuter ce projet ?

```hcl
required_version = ">= 1.16.0, < 2.0.0"
```

Nous acceptons ici la branche Terraform 1.x à partir de 1.16.

Nous n'autorisons pas automatiquement une future version majeure 2.x.

Une montée de version majeure doit être testée avant d'être utilisée.

### Quel provider devons-nous utiliser ?

```hcl
required_providers {
  azurerm = {
    source  = "hashicorp/azurerm"
    version = "~> 5.0"
  }
}
```

Terraform lui-même ne contient pas toute la logique permettant de gérer Azure.

Il s'appuie sur un plugin :

```text
Terraform
    │
    ▼
Provider AzureRM
    │
    ▼
Azure APIs
```

Le provider AzureRM traduit donc une partie des intentions exprimées dans notre configuration Terraform en opérations compréhensibles par les API Azure.

---

## 8. Contrainte de version et version réellement utilisée

Il faut distinguer :

```hcl
version = "~> 5.0"
```

de la version précise qui sera réellement installée.

La contrainte définit les versions acceptables.

Après l'initialisation, Terraform crée :

```text
.terraform.lock.hcl
```

Le lock file enregistre notamment la version du provider effectivement sélectionnée.

Nous versionnerons ce fichier dans Git.

En pratique :

```text
versions.tf
     │
     │ définit les versions autorisées
     ▼
terraform init
     │
     │ choisit une version compatible
     ▼
.terraform.lock.hcl
     │
     │ mémorise la sélection
     ▼
Exécutions suivantes
```

---

# 9. `providers.tf` - Configurer AzureRM

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

Le bloc :

```hcl
provider "azurerm"
```

configure le provider que nous avons déclaré dans `versions.tf`.

Le bloc :

```hcl
features {}
```

est propre au provider AzureRM.

Pour ce Lab, nous conservons son comportement par défaut.

L'authentification repose sur le contexte Azure préparé précédemment.

Nous n'écrivons donc pas de :

```text
client_secret
password
access_key
```

dans le code Terraform.

---

# 10. `variables.tf` - Définir les entrées

Créer :

```text
variables.tf
```

avec :

```hcl
variable "project_name" {
  description = "Nom court du projet utilisé dans les conventions de nommage."
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
  description = "Région Azure utilisée pour les ressources."
  type        = string
  default     = "francecentral"

  validation {
    condition = contains(
      ["francecentral", "westeurope"],
      lower(var.location)
    )

    error_message = "location doit être francecentral ou westeurope pour ce Lab."
  }
}

variable "owner" {
  description = "Équipe responsable des ressources."
  type        = string
  default     = "cloud-platform"
}
```

Une variable représente une **entrée** de notre configuration.

Nous pouvons représenter cette relation ainsi :

```text
Valeurs externes
      │
      ▼
variables
      │
      ▼
configuration Terraform
```

Par exemple :

```hcl
var.environment
```

représentera :

```text
dev
```

---

## 11. Pourquoi typer les variables ?

Nous avons utilisé :

```hcl
type = string
```

Terraform sait donc que la valeur attendue doit être une chaîne de caractères.

Dans des Labs futurs, nous utiliserons également :

```text
bool
number
list
set
map
object
```

Le typage permet de détecter certaines erreurs avant de commencer le déploiement Azure.

---

## 12. Pourquoi utiliser une validation ?

Pour `environment`, nous autorisons uniquement :

```text
dev
qual
prod
```

Une valeur comme :

```text
production-test
```

sera refusée.

La validation ne garantit pas que toute notre architecture est correcte.

Elle permet simplement d'empêcher certaines entrées que nous considérons invalides.

---

# 13. `locals.tf` - Construire nos valeurs internes

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

  resource_group_name = "rg-${var.project_name}-${local.environment}-${local.location_code}-001"

  common_tags = {
    environment = local.environment
    project     = var.project_name
    managed_by  = "terraform"
    owner       = var.owner
    lab         = "lab-01"
  }
}
```

Nous introduisons ici les valeurs locales.

Une distinction utile est :

```text
variable
   │
   └── entrée venant de l'extérieur

local
   │
   └── valeur calculée ou centralisée à l'intérieur du module
```

Nous ne demandons donc pas à l'utilisateur d'écrire directement :

```text
rg-mla-dev-fce-001
```

Terraform le construit à partir de plusieurs informations.

---

## 14. Construction du nom

Cette expression :

```hcl
resource_group_name = "rg-${var.project_name}-${local.environment}-${local.location_code}-001"
```

combine plusieurs valeurs.

Avec :

```text
project_name = mla
environment  = dev
location     = francecentral
```

Terraform calcule :

```text
rg-mla-dev-fce-001
```

Cela commence à nous permettre de centraliser nos conventions plutôt que de saisir manuellement des noms différents dans chaque ressource.

---

# 15. `main.tf` - Notre première ressource

Créer :

```text
main.tf
```

avec :

```hcl
resource "azurerm_resource_group" "main" {
  name     = local.resource_group_name
  location = local.location

  tags = local.common_tags
}
```

C'est notre première vraie déclaration de ressource Terraform.

Décomposons :

```hcl
resource "azurerm_resource_group" "main"
```

### `resource`

Indique que Terraform doit gérer le cycle de vie d'une ressource.

### `azurerm_resource_group`

C'est le type de ressource fourni par AzureRM.

### `main`

C'est le nom logique que nous avons choisi à l'intérieur de notre configuration Terraform.

Terraform peut donc identifier cette ressource avec l'adresse :

```text
azurerm_resource_group.main
```

---

## 16. Adresse Terraform et nom Azure

Il faut immédiatement distinguer deux éléments.

### Adresse Terraform

```text
azurerm_resource_group.main
```

### Nom dans Azure

```text
rg-mla-dev-fce-001
```

Ils n'ont pas le même rôle.

L'adresse Terraform sert notamment à identifier la ressource dans :

- la configuration ;
- le state ;
- les dépendances ;
- les commandes Terraform.

Le nom Azure identifie la ressource dans Microsoft Azure.

Cette distinction deviendra essentielle avec les modules et le state.

---

# 17. `outputs.tf` - Exposer les résultats utiles

Créer :

```text
outputs.tf
```

avec :

```hcl
output "resource_group_id" {
  description = "Identifiant Azure du Resource Group."
  value       = azurerm_resource_group.main.id
}

output "resource_group_name" {
  description = "Nom du Resource Group."
  value       = azurerm_resource_group.main.name
}

output "resource_group_location" {
  description = "Région Azure du Resource Group."
  value       = azurerm_resource_group.main.location
}

output "resource_group_tags" {
  description = "Tags appliqués au Resource Group."
  value       = azurerm_resource_group.main.tags
}
```

Un output permet à notre configuration d'exposer une information.

Par exemple :

```hcl
azurerm_resource_group.main.id
```

signifie :

```text
azurerm_resource_group
          │
          ▼
         main
          │
          ▼
          id
```

Terraform récupérera l'identifiant réel attribué par Azure après la création de la ressource.

---

# 18. `terraform.tfvars.example`

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
```

Ce fichier sert de modèle versionnable.

Créer ensuite le fichier réellement utilisé :

```powershell
Copy-Item `
  ".\terraform.tfvars.example" `
  ".\terraform.tfvars"
```

Nous obtenons :

```text
terraform.tfvars.example
        │
        └── modèle versionné

terraform.tfvars
        │
        └── valeurs locales utilisées par Terraform
```

Dans le LAB-00, nous avons choisi une règle `.gitignore` conservatrice qui ignore les fichiers `*.tfvars`.

Un fichier `.tfvars` n'est pas intrinsèquement secret.

Il peut parfaitement être versionné lorsqu'il contient uniquement de la configuration non sensible.

Nous étudierons cette décision plus précisément lorsque nous construirons DEV, QUALIF et PROD.

Pour le moment, le fichier local reste ignoré.

---

# 19. Configuration finale

Nous avons maintenant :

```text
lab-01/
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

Avant de continuer, enregistrer tous les fichiers.

---

# 20. Formater le code

Exécuter :

```powershell
terraform fmt -recursive
```

Cette commande normalise la présentation du code HCL.

Elle peut modifier automatiquement certains fichiers.

Vérifier ensuite :

```powershell
git diff
```

`terraform fmt` contrôle principalement le format.

Il ne vérifie pas :

- les permissions Azure ;
- la sécurité de l'architecture ;
- les coûts ;
- la cohérence fonctionnelle ;
- l'existence d'une ressource distante ;
- la conformité à nos règles d'entreprise.

Il ne faut donc pas confondre :

```text
Code formaté
```

avec :

```text
Infrastructure correcte
```

---

# 21. Initialiser Terraform

Exécuter :

```powershell
terraform init
```

Terraform va notamment :

1. lire `versions.tf` ;
2. identifier le provider `hashicorp/azurerm` ;
3. rechercher une version compatible ;
4. télécharger le provider ;
5. créer le répertoire `.terraform` ;
6. créer ou mettre à jour `.terraform.lock.hcl`.

L'arborescence contient maintenant :

```text
lab-01/
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

## 22. `.terraform` et `.terraform.lock.hcl` ne jouent pas le même rôle

### `.terraform/`

Contient des données de travail locales, notamment les providers téléchargés.

Nous ne versionnons pas ce dossier.

### `.terraform.lock.hcl`

Enregistre les sélections de providers et leurs informations de vérification.

Nous versionnons ce fichier.

Cette distinction est importante :

```text
.terraform/
     │
     └── cache / données locales

.terraform.lock.hcl
     │
     └── contrôle des dépendances
```

---

# 23. Vérifier le provider utilisé

Exécuter :

```powershell
terraform providers
```

Terraform doit afficher une dépendance vers :

```text
registry.terraform.io/hashicorp/azurerm
```

Vérifier également :

```powershell
terraform version
```

Terraform affiche notamment la version du CLI et les providers utilisés par la configuration.

---

# 24. Valider la configuration

Exécuter :

```powershell
terraform validate
```

Résultat attendu :

```text
Success! The configuration is valid.
```

Ce message signifie que Terraform considère la configuration comme syntaxiquement et structurellement valide dans le contexte qu'il peut analyser.

Il ne signifie pas :

```text
La production est sécurisée.
```

Il ne signifie pas non plus :

```text
Azure acceptera nécessairement le déploiement.
```

Par exemple, une erreur de Role-Based Access Control (RBAC) peut encore apparaître pendant les opérations réelles.

---

# 25. Générer notre premier plan

Exécuter :

```powershell
terraform plan -out="lab-01.tfplan"
```

Terraform doit proposer quelque chose proche de :

```text
Plan: 1 to add, 0 to change, 0 to destroy.
```

Nous avons :

```text
1 to add
```

car le Resource Group n'existe pas encore dans l'état géré par cette configuration.

Nous n'avons :

```text
0 to change
0 to destroy
```

car aucune ressource existante n'a besoin d'être modifiée ou supprimée.

---

# 26. Comprendre `terraform plan`

Conceptuellement, Terraform raisonne autour de plusieurs sources d'information :

```text
Configuration Terraform
          │
          │
Terraform State
          │
          │
Infrastructure distante
          │
          ▼
   Terraform Plan
          │
          ▼
Actions proposées
```

Il faut éviter de résumer `terraform plan` à :

> montrer ce que Terraform va faire.

Le plan constitue surtout le résultat du raisonnement de Terraform à un instant donné.

Dans une chaîne CI/CD professionnelle, ce plan deviendra un élément essentiel de la revue avant déploiement.

---

# 27. Sauvegarder le plan

Nous avons utilisé :

```powershell
terraform plan -out="lab-01.tfplan"
```

au lieu de simplement :

```powershell
terraform plan
```

Cela produit un plan sauvegardé.

Nous pouvons l'inspecter :

```powershell
terraform show ".\lab-01.tfplan"
```

Puis appliquer précisément ce plan.

Attention : un plan sauvegardé peut contenir des valeurs de configuration, y compris des informations sensibles selon l'infrastructure concernée.

C'est pour cette raison que notre `.gitignore` contient :

```gitignore
*.tfplan
```

Un plan Terraform ne doit pas être considéré comme un simple fichier texte sans risque.

---

# 28. Premier `terraform apply`

Après avoir lu le plan :

```powershell
terraform apply ".\lab-01.tfplan"
```

Résultat attendu :

```text
Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

Lorsque nous appliquons un plan sauvegardé, Terraform exécute directement le plan fourni.

Il ne redemande pas l'approbation interactive comme lors d'un `terraform apply` classique sans plan sauvegardé.

C'est précisément cette séparation :

```text
PLAN
  │
  ▼
REVIEW
  │
  ▼
APPLY
```

qui nous intéressera plus tard dans GitHub Actions.

---

# 29. Observer les outputs

Exécuter :

```powershell
terraform output
```

Nous devons retrouver les informations définies dans :

```text
outputs.tf
```

Pour récupérer uniquement le nom :

```powershell
terraform output -raw resource_group_name
```

Nous pouvons l'enregistrer dans PowerShell :

```powershell
$rgName = terraform output -raw resource_group_name
```

Puis :

```powershell
$rgName
```

Résultat :

```text
rg-mla-dev-fce-001
```

---

# 30. Vérifier Azure

Utiliser Azure CLI :

```powershell
az group show `
  --name $rgName `
  --output table
```

Vérifier les tags :

```powershell
az group show `
  --name $rgName `
  --query tags `
  --output json
```

Nous devons retrouver :

```json
{
  "environment": "dev",
  "lab": "lab-01",
  "managed_by": "terraform",
  "owner": "cloud-platform",
  "project": "mla"
}
```

Nous avons donc vérifié la ressource avec un outil différent de Terraform.

C'est une bonne habitude de troubleshooting.

---

# 31. Observer le Terraform State

Exécuter :

```powershell
terraform state list
```

Résultat :

```text
azurerm_resource_group.main
```

Cette adresse correspond exactement à la ressource déclarée :

```hcl
resource "azurerm_resource_group" "main"
```

Nous pouvons obtenir davantage d'informations :

```powershell
terraform state show azurerm_resource_group.main
```

Terraform connaît maintenant notamment :

- le nom ;
- la région ;
- les tags ;
- l'identifiant Azure de la ressource.

---

## 32. Le premier `terraform.tfstate`

Après l'`apply`, Terraform a créé :

```text
terraform.tfstate
```

Notre architecture actuelle ressemble donc à :

```text
versions.tf
providers.tf
variables.tf
locals.tf
main.tf
outputs.tf
        │
        ▼
   Terraform CLI
        │
        ├──────────────► Microsoft Azure
        │
        ▼
terraform.tfstate
```

Pour le moment, le state est local.

Cela convient pour observer Terraform pendant ce premier Lab.

Ce n'est pas notre architecture cible pour une équipe.

Nous traiterons spécifiquement :

- remote state ;
- backend Azure ;
- locking ;
- chiffrement ;
- droits d'accès ;
- séparation des states ;
- récupération après incident ;

dans les Labs consacrés au state.

---

# 33. Tester l'idempotence

Sans modifier le code, exécuter :

```powershell
terraform plan
```

Le résultat attendu est :

```text
No changes.
Your infrastructure matches the configuration.
```

Pourquoi Terraform ne recrée-t-il pas le Resource Group ?

Parce que son objectif n'est pas :

```text
exécuter toujours CREATE
```

mais de faire converger l'infrastructure vers l'état déclaré.

Nous avons actuellement :

```text
Configuration
     │
     │
     ├────────── rg-mla-dev-fce-001
     │
State│
     ├────────── rg-mla-dev-fce-001
     │
Azure│
     └────────── rg-mla-dev-fce-001
```

Il n'y a donc aucun changement nécessaire.

C'est une première manifestation du comportement déclaratif de Terraform.

---

# 34. Effectuer notre premier changement

Nous allons ajouter un tag.

Dans :

```text
locals.tf
```

remplacer :

```hcl
common_tags = {
  environment = local.environment
  project     = var.project_name
  managed_by  = "terraform"
  owner       = var.owner
  lab         = "lab-01"
}
```

par :

```hcl
common_tags = {
  environment = local.environment
  project     = var.project_name
  managed_by  = "terraform"
  owner       = var.owner
  lab         = "lab-01"
  cost_center = "training"
}
```

Puis :

```powershell
terraform fmt
terraform validate
terraform plan
```

Terraform devrait proposer approximativement :

```text
Plan: 0 to add, 1 to change, 0 to destroy.
```

Le Resource Group n'a pas besoin d'être recréé.

Seuls les tags doivent être mis à jour.

---

# 35. Lire le plan avant de l'appliquer

Ne pas lancer immédiatement `apply`.

Observer le plan.

Terraform utilise notamment les symboles :

```text
+ create
~ update in-place
- destroy
-/+ replace
```

Pour notre changement de tag, nous devons nous attendre à une modification en place :

```text
~
```

et non à :

```text
-/+
```

Si Terraform proposait de remplacer une ressource importante alors que nous pensions effectuer une simple modification de tag, ce serait un signal d'alerte.

La lecture du plan fait partie du travail.

---

# 36. Appliquer le changement

Créer un plan sauvegardé :

```powershell
terraform plan -out="lab-01-update.tfplan"
```

Puis :

```powershell
terraform apply ".\lab-01-update.tfplan"
```

Vérifier :

```powershell
az group show `
  --name $rgName `
  --query tags `
  --output json
```

Le tag suivant doit apparaître :

```text
cost_center = training
```

---

# 37. Ce que Terraform vient réellement de faire

Nous n'avons pas demandé :

```text
Ajoute le tag cost_center.
```

Nous avons modifié l'état désiré :

```hcl
cost_center = "training"
```

Terraform a ensuite calculé la différence entre l'état désiré et la ressource existante.

Conceptuellement :

```text
AVANT

Configuration
cost_center absent

Azure
cost_center absent


MODIFICATION DU CODE

Configuration
cost_center = training


PLAN

Terraform détecte la différence


APPLY

Azure
cost_center = training
```

C'est l'une des idées fondamentales de l'Infrastructure as Code (IaC).

---

# 38. Configuration, state et Azure

À partir de ce Lab, il faut prendre l'habitude de raisonner avec ces trois éléments.

```text
┌───────────────────────┐
│     CONFIGURATION     │
│                       │
│ Ce que nous déclarons │
│ dans les fichiers .tf │
└───────────┬───────────┘
            │
            ▼
┌───────────────────────┐
│         STATE         │
│                       │
│ Ce que Terraform      │
│ connaît et associe    │
└───────────┬───────────┘
            │
            ▼
┌───────────────────────┐
│         AZURE         │
│                       │
│ Ce qui existe         │
│ réellement            │
└───────────────────────┘
```

Cette représentation est volontairement simplifiée.

Terraform ne travaille pas simplement sur une copie statique du state.

Lors de certaines opérations, il interroge également les systèmes distants via le provider.

Nous approfondirons ce fonctionnement dans le Lab consacré au state.

---

# 39. Ce que `terraform validate` ne détecte pas

Une configuration peut réussir :

```powershell
terraform validate
```

et échouer ensuite pendant :

```powershell
terraform plan
```

ou :

```powershell
terraform apply
```

Quelques exemples :

```text
Configuration HCL correcte
          │
          ├── mais aucun accès à Azure
          ├── mauvaise subscription
          ├── permissions RBAC insuffisantes
          ├── politique Azure bloquante
          └── erreur retournée par une API Azure
```

Il faut donc penser en plusieurs niveaux de contrôle.

```text
terraform fmt
      ↓
Format

terraform validate
      ↓
Structure Terraform

terraform plan
      ↓
Analyse du changement

terraform apply
      ↓
Exécution réelle

Vérifications Azure
      ↓
Validation du résultat
```

Aucun de ces contrôles ne remplace complètement les autres.

---

# 40. Versionner le code

Revenir à la racine du repository :

```powershell
Set-Location "C:\Labs\MonLabAzure\terraform-azure-platform"
```

Vérifier :

```powershell
git status
```

Nous devons notamment pouvoir versionner :

```text
lab-01/versions.tf
lab-01/providers.tf
lab-01/variables.tf
lab-01/locals.tf
lab-01/main.tf
lab-01/outputs.tf
lab-01/terraform.tfvars.example
lab-01/.terraform.lock.hcl
```

Nous ne devons pas retrouver dans le commit :

```text
.terraform/
terraform.tfstate
terraform.tfstate.backup
terraform.tfvars
*.tfplan
```

Ajouter les fichiers :

```powershell
git add .
```

Puis contrôler ce qui sera réellement commité :

```powershell
git diff --cached --name-only
```

Cette commande est importante.

Ne pas considérer :

```powershell
git add .
```

comme une opération sans risque.

Toujours vérifier ce qui va être publié.

---

## 41. Créer le commit

Si la liste est correcte :

```powershell
git commit -m "feat(terraform): add lab 01 resource group"
```

Puis :

```powershell
git status
```

Notre code Terraform est maintenant versionné localement.

Nous n'utilisons pas encore GitHub.

Cette étape viendra plus tard avec :

```text
branch
   ↓
Pull Request
   ↓
CI
   ↓
terraform plan
   ↓
review
   ↓
approval
   ↓
apply
```

---

# 42. Nettoyer l'infrastructure

Une infrastructure de Lab ne doit pas rester active inutilement.

Revenir dans :

```powershell
Set-Location "C:\Labs\MonLabAzure\terraform-azure-platform\lab-01"
```

Créer un plan de destruction :

```powershell
terraform plan `
  -destroy `
  -out="lab-01-destroy.tfplan"
```

Lire le plan :

```powershell
terraform show ".\lab-01-destroy.tfplan"
```

Le résultat attendu doit concerner uniquement notre Resource Group :

```text
Plan: 0 to add, 0 to change, 1 to destroy.
```

Si Terraform propose de supprimer autre chose, ne pas continuer.

---

# 43. Appliquer le plan de destruction

Une fois le plan vérifié :

```powershell
terraform apply ".\lab-01-destroy.tfplan"
```

Résultat attendu :

```text
Apply complete! Resources: 0 added, 0 changed, 1 destroyed.
```

---

# 44. Vérifier la suppression

Avec Azure CLI :

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

La commande ne doit plus afficher :

```text
azurerm_resource_group.main
```

Notre configuration Terraform existe toujours.

La ressource Azure, elle, a disparu.

---

# 45. Attention à la suppression d'un Resource Group

Un Resource Group Azure est un conteneur logique pour des ressources Azure.

Supprimer un Resource Group peut entraîner la suppression des ressources qu'il contient.

Dans ce Lab, ce comportement est sans conséquence particulière puisque notre Resource Group est vide.

Ce ne sera plus le cas lorsque nous y ajouterons :

- Virtual Network ;
- subnets ;
- Network Security Groups ;
- Private Endpoints ;
- Key Vault ;
- Container Apps ;
- Application Gateway.

À partir de ce moment, un :

```text
destroy
```

devra être analysé avec beaucoup plus de prudence.

---

# 46. État actuel de notre architecture

Nous sommes partis de :

```text
Terraform installé
Azure CLI connecté
Git initialisé
```

Nous disposons maintenant de :

```text
Git repository
│
└── lab-01
     │
     ├── versions.tf
     ├── providers.tf
     ├── variables.tf
     ├── locals.tf
     ├── main.tf
     ├── outputs.tf
     └── .terraform.lock.hcl
             │
             ▼
        Terraform CLI
             │
             ▼
      Provider AzureRM
             │
             ▼
       Microsoft Azure
```

Nous avons également observé notre premier state local.

---

# 47. Ce qui est déjà correct

Plusieurs pratiques utilisées dans ce Lab pourront être conservées :

- version explicite de Terraform ;
- contrainte de version du provider ;
- `.terraform.lock.hcl` versionné ;
- variables typées ;
- validations simples ;
- conventions de nommage centralisées ;
- tags centralisés ;
- outputs documentés ;
- `terraform fmt` ;
- `terraform validate` ;
- plan avant application ;
- vérification de la cible Azure ;
- contrôle de ce qui entre dans Git.

Mais cela ne suffit pas encore pour parler d'une plateforme Terraform de production.

---

# 48. Ce qui n'est pas encore production-ready

Notre architecture comporte volontairement plusieurs limites.

### State local

```text
terraform.tfstate
```

reste sur le poste de l'ingénieur.

Pour une équipe, ce modèle devient rapidement problématique.

---

### Pas de locking partagé

Deux ingénieurs pourraient travailler sans mécanisme de coordination centralisé autour du state.

---

### Authentification humaine

Nous utilisons :

```powershell
az login
```

Cette méthode est adaptée au travail interactif.

Elle ne sera pas utilisée comme modèle d'authentification de notre pipeline.

---

### Pas de séparation DEV / QUALIF / PROD

Nous avons uniquement :

```text
dev
```

La stratégie multi-environnements n'est pas encore construite.

---

### Pas de CI/CD

Terraform est encore exécuté manuellement depuis le poste local.

---

### Pas de revue de Pull Request

Le code n'est pas encore envoyé vers GitHub.

---

### Pas de contrôles de sécurité automatisés

Nous n'avons pas encore de :

```text
linting
security scanning
policy as code
quality gates
```

---

### Pas de modules

Le code appartient uniquement au root module.

Cela est parfaitement acceptable pour une infrastructure aussi petite.

Créer immédiatement plusieurs modules pour un seul Resource Group ajouterait surtout de la complexité.

Nous introduirons les modules lorsque le besoin d'encapsulation et de réutilisation apparaîtra réellement.

---

# 49. Erreurs fréquentes

## `terraform` n'est pas reconnu

Vérifier :

```powershell
Get-Command terraform
```

---

## Azure n'est pas authentifié

Vérifier :

```powershell
az account show
```

Si nécessaire :

```powershell
az login
```

---

## Mauvaise subscription

Vérifier :

```powershell
az account show `
  --query "{Subscription:name,SubscriptionId:id,TenantId:tenantId}" `
  --output table
```

Puis sélectionner explicitement la bonne subscription :

```powershell
az account set --subscription "<SUBSCRIPTION_ID>"
```

---

## `ARM_SUBSCRIPTION_ID` absent

Vérifier :

```powershell
$env:ARM_SUBSCRIPTION_ID
```

Si nécessaire :

```powershell
$env:ARM_SUBSCRIPTION_ID = az account show --query id -o tsv
```

---

## Provider non initialisé

Symptôme possible :

```text
Required plugins are not installed
```

Exécuter :

```powershell
terraform init
```

---

## Code mal formaté

Exécuter :

```powershell
terraform fmt -recursive
```

---

## Configuration invalide

Exécuter :

```powershell
terraform validate
```

Lire l'erreur avant de modifier plusieurs fichiers à la fois.

---

## Permission Azure insuffisante

Dans ce cas, le HCL peut être parfaitement correct.

Il faut distinguer :

```text
Erreur Terraform
```

de :

```text
Erreur d'autorisation Azure
```

Vérifier l'identité et les rôles attribués avant de commencer à modifier le code.

---

# 50. Méthode de troubleshooting à conserver

Lorsqu'un déploiement échoue, éviter de modifier plusieurs choses au hasard.

Commencer par vérifier méthodiquement :

```text
1. Terraform fonctionne-t-il ?
             │
             ▼
2. Azure CLI est-il authentifié ?
             │
             ▼
3. Sommes-nous dans le bon tenant ?
             │
             ▼
4. Sommes-nous dans la bonne subscription ?
             │
             ▼
5. Le provider est-il correctement initialisé ?
             │
             ▼
6. terraform validate réussit-il ?
             │
             ▼
7. Que dit précisément terraform plan ?
             │
             ▼
8. L'identité possède-t-elle les permissions nécessaires ?
             │
             ▼
9. Que retourne Azure ?
```

Le diagnostic fait partie de la compétence Terraform.

---

# 51. Ce qu'il faut retenir

À ce stade, les notions les plus importantes ne sont pas les commandes.

Il faut surtout comprendre les relations suivantes.

### Terraform décrit un état souhaité

```text
Configuration HCL
      │
      ▼
État souhaité
```

---

### Le provider permet à Terraform d'interagir avec Azure

```text
Terraform
    │
    ▼
AzureRM
    │
    ▼
Azure
```

---

### Le plan représente les actions calculées

```text
Configuration
      +
State
      +
Réalité distante
      │
      ▼
Plan
```

---

### Le state établit une relation entre Terraform et les ressources gérées

```text
azurerm_resource_group.main
            │
            ▼
     terraform.tfstate
            │
            ▼
/subscriptions/.../resourceGroups/rg-mla-dev-fce-001
```

---

### `apply` n'est pas le début du raisonnement

La bonne séquence est plutôt :

```text
Écrire
  ↓
Formatter
  ↓
Valider
  ↓
Planifier
  ↓
Lire
  ↓
Appliquer
  ↓
Vérifier
```

---

# 52. Validation du LAB-01

Le Lab est considéré comme terminé lorsque les étapes suivantes ont été réalisées avec succès :

```text
[ ] Configuration Terraform créée

[ ] terraform fmt réussi

[ ] terraform init réussi

[ ] .terraform.lock.hcl généré

[ ] terraform validate réussi

[ ] terraform plan = 1 add

[ ] Resource Group créé

[ ] Vérification avec Azure CLI

[ ] terraform state list exécuté

[ ] terraform state show exécuté

[ ] Deuxième plan sans changement

[ ] Tag cost_center ajouté par Terraform

[ ] Plan = modification en place

[ ] Modification appliquée

[ ] Code ajouté dans Git

[ ] State absent du commit

[ ] Plan files absents du commit

[ ] Resource Group détruit proprement

[ ] Suppression vérifiée dans Azure
```

---

# 53. Suite

Nous savons maintenant utiliser Terraform pour créer une première ressource.

Mais nous avons volontairement utilisé plusieurs éléments sans encore les étudier profondément :

```text
resource
variable
local
output
reference
expression
provider
state
```

Le prochain Lab va donc revenir sur le langage lui-même.

Nous allons comprendre comment Terraform interprète le HashiCorp Configuration Language (HCL), comment fonctionne son modèle déclaratif et comment les références créent progressivement un graphe de dépendances.

**LAB-02 - Comprendre HCL et le modèle déclaratif de Terraform**

---

## Références officielles

Pour approfondir les mécanismes utilisés dans ce Lab :

- HashiCorp Developer - Terraform Provider Requirements
- HashiCorp Developer - Dependency Lock File
- HashiCorp Developer - `terraform plan`
- HashiCorp Developer - `terraform apply`
- Terraform Registry - HashiCorp AzureRM Provider
- Terraform Registry - `azurerm_resource_group`
- Microsoft Learn - Terraform sur Microsoft Azure
- Microsoft Learn - Authentification Azure CLI et Terraform

Avant toute utilisation en production, vérifier la documentation correspondant aux versions réellement utilisées par le projet.