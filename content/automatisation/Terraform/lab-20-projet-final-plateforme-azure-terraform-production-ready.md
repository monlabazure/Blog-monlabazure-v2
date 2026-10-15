---
title: "LAB-20 — Projet final : plateforme Azure Terraform production-ready"
date: 2026-10-15
draft: false
description: "Assembler une plateforme Azure complète avec Terraform : Remote State, modules, environnements, OIDC, réseau privé, Key Vault, ACR, Container Apps, Application Gateway WAF, monitoring et CI/CD."
summary: "Projet final de la série Terraform Azure : concevoir, déployer, sécuriser, tester et exploiter une plateforme proche d’un contexte de production."
slug: "terraform-azure-lab-20-projet-final-production-ready"
categories:
  - "Automatisation"
tags:
  - "Terraform"
  - "Azure"
  - "Production"
  - "GitHub Actions"
  - "WAF"
  - "Private Link"
  - "DevOps"
series:
  - "Terraform Azure — De zéro à production"
weight: 21
ShowToc: true
TocOpen: true
---

# LAB-20 — Projet final : plateforme Azure Terraform production-ready

Ce dernier Lab n’introduit pas un nouveau concept isolé.

Il assemble toute la série dans une plateforme cohérente.

L’objectif est de pouvoir regarder l’architecture et expliquer :

- comment le code est structuré ;
- où se trouve le state ;
- quelle identité exécute Terraform ;
- comment les environnements sont séparés ;
- comment l’application accède aux services Azure ;
- où passe le trafic ;
- comment le DNS privé fonctionne ;
- comment un changement est validé puis promu ;
- comment l’équipe réagit en cas d’incident.

## 1. Architecture finale

```text
Developer
   │
   ▼
Git / GitHub
   │
   ▼
Pull Request
   │
   ├── fmt
   ├── validate
   ├── TFLint
   ├── terraform test
   ├── Checkov
   └── Terraform Plan
          │
          ▼
        Review
          │
          ▼
         main
          │
          ▼
GitHub Actions CD
          │
          │ OIDC
          ▼
Microsoft Entra ID
          │
          ▼
UAMI DEV / QUAL / PROD
          │
          ▼
Azure RBAC
          │
          ├───────────────┐
          ▼               ▼
Azure Resources      Remote State
                          │
                          ▼
                    Azure Storage
```

Côté application :

```text
Internet
   │
   ▼
Application Gateway WAF_v2
   │
   ▼
Internal Container Apps Environment
   │
   └── Container App
         │
         ├── ACR via Managed Identity
         └── Key Vault via Managed Identity
                │
                ▼
            Private Endpoint
                │
                ▼
            Private DNS
```

## 2. Objectifs

Le projet final doit démontrer :

```text
Architecture Terraform
State Management
Identity
RBAC
Networking
Private DNS
Private Link
Secrets
Application Platform
WAF
Observability
CI
CD
Security Gates
Troubleshooting
Recovery
```

## 3. Repository final

```text
terraform-azure-platform/
│
├── .github/
│   ├── CODEOWNERS
│   ├── pull_request_template.md
│   └── workflows/
│       ├── terraform-quality.yml
│       ├── terraform-plan-dev.yml
│       └── terraform-cd.yml
│
├── bootstrap/
│   ├── backend/
│   └── identity/
│
├── modules/
│   ├── network/
│   ├── network-security/
│   ├── key-vault/
│   ├── private-endpoint/
│   ├── container-platform/
│   └── application-gateway/
│
├── environments/
│   ├── dev/
│   ├── qual/
│   └── prod/
│
├── tests/
├── .gitignore
├── .tflint.hcl
├── .checkov.yml
└── README.md
```

## 4. Principes non négociables

Pour le projet final :

```text
Pas de Terraform State dans Git
Pas de Client Secret Azure permanent dans GitHub
Pas d'admin ACR
Pas de secret applicatif en clair dans les fichiers Terraform
Pas de même state pour DEV/QUAL/PROD
Pas d'identité DEV avec droits PROD
Pas d'apply automatique depuis une Pull Request
Pas de public Key Vault Data Plane en cible finale
```

## 5. Backend

La plateforme doit utiliser :

```text
Azure Storage
+
Microsoft Entra ID
+
Storage Blob Data Contributor
+
State Locking
+
Blob Versioning
+
Soft Delete
```

Isolation :

```text
tfstate-dev/platform.tfstate
tfstate-qual/platform.tfstate
tfstate-prod/platform.tfstate
```

Chaque identité d’environnement ne reçoit accès qu’à son container.

## 6. Bootstrap séparé

Le bootstrap privilégié gère :

```text
Storage backend
Managed Identities
Federated Identity Credentials
Workload Resource Groups
RBAC
```

Le workload Terraform ne doit pas disposer automatiquement des permissions nécessaires pour modifier son propre modèle IAM.

## 7. Root Modules

Nous conservons :

```text
environments/dev
environments/qual
environments/prod
```

Chaque dossier représente :

```text
1 Root Module
1 Backend
1 Identity
1 State
1 Pipeline target
```

## 8. Variables d’environnement

Exemple DEV :

```hcl
environment = "dev"
location    = "francecentral"

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

  appgw_subnet_prefixes = [
    "10.20.3.0/24"
  ]

  container_apps_subnet_prefixes = [
    "10.20.8.0/21"
  ]
}
```

QUALIF :

```text
10.30.0.0/16
```

PROD :

```text
10.40.0.0/16
```

Les plages ne doivent pas se chevaucher si les réseaux doivent être interconnectés.

## 9. Module réseau

Responsabilités :

```text
VNet
Subnet Application
Subnet Private Endpoints
Subnet Application Gateway
Subnet Container Apps
Delegations
```

Le module ne crée pas :

```text
Key Vault
Container App
WAF
```

Les frontières restent cohérentes.

## 10. Private DNS

Les zones privées doivent être traitées comme un composant d’architecture.

Exemples :

```text
privatelink.vaultcore.azure.net
privatelink.azurecr.io
```

Pour Container Apps interne, créer la zone correspondant au default domain de l’environnement avec le wildcard A record vers la static IP.

Dans une Landing Zone réelle, envisager la centralisation DNS dans la subscription Connectivity.

## 11. Key Vault

Cible :

```hcl
enable_rbac_authorization     = true
public_network_access_enabled = false
```

avec :

```text
Private Endpoint
Private DNS
Managed Identity
RBAC Data Plane
```

Éviter que Terraform ait connaissance de la valeur d’un secret si ce n’est pas nécessaire.

## 12. ACR

Cible :

```hcl
admin_enabled = false
```

La Container App récupère l’image via :

```text
User Assigned Managed Identity
+
AcrPull
```

Pour une production durcie :

```text
ACR Premium
+
Private Endpoint
+
privatelink.azurecr.io
+
public network disabled
```

## 13. Container Apps

Cible :

```text
Dedicated infrastructure subnet
VNet integration
Internal environment
Managed Identity
Log Analytics
```

L’application n’est pas directement publiée sur Internet.

## 14. Application Gateway

Cible :

```text
WAF_v2
Dedicated subnet
WAF Policy
Autoscale
HTTPS frontend
Private backend
```

Pour le projet final, le mode de WAF attendu après période d’observation est :

```text
Prevention
```

Les exclusions doivent être documentées.

## 15. HTTPS

Le LAB-13 utilisait HTTP sur le frontend pour simplifier l’apprentissage.

Le projet final doit prévoir :

```text
DNS public
Certificate
HTTPS listener
TLS policy
HTTP -> HTTPS redirect
```

Le certificat peut être stocké dans Key Vault et lu par une Managed Identity dédiée Application Gateway.

Ne pas mettre le PFX ou son mot de passe dans Git.

## 16. Monitoring

La plateforme doit disposer de :

```text
Log Analytics Workspace
Application Gateway diagnostics
WAF logs
Container Apps logs
Key Vault diagnostics
Azure Activity Log
```

Les catégories exactes de diagnostic disponibles peuvent évoluer par service. Les vérifier avec Azure avant de les coder en dur.

## 17. Exemple Diagnostic Setting

Pattern Terraform :

```hcl
resource "azurerm_monitor_diagnostic_setting" "example" {
  name = "diag-${var.resource_name}"

  target_resource_id = var.target_resource_id

  log_analytics_workspace_id = var.log_analytics_workspace_id

  enabled_log {
    category_group = "allLogs"
  }

  enabled_metric {
    category = "AllMetrics"
  }
}
```

Tous les services ne supportent pas nécessairement les mêmes groupes/catégories. Le module monitoring doit tenir compte du schéma réel du service.

## 18. Tags

Minimum recommandé dans cette série :

```text
environment
project
managed_by
owner
criticality
```

En entreprise, ajouter selon les standards :

```text
cost_center
application
data_classification
business_owner
technical_owner
```

Les tags ne remplacent pas Azure Policy.

## 19. CI finale

La Pull Request doit au minimum passer :

```text
terraform fmt -check
terraform validate
TFLint
terraform test
Checkov
terraform plan DEV
```

Selon les exigences :

```text
policy-as-code
cost estimation
documentation generation
```

peuvent être ajoutés.

## 20. CD finale

```text
Merge main
   │
   ▼
DEV
   │
   ├── plan
   ├── apply
   └── smoke test
         │
         ▼
       QUALIF
         │
         ├── plan
         ├── apply
         └── validation
               │
               ▼
           PROD approval
               │
               ▼
             PROD
               │
               ├── plan
               ├── review
               ├── apply
               └── validation
```

## 21. GitHub OIDC

Chaque environnement utilise sa propre UAMI.

```text
GitHub Environment DEV
      │
      ▼
FIC DEV
      │
      ▼
UAMI DEV
```

Le subject OIDC exact doit être vérifié selon la politique actuelle du repository GitHub. Ne pas copier une ancienne chaîne `sub` depuis un tutoriel.

## 22. GitHub permissions

Workflow privilégié :

```yaml
permissions:
  contents: read
  id-token: write
```

Ne pas ajouter :

```text
contents: write
packages: write
actions: write
```

si le workflow n’en a pas besoin.

## 23. Supply Chain

Pour une cible exigeante :

- épingler GitHub Actions sur SHA ;
- activer Dependabot/Renovate ;
- revoir les changements de `.terraform.lock.hcl` ;
- versionner les modules ;
- limiter les sources de providers/modules ;
- protéger `.github/workflows/`.

## 24. Terraform Provider Upgrade

Processus :

```text
Branch dédiée
   │
   ▼
mettre à jour contrainte
   │
   ▼
terraform init -upgrade
   │
   ▼
lock file diff
   │
   ▼
tests
   │
   ▼
DEV
   │
   ▼
QUAL
   │
   ▼
PROD
```

Ne pas faire une montée provider implicite pendant un changement fonctionnel important.

## 25. State Recovery

Le runbook doit documenter :

```text
Backend inaccessible
Lock orphelin
Blob state supprimé
Restore version
State import
State address move
```

Tester la restauration avant d’en avoir besoin en production.

## 26. Drift

Politique recommandée :

```text
Infrastructure gérée Terraform
      │
      ▼
changement manuel interdit sauf procédure d'urgence
```

Après un break-glass :

```text
documenter
mettre à jour le code
planifier
réconcilier
```

## 27. Azure Policy

La plateforme Terraform ne doit pas être la seule ligne de défense.

Azure Policy peut imposer :

```text
locations autorisées
tags requis
SKUs
private networking
TLS
diagnostic settings
public access restrictions
```

La CI et Azure Policy protègent deux chemins différents.

## 28. FinOps

Avant production :

```text
Application Gateway WAF_v2
Container Apps
Log Analytics
Private Endpoints
ACR Premium
```

ont des coûts.

La plateforme doit avoir :

- budgets ;
- tags de coûts ;
- revue des rétentions Log Analytics ;
- scaling approprié ;
- suppression des ressources de test.

## 29. Haute disponibilité

Production doit évaluer :

```text
zone redundancy
autoscaling
regional availability
backend resiliency
DNS architecture
state backend replication
```

Une option disponible dans Azure n’est pas automatiquement nécessaire partout. Décider selon RTO, RPO et criticité.

## 30. Security Review

Checklist :

```text
[ ] public Key Vault disabled
[ ] ACR admin disabled
[ ] secrets absents de Git
[ ] no Client Secret GitHub
[ ] OIDC scopes contrôlés
[ ] PROD identity séparée
[ ] state PROD isolé
[ ] WAF Prevention
[ ] TLS frontend
[ ] Private DNS correct
[ ] RBAC least privilege
[ ] logging activé
[ ] state recovery testée
```

## 31. Operational Review

```text
[ ] runbook backend
[ ] runbook state lock
[ ] runbook OIDC
[ ] runbook Private DNS
[ ] runbook App Gateway health
[ ] runbook ACR image pull
[ ] rollback documenté
[ ] ownership identifié
[ ] cost owner identifié
```

## 32. Test final DEV

Le candidat doit pouvoir exécuter :

```powershell
terraform `
  -chdir="environments/dev" `
  plan
```

et expliquer chaque ressource modifiée.

Puis vérifier :

```text
Application accessible via WAF
Container App privée
Key Vault privé
DNS privé correct
ACR auth par Managed Identity
```

## 33. Test final CI

Créer une PR qui modifie un tag non critique.

Vérifier :

```text
fmt
validate
TFLint
terraform test
Checkov
plan DEV
```

Puis faire relire le plan.

## 34. Test final CD

Merger.

Observer :

```text
DEV success
QUAL success
PROD waiting approval
```

Avant d’approuver PROD, comparer le plan aux attentes.

## 35. Test de sécurité

Créer volontairement une configuration interdite, par exemple :

```hcl
admin_enabled = true
```

sur ACR.

Le changement doit idéalement être détecté avant PROD par :

```text
test
scanner
review
policy
```

Le projet final n’est réussi que si la plateforme refuse aussi certaines mauvaises configurations.

## 36. Test d’incident

Provoquer en DEV :

```text
Role Assignment backend supprimé
```

Vérifier que la pipeline échoue proprement.

Restaurer le rôle.

Puis provoquer :

```text
Private DNS link absent
```

et diagnostiquer sans modifier au hasard le WAF ou RBAC.

## 37. Critères de réussite

La plateforme est considérée terminée lorsque :

```text
[ ] architecture documentée
[ ] backend sécurisé
[ ] state isolé DEV/QUAL/PROD
[ ] modules structurés
[ ] OIDC opérationnel
[ ] aucune credential Azure permanente dans GitHub
[ ] RBAC séparé
[ ] Key Vault privé
[ ] Container Apps intégré au VNet
[ ] ACR sans admin
[ ] application accessible via WAF
[ ] WAF en Prevention pour PROD
[ ] logs disponibles
[ ] CI obligatoire
[ ] PROD protégé par approval
[ ] troubleshooting runbook disponible
[ ] recovery state documentée et testée
```

## 38. Ce que nous avons construit depuis LAB-00

```text
LAB-00  Workstation
LAB-01  Premier déploiement
LAB-02  HCL
LAB-03  Variables et types
LAB-04  Dependency Graph
LAB-05  Terraform State
LAB-06  Remote State
LAB-07  Modules
LAB-08  DEV / QUAL / PROD
LAB-09  Identity & RBAC
LAB-10  Key Vault
LAB-11  Private Endpoint & DNS
LAB-12  ACR & Container Apps
LAB-13  Application Gateway WAF
LAB-14  Git Workflow
LAB-15  CI
LAB-16  GitHub OIDC
LAB-17  CD
LAB-18  Quality Gates
LAB-19  Troubleshooting
LAB-20  Production-ready platform
```

## 39. Compétences attendues après la série

Il faut maintenant pouvoir expliquer et défendre :

```text
Pourquoi ce backend ?
Pourquoi cette frontière de state ?
Pourquoi cette identity ?
Pourquoi ce scope RBAC ?
Pourquoi ce module ?
Pourquoi ce Private Endpoint ?
Pourquoi cette zone DNS ?
Pourquoi ce plan remplace la ressource ?
Pourquoi PROD est protégée différemment ?
Comment restaurer après perte du state ?
Comment diagnostiquer un 403 ?
Comment diagnostiquer un backend unhealthy ?
```

C’est cette capacité de raisonnement qui fait passer Terraform d’un outil de provisioning à une compétence d’architecture et d’exploitation.

## 40. Conclusion

Une plateforme Terraform production-ready n’est pas définie par la présence d’un fichier `main.tf`.

Elle repose sur un ensemble cohérent :

```text
Code
+
State
+
Identity
+
Network
+
Security
+
Testing
+
CI/CD
+
Observability
+
Operations
```

L’objectif de la série était précisément de construire ces éléments progressivement plutôt que de copier une architecture finale sans comprendre ses dépendances.

La suite naturelle n’est plus un nouveau Lab fondamental.

À partir d’ici, la plateforme peut évoluer par domaines spécialisés :

- Azure Policy et gouvernance ;
- Landing Zone multi-subscriptions ;
- Private DNS Resolver et DNS hybride ;
- AKS ;
- Azure Firewall ;
- API Management ;
- observabilité avancée ;
- FinOps ;
- disaster recovery ;
- modules Azure Verified Modules ;
- tests d’intégration et policy-as-code.

## Références officielles

- HashiCorp Terraform Documentation
- HashiCorp AzureRM Provider
- Microsoft Learn — Terraform on Azure
- Microsoft Learn — Azure Well-Architected Framework
- Microsoft Learn — Azure Landing Zones
- Microsoft Learn — Azure Private Link
- Microsoft Learn — Azure Container Apps
- Microsoft Learn — Application Gateway WAF
- GitHub Docs — OpenID Connect and Environments
