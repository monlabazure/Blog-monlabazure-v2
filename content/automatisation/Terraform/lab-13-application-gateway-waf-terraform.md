---
title: "LAB-13 — Application Gateway WAF_v2 avec Terraform"
date: 2026-10-15
draft: false
description: "Déployer Azure Application Gateway WAF_v2 avec Terraform, associer une WAF Policy et publier une application Azure Container Apps interne."
summary: "Créer un frontend Application Gateway, un backend privé Container Apps, une WAF Policy et valider le flux HTTP de bout en bout."
slug: "terraform-azure-lab-13-application-gateway-waf"
categories:
  - "Automatisation"
tags:
  - "Terraform"
  - "Azure"
  - "Application Gateway"
  - "WAF"
  - "WAF_v2"
  - "OWASP"
series:
  - "Terraform Azure — De zéro à production"
weight: 14
ShowToc: true
TocOpen: true
---

# LAB-13 — Application Gateway WAF_v2 avec Terraform

Nous disposons maintenant d’une application Container Apps interne au VNet. Elle n’est pas directement exposée à Internet.

Nous allons placer devant elle :

```text
Internet
   │
   ▼
Public IP
   │
   ▼
Application Gateway WAF_v2
   │
   ▼
Private backend
   │
   ▼
Container App
```

Ce Lab reste volontairement en HTTP sur le frontend pour se concentrer sur Terraform, la composition Application Gateway et WAF. En production, le frontend doit être basculé en HTTPS avec certificat et politique TLS appropriée.

## 1. Objectifs

Nous allons :

- créer un subnet dédié Application Gateway ;
- créer une Public IP Standard ;
- créer une WAF Policy ;
- utiliser le mode Detection puis Prevention ;
- créer Application Gateway WAF_v2 ;
- configurer frontend IP, port, listener, backend pool, backend settings et routing rule ;
- utiliser le FQDN privé de la Container App comme backend ;
- ajouter un health probe ;
- comprendre `Host` et le backend hostname ;
- vérifier le backend health ;
- tester une règle WAF ;
- préparer le HTTPS.

## 2. Architecture

```text
Internet
   │
   ▼
Public IP
   │
   ▼
snet-appgw
   │
   ▼
Application Gateway WAF_v2
   │
   │ DNS privé
   ▼
Container App FQDN
   │
   ▼
Internal Container Apps Environment
```

## 3. Subnet Application Gateway

Ajouter au module réseau :

```hcl
resource "azurerm_subnet" "application_gateway" {
  name = var.application_gateway_subnet_name

  resource_group_name  = var.resource_group_name
  virtual_network_name = azurerm_virtual_network.this.name

  address_prefixes = var.application_gateway_subnet_prefixes
}
```

Pour DEV :

```hcl
application_gateway_subnet_name = "snet-appgw-dev-fce-001"

application_gateway_subnet_prefixes = [
  "10.20.3.0/24"
]
```

Un subnet Application Gateway doit être dédié à cette fonction.

## 4. Module Application Gateway

Créer :

```text
modules/application-gateway/
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
variable "name" {
  type = string
}

variable "resource_group_name" {
  type = string
}

variable "location" {
  type = string
}

variable "subnet_id" {
  type = string
}

variable "backend_fqdn" {
  type = string
}

variable "tags" {
  type    = map(string)
  default = {}
}
```

## 7. `main.tf`

```hcl
locals {
  pip_name = "${var.name}-pip"

  frontend_ip_configuration_name = "frontend-public"
  frontend_port_name             = "frontend-http"
  backend_address_pool_name      = "backend-container-apps"
  backend_http_settings_name     = "backend-https"
  listener_name                  = "listener-http"
  routing_rule_name              = "rule-main"
  probe_name                     = "probe-container-app"
}

resource "azurerm_public_ip" "main" {
  name                = local.pip_name
  location            = var.location
  resource_group_name = var.resource_group_name

  allocation_method = "Static"
  sku               = "Standard"

  tags = var.tags
}

resource "azurerm_web_application_firewall_policy" "main" {
  name                = "${var.name}-policy"
  resource_group_name = var.resource_group_name
  location            = var.location

  policy_settings {
    enabled = true
    mode    = "Detection"

    request_body_check = true
  }

  managed_rules {
    managed_rule_set {
      type    = "OWASP"
      version = "3.2"
    }
  }

  tags = var.tags
}

resource "azurerm_application_gateway" "main" {
  name                = var.name
  location            = var.location
  resource_group_name = var.resource_group_name

  firewall_policy_id = azurerm_web_application_firewall_policy.main.id

  sku {
    name = "WAF_v2"
    tier = "WAF_v2"
  }

  autoscale_configuration {
    min_capacity = 1
    max_capacity = 2
  }

  gateway_ip_configuration {
    name      = "gateway-ip-config"
    subnet_id = var.subnet_id
  }

  frontend_ip_configuration {
    name                 = local.frontend_ip_configuration_name
    public_ip_address_id = azurerm_public_ip.main.id
  }

  frontend_port {
    name = local.frontend_port_name
    port = 80
  }

  backend_address_pool {
    name = local.backend_address_pool_name

    fqdns = [
      var.backend_fqdn
    ]
  }

  probe {
    name = local.probe_name

    protocol = "Https"
    path     = "/"

    host = var.backend_fqdn

    interval            = 30
    timeout             = 30
    unhealthy_threshold = 3

    pick_host_name_from_backend_http_settings = false

    match {
      status_code = [
        "200-399"
      ]
    }
  }

  backend_http_settings {
    name = local.backend_http_settings_name

    cookie_based_affinity = "Disabled"

    protocol = "Https"
    port     = 443

    request_timeout = 30

    host_name = var.backend_fqdn

    probe_name = local.probe_name
  }

  http_listener {
    name = local.listener_name

    frontend_ip_configuration_name = local.frontend_ip_configuration_name
    frontend_port_name             = local.frontend_port_name

    protocol = "Http"
  }

  request_routing_rule {
    name     = local.routing_rule_name
    priority = 100

    rule_type = "Basic"

    http_listener_name        = local.listener_name
    backend_address_pool_name = local.backend_address_pool_name

    backend_http_settings_name = local.backend_http_settings_name
  }

  tags = var.tags
}
```

## 8. Pourquoi le backend utilise HTTPS ?

L’Application Gateway termine ici le frontend HTTP du Lab mais se connecte à la Container App via HTTPS :

```text
Client
  │ HTTP
  ▼
Application Gateway
  │ HTTPS
  ▼
Container App
```

Cela évite de désactiver TLS côté backend.

## 9. Host Header

Azure Container Apps route les requêtes selon le hostname.

Nous configurons donc :

```hcl
host_name = var.backend_fqdn
```

dans les backend HTTP settings.

Sans cela, nous pouvons obtenir :

- backend unhealthy ;
- HTTP 404 ;
- routage vers une mauvaise application.

Le problème n’est alors pas nécessairement réseau. Il peut être lié au Host header.

## 10. `outputs.tf`

```hcl
output "public_ip_address" {
  value = azurerm_public_ip.main.ip_address
}

output "application_gateway_id" {
  value = azurerm_application_gateway.main.id
}

output "waf_policy_id" {
  value = azurerm_web_application_firewall_policy.main.id
}
```

## 11. Appeler le module

Dans le Root Module DEV :

```hcl
module "application_gateway" {
  source = "../../modules/application-gateway"

  name                = "agw-mla-dev-fce-001"
  location            = data.azurerm_resource_group.main.location
  resource_group_name = data.azurerm_resource_group.main.name

  subnet_id = module.network.subnet_ids["application_gateway"]

  backend_fqdn = module.container_platform.container_app_fqdn

  tags = local.common_tags
}
```

Le VNet doit disposer de la zone DNS privée créée au LAB-12 afin que l’Application Gateway puisse résoudre le FQDN interne de la Container App.

## 12. Plan

```powershell
terraform init
terraform fmt -recursive
terraform validate
terraform plan -out="lab-13.tfplan"
```

Appliquer :

```powershell
terraform apply ".\lab-13.tfplan"
```

La création d’Application Gateway peut prendre plusieurs minutes.

## 13. Vérifier l’IP publique

```powershell
terraform output
```

Puis :

```powershell
$publicIp = terraform output -raw application_gateway_public_ip
```

Si le Root Module ne réexpose pas encore l’output :

```hcl
output "application_gateway_public_ip" {
  value = module.application_gateway.public_ip_address
}
```

## 14. Tester le flux

```powershell
Invoke-WebRequest "http://$publicIp"
```

Nous devons obtenir la réponse de l’application Container Apps.

Le chemin complet est maintenant :

```text
Internet
  ↓
Public IP
  ↓
Application Gateway
  ↓
Private DNS
  ↓
Container Apps internal endpoint
  ↓
Application
```

## 15. Backend Health

```powershell
az network application-gateway show-backend-health `
  --name "agw-mla-dev-fce-001" `
  --resource-group "rg-mla-dev-fce-001"
```

Chercher :

```text
Healthy
```

## 16. Si le backend est `Unhealthy`

Vérifier :

```text
1. FQDN exact de la Container App
2. Private DNS Zone de l’environnement
3. VNet Link
4. résolution DNS depuis le VNet
5. backend port 443
6. protocol Https
7. host_name
8. probe host
9. NSG / UDR éventuels
10. ingress de la Container App
```

## 17. WAF Detection

Nous démarrons avec :

```hcl
mode = "Detection"
```

Le trafic n’est pas bloqué par les managed rules. Les détections servent à observer le comportement.

C’est une méthode plus prudente lors de l’introduction initiale d’un WAF sur une application existante.

## 18. Passer en Prevention

Une fois les faux positifs étudiés :

```hcl
policy_settings {
  enabled = true
  mode    = "Prevention"
}
```

Puis :

```powershell
terraform plan
```

Le changement doit mettre à jour la WAF Policy sans recréer Application Gateway.

## 19. Test WAF

En environnement de Lab uniquement, envoyer une requête contenant un motif typique de test :

```powershell
Invoke-WebRequest `
  "http://$publicIp/?id=%27%20OR%201%3D1--"
```

En mode Detection, le trafic peut atteindre le backend mais générer un événement WAF.

En mode Prevention, une règle managée peut bloquer la requête selon la règle déclenchée.

Ne pas utiliser ce type de test contre un environnement qui ne vous appartient pas.

## 20. HTTPS frontend

La cible de production sera :

```text
Client
   │ HTTPS
   ▼
Application Gateway
   │
   ├── certificat
   ├── TLS policy
   └── WAF
```

Le certificat peut être intégré via Azure Key Vault et une User Assigned Managed Identity Application Gateway.

Cela ajoute plusieurs dépendances :

```text
App Gateway UAMI
      │
      ▼
Key Vault certificate permission
      │
      ▼
SSL certificate
      │
      ▼
HTTPS listener
```

Nous gardons HTTP dans ce Lab afin de ne pas mélanger certificat, DNS public et domaine personnalisé avec l’apprentissage du composant Terraform.

## 21. WAF Policy séparée

Nous utilisons :

```hcl
azurerm_web_application_firewall_policy
```

et :

```hcl
firewall_policy_id = ...
```

Cette séparation permet de gérer :

- managed rules ;
- custom rules ;
- exclusions ;
- policy mode ;
- associations futures.

La policy devient un objet à part entière plutôt qu’une configuration WAF enfouie dans Application Gateway.

## 22. Production : custom rules

Exemple de règle simple de blocage IP :

```hcl
custom_rules {
  name      = "BlockKnownBadIP"
  priority  = 10
  rule_type = "MatchRule"
  action    = "Block"

  match_conditions {
    match_variables {
      variable_name = "RemoteAddr"
    }

    operator           = "IPMatch"
    negation_condition = false

    match_values = [
      "203.0.113.10/32"
    ]
  }
}
```

Utiliser uniquement des valeurs adaptées à votre contexte réel.

## 23. Coût

Application Gateway WAF_v2 a un coût significativement supérieur aux ressources pédagogiques précédentes.

Ne pas laisser le Lab actif inutilement.

C’est particulièrement important pour une série d’apprentissage répétée plusieurs fois.

## 24. Validation

```text
[ ] subnet Application Gateway dédié
[ ] Public IP Standard créée
[ ] WAF Policy séparée créée
[ ] mode Detection testé
[ ] Application Gateway WAF_v2 créé
[ ] backend pool utilise le FQDN Container Apps
[ ] backend HTTP settings en HTTPS
[ ] Host header configuré
[ ] health probe créé
[ ] backend Health = Healthy
[ ] requête frontend testée
[ ] mode Prevention compris
[ ] stratégie HTTPS future comprise
```

## 25. À retenir

Notre fil rouge Azure est maintenant complet côté infrastructure applicative :

```text
Internet
  │
  ▼
Application Gateway WAF_v2
  │
  ▼
Container Apps
  │
  ├── Managed Identity
  ├── ACR
  └── Key Vault privé
```

Les prochains Labs vont maintenant industrialiser **la manière de modifier et déployer ce code**.

**LAB-14 — Git et workflow professionnel pour Terraform**

## Références officielles

- Microsoft Learn — Application Gateway WAF
- Microsoft Learn — WAF Policy
- HashiCorp AzureRM — `azurerm_application_gateway`
- HashiCorp AzureRM — `azurerm_web_application_firewall_policy`
