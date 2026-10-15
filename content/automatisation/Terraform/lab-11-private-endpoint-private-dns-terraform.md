---
title: "LAB-11 — Private Endpoint, Private DNS et réseau privé Azure"
date: 2026-10-15
draft: false
description: "Sécuriser Azure Key Vault avec Private Endpoint et Azure Private DNS, désactiver l’accès public et comprendre le chemin DNS d’un service PaaS privé."
summary: "Déployer un Private Endpoint pour Key Vault, créer la zone privatelink.vaultcore.azure.net et vérifier la résolution privée."
slug: "terraform-azure-lab-11-private-endpoint-private-dns"
categories:
  - "Automatisation"
tags:
  - "Terraform"
  - "Azure"
  - "Private Endpoint"
  - "Private DNS"
  - "Key Vault"
  - "Private Link"
series:
  - "Terraform Azure — De zéro à production"
weight: 12
ShowToc: true
TocOpen: true
---

# LAB-11 — Private Endpoint, Private DNS et réseau privé Azure

Le LAB-10 a sécurisé l’accès logique au Key Vault avec Microsoft Entra ID et Azure RBAC. Le Data Plane reste toutefois exposé sur un endpoint public.

Dans ce Lab, nous allons traiter la couche réseau.

L’architecture cible devient :

```text
VNet
│
├── snet-app
│
└── snet-pe
     │
     └── Private Endpoint
          │
          ▼
       Key Vault
          │
          ▼
privatelink.vaultcore.azure.net
```

À la fin, le Key Vault n’acceptera plus le trafic via son endpoint public.

## 1. Objectifs

Nous allons apprendre à :

- comprendre Azure Private Link ;
- créer un Private Endpoint avec Terraform ;
- utiliser un subnet dédié aux Private Endpoints ;
- créer une Azure Private DNS Zone ;
- lier cette zone au VNet ;
- associer le Private Endpoint à la zone DNS ;
- comprendre le CNAME public `vault.azure.net` → `privatelink.vaultcore.azure.net` ;
- désactiver le public network access ;
- vérifier les objets DNS créés ;
- comprendre pourquoi un client hors du VNet ne peut plus accéder au Data Plane.

## 2. Private Endpoint vs Service Endpoint

Un Private Endpoint crée une interface réseau privée dans le VNet et attribue une IP privée au service PaaS.

```text
Client 10.20.x.x
      │
      ▼
Private IP
10.20.2.x
      │
      ▼
Private Link
      │
      ▼
Azure Key Vault
```

Le service continue d’exister comme PaaS Azure, mais son accès peut être ramené dans notre périmètre réseau privé.

## 3. Zone DNS Key Vault

Pour Azure Key Vault, la zone privée utilisée avec Private Link est :

```text
privatelink.vaultcore.azure.net
```

Le groupe de sous-ressource du Private Endpoint est :

```text
vault
```

Ces valeurs ne doivent pas être inventées. Elles sont définies par le service Azure.

## 4. Arborescence

Nous ajoutons un module :

```text
lab-11/
├── modules/
│   └── key-vault-private-endpoint/
│       ├── versions.tf
│       ├── variables.tf
│       ├── main.tf
│       └── outputs.tf
└── environments/
    └── dev/
        ├── ...
        └── private-link.tf
```

Le Lab suppose que le VNet, le subnet Private Endpoints et le Key Vault existent déjà.

## 5. `modules/key-vault-private-endpoint/versions.tf`

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
  description = "Nom du Private Endpoint."
  type        = string
}

variable "location" {
  description = "Région Azure."
  type        = string
}

variable "resource_group_name" {
  description = "Resource Group du Private Endpoint et de la zone DNS."
  type        = string
}

variable "subnet_id" {
  description = "Subnet accueillant le Private Endpoint."
  type        = string
}

variable "virtual_network_id" {
  description = "VNet à lier à la zone DNS privée."
  type        = string
}

variable "key_vault_id" {
  description = "Resource ID du Key Vault."
  type        = string
}

variable "key_vault_name" {
  description = "Nom du Key Vault."
  type        = string
}

variable "tags" {
  description = "Tags."
  type        = map(string)
  default     = {}
}
```

## 7. `main.tf`

```hcl
locals {
  private_dns_zone_name = "privatelink.vaultcore.azure.net"
}

resource "azurerm_private_dns_zone" "key_vault" {
  name                = local.private_dns_zone_name
  resource_group_name = var.resource_group_name

  tags = var.tags
}

resource "azurerm_private_dns_zone_virtual_network_link" "key_vault" {
  name = "link-${var.key_vault_name}"

  resource_group_name = var.resource_group_name
  private_dns_zone_id = azurerm_private_dns_zone.key_vault.id
  virtual_network_id  = var.virtual_network_id

  registration_enabled = false

  tags = var.tags
}

resource "azurerm_private_endpoint" "key_vault" {
  name                = var.name
  location            = var.location
  resource_group_name = var.resource_group_name
  subnet_id           = var.subnet_id

  private_service_connection {
    name = "psc-${var.key_vault_name}"

    private_connection_resource_id = var.key_vault_id
    subresource_names              = ["vault"]
    is_manual_connection           = false
  }

  private_dns_zone_group {
    name = "default"

    private_dns_zone_ids = [
      azurerm_private_dns_zone.key_vault.id
    ]
  }

  tags = var.tags
}
```

### Pourquoi `private_dns_zone_group` ?

Sans cette association, le Private Endpoint peut exister mais le client peut encore résoudre le service vers une adresse non souhaitée.

Le réseau privé et le DNS privé doivent être conçus ensemble.

## 8. `outputs.tf`

```hcl
output "private_endpoint_id" {
  value = azurerm_private_endpoint.key_vault.id
}

output "private_dns_zone_id" {
  value = azurerm_private_dns_zone.key_vault.id
}

output "private_ip_addresses" {
  value = flatten([
    for config in azurerm_private_endpoint.key_vault.custom_dns_configs :
    config.ip_addresses
  ])
}
```

## 9. Référencer les ressources existantes

Dans le Root Module DEV :

```hcl
data "azurerm_resource_group" "main" {
  name = "rg-mla-dev-fce-001"
}

data "azurerm_virtual_network" "main" {
  name                = "vnet-mla-dev-fce-001"
  resource_group_name = data.azurerm_resource_group.main.name
}

data "azurerm_subnet" "private_endpoints" {
  name                 = "snet-pe-dev-fce-001"
  virtual_network_name = data.azurerm_virtual_network.main.name
  resource_group_name  = data.azurerm_resource_group.main.name
}

data "azurerm_key_vault" "main" {
  name                = var.key_vault_name
  resource_group_name = data.azurerm_resource_group.main.name
}
```

Ajouter la variable :

```hcl
variable "key_vault_name" {
  type        = string
  description = "Nom du Key Vault créé au LAB-10."
}
```

## 10. Appeler le module

Créer `private-link.tf` :

```hcl
module "key_vault_private_endpoint" {
  source = "../../modules/key-vault-private-endpoint"

  name                = "pe-kv-dev-fce-001"
  location            = data.azurerm_resource_group.main.location
  resource_group_name = data.azurerm_resource_group.main.name

  subnet_id          = data.azurerm_subnet.private_endpoints.id
  virtual_network_id = data.azurerm_virtual_network.main.id

  key_vault_id   = data.azurerm_key_vault.main.id
  key_vault_name = data.azurerm_key_vault.main.name

  tags = {
    environment = "dev"
    project     = "mla"
    managed_by  = "terraform"
    lab         = "lab-11"
  }
}
```

## 11. Désactiver l’accès public du Key Vault

Revenir dans la configuration qui gère le Key Vault et modifier :

```hcl
public_network_access_enabled = false
```

Puis :

```hcl
network_acls {
  bypass         = "AzureServices"
  default_action = "Deny"
}
```

Important : ne pas couper le réseau public avant d’avoir vérifié que le Private Endpoint et le DNS privé sont prêts, surtout dans un environnement réel.

## 12. Plan

```powershell
terraform init
terraform validate
terraform plan -out="lab-11.tfplan"
```

Le plan doit créer :

```text
Private DNS Zone
VNet Link
Private Endpoint
Private DNS Zone Group
```

Puis appliquer :

```powershell
terraform apply ".\lab-11.tfplan"
```

## 13. Vérifier le Private Endpoint

```powershell
az network private-endpoint show `
  --name "pe-kv-dev-fce-001" `
  --resource-group "rg-mla-dev-fce-001" `
  --query "{Name:name,State:privateLinkServiceConnections[0].privateLinkServiceConnectionState.status}" `
  --output json
```

Résultat attendu :

```text
Approved
```

## 14. Vérifier la zone DNS

```powershell
az network private-dns zone show `
  --name "privatelink.vaultcore.azure.net" `
  --resource-group "rg-mla-dev-fce-001" `
  --output table
```

Lister les enregistrements :

```powershell
az network private-dns record-set a list `
  --zone-name "privatelink.vaultcore.azure.net" `
  --resource-group "rg-mla-dev-fce-001" `
  --output table
```

Un A record correspondant au Key Vault doit pointer vers l’adresse privée du Private Endpoint.

## 15. Vérifier l’adresse privée

```powershell
az network private-endpoint show `
  --name "pe-kv-dev-fce-001" `
  --resource-group "rg-mla-dev-fce-001" `
  --query "customDnsConfigs[].{FQDN:fqdn,IPs:ipAddresses}" `
  --output json
```

## 16. Comprendre la résolution DNS

Depuis un client qui utilise la résolution Azure Private DNS liée au VNet :

```text
<kv>.vault.azure.net
       │
       ▼
CNAME
<kv>.privatelink.vaultcore.azure.net
       │
       ▼
A record privé
10.20.2.x
```

Depuis un poste situé hors du réseau privé et sans résolution hybride appropriée, le nom peut continuer à être visible dans le DNS public, mais l’accès Data Plane échoue puisque le public network access est désactivé.

## 17. Test depuis le poste local

```powershell
Resolve-DnsName "$($kvName).vault.azure.net"
```

Ne pas s’attendre nécessairement à obtenir l’IP privée si le poste local n’est pas connecté au VNet ou à un DNS hybride capable de résoudre la zone privée.

C’est normal.

La Private DNS Zone Azure est liée au VNet, pas automatiquement au poste de travail de l’ingénieur.

## 18. Test d’accès hors réseau privé

Avec public network access désactivé :

```powershell
az keyvault secret list `
  --vault-name $kvName
```

Depuis un poste non connecté au réseau privé, la requête Data Plane doit échouer au niveau réseau même si l’identité possède un rôle Data Plane.

Nous démontrons ainsi que :

```text
RBAC correct
+
réseau incorrect
=
accès impossible
```

## 19. DNS hybride

Dans une entreprise, les clients on-premises ou des réseaux connectés doivent également résoudre les zones Private Link.

Nous pouvons utiliser :

```text
On-Prem DNS
   │
   ▼
Conditional Forwarder
   │
   ▼
Azure Private DNS Resolver
   │
   ▼
Azure Private DNS Zone
```

Le `/etc/hosts` ou le fichier `hosts` Windows n’est pas une solution d’architecture durable.

## 20. Private DNS centralisé

Dans une Landing Zone Hub & Spoke, on évitera souvent de créer les mêmes zones Private DNS dans chaque spoke.

Une cible plus mature est :

```text
Hub / Connectivity
│
├── Private DNS Zones
├── Private DNS Resolver
└── VNet Links
     │
     ├── Spoke DEV
     ├── Spoke QUAL
     └── Spoke PROD
```

Le Lab conserve volontairement la zone dans le Resource Group DEV pour rester lisible.

## 21. Troubleshooting

Si l’accès ne fonctionne pas, vérifier dans cet ordre :

```text
1. Private Endpoint existe
2. Connection State = Approved
3. Private IP attribuée
4. Zone DNS exacte
5. VNet Link présent
6. A record présent
7. Client utilise un DNS capable d'interroger la zone privée
8. Public Network Access correctement configuré
9. RBAC Data Plane correct
```

## 22. Validation

```text
[ ] Private Endpoint créé
[ ] subresource = vault
[ ] zone privatelink.vaultcore.azure.net créée
[ ] zone liée au VNet
[ ] zone group associé au Private Endpoint
[ ] IP privée identifiée
[ ] A record privé observé
[ ] public network access désactivé
[ ] comportement DNS hors VNet compris
[ ] différence DNS privé / RBAC comprise
[ ] architecture DNS hybride comprise
```

## 23. À retenir

Private Endpoint sans DNS privé est une architecture incomplète.

Pour un service PaaS privé, nous devons raisonner :

```text
Route
+
Private IP
+
DNS
+
Identity
+
Authorization
```

Le prochain Lab va ajouter une application réelle à notre plateforme.

**LAB-12 — Déployer une application Azure avec Terraform : ACR et Container Apps**

## Références officielles

- Microsoft Learn — Azure Private Endpoint DNS configuration
- Microsoft Learn — Integrate Key Vault with Azure Private Link
- HashiCorp AzureRM — `azurerm_private_endpoint`
- HashiCorp AzureRM — `azurerm_private_dns_zone`
- HashiCorp AzureRM — `azurerm_private_dns_zone_virtual_network_link`
