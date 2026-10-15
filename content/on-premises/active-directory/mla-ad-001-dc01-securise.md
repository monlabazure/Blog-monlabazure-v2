---
title: "MLA-AD-001 — Déployer et sécuriser le premier contrôleur de domaine MLA-DC01"
date: 2026-09-23
draft: false
description: "Déploiement sécurisé du premier contrôleur de domaine MonLabAzure sous Windows Server 2025 : AD DS, DNS, forêt ad.monlabazure.fr, OSConfig Security Baseline, Defender et contrôles de santé."
categories:
  - "On-Premises"
tags:
  - "MonLabAzure"
  - "Active Directory"
  - "AD DS"
  - "DNS"
  - "Windows Server 2025"
  - "Domain Controller"
  - "Security Baseline"
  - "OSConfig"
  - "Tier 0"
  - "Sécurité"
series:
  - "Active Directory"
weight: 30
---

# MLA-AD-001 — Déployer et sécuriser le premier contrôleur de domaine MLA-DC01

Le réseau MonLabAzure est maintenant segmenté avec pfSense et nous disposons d'un hôte d'administration dédié avec `MLA-MGMT01`.

Nous pouvons donc commencer la construction d'Active Directory.

Cette étape ne consiste pas simplement à installer le rôle **Active Directory Domain Services (AD DS)** et cliquer sur *Promote this server to a domain controller*.

Un contrôleur de domaine constitue une partie critique de l'infrastructure d'identité.

Une compromission d'un Domain Controller (DC) peut donner à un attaquant un niveau de contrôle très important sur le système d'information.

Nous allons donc construire `MLA-DC01` avec une approche différente d'un serveur Windows généraliste :

- rôle dédié ;
- réseau dédié ;
- administration contrôlée ;
- services inutiles réduits ;
- Microsoft Defender actif ;
- Windows Firewall actif ;
- Security Baseline Windows Server 2025 ;
- DNS intégré à Active Directory ;
- contrôles de santé après promotion ;
- aucun usage bureautique ou applicatif.

Le principe retenu pour MonLabAzure est simple :

> Un Domain Controller doit être sécurisé dès son déploiement, pas après sa mise en production.

---

## Prérequis

Cet article s'appuie sur les étapes précédentes :

[**MLA-NET-001 — Construire le réseau MonLabAzure et déployer pfSense**]({{< relref "on-premises/reseau/mla-net-001-reseau-pfsense.md" >}})

[**MLA-MGMT-001 — Déployer et sécuriser MLA-MGMT01**]({{< relref "on-premises/administration/mla-mgmt-001-administration-securisee.md" >}})

---

# Architecture

![Architecture sécurisée du premier contrôleur de domaine MonLabAzure](/images/on-premises/active-directory/mla-ad-001-architecture-dc01-securise.png)

La première forêt Active Directory utilisera :

| Élément | Valeur |
|---|---|
| Serveur | `MLA-DC01` |
| OS | Windows Server 2025 |
| VLAN | VLAN 20 — IDENTITY |
| Adresse IPv4 | `10.10.20.10/24` |
| Passerelle | `10.10.20.1` |
| Domaine AD DS | `ad.monlabazure.fr` |
| NetBIOS | `MONLABAZURE` |
| Rôles | AD DS + DNS |
| Administration | `MLA-MGMT01` |

Le domaine public reste :

```text
monlabazure.fr
````

Le domaine Active Directory est volontairement :

```text
ad.monlabazure.fr
```

Nous évitons ainsi les anciens schémas utilisant des suffixes comme :

```text
.local
```

Le futur User Principal Name (UPN) des utilisateurs pourra malgré tout être :

```text
user01@monlabazure.fr
```

Le suffixe UPN public sera ajouté ultérieurement.

---

# 1. Pourquoi AD DS et DNS sur le même serveur ?

Active Directory dépend fortement du Domain Name System (DNS).

Les clients et serveurs utilisent notamment DNS pour localiser :

* les contrôleurs de domaine ;
* les services Kerberos ;
* les Global Catalog ;
* les services LDAP ;
* les différents services Active Directory publiés via des enregistrements SRV.

Pour le premier DC, nous installons donc :

```text
MLA-DC01
├── Active Directory Domain Services
└── DNS Server
```

La zone :

```text
ad.monlabazure.fr
```

sera intégrée à Active Directory.

Cela permettra plus tard sa réplication automatiquement vers `MLA-DC02`.

---

# 2. Positionnement réseau

`MLA-DC01` est placé exclusivement dans :

```text
VLAN 20 — IDENTITY
10.10.20.0/24
```

Configuration :

```text
IPv4       10.10.20.10
Mask       255.255.255.0
Gateway    10.10.20.1
```

La passerelle :

```text
10.10.20.1
```

correspond à l'interface pfSense du VLAN IDENTITY.

Le DC ne possède qu'une seule interface réseau.

Nous évitons un DC multi-homed avec plusieurs cartes réseau, qui compliquerait notamment :

* les enregistrements DNS ;
* la localisation des DC ;
* le routage ;
* le filtrage ;
* le diagnostic réseau.

---

# 3. Préparer la VM

Configuration recommandée pour le lab :

| Ressource   |      Configuration |
| ----------- | -----------------: |
| vCPU        |                  4 |
| RAM         |               8 Go |
| Disque OS   |              80 Go |
| Réseau      | VLAN 20 — IDENTITY |
| vTPM        |  Oui si disponible |
| Secure Boot |  Oui si disponible |

Le serveur utilise :

```text
Windows Server 2025
```

avec Desktop Experience pour faciliter les premières manipulations du lab.

Le rôle du serveur reste néanmoins dédié à Active Directory et DNS.

---

# 4. Installer Windows Server

Après installation :

1. terminer la configuration initiale ;
2. vérifier la date et l'heure ;
3. installer les mises à jour ;
4. mettre Microsoft Defender à jour ;
5. ne pas installer d'applications supplémentaires ;
6. vérifier Windows Firewall.

À ce stade le serveur est encore dans un workgroup.

---

# 5. Renommer le serveur

Vérifier le nom actuel :

```powershell
hostname
```

Renommer :

```powershell
Rename-Computer -NewName "MLA-DC01" -Restart
```

Après redémarrage :

```powershell
hostname
```

Résultat attendu :

```text
MLA-DC01
```

---

# 6. Configurer l'adresse IP

Identifier l'interface :

```powershell
Get-NetAdapter
```

Puis :

```powershell
Get-NetIPConfiguration
```

Configurer l'IPv4 statique :

```powershell
New-NetIPAddress `
  -InterfaceAlias "Ethernet" `
  -IPAddress "10.10.20.10" `
  -PrefixLength 24 `
  -DefaultGateway "10.10.20.1"
```

Adapter le nom `Ethernet` si nécessaire.

Vérifier :

```powershell
Get-NetIPConfiguration
```

---

# 7. DNS avant la création du domaine

C'est une phase temporaire.

Le DNS Active Directory n'existe pas encore.

Le serveur peut utiliser provisoirement le résolveur prévu pour la phase de bootstrap afin d'accéder aux services nécessaires :

* Windows Update ;
* PowerShell Gallery ;
* dépôts Microsoft.

Cette configuration ne doit pas rester après la promotion.

Une fois `MLA-DC01` devenu contrôleur de domaine et serveur DNS, sa configuration DNS devra pointer vers le DNS Active Directory.

Dans notre cas :

```text
DNS préféré
10.10.20.10
```

Nous n'ajouterons pas :

```text
8.8.8.8
1.1.1.1
```

comme DNS secondaire directement sur la carte réseau du Domain Controller.

Les résolutions Internet seront prises en charge par le service DNS via ses redirecteurs ou ses mécanismes de résolution.

---

# 8. Vérifier le réseau avant AD

Tester la passerelle :

```powershell
Test-NetConnection 10.10.20.1
```

Vérifier l'adresse :

```powershell
Get-NetIPAddress -AddressFamily IPv4
```

Vérifier la route par défaut :

```powershell
Get-NetRoute -DestinationPrefix "0.0.0.0/0"
```

À ce stade, les problèmes réseau doivent être corrigés avant d'installer Active Directory.

Un problème d'IP, de DNS ou de temps avant la promotion deviendra généralement plus difficile à diagnostiquer après.

---

# 9. Vérifier l'heure

Kerberos dépend fortement d'une heure cohérente entre les systèmes.

Avant la promotion :

```powershell
Get-Date
```

Puis :

```powershell
w32tm /query /status
```

Nous vérifierons également :

```powershell
w32tm /query /source
```

Après création de la forêt, `MLA-DC01` détiendra initialement le rôle **Primary Domain Controller Emulator (PDC Emulator)**.

Le PDC Emulator de la racine de forêt doit utiliser une source NTP fiable plutôt que de dépendre uniquement de l'horloge de la VM.

La configuration NTP définitive sera détaillée dans l'article consacré au DNS, au temps et aux contrôles de santé Active Directory.

---

# 10. Préparer OSConfig

Avant de retirer l'accès Internet de bootstrap, nous pouvons installer le module OSConfig :

```powershell
Install-Module `
  -Name Microsoft.OSConfig `
  -Scope AllUsers `
  -Force
```

Vérifier :

```powershell
Get-Module -ListAvailable -Name Microsoft.OSConfig
```

Nous ne sommes pas encore Domain Controller.

Nous n'appliquons donc pas immédiatement la baseline `DomainController`.

Elle sera appliquée après la promotion, lorsque le rôle réel du serveur sera connu.

---

# 11. Microsoft Defender

Vérifier son état :

```powershell
Get-MpComputerStatus
```

Contrôler notamment :

```text
AntivirusEnabled
RealTimeProtectionEnabled
AntispywareEnabled
```

Microsoft Defender reste actif sur le Domain Controller.

Nous n'ajoutons pas d'exclusions antivirus arbitraires.

Windows Defender connaît déjà les exclusions automatiques nécessaires aux rôles Windows Server pris en charge.

---

# 12. Windows Firewall

Vérifier :

```powershell
Get-NetFirewallProfile |
Select-Object Name, Enabled, DefaultInboundAction, DefaultOutboundAction
```

Le firewall Windows doit rester actif.

Nous avons donc deux niveaux de filtrage :

```text
pfSense
    │
    │ segmentation inter-VLAN
    ▼
Windows Firewall
    │
    │ protection locale
    ▼
MLA-DC01
```

pfSense ne remplace pas Windows Firewall.

Windows Firewall ne remplace pas non plus la segmentation réseau.

Les deux contrôles sont complémentaires.

---

# 13. Snapshot avant promotion

Dans le contexte du lab, c'est un bon moment pour prendre un snapshot.

Par exemple :

```text
01-dc01-pre-promotion
```

Ce snapshot représente :

```text
Windows installé
IP configurée
Patching terminé
Defender actif
OSConfig disponible
AD DS non installé
```

En cas d'erreur importante pendant le bootstrap, nous pouvons revenir à cet état.

Ce snapshot n'est pas une sauvegarde Active Directory.

Une vraie stratégie de sauvegarde AD sera mise en place après la construction des DC.

---

# 14. Installer Active Directory Domain Services

Installer le rôle :

```powershell
Install-WindowsFeature `
  AD-Domain-Services `
  -IncludeManagementTools
```

Vérifier :

```powershell
Get-WindowsFeature AD-Domain-Services
```

Résultat attendu :

```text
Install State : Installed
```

---

# 15. Créer la forêt Active Directory

Nous allons créer une nouvelle forêt :

```text
ad.monlabazure.fr
```

avec le nom NetBIOS :

```text
MONLABAZURE
```

Le rôle DNS sera installé en même temps.

Créer d'abord le mot de passe **Directory Services Restore Mode (DSRM)** :

```powershell
$DSRM = Read-Host `
  "Mot de passe DSRM" `
  -AsSecureString
```

Ce mot de passe doit être :

* unique ;
* complexe ;
* différent d'un mot de passe administrateur quotidien ;
* stocké dans un emplacement sécurisé en dehors du DC.

Le DSRM est utilisé pour certaines opérations de récupération d'Active Directory.

---

## Promotion

Exécuter :

```powershell
Install-ADDSForest `
  -DomainName "ad.monlabazure.fr" `
  -DomainNetbiosName "MONLABAZURE" `
  -InstallDns:$true `
  -SafeModeAdministratorPassword $DSRM
```

Nous ne désactivons pas les contrôles préalables.

En particulier, nous évitons :

```text
-SkipPreChecks
```

dans un déploiement normal.

Le serveur redémarrera après la promotion.

Microsoft installe également DNS par défaut lors de la création d'une nouvelle forêt avec `Install-ADDSForest`.

---

# 16. Niveau fonctionnel Windows Server 2025

MonLabAzure est une nouvelle forêt composée de contrôleurs de domaine Windows Server 2025.

Nous pouvons donc utiliser le niveau fonctionnel Windows Server 2025.

Microsoft recommande d'utiliser le niveau fonctionnel le plus élevé compatible avec tous les contrôleurs de domaine de l'environnement.

Après la promotion, vérifier :

```powershell
Get-ADDomain |
Select-Object DNSRoot, DomainMode
```

Puis :

```powershell
Get-ADForest |
Select-Object Name, ForestMode
```

Dans notre architecture cible :

```text
DomainMode : Windows Server 2025
ForestMode : Windows Server 2025
```

Un point important : un niveau fonctionnel Windows Server 2025 implique que les Domain Controllers doivent eux-mêmes être compatibles avec ce niveau.

Nous ne pourrons donc pas introduire ensuite un ancien DC Windows Server 2019 ou 2022 dans une forêt fonctionnant au niveau 2025.

C'est un choix volontaire dans MonLabAzure.

---

# 17. Première connexion après promotion

Après redémarrage, la machine appartient maintenant au domaine :

```text
ad.monlabazure.fr
```

Vérifier :

```powershell
Get-CimInstance Win32_ComputerSystem |
Select-Object Name, Domain
```

Résultat attendu :

```text
Name    : MLA-DC01
Domain  : ad.monlabazure.fr
```

---

# 18. Vérifier les rôles FSMO

Comme `MLA-DC01` est le premier et seul DC de la forêt, il détient initialement les cinq Flexible Single Master Operations (FSMO).

Vérifier :

```powershell
netdom query fsmo
```

Nous devons retrouver :

```text
Schema Master
Domain Naming Master
PDC Emulator
RID Master
Infrastructure Master
```

sur :

```text
MLA-DC01
```

Cela changera éventuellement plus tard selon l'architecture retenue.

---

# 19. Corriger la configuration DNS du serveur

Après la promotion, le DNS du serveur doit pointer vers Active Directory DNS.

Vérifier :

```powershell
Get-DnsClientServerAddress `
  -AddressFamily IPv4
```

La configuration cible à ce stade est :

```text
MLA-DC01
DNS : 10.10.20.10
```

Lorsque `MLA-DC02` sera installé, nous réévaluerons la configuration des DNS préférés et secondaires des deux Domain Controllers.

---

# 20. Vérifier la zone DNS

Lister les zones :

```powershell
Get-DnsServerZone
```

Nous devons notamment retrouver :

```text
ad.monlabazure.fr
```

et la zone associée à :

```text
_msdcs.ad.monlabazure.fr
```

Les enregistrements DNS d'Active Directory sont essentiels au fonctionnement du domaine.

---

# 21. Vérifier les enregistrements SRV

Active Directory publie ses services via des enregistrements DNS de type **Service Record (SRV)**.

Tester :

```powershell
nslookup -type=SRV `
  _ldap._tcp.dc._msdcs.ad.monlabazure.fr
```

Le résultat doit référencer :

```text
MLA-DC01.ad.monlabazure.fr
```

Tester également :

```powershell
Resolve-DnsName `
  _kerberos._tcp.ad.monlabazure.fr `
  -Type SRV
```

Ces tests sont beaucoup plus utiles qu'un simple ping pour déterminer si la découverte Active Directory fonctionne correctement.

---

# 22. Vérifier SYSVOL et NETLOGON

Active Directory utilise notamment les partages :

```text
SYSVOL
NETLOGON
```

Vérifier :

```powershell
Get-SmbShare |
Where-Object Name -in "SYSVOL","NETLOGON"
```

Les deux partages doivent être présents.

Ils seront notamment utilisés pour :

* les Group Policy Objects (GPO) ;
* les scripts ;
* différents mécanismes de domaine.

---

# 23. Vérifier les services essentiels

Contrôler :

```powershell
Get-Service `
  NTDS,
  DNS,
  Netlogon,
  Kdc,
  W32Time
```

Les services importants doivent être dans l'état attendu.

En particulier :

```text
NTDS       Active Directory Domain Services
DNS        DNS Server
Netlogon   Net Logon
Kdc        Kerberos Key Distribution Center
W32Time    Windows Time
```

---

# 24. Exécuter DCDIAG

L'outil `dcdiag` est indispensable pour vérifier la santé d'un Domain Controller.

Exécuter :

```powershell
dcdiag
```

Puis une analyse détaillée :

```powershell
dcdiag /v
```

Pour DNS :

```powershell
dcdiag /test:dns /v
```

Nous ne devons pas considérer la promotion comme terminée tant que les erreurs significatives de `dcdiag` ne sont pas comprises.

Un warning n'est pas automatiquement critique.

Une erreur ne doit pas non plus être ignorée simplement parce que l'authentification semble fonctionner.

---

# 25. Appliquer la Security Baseline Domain Controller

`MLA-DC01` est maintenant réellement un Domain Controller.

Nous pouvons appliquer la baseline correspondant à son rôle :

```powershell
Set-OSConfigDesiredConfiguration `
  -Scenario SecurityBaseline/WindowsServer/2025/DomainController `
  -Default
```

Redémarrer si nécessaire :

```powershell
Restart-Computer
```

Puis vérifier :

```powershell
Get-OSConfigDesiredConfiguration `
  -Scenario SecurityBaseline/WindowsServer/2025/DomainController
```

Pour examiner la conformité :

```powershell
Get-OSConfigDesiredConfiguration `
  -Scenario SecurityBaseline/WindowsServer/2025/DomainController |
Format-Table `
  Name,
  @{Name="Status";Expression={$_.Compliance.Status}},
  @{Name="Reason";Expression={$_.Compliance.Reason}} `
  -AutoSize -Wrap
```

La baseline Windows Server 2025 fournit un socle de paramètres adapté spécifiquement au rôle Domain Controller.

Elle ne remplace pas notre architecture de sécurité, mais évite de repartir d'une configuration Windows générique et de durcir manuellement des centaines de paramètres.

OSConfig surveille également les paramètres de la baseline afin de limiter la dérive de configuration.

---

# 26. Baseline Microsoft Defender

Appliquer également le scénario Microsoft Defender :

```powershell
Set-OSConfigDesiredConfiguration `
  -Scenario Defender/Antivirus/WindowsServer/2025 `
  -Default
```

Vérifier :

```powershell
Get-OSConfigDesiredConfiguration `
  -Scenario Defender/Antivirus/WindowsServer/2025
```

Puis :

```powershell
Get-MpComputerStatus
```

---

# 27. LDAP Signing

Lightweight Directory Access Protocol (LDAP) est utilisé par de nombreuses applications pour communiquer avec Active Directory.

Windows Server 2025 améliore le comportement de sécurité des nouveaux déploiements Active Directory.

Dans une nouvelle infrastructure Windows Server 2025, Microsoft impose notamment par défaut la signature LDAP côté Domain Controller.

La signature LDAP apporte une protection d'intégrité et d'authenticité aux communications LDAP.

Elle permet notamment de réduire les risques :

* d'altération du trafic ;
* de replay ;
* d'attaques de type man-in-the-middle.

Nous ne diminuerons donc pas ce niveau de sécurité pour résoudre rapidement un problème applicatif.

Si une application ancienne rencontre un problème, nous commencerons par identifier précisément son mode de connexion LDAP.

---

# 28. LDAP Channel Binding

LDAP Channel Binding ajoute une protection supplémentaire pour certaines communications LDAP utilisant TLS.

Windows Server 2025 active une posture plus stricte que les anciennes versions, avec notamment le support et l'audit du Channel Binding.

L'enforcement complet sera traité après avoir validé les clients et applications.

Dans un SI existant, ce point mérite toujours une phase d'audit avant durcissement.

Dans MonLabAzure, nous avons l'avantage de construire l'environnement à partir de zéro : les futurs services devront s'adapter aux exigences de sécurité de l'annuaire, et non l'inverse.

---

# 29. Désactiver Print Spooler

Un Domain Controller n'a normalement aucune raison d'imprimer.

Microsoft recommande de désactiver le service Print Spooler sur les Domain Controllers qui n'en ont pas besoin.

Vérifier :

```powershell
Get-Service Spooler
```

Arrêter le service :

```powershell
Stop-Service Spooler
```

Puis :

```powershell
Set-Service `
  Spooler `
  -StartupType Disabled
```

Contrôler :

```powershell
Get-Service Spooler |
Select-Object Name, Status, StartType
```

Résultat cible :

```text
Status    : Stopped
StartType : Disabled
```

Cette configuration sera ensuite centralisée par GPO lorsque la structure Active Directory sera en place.

---

# 30. Ne pas utiliser le DC comme poste d'administration

Une erreur classique consiste à se connecter directement sur le Domain Controller pour effectuer toutes les opérations Active Directory.

Nous voulons éviter cette habitude.

La console locale de `MLA-DC01` reste utilisée pour :

* installation ;
* récupération ;
* dépannage nécessitant un accès local ;
* situations exceptionnelles.

L'administration quotidienne sera effectuée depuis :

```text
MLA-MGMT01
```

après son intégration au domaine.

Architecture cible :

```text
Administrateur privilégié
          │
          ▼
     MLA-MGMT01
          │
          ▼
     MLA-DC01 / DC02
```

et non :

```text
Administrateur
     │
     ▼
connexion quotidienne au DC
```

---

# 31. Comptes d'administration

Pour construire la première forêt, le compte administrateur bootstrap est nécessaire.

Mais ce compte ne doit pas devenir notre compte d'administration quotidien.

Nous construirons ensuite des comptes dédiés, par exemple :

```text
user01
```

pour l'utilisation standard,

et :

```text
adm-domain01
```

pour certaines tâches d'administration Active Directory.

La structuration complète :

* comptes ;
* groupes ;
* délégations ;
* comptes Tier 0 ;
* restrictions de logon ;

sera traitée dans les articles suivants.

---

# 32. pfSense et les flux Active Directory

Nous conservons le principe :

```text
DENY BY DEFAULT
```

Les flux seront ouverts uniquement lorsqu'ils sont nécessaires.

Active Directory utilise cependant plusieurs protocoles.

Parmi les principaux :

| Service             | Port            |
| ------------------- | --------------- |
| DNS                 | TCP/UDP 53      |
| Kerberos            | TCP/UDP 88      |
| NTP                 | UDP 123         |
| RPC Endpoint Mapper | TCP 135         |
| LDAP                | TCP/UDP 389     |
| SMB                 | TCP 445         |
| Kerberos password   | TCP/UDP 464     |
| LDAPS               | TCP 636         |
| Global Catalog      | TCP 3268        |
| Global Catalog SSL  | TCP 3269        |
| RPC dynamique       | TCP 49152-65535 |

Nous n'allons pas créer une règle :

```text
MGMT → IDENTITY → ANY
```

simplement parce qu'Active Directory utilise plusieurs ports.

Les règles seront regroupées avec des alias pfSense documentés.

Par exemple :

```text
SRV_DOMAIN_CONTROLLERS
PORTS_AD_CORE
PORTS_AD_ADMIN
```

La matrice détaillée des flux sera affinée lors de l'intégration de `MLA-MGMT01` et de l'arrivée de `MLA-DC02`.

---

# 33. Aucun accès Internet direct permanent

Un Domain Controller ne doit pas devenir une machine utilisée pour :

* naviguer sur Internet ;
* télécharger des outils ;
* consulter une messagerie ;
* installer des utilitaires divers.

L'accès utilisé pour :

* patching ;
* téléchargement d'OSConfig ;
* mises à jour Defender ;

doit être contrôlé.

À terme :

```text
MLA-DC01
   │
   ├── services AD nécessaires
   ├── DNS contrôlé
   ├── NTP approuvé
   ├── mises à jour contrôlées
   │
   X
Internet général
```

---

# 34. NTP et PDC Emulator

Le premier DC possède le rôle :

```text
PDC Emulator
```

dans la racine de forêt.

Il devient donc un élément central de la hiérarchie temporelle du domaine.

Les membres du domaine synchroniseront leur temps avec la hiérarchie Active Directory.

Le PDC Emulator racine devra lui-même utiliser une source NTP externe ou interne faisant autorité.

Vérifier :

```powershell
w32tm /query /configuration
```

et :

```powershell
w32tm /query /source
```

La configuration définitive sera réalisée dans :

```text
MLA-AD-003 — DNS, NTP et contrôles de santé Active Directory
```

Nous ne choisirons pas arbitrairement un serveur NTP public sans documenter cette dépendance.

---

# 35. Vérifier les journaux

Consulter Event Viewer.

Les journaux principaux sont :

```text
Directory Service
DNS Server
System
Security
```

Une promotion réussie n'implique pas que tous les composants fonctionnent correctement.

Il faut rechercher notamment :

* erreurs NTDS ;
* erreurs DNS ;
* erreurs Netlogon ;
* problèmes Kerberos ;
* problèmes W32Time.

---

# 36. Ce que nous ne faisons pas encore

Nous ne cherchons pas à tout configurer dans le premier article.

Ne sont pas encore traités :

* deuxième Domain Controller ;
* réplication inter-DC ;
* Sites and Services ;
* structure des Organizational Units ;
* comptes utilisateurs ;
* groupes ;
* délégations ;
* Windows LAPS ;
* politiques Tier 0 complètes ;
* Restricted Groups ;
* Authentication Policies ;
* Authentication Policy Silos ;
* Protected Users ;
* PKI ;
* LDAPS avec certificat ;
* sauvegarde System State ;
* restauration AD ;
* Active Directory Recycle Bin ;
* supervision Wazuh ;
* Microsoft Defender for Identity.

Ces sujets seront ajoutés progressivement plutôt que d'être configurés sans contexte.

---

# 37. Séquence de déploiement

![Séquence de déploiement de MLA-DC01](/images/on-premises/active-directory/mla-ad-001-sequence-deploiement-dc01.png)

Le déroulement retenu est :

```text
Windows Server 2025
        ↓
IP statique
        ↓
Patching / Defender
        ↓
AD DS + DNS
        ↓
Création forêt
        ↓
ad.monlabazure.fr
        ↓
Security Baseline DC
        ↓
Contrôles DNS / AD
        ↓
Durcissement
        ↓
Administration depuis MLA-MGMT01
```

Chaque étape doit être validée avant de poursuivre.

---

# 38. Checklist finale

Avant de considérer `MLA-DC01` comme opérationnel :

## Système

* [ ] Windows Server 2025 est à jour.
* [ ] Le serveur se nomme `MLA-DC01`.
* [ ] L'adresse IP est statique.
* [ ] Microsoft Defender est actif.
* [ ] Windows Firewall est actif.
* [ ] Aucun logiciel inutile n'est installé.

## Active Directory

* [ ] La forêt `ad.monlabazure.fr` existe.
* [ ] Le domaine `ad.monlabazure.fr` existe.
* [ ] Le NetBIOS est `MONLABAZURE`.
* [ ] `MLA-DC01` est Global Catalog.
* [ ] Les rôles FSMO sont identifiés.
* [ ] SYSVOL est partagé.
* [ ] NETLOGON est partagé.

## DNS

* [ ] La zone AD existe.
* [ ] `_msdcs` est présent.
* [ ] Les enregistrements SRV sont présents.
* [ ] Le DC utilise le DNS AD.
* [ ] Aucun DNS public n'est configuré directement sur la NIC du DC.

## Sécurité

* [ ] La baseline OSConfig `DomainController` est appliquée.
* [ ] La baseline Defender est appliquée.
* [ ] Print Spooler est désactivé.
* [ ] LDAP Signing n'a pas été affaibli.
* [ ] L'accès Internet général n'est pas utilisé.
* [ ] L'administration quotidienne ne se fait pas directement sur le DC.

## Santé

* [ ] `dcdiag` ne remonte aucune erreur critique inexpliquée.
* [ ] `dcdiag /test:dns` est contrôlé.
* [ ] Les journaux Directory Service et DNS sont vérifiés.
* [ ] La source de temps est connue.

---

# Lab versus production

Dans MonLabAzure, le domaine commence avec un seul Domain Controller.

Pendant cette courte phase :

```text
MLA-DC01
```

constitue donc un **Single Point of Failure (SPOF)** pour Active Directory et DNS.

Cette situation n'est acceptable que pendant la construction initiale du lab.

L'étape suivante côté AD consistera à déployer :

```text
MLA-DC02
10.10.20.11
```

afin d'obtenir :

* redondance Active Directory ;
* second serveur DNS ;
* réplication ;
* meilleure résilience.

Nous ne considérerons donc pas l'infrastructure Active Directory comme résiliente tant que `MLA-DC02` n'est pas opérationnel.

---

# Références officielles

Les points techniques et de sécurité de cet article s'appuient notamment sur les documentations Microsoft suivantes :

* **Installer les services de domaine Active Directory sur Windows Server**
* **Install-ADDSForest — ADDSDeployment**
* **Niveaux fonctionnels Active Directory Domain Services**
* **Configurer les bases de référence de sécurité Windows Server 2025 avec OSConfig**
* **LDAP Signing for Active Directory Domain Services**
* **Windows Time Service**
* **Configurer le PDC Emulator avec une source de temps faisant autorité**
* **Microsoft Defender for Identity — Disable Print Spooler service on Domain Controllers**

Les recommandations de sécurité doivent toujours être validées avec les contraintes des applications et services existants avant application dans un environnement de production.

---

# Résultat obtenu

Nous disposons maintenant du premier composant d'identité MonLabAzure :

```text
MLA-DC01
│
├── Windows Server 2025
├── AD DS
├── DNS
├── ad.monlabazure.fr
├── MONLABAZURE
├── Global Catalog
├── FSMO
├── Microsoft Defender
├── Windows Firewall
├── OSConfig Security Baseline
├── LDAP Signing
└── Print Spooler désactivé
```

Mais il reste une étape importante avant de continuer à construire Active Directory.

`MLA-MGMT01` a été préparé en workgroup avant la création du domaine.

Maintenant que :

```text
ad.monlabazure.fr
```

existe, nous allons pouvoir l'intégrer au domaine et terminer sa configuration comme hôte d'administration privilégié.

## Prochaine étape

[**MLA-MGMT-002 — Intégrer MLA-MGMT01 au domaine et sécuriser l'administration Tier 0**]({{< relref "on-premises/administration/mla-mgmt-002-integration-domaine-tier0.md" >}})

Après cette étape, nous reviendrons sur Active Directory avec :

**MLA-AD-002 — Déployer MLA-DC02 et valider la réplication Active Directory et DNS.**



J’ai vérifié les points susceptibles d’avoir évolué avec Windows Server 2025 : une nouvelle forêt uniquement en Windows Server 2025 peut utiliser le niveau fonctionnel 2025 ; Microsoft fournit bien une baseline OSConfig spécifique `DomainController` et OSConfig protège les paramètres de baseline contre la dérive.

Pour les nouveaux déploiements Windows Server 2025, Microsoft indique aussi que LDAP signing est requis par défaut côté DC, avec Channel Binding configuré sur « when supported » et audit activé. La désactivation du Print Spooler sur les DC qui n’en ont pas besoin reste une recommandation Microsoft Defender for Identity.

Enfin, le premier DC de la racine détient le rôle PDC Emulator et se trouve au sommet de la hiérarchie de temps AD ; Microsoft recommande de lui fournir une source de temps faisant autorité plutôt que de le laisser dépendre de sa seule horloge locale.

