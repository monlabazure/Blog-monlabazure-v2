---
title: "LAB-00 - Préparer son environnement Terraform pour Microsoft Azure"
date: 2026-10-06
draft: false
description: "Préparer un poste Windows propre pour travailler avec Terraform, Microsoft Azure, Azure CLI, Git et Visual Studio Code avant de commencer les déploiements Infrastructure as Code."
summary: "Installation, configuration et validation d'un environnement de travail Terraform Azure sous Windows."
slug: "terraform-azure-lab-00-preparer-environnement"
categories:
  - "Automatisation"
tags:
  - "Terraform"
  - "Azure"
  - "Infrastructure as Code"
  - "Azure CLI"
  - "Git"
  - "Visual Studio Code"
series:
  - "Terraform Azure - De zéro à production"
weight: 1
ShowToc: false
TocOpen: false
---


Avant d'écrire la première ressource Terraform, il faut disposer d'un environnement de travail propre et surtout savoir exactement **quel outil intervient à quel moment**.

L'objectif de ce premier Lab n'est donc pas encore de créer une infrastructure Azure.

Nous allons préparer la chaîne de travail qui sera utilisée pendant toute la série :

```text
Développeur
    │
    ▼
Visual Studio Code
    │
    ├── Git
    ├── Terraform CLI
    └── Azure CLI
            │
            ▼
       Microsoft Entra ID
            │
            ▼
      Subscription Azure
```

À la fin du Lab, le poste devra être capable :

- d'exécuter Terraform ;
- de communiquer avec Microsoft Azure ;
- de sélectionner explicitement une subscription ;
- d'utiliser Git ;
- d'éditer correctement du HashiCorp Configuration Language (HCL) ;
- de préparer le dépôt qui servira de fil rouge à toute la série.

---

## 1. Ce que nous allons utiliser

La série repose principalement sur la stack suivante :

```text
Git
  ↓
GitHub
  ↓
GitHub Actions
  ↓
Terraform
  ↓
Microsoft Azure
  ↓
Microsoft Entra ID
  ↓
Azure Key Vault
```

Nous n'allons pas tout mettre en place dans ce premier Lab.

Le principe sera d'ajouter chaque composant lorsque son utilité devient concrète.

Pour commencer, quatre outils suffisent :

| Outil | Rôle |
|---|---|
| Terraform CLI | Lire la configuration Terraform, construire un plan et appliquer les changements |
| Azure CLI | S'authentifier et interagir avec Azure depuis le terminal |
| Git | Versionner le code Infrastructure as Code |
| Visual Studio Code | Écrire et analyser les fichiers Terraform |

---

## 2. Versions utilisées dans la série

La série est construite autour de versions modernes de Terraform et du provider AzureRM.

L'objectif n'est cependant pas de dépendre inutilement d'un numéro de patch précis.

Nous utiliserons plus tard des contraintes de versions de ce type :

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

Cette approche signifie notamment :

- Terraform 1.16 ou une version compatible supérieure est autorisé ;
- Terraform 2.x n'est pas accepté automatiquement ;
- le provider AzureRM reste dans la branche majeure 5.x.

Le verrouillage précis des versions sera ensuite renforcé avec :

```text
.terraform.lock.hcl
```

> Une version plus récente n'est pas automatiquement synonyme de meilleure décision en production. Une montée de version doit être testée, particulièrement lorsqu'elle concerne une version majeure d'un provider.

---

## 3. Vérifier Windows Package Manager

Sur Windows 11, nous pouvons utiliser Windows Package Manager (`winget`) pour installer plusieurs outils.

Ouvrir PowerShell :

```powershell
winget --version
```

Si la commande retourne une version, `winget` est disponible.

---

## 4. Installer Terraform

### Installation avec WinGet

```powershell
winget install --exact --id Hashicorp.Terraform
```

Fermer puis rouvrir le terminal après l'installation.

Vérifier :

```powershell
terraform version
```

Exemple :

```text
Terraform v1.x.x
on windows_amd64
```

La commande importante ici n'est pas le numéro exact retourné mais le fait que :

```text
terraform.exe
```

soit accessible depuis le `PATH` Windows.

### Localiser Terraform

```powershell
Get-Command terraform
```

Puis :

```powershell
Get-Command terraform | Select-Object Source
```

Cela permet de savoir précisément quel exécutable est appelé lorsque nous lançons :

```powershell
terraform
```

C'est particulièrement utile lorsqu'un poste possède plusieurs versions de Terraform.

---

## 5. Installer Azure CLI

Azure Command-Line Interface (Azure CLI) sera utilisée pour l'authentification locale et pour certaines vérifications Azure.

Installation :

```powershell
winget install --exact --id Microsoft.AzureCLI
```

Fermer puis rouvrir PowerShell.

Vérifier :

```powershell
az version
```

Puis :

```powershell
az --version
```

---

## 6. Installer Git

Installation :

```powershell
winget install --exact --id Git.Git
```

Vérifier :

```powershell
git --version
```

Git n'est pas nécessaire au fonctionnement interne de Terraform.

Terraform peut parfaitement être exécuté depuis un simple dossier sans Git.

En revanche, ce serait une très mauvaise base pour une utilisation professionnelle.

Le code d'infrastructure doit pouvoir être :

- versionné ;
- relu ;
- comparé ;
- audité ;
- restauré ;
- soumis à Pull Request ;
- intégré à une chaîne de Continuous Integration / Continuous Delivery (CI/CD).

Git deviendra donc rapidement un composant central de notre architecture.

---

## 7. Installer Visual Studio Code

Installation :

```powershell
winget install --exact --id Microsoft.VisualStudioCode
```

Vérifier :

```powershell
code --version
```

---

## 8. Installer l'extension Terraform

Depuis PowerShell :

```powershell
code --install-extension hashicorp.terraform
```

Installer également l'extension PowerShell :

```powershell
code --install-extension ms-vscode.powershell
```

Vérifier les extensions :

```powershell
code --list-extensions
```

Nous devons notamment retrouver :

```text
hashicorp.terraform
ms-vscode.powershell
```

L'extension Terraform améliore notamment :

- la coloration HCL ;
- l'autocomplétion ;
- la navigation dans la configuration ;
- la détection de certaines erreurs ;
- le formatage.

Elle ne remplace cependant pas :

```powershell
terraform fmt
terraform validate
terraform plan
```

L'éditeur aide le développeur.

Terraform reste l'autorité pour interpréter la configuration Terraform.

---

## 9. Vérification globale du poste

Exécuter :

```powershell
terraform version
az version
git --version
code --version
```

À ce stade, les quatre outils doivent répondre correctement.

Architecture obtenue :

```text
Windows 11
│
├── Visual Studio Code
│      └── Extension HashiCorp Terraform
│
├── Terraform CLI
│
├── Azure CLI
│
└── Git
```

---

## 10. Se connecter à Microsoft Azure

Pour le travail interactif depuis un poste local, nous allons utiliser Azure CLI.

Exécuter :

```powershell
az login
```

Azure ouvre le processus d'authentification.

Une fois connecté, vérifier le contexte :

```powershell
az account show
```

Pour une sortie plus lisible :

```powershell
az account show `
  --query "{Subscription:name,SubscriptionId:id,TenantId:tenantId}" `
  --output table
```

---

## 11. Comprendre le contexte Azure

Une authentification Azure ne suffit pas.

Un compte peut avoir accès à :

- plusieurs tenants ;
- plusieurs subscriptions ;
- plusieurs environnements.

Terraform doit travailler sur la bonne cible.

Lister les subscriptions disponibles :

```powershell
az account list --output table
```

Une erreur de subscription peut être beaucoup plus grave qu'une erreur de syntaxe Terraform.

Dans un environnement réel, lancer Terraform sur la mauvaise subscription peut signifier :

```text
configuration DEV
       │
       │ mauvaise cible
       ▼
subscription PROD
```

Le code peut être parfaitement valide tout en étant appliqué au mauvais endroit.

---

## 12. Sélectionner explicitement la subscription

Utiliser :

```powershell
az account set --subscription "<SUBSCRIPTION_ID>"
```

Puis vérifier immédiatement :

```powershell
az account show `
  --query "{Subscription:name,SubscriptionId:id,TenantId:tenantId}" `
  --output table
```

Cette seconde vérification est importante.

Ne pas considérer que :

```powershell
az account set
```

a sélectionné la bonne cible simplement parce que la commande n'a retourné aucune erreur.

---

## 13. Préparer `ARM_SUBSCRIPTION_ID`

Pour rendre le contexte Terraform plus explicite, récupérer l'identifiant de la subscription sélectionnée :

```powershell
$env:ARM_SUBSCRIPTION_ID = az account show --query id -o tsv
```

Vérifier :

```powershell
$env:ARM_SUBSCRIPTION_ID
```

Nous utilisons ici une variable d'environnement.

Elle est définie uniquement pour la session PowerShell courante.

Fermer le terminal la supprimera.

C'est volontaire pour le moment.

Nous éviterons de commencer la formation en enregistrant globalement des informations de contexte qui pourraient ensuite être utilisées accidentellement depuis un autre projet.

---

## 14. Vérifier le tenant

Récupérer également le tenant actuellement utilisé :

```powershell
$env:ARM_TENANT_ID = az account show --query tenantId -o tsv
```

Puis :

```powershell
$env:ARM_TENANT_ID
```

Notre contexte local contient maintenant :

```text
Utilisateur Azure
      │
      ▼
Microsoft Entra ID tenant
      │
      ▼
Subscription sélectionnée
      │
      ├── ARM_TENANT_ID
      │
      └── ARM_SUBSCRIPTION_ID
```

---

## 15. Terraform et Azure CLI : qui fait quoi ?

Il faut éviter une confusion fréquente.

Azure CLI et Terraform ne remplissent pas la même fonction.

### Azure CLI

Une commande comme :

```powershell
az group create
```

demande directement à Azure d'exécuter une opération.

Nous sommes principalement dans une logique impérative :

```text
"Crée cette ressource."
```

### Terraform

Terraform utilise une configuration qui décrit un état souhaité.

Exemple conceptuel :

```hcl
resource "azurerm_resource_group" "main" {
  name     = "rg-mla-dev-fce-001"
  location = "France Central"
}
```

La logique devient :

```text
"Voici l'état que je souhaite obtenir."
```

Terraform doit ensuite déterminer les actions nécessaires pour converger vers cet état.

Cette différence sera étudiée beaucoup plus précisément dans les prochains Labs.

---

## 16. Vérifier l'accès Azure

Avant d'impliquer Terraform, vérifier qu'Azure CLI peut correctement interroger la subscription :

```powershell
az group list --output table
```

Nous ne créons encore aucune ressource.

Cette commande permet simplement de vérifier que :

```text
Poste local
    │
    ▼
Azure CLI
    │
    ▼
Microsoft Entra ID
    │
    ▼
Azure Resource Manager
```

fonctionne correctement.

---

## 17. Préparer le répertoire du projet

Nous allons maintenant créer le projet fil rouge de la série.

Exemple :

```powershell
New-Item -ItemType Directory -Path "C:\Labs\MonLabAzure" -Force
```

Puis :

```powershell
Set-Location "C:\Labs\MonLabAzure"
```

Créer le projet :

```powershell
New-Item -ItemType Directory -Name "terraform-azure-platform"
Set-Location ".\terraform-azure-platform"
```

Notre emplacement devient :

```text
C:\Labs\MonLabAzure\terraform-azure-platform
```

---

## 18. Initialiser Git

Initialiser le dépôt :

```powershell
git init -b main
```

Vérifier :

```powershell
git status
```

Résultat attendu proche de :

```text
On branch main

No commits yet
```

À ce stade :

```text
terraform-azure-platform/
└── .git/
```

Nous avons un dépôt Git local mais aucun dépôt GitHub.

GitHub sera introduit plus tard afin de bien distinguer :

```text
Git
```

du service :

```text
GitHub
```

Les deux sont liés mais ne désignent pas la même chose.

---

## 19. Créer l'arborescence de travail

Nous allons organiser la série par Labs :

```powershell
New-Item -ItemType Directory -Name "lab-01"
```

Arborescence :

```text
terraform-azure-platform/
│
├── .git/
│
└── lab-01/
```

Le premier vrai code Terraform sera ajouté dans `lab-01`.

Nous ne lançons pas encore :

```powershell
terraform init
```

Il n'existe encore aucune configuration Terraform à initialiser.

---

## 20. Créer le `.gitignore`

À la racine du dépôt :

```powershell
New-Item -ItemType File -Name ".gitignore"
```

Ouvrir le projet :

```powershell
code .
```

Ajouter dans `.gitignore` :

```gitignore
# Terraform working directories
**/.terraform/*

# Terraform state
*.tfstate
*.tfstate.*

# Terraform plan files
*.tfplan

# Crash logs
crash.log
crash.*.log

# Variable files potentially containing sensitive data
*.tfvars
*.tfvars.json

# Local override files
override.tf
override.tf.json
*_override.tf
*_override.tf.json

# Terraform CLI configuration
.terraformrc
terraform.rc

# Operating system files
.DS_Store
Thumbs.db
```

Un élément est volontairement absent :

```text
.terraform.lock.hcl
```

Nous versionnerons ce fichier.

Il permet notamment de conserver les sélections de providers et les checksums associés.

---

## 21. Pourquoi ignorer le Terraform state ?

Nous avons ajouté :

```gitignore
*.tfstate
*.tfstate.*
```

Le state Terraform peut contenir des informations importantes sur l'infrastructure gérée.

Selon les ressources utilisées, il peut également contenir des données sensibles.

Le publier dans Git constituerait donc un risque.

Plus tard, nous remplacerons le state local par un **remote state** protégé dans Azure.

Architecture future :

```text
Terraform
    │
    ▼
Azure Storage Account
    │
    └── Terraform State
```

Avec notamment :

- authentification ;
- contrôle d'accès ;
- Role-Based Access Control (RBAC) ;
- chiffrement ;
- verrouillage ;
- séparation des environnements.

Pour le moment, il suffit de retenir :

> Le state n'est pas un simple fichier temporaire que l'on peut traiter sans précaution.

---

## 22. Vérification Git

Exécuter :

```powershell
git status
```

Nous devons retrouver au minimum :

```text
.gitignore
lab-01/
```

Un dossier vide n'étant pas suivi par Git, `lab-01` peut ne pas apparaître tant qu'aucun fichier n'y a été créé.

C'est un comportement Git normal.

---

## 23. Contrôle final automatisé

Nous pouvons effectuer une dernière vérification depuis PowerShell :

```powershell
$commands = @(
    "terraform",
    "az",
    "git",
    "code"
)

foreach ($command in $commands) {

    if (Get-Command $command -ErrorAction SilentlyContinue) {
        Write-Host "[OK] $command"
    }
    else {
        Write-Host "[ERREUR] $command introuvable"
    }
}
```

Résultat attendu :

```text
[OK] terraform
[OK] az
[OK] git
[OK] code
```

Puis :

```powershell
az account show `
  --query "{Subscription:name,SubscriptionId:id,TenantId:tenantId}" `
  --output table
```

Et :

```powershell
$env:ARM_SUBSCRIPTION_ID
$env:ARM_TENANT_ID
```

---

## 24. État de notre environnement

À la fin du LAB-00, nous disposons de :

```text
Windows 11
│
├── Terraform CLI
│
├── Azure CLI
│      │
│      └── Authentification Microsoft Entra ID
│
├── Git
│      │
│      └── dépôt local
│
├── Visual Studio Code
│      │
│      └── extension Terraform
│
└── Projet
       │
       └── terraform-azure-platform/
```

Nous avons volontairement encore :

```text
Pas de ressources Azure
Pas de Terraform State
Pas de provider AzureRM téléchargé
Pas de backend distant
Pas de modules
Pas de GitHub
Pas de GitHub Actions
Pas d'OIDC
```

C'est normal.

Le but du Lab était uniquement de préparer une base fiable.

---

## 25. Ce qui serait différent en production

L'utilisation de :

```powershell
az login
```

est adaptée à un ingénieur qui travaille interactivement depuis son poste.

Ce n'est pas la méthode que nous utiliserons pour une pipeline automatisée.

Une pipeline GitHub Actions ne doit pas dépendre de la session Azure personnelle d'un ingénieur.

Nous irons progressivement vers :

```text
GitHub Actions
      │
      ▼
OpenID Connect (OIDC)
      │
      ▼
Microsoft Entra ID
      │
      ▼
Workload Identity
      │
      ▼
RBAC Azure
      │
      ▼
Terraform
```

L'objectif sera d'éviter les credentials Azure permanents stockés dans GitHub.

---

## 26. Points de sécurité

Quelques règles sont déjà applicables.

### Ne pas stocker de mot de passe Azure dans un script

À éviter :

```powershell
$user     = "user@example"
$password = "MyPassword123"
```

Nous utilisons l'authentification interactive pour le poste local.

---

### Ne pas versionner le state

Les fichiers :

```text
terraform.tfstate
terraform.tfstate.backup
```

ne doivent pas être publiés dans Git.

---

### Ne pas considérer `.gitignore` comme une protection absolue

Si un secret a déjà été commité, l'ajouter ensuite dans `.gitignore` ne le supprime pas automatiquement de l'historique Git.

Il faut donc éviter le secret **avant le premier commit**.

---

### Toujours vérifier la subscription

Avant une opération Terraform sensible :

```powershell
az account show `
  --query "{Subscription:name,SubscriptionId:id,TenantId:tenantId}" `
  --output table
```

Cette habitude devient particulièrement importante lorsque plusieurs environnements Azure existent.

---

## 27. Ce qu'il faut retenir

Terraform n'est pas encore intervenu sur Azure.

Nous avons volontairement séparé :

```text
installation des outils
        ↓
authentification
        ↓
sélection de la subscription
        ↓
préparation du dépôt
        ↓
déploiement Terraform
```

Cette séparation facilite énormément le diagnostic.

Lorsque quelque chose échoue plus tard, nous pourrons nous demander :

```text
Terraform est-il installé ?

        ↓ oui

L'authentification Azure fonctionne-t-elle ?

        ↓ oui

La bonne subscription est-elle sélectionnée ?

        ↓ oui

Le provider fonctionne-t-il ?

        ↓ oui

La configuration Terraform est-elle correcte ?
```

C'est beaucoup plus efficace que de modifier du code Terraform au hasard.

---

## 28. Validation du LAB-00

Le Lab est terminé lorsque toutes les commandes suivantes fonctionnent :

```powershell
terraform version
az version
git --version
code --version
```

L'authentification Azure fonctionne :

```powershell
az account show
```

La bonne subscription est sélectionnée :

```powershell
az account show `
  --query "{Subscription:name,SubscriptionId:id,TenantId:tenantId}" `
  --output table
```

Et le projet existe :

```text
C:\Labs\MonLabAzure\terraform-azure-platform
```

avec :

```text
terraform-azure-platform/
│
├── .git/
├── .gitignore
└── lab-01/
```

---

## 29. Suite de la série

Dans le prochain Lab, nous allons enfin créer notre première infrastructure :

```text
Terraform configuration
        │
        ▼
AzureRM Provider
        │
        ▼
Microsoft Azure
        │
        ▼
Resource Group
```

Nous introduirons réellement :

- Infrastructure as Code (IaC) ;
- HashiCorp Configuration Language (HCL) ;
- `terraform init` ;
- providers ;
- resources ;
- variables ;
- locals ;
- outputs ;
- `terraform fmt` ;
- `terraform validate` ;
- `terraform plan` ;
- `terraform apply` ;
- Terraform State ;
- idempotence ;
- `terraform destroy`.

Le prochain article sera :

**LAB-01 - Premier déploiement Microsoft Azure avec Terraform**

---

## Références officielles

Les informations et commandes de ce Lab s'appuient principalement sur les documentations officielles suivantes :

- HashiCorp Developer - installation de Terraform ;
- Terraform Registry - provider HashiCorp AzureRM ;
- Microsoft Learn - installation d'Azure CLI sur Windows ;
- Microsoft Learn - authentification Terraform vers Azure ;
- Microsoft Learn - authentification interactive avec Azure CLI.

> Les versions de Terraform, Azure CLI et du provider AzureRM évoluent régulièrement. Avant une utilisation en production, vérifier les versions stables et les éventuels breaking changes dans les documentations officielles.