---
title: "LAB-12 — Déployer une application Azure avec Terraform : ACR et Container Apps"
date: 2026-10-15
draft: false
description: "Déployer une application conteneurisée sur Azure Container Apps avec Terraform, Azure Container Registry, Managed Identity, Log Analytics et réseau privé."
summary: "Créer ACR, une identité de pull dédiée, un environnement Container Apps intégré au VNet et une application prête à être publiée derrière Application Gateway."
slug: "terraform-azure-lab-12-acr-container-apps"
categories:
  - "Automatisation"
tags:
  - "Terraform"
  - "Azure"
  - "Container Apps"
  - "Azure Container Registry"
  - "Managed Identity"
  - "Log Analytics"
series:
  - "Terraform Azure — De zéro à production"
weight: 13
ShowToc: true
TocOpen: true
---

# LAB-12 — Déployer une application Azure avec Terraform : ACR et Container Apps

Nous avons maintenant un réseau Azure, des Private Endpoints, des identités et un Key Vault. Il manque encore un workload réel.

Nous allons déployer :

```text
Azure Container Registry
      │
      │ AcrPull
      ▼
User Assigned Managed Identity
      │
      ▼
Azure Container App
      │
      ▼
Internal Container Apps Environment
      │
      ▼
VNet
```

L’application sera volontairement simple. L’objectif est de construire une plateforme cohérente avant de placer Application Gateway WAF devant elle au LAB-13.

## 1. Objectifs

Nous allons :

- créer Azure Container Registry ;
- laisser `admin_enabled = false` ;
- créer une UAMI dédiée au pull d’image ;
- attribuer `AcrPull` au scope du registry ;
- importer une image de démonstration dans ACR ;
- ajouter un subnet dédié à Container Apps ;
- créer un Log Analytics Workspace ;
- créer un Container Apps Environment intégré au VNet ;
- le rendre interne ;
- créer une Container App ;
- utiliser Managed Identity pour ACR ;
- comprendre les révisions et l’ingress ;
- préparer Application Gateway.

## 2. Architecture

```text
VNet 10.20.0.0/16
│
├── snet-app
├── snet-pe
└── snet-aca 10.20.8.0/21
     │
     ▼
Container Apps Environment
     │
     └── Container App
          │
          ├── System Assigned Identity
          │
          └── UAMI ACR Pull
                 │
                 ▼
                ACR
```

## 3. Subnet Container Apps

Pour un environnement Workload Profiles intégré à notre VNet, nous utilisons un subnet dédié et délégué à :

```text
Microsoft.App/environments
```

Ajouter au module réseau :

```hcl
variable "container_apps_subnet_name" {
  type = string
}

variable "container_apps_subnet_prefixes" {
  type = list(string)
}
```

Puis dans `main.tf` du module réseau :

```hcl
resource "azurerm_subnet" "container_apps" {
  name = var.container_apps_subnet_name

  resource_group_name  = var.resource_group_name
  virtual_network_name = azurerm_virtual_network.this.name
  address_prefixes     = var.container_apps_subnet_prefixes

  delegation {
    name = "container-apps"

    service_delegation {
      name = "Microsoft.App/environments"
    }
  }
}
```

Ajouter à l’output :

```hcl
output "subnet_ids" {
  value = {
    app               = azurerm_subnet.app.id
    private_endpoints = azurerm_subnet.private_endpoints.id
    container_apps    = azurerm_subnet.container_apps.id
  }
}
```

Pour DEV :

```hcl
container_apps_subnet_name = "snet-aca-dev-fce-001"

container_apps_subnet_prefixes = [
  "10.20.8.0/21"
]
```

## 4. Module application

Créer :

```text
modules/container-platform/
├── versions.tf
├── variables.tf
├── main.tf
└── outputs.tf
```

## 5. `versions.tf`

```hcl
terraform {
  required_version = ">= 1.16.0, < 2.0.0"

  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = ">= 5.0, < 6.0"
    }
  }
}
```

## 6. `variables.tf`

```hcl
variable "resource_group_name" {
  type = string
}

variable "location" {
  type = string
}

variable "environment" {
  type = string
}

variable "project_name" {
  type = string
}

variable "infrastructure_subnet_id" {
  type = string
}

variable "subscription_id" {
  type = string
}

variable "image_name" {
  type    = string
  default = "monlabazure/hello"
}

variable "image_tag" {
  type    = string
  default = "v1"
}

variable "tags" {
  type    = map(string)
  default = {}
}
```

## 7. `main.tf`

```hcl
locals {
  suffix = substr(md5(var.subscription_id), 0, 8)

  acr_name = substr(
    "acr${var.project_name}${var.environment}${local.suffix}",
    0,
    50
  )

  log_analytics_name = "log-${var.project_name}-${var.environment}-fce-001"
  environment_name   = "cae-${var.project_name}-${var.environment}-fce-001"
  app_name           = "ca-${var.project_name}-${var.environment}-fce-001"
  pull_identity_name = "id-${var.project_name}-acr-${var.environment}-fce-001"
}

resource "azurerm_log_analytics_workspace" "main" {
  name                = local.log_analytics_name
  location            = var.location
  resource_group_name = var.resource_group_name

  sku               = "PerGB2018"
  retention_in_days = 30

  tags = var.tags
}

resource "azurerm_container_registry" "main" {
  name                = local.acr_name
  resource_group_name = var.resource_group_name
  location            = var.location

  sku           = "Premium"
  admin_enabled = false

  public_network_access_enabled = true

  tags = var.tags
}

resource "azurerm_user_assigned_identity" "acr_pull" {
  name                = local.pull_identity_name
  location            = var.location
  resource_group_name = var.resource_group_name

  tags = var.tags
}

resource "azurerm_role_assignment" "acr_pull" {
  scope                = azurerm_container_registry.main.id
  role_definition_name = "AcrPull"

  principal_id   = azurerm_user_assigned_identity.acr_pull.principal_id
  principal_type = "ServicePrincipal"

  skip_service_principal_aad_check = true
}

resource "azurerm_container_app_environment" "main" {
  name                = local.environment_name
  location            = var.location
  resource_group_name = var.resource_group_name

  infrastructure_subnet_id = var.infrastructure_subnet_id

  internal_load_balancer_enabled = true

  log_analytics_workspace_id = azurerm_log_analytics_workspace.main.id

  tags = var.tags
}

resource "azurerm_container_app" "main" {
  name                         = local.app_name
  container_app_environment_id = azurerm_container_app_environment.main.id
  resource_group_name          = var.resource_group_name

  revision_mode = "Single"

  identity {
    type = "SystemAssigned, UserAssigned"

    identity_ids = [
      azurerm_user_assigned_identity.acr_pull.id
    ]
  }

  registry {
    server   = azurerm_container_registry.main.login_server
    identity = azurerm_user_assigned_identity.acr_pull.id
  }

  ingress {
    external_enabled = true
    target_port      = 80

    traffic_weight {
      latest_revision = true
      percentage      = 100
    }
  }

  template {
    min_replicas = 1
    max_replicas = 2

    container {
      name   = "hello"
      image  = "${azurerm_container_registry.main.login_server}/${var.image_name}:${var.image_tag}"
      cpu    = 0.25
      memory = "0.5Gi"
    }
  }

  tags = var.tags

  depends_on = [
    azurerm_role_assignment.acr_pull
  ]
}
```

### Pourquoi `depends_on` ici ?

La Container App référence déjà l’UAMI et ACR. En revanche, le besoin de disposer de l’autorisation `AcrPull` avant le premier pull d’image est un effet d’autorisation qui n’est pas intégralement exprimé par une donnée consommée.

Ici, le `depends_on` documente une dépendance opérationnelle réelle.

## 8. `outputs.tf`

```hcl
output "container_registry_name" {
  value = azurerm_container_registry.main.name
}

output "container_registry_login_server" {
  value = azurerm_container_registry.main.login_server
}

output "container_app_environment_id" {
  value = azurerm_container_app_environment.main.id
}

output "container_app_environment_default_domain" {
  value = azurerm_container_app_environment.main.default_domain
}

output "container_app_environment_static_ip" {
  value = azurerm_container_app_environment.main.static_ip_address
}

output "container_app_name" {
  value = azurerm_container_app.main.name
}

output "container_app_fqdn" {
  value = azurerm_container_app.main.latest_revision_fqdn
}
```

## 9. Importer l’image avant de créer la Container App

Le premier plan peut créer ACR avant la Container App, mais l’image doit exister au moment du premier démarrage.

Une méthode pratique de Lab consiste à créer d’abord ACR, puis importer une image Microsoft publique.

Appliquer uniquement les briques préalables :

```powershell
terraform apply `
  -target="module.container_platform.azurerm_container_registry.main" `
  -target="module.container_platform.azurerm_user_assigned_identity.acr_pull" `
  -target="module.container_platform.azurerm_role_assignment.acr_pull"
```

L’usage de `-target` reste exceptionnel. Ici il sert uniquement à initialiser l’image de démonstration.

Récupérer le nom ACR :

```powershell
$acrName = terraform output -raw container_registry_name
```

Importer :

```powershell
az acr import `
  --name $acrName `
  --source "mcr.microsoft.com/azuredocs/containerapps-helloworld:latest" `
  --image "monlabazure/hello:v1"
```

Vérifier :

```powershell
az acr repository show-tags `
  --name $acrName `
  --repository "monlabazure/hello" `
  --output table
```

## 10. Plan complet

```powershell
terraform plan -out="lab-12.tfplan"
terraform apply ".\lab-12.tfplan"
```

Puis :

```powershell
terraform plan
```

Résultat attendu :

```text
No changes.
```

## 11. Pourquoi `admin_enabled = false` ?

Nous ne voulons pas :

```text
ACR username
+
ACR password
```

dans la configuration de la Container App.

Nous utilisons :

```text
Managed Identity
+
AcrPull
```

L’application peut ainsi récupérer son image sans credential statique.

## 12. Identité de pull dédiée

Nous avons volontairement séparé :

```text
UAMI ACR Pull
```

de l’identité système du workload.

Une identité compromise dans le container ne doit pas nécessairement disposer des mêmes droits que l’identité utilisée uniquement par la plateforme pour récupérer l’image.

## 13. Environnement interne

Nous avons :

```hcl
internal_load_balancer_enabled = true
```

L’environnement Container Apps est donc publié via une IP privée dans le subnet d’infrastructure.

Une application dont l’ingress est `external_enabled = true` est publiée à la frontière de cet environnement interne, pas directement sur Internet.

C’est une distinction importante.

## 14. DNS interne du Container Apps Environment

Récupérer :

```powershell
terraform output container_app_environment_default_domain
terraform output container_app_environment_static_ip
```

Pour qu’un client dans le VNet, notamment Application Gateway au LAB-13, résolve correctement le FQDN interne, nous devons créer une zone DNS privée correspondant au default domain de l’environnement.

Exemple Terraform :

```hcl
resource "azurerm_private_dns_zone" "container_apps" {
  name                = module.container_platform.container_app_environment_default_domain
  resource_group_name = data.azurerm_resource_group.main.name
}

resource "azurerm_private_dns_zone_virtual_network_link" "container_apps" {
  name = "link-container-apps"

  private_dns_zone_id = azurerm_private_dns_zone.container_apps.id
  virtual_network_id  = module.network.virtual_network_id

  resource_group_name = data.azurerm_resource_group.main.name

  registration_enabled = false
}

resource "azurerm_private_dns_a_record" "container_apps_wildcard" {
  name                = "*"
  zone_name           = azurerm_private_dns_zone.container_apps.name
  resource_group_name = data.azurerm_resource_group.main.name
  ttl                 = 300

  records = [
    module.container_platform.container_app_environment_static_ip
  ]
}
```

Cette zone résout les noms des applications de l’environnement vers son IP interne.

## 15. Tester le backend

Depuis une machine ayant accès au VNet et à la Private DNS Zone :

```powershell
Resolve-DnsName "<container-app-fqdn>"
```

Puis :

```powershell
Invoke-WebRequest "https://<container-app-fqdn>"
```

Depuis un poste hors VNet, le comportement dépend de votre connectivité et de votre DNS hybride.

## 16. Préparation Key Vault

La Container App possède une System Assigned Managed Identity.

Nous pourrons lui attribuer :

```text
Key Vault Secrets User
```

sur le coffre du LAB-10.

Cette étape devra être réalisée par le composant IAM autorisé à créer les Role Assignments, pas par un workload Terraform limité à Contributor si celui-ci ne dispose pas de `roleAssignments/write`.

## 17. Production : ACR privé

Dans ce Lab :

```hcl
public_network_access_enabled = true
```

sur ACR afin de simplifier l’import initial d’image.

Une cible plus stricte est :

```text
Container Apps
   │
   ▼
Private DNS
   │
   ▼
Private Endpoint ACR
   │
   ▼
ACR Premium
```

avec la zone :

```text
privatelink.azurecr.io
```

Nous pourrons appliquer le même pattern Private Endpoint que dans le LAB-11.

## 18. Validation

```text
[ ] subnet Container Apps dédié
[ ] délégation Microsoft.App/environments
[ ] Log Analytics créé
[ ] ACR Premium créé
[ ] admin ACR désactivé
[ ] UAMI ACR Pull créée
[ ] rôle AcrPull assigné
[ ] image importée dans ACR
[ ] environnement Container Apps interne créé
[ ] Container App déployée
[ ] Managed Identity utilisée pour ACR
[ ] Private DNS de l’environnement créée
[ ] wildcard A record créé
[ ] terraform plan = No changes
```

## 19. À retenir

La plateforme applicative suit maintenant cette chaîne :

```text
Code container
      │
      ▼
ACR
      │
      │ Managed Identity + AcrPull
      ▼
Container App
      │
      ▼
Internal Container Apps Environment
      │
      ▼
VNet
```

Le prochain Lab va ajouter une couche d’entrée sécurisée.

**LAB-13 — Application Gateway WAF_v2 avec Terraform**

## Références officielles

- Microsoft Learn — Azure Container Apps networking
- Microsoft Learn — Container Apps Managed Identity
- Microsoft Learn — Container Apps image pull with Managed Identity
- HashiCorp AzureRM — `azurerm_container_app`
- HashiCorp AzureRM — `azurerm_container_app_environment`
- HashiCorp AzureRM — `azurerm_container_registry`
