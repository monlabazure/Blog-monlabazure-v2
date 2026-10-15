---
title: "LAB-19 — Troubleshooting Terraform en conditions réelles"
date: 2026-10-15
draft: false
description: "Diagnostiquer méthodiquement les incidents Terraform : backend, state lock, drift, import, OIDC, RBAC, Private DNS, Container Apps et Application Gateway."
summary: "Construire une méthode de troubleshooting Terraform reproductible, depuis le code et le state jusqu’aux APIs Azure et au réseau."
slug: "terraform-azure-lab-19-troubleshooting"
categories:
  - "Automatisation"
tags:
  - "Terraform"
  - "Azure"
  - "Troubleshooting"
  - "State"
  - "OIDC"
  - "Private DNS"
series:
  - "Terraform Azure — De zéro à production"
weight: 20
ShowToc: true
TocOpen: true
---

# LAB-19 — Troubleshooting Terraform en conditions réelles

Un ingénieur Terraform senior ne se distingue pas uniquement par la quantité de HCL qu’il sait écrire.

Il doit être capable de répondre à :

> Où se situe exactement le problème ?

Dans une plateforme moderne, une erreur peut venir de :

```text
Git
GitHub Actions
OIDC
Microsoft Entra ID
Azure RBAC
Terraform Backend
Terraform State
AzureRM Provider
Azure API
Private DNS
Routing
NSG
PaaS service
Application
```

Modifier du code au hasard est une mauvaise méthode de diagnostic.

## 1. Méthode générale

Toujours commencer par classifier l’incident.

```text
1. Code ?
2. Terraform init ?
3. Provider ?
4. Backend ?
5. State ?
6. Authentication ?
7. Authorization ?
8. Azure API ?
9. Network ?
10. DNS ?
11. Application ?
```

Puis collecter les faits avant de corriger.

## 2. Capturer le contexte

```powershell
terraform version
terraform providers
terraform workspace show
git rev-parse HEAD
git status
```

Azure :

```powershell
az account show `
  --query "{Subscription:name,SubscriptionId:id,TenantId:tenantId,User:user.name}" `
  --output json
```

Ne pas publier ces informations avec des tokens ou secrets.

## 3. Incident 1 — `terraform init` échoue sur le backend

Symptômes :

```text
403
AuthorizationPermissionMismatch
AuthorizationFailure
```

Diagnostic :

```text
Backend Storage Account correct ?
Container correct ?
Key correcte ?
ARM_CLIENT_ID correct ?
ARM_TENANT_ID correct ?
ARM_USE_OIDC / ARM_USE_CLI correct ?
Storage Blob Data Contributor présent ?
```

Tester le Data Plane avec la même identité :

```powershell
az storage blob list `
  --account-name "<storage>" `
  --container-name "<container>" `
  --auth-mode login `
  --output table
```

Un rôle `Contributor` sur le Resource Group n’implique pas automatiquement l’accès Blob Data Plane.

## 4. Incident 2 — state lock

Erreur :

```text
Error acquiring the state lock
```

Ne pas répondre immédiatement par :

```bash
-lock=false
```

Procédure :

```text
1. lire Lock Info
2. identifier Who
3. identifier Operation
4. vérifier pipeline en cours
5. attendre si opération légitime
6. force-unlock uniquement si verrou réellement orphelin
```

Commande :

```powershell
terraform force-unlock "<LOCK_ID>"
```

uniquement après vérification.

## 5. Incident 3 — drift

Quelqu’un modifie Azure Portal.

Exécuter :

```powershell
terraform plan
```

Puis, pour isoler uniquement la mise à jour de state :

```powershell
terraform plan -refresh-only
```

Ne pas utiliser `apply -refresh-only` simplement pour faire disparaître le warning.

Décider d’abord si :

```text
Azure doit revenir au code
```

ou :

```text
le code doit adopter le changement
```

## 6. Incident 4 — ressource existe déjà

Erreur typique :

```text
A resource with the ID ... already exists
```

Questions :

```text
Ressource créée manuellement ?
Ancien state perdu ?
state rm exécuté ?
mauvaise backend key ?
mauvais environnement ?
```

Ne pas renommer artificiellement la ressource pour contourner le problème.

Si elle doit être gérée par ce state, utiliser un import contrôlé.

## 7. Import block

Exemple :

```hcl
import {
  to = azurerm_resource_group.main
  id = "/subscriptions/.../resourceGroups/rg-mla-dev-fce-001"
}
```

Puis :

```powershell
terraform plan
```

L’objectif est :

```text
import
+
configuration alignée
+
pas de changement destructif inattendu
```

## 8. Incident 5 — remplacement inattendu

Le plan affiche :

```text
-/+
```

Avant d’appliquer :

```powershell
terraform show tfplan
```

Rechercher l’argument qui force le remplacement.

Questions :

```text
Nom changé ?
Location changée ?
Subnet changé ?
Type/SKU changé ?
Adresse Terraform changée ?
Provider upgrade ?
```

Pour un refactoring d’adresse, vérifier si un `moved` block manque.

## 9. Ne pas utiliser `-replace` sans raison

Terraform permet :

```powershell
terraform apply `
  -replace="azurerm_resource.example"
```

Cette option sert lorsqu’on veut volontairement recréer un objet.

Elle n’est pas une solution générique à un incident mal compris.

## 10. `terraform taint`

Les workflows modernes préfèrent généralement :

```text
-replace
```

à la commande historique `terraform taint`.

La décision de recréer une ressource doit rester visible dans le plan de l’opération.

## 11. Incident 6 — cycle Terraform

Erreur :

```text
Cycle
```

Utiliser :

```powershell
terraform graph
```

Analyser :

```text
A référence B
B référence C
C référence A
```

Retirer la dépendance artificielle ou redécouper les responsabilités.

Ajouter davantage de `depends_on` est rarement la solution.

## 12. Incident 7 — provider inconsistant

Après changement de version :

```text
Inconsistent dependency lock file
Required plugins are not installed
```

Vérifier :

```powershell
terraform version
terraform providers
git diff -- .terraform.lock.hcl
```

Puis :

```powershell
terraform init
```

Ne pas supprimer systématiquement `.terraform.lock.hcl`.

Une montée provider doit être un changement contrôlé.

## 13. Incident 8 — OIDC GitHub échoue

`azure/login` échoue.

Vérifier :

```text
id-token: write ?
Client ID ?
Tenant ID ?
Federated credential existe ?
Issuer exact ?
Audience exacte ?
Subject exact ?
GitHub Environment attendu ?
Repository renommé/transféré ?
```

Depuis les changements GitHub OIDC de 2026, ne pas déduire le `sub` depuis un vieux tutoriel. Vérifier la politique actuelle du repository.

## 14. OIDC fonctionne mais Terraform reçoit 403

Cela signifie souvent :

```text
Authentication = OK
Authorization = KO
```

Vérifier les Role Assignments :

```powershell
az role assignment list `
  --assignee-object-id "<PRINCIPAL_ID>" `
  --all `
  --output table
```

Puis comparer le scope réellement ciblé par Terraform.

## 15. Incident 9 — Key Vault 403

Séparer :

```text
Management Plane
```

et :

```text
Data Plane
```

Une identité peut créer le Key Vault avec Contributor mais ne pas pouvoir lire le secret.

Vérifier les rôles Data Plane :

```text
Key Vault Secrets User
Key Vault Secrets Officer
```

selon le besoin.

## 16. Incident 10 — Key Vault inaccessible avec Private Endpoint

Avant de modifier RBAC, vérifier le réseau.

```powershell
Resolve-DnsName "<vault>.vault.azure.net"
```

Depuis le VNet, nous voulons :

```text
CNAME privatelink.vaultcore.azure.net
+
IP privée
```

Puis vérifier :

```text
Private Endpoint Approved
Private DNS Zone
VNet Link
A record
routing
```

Un 403 d’autorisation et un timeout réseau ne se diagnostiquent pas de la même manière.

## 17. Incident 11 — Container App ne démarre pas

Vérifier l’image :

```powershell
az acr repository show-tags `
  --name "<acr>" `
  --repository "monlabazure/hello" `
  --output table
```

Vérifier la Managed Identity :

```text
UAMI attachée à Container App ?
registry.identity correct ?
AcrPull présent ?
```

Puis consulter les logs Container Apps.

## 18. Incident 12 — Application Gateway Backend Unhealthy

Commande :

```powershell
az network application-gateway show-backend-health `
  --resource-group "<rg>" `
  --name "<appgw>"
```

Vérifier :

```text
DNS backend
Host header
Probe host
Probe path
Port 443
TLS
NSG
UDR
Container App ingress
```

Le backend peut répondre 404 simplement parce que le Host header ne correspond pas au FQDN Container Apps.

## 19. Incident 13 — `No changes` alors qu’Azure est différent

Vérifier :

```text
bonne backend key ?
bonne subscription ?
bon Root Module ?
bonne resource address ?
ressource réellement gérée par ce state ?
```

Puis :

```powershell
terraform state list
terraform state show <address>
terraform plan -refresh-only
```

Ne pas conclure trop vite à un bug Terraform.

## 20. Incident 14 — plan énorme après changement de module

Si un refactoring provoque :

```text
50 destroy
50 create
```

arrêter.

Vérifier les adresses avant/après :

```powershell
terraform state list
```

Ajouter des `moved` blocks ou préparer une migration de state documentée.

## 21. Logs Terraform

Pour un diagnostic ponctuel :

```powershell
$env:TF_LOG = "DEBUG"
$env:TF_LOG_PATH = ".\terraform-debug.log"
```

Puis exécuter la commande problématique.

Après diagnostic :

```powershell
Remove-Item Env:TF_LOG
Remove-Item Env:TF_LOG_PATH
```

Le fichier peut contenir des informations sensibles.

Ne pas le committer.

## 22. `TF_LOG=TRACE`

Le niveau TRACE est très verbeux.

Ne l’utiliser que si DEBUG ne suffit pas et limiter la collecte.

Avant de partager le log, le traiter comme une donnée potentiellement sensible.

## 23. JSON Plan

Pour une analyse automatisée :

```powershell
terraform plan -out="tfplan"
terraform show -json ".\tfplan" > ".\tfplan.json"
```

Le JSON peut contenir des valeurs sensibles.

Ne pas le publier comme artifact non protégé.

## 24. `-detailed-exitcode`

```powershell
terraform plan -detailed-exitcode
```

Interprétation :

```text
0 = aucun changement
1 = erreur
2 = changements
```

Dans un script, ne pas traiter `2` comme une panne.

## 25. Troubleshooting Matrix

| Symptôme | Première zone à vérifier |
|---|---|
| `terraform init` 403 | Backend RBAC/Data Plane |
| `azure/login` échoue | OIDC/FIC |
| `plan` Azure 403 | RBAC Management Plane |
| Key Vault secret 403 | RBAC Data Plane |
| Timeout Key Vault | Private Endpoint/DNS/network |
| `already exists` | State/import/backend key |
| `Cycle` | Graphe/dépendances |
| Backend unhealthy AppGW | DNS/Host/probe/route |
| ImagePull error | ACR/UAMI/AcrPull |
| Nombreux replacements | State addresses/provider/refactoring |

## 26. Règle d’or

Ne pas effectuer plusieurs corrections simultanément.

Mauvais troubleshooting :

```text
changer NSG
changer DNS
changer provider
changer role
recreate resource
```

puis constater que cela fonctionne sans savoir pourquoi.

Préférer :

```text
hypothèse
  │
  ▼
test unique
  │
  ▼
observation
  │
  ▼
conclusion
```

## 27. Runbook d’incident Terraform

Pour une plateforme d’entreprise, conserver un runbook avec :

```text
Backend inaccessible
State lock orphelin
State recovery
Import
Provider upgrade failure
OIDC failure
RBAC failure
Private DNS failure
Emergency rollback
```

Le jour d’un incident PROD, ce n’est pas le moment d’inventer la procédure.

## 28. Validation

```text
[ ] backend 403 diagnostiqué
[ ] state lock compris
[ ] drift diagnostiqué
[ ] import block compris
[ ] replacement inattendu analysé
[ ] cycle analysé
[ ] provider lock file compris
[ ] OIDC failure diagnostiquée
[ ] RBAC Management/Data Plane distingué
[ ] Private DNS troubleshooting effectué
[ ] ACR pull troubleshooting compris
[ ] App Gateway health troubleshooting compris
[ ] TF_LOG utilisé avec prudence
[ ] plan JSON considéré sensible
```

## 29. Suite

Nous disposons maintenant des compétences nécessaires pour assembler la plateforme complète.

**LAB-20 — Projet final : plateforme Azure Terraform production-ready**

## Références officielles

- HashiCorp — Terraform Troubleshooting
- HashiCorp — Terraform State
- HashiCorp — Import
- HashiCorp — `-replace`
- HashiCorp — Logging
- Microsoft Learn — Azure Private Link troubleshooting
- Microsoft Learn — Application Gateway backend health
