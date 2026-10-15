---
title: "MLA-MGMT-001 - Déployer et sécuriser MLA-MGMT01"
date: 2026-09-22
draft: false
description: "Déploiement de MLA-MGMT01 comme hôte d'administration privilégié : Windows Server 2025, réseau Management, durcissement, OSConfig, Credential Guard, Defender et RSAT."
categories:
  - "On-Premises"
tags:
  - "MonLabAzure"
  - "Administration"
  - "Active Directory"
  - "Tier 0"
  - "Windows Server 2025"
  - "RSAT"
  - "Security Baseline"
  - "Credential Guard"
  - "Windows LAPS"
series:
  - "Infrastructure On-Premises"
weight: 20
---

# MLA-MGMT-001 - Déployer et sécuriser MLA-MGMT01

Avant de déployer le premier contrôleur de domaine, nous allons préparer la machine depuis laquelle il sera administré.

Cette étape est importante.

Un Domain Controller (DC) peut être correctement durci et rester exposé si les comptes privilégiés utilisés pour l'administrer transitent par un poste standard sur lequel on consulte ses mails, navigue sur Internet ou installe différents outils.

Dans MonLabAzure, l'administration d'Active Directory ne sera donc pas réalisée depuis le poste utilisateur ni directement depuis les Domain Controllers.

Nous allons construire un hôte dédié :

```text
MLA-MGMT01
````

Son rôle sera simple : **fournir une plateforme de confiance depuis laquelle les opérations d'administration privilégiées seront réalisées**.

Microsoft recommande de ne pas administrer un système de confiance depuis une machine moins sécurisée et décrit les Secure Administrative Hosts comme des postes ou serveurs dédiés à l'administration, sans outils bureautiques, messagerie ou navigation Web classique.

---

## Architecture d'administration

![Architecture d'administration sécurisée MonLabAzure](/images/on-premises/administration/mla-mgmt-001-administration-securisee.png)

*MLA-MGMT01 devient le point d'administration privilégié des composants critiques du lab.*


> Pour Microsoft, la restriction des connexions de comptes fortement privilégiés vers les serveurs et postes de confiance inférieure constitue une mesure importante contre le vol d'identifiants.

---

# 1. Quel rôle pour MLA-MGMT01 ?

`MLA-MGMT01` ne doit pas héberger :

* Active Directory Domain Services ;
* DNS Server ;
* IIS ;
* SQL Server ;
* une autorité de certification ;
* des applications métiers ;
* un navigateur utilisé pour surfer sur Internet ;
* une messagerie ;
* Microsoft Office ;
* Teams ;
* des outils personnels.

Son rôle est l'administration, nous y installerons principalement :

* Remote Server Administration Tools (RSAT) ;
* PowerShell ;
* Active Directory Users and Computers ;
* Active Directory Administrative Center ;
* Active Directory Sites and Services ;
* DNS Manager ;
* Group Policy Management Console ;
* outils PKI lorsque la PKI sera construite ;
* outils de diagnostic nécessaires.

>Microsoft fournit RSAT sur Windows Server 2025 pour administrer à distance les rôles Windows Server à l'aide des consoles Microsoft Management Console (MMC), de Server Manager, de PowerShell et des outils en ligne de commande.

---

# 2. Une évolution importante de l'architecture MonLabAzure

Dans la première version de l'architecture, `MLA-MGMT01` devait également recevoir des outils comme Git, Terraform ou Azure CLI.

Nous allons volontairement revenir sur ce choix.

À partir du moment où `MLA-MGMT01` sert à administrer Active Directory et les futurs composants Tier 0, il ne doit plus devenir une machine d'administration générale.

Nous conserverons donc :

```text
MLA-MGMT01
Administration Tier 0
```

pour :

* Active Directory ;
* DNS ;
* GPO ;
* comptes privilégiés ;
* PKI ;
* opérations directement liées à l'identité.

Une seconde machine d'administration pourra être introduite plus tard :

```text
MLA-MGMT02
```

pour les activités :

* serveurs membres ;
* Azure ;
* Terraform ;
* Ansible ;
* Git ;
* Azure CLI ;
* automatisation ;
* services Cloud.

Cette séparation évite de déposer des identifiants Active Directory très privilégiés sur une machine utilisée pour des activités beaucoup plus larges.

---

# 3. Configuration de la VM


Système retenu pour le lab :

```text
Nom : MLA-MGMT01
Windows Server 2025 Standard
Desktop Experience
```

Configuration recommandée :

| Ressource      |            Valeur |
| -------------- | ----------------: |
| vCPU           |                 4 |
| RAM            |              8 Go |
| Disque système |             80 Go |
| Carte réseau   |         VLAN MGMT |
| vTPM           | Oui si disponible |
| Secure Boot    | Oui si disponible |



>Dans une architecture d'entreprise, un **Privileged Access Workstation (PAW)** Windows dédié peut également être utilisé. Microsoft considère aussi bien les workstations dédiées que certains jump servers durcis comme des modèles possibles d'administration sécurisée.

---

# 4. Configuration réseau

`MLA-MGMT01` est placé dans le VLAN Management défini dans `MLA-NET-001`.

```text
VLAN      10
Réseau    10.10.10.0/24
Gateway   10.10.10.1
```

Configuration de la machine :

```text
Hostname   MLA-MGMT01
IPv4       10.10.10.10
Mask       255.255.255.0
Gateway    10.10.10.1
```

Avant la création d'Active Directory, le DNS utilisé est temporaire.

Une fois `MLA-DC01` et `MLA-DC02` opérationnels, nous utiliserons :

```text
DNS primaire     10.10.20.10
DNS secondaire   10.10.20.11
```

La machine ne devra pas continuer à utiliser directement un DNS public une fois membre du domaine.

---

# 5. Installation et renommage

Après l'installation de Windows Server 2025, ouvrir PowerShell en administrateur.

Vérifier le nom actuel :

```powershell
hostname
```

Renommer la machine :

```powershell
Rename-Computer -NewName "MLA-MGMT01" -Restart
```

Après redémarrage :

```powershell
hostname
```

Résultat attendu :

```text
MLA-MGMT01
```

---

# 6. Configurer l'adresse IP

Identifier l'interface :

```powershell
Get-NetAdapter
```

Afficher la configuration :

```powershell
Get-NetIPConfiguration
```

Configurer l'adresse en adaptant le nom de l'interface si nécessaire :

```powershell
New-NetIPAddress `
  -InterfaceAlias "Ethernet" `
  -IPAddress "10.10.10.10" `
  -PrefixLength 24 `
  -DefaultGateway "10.10.10.1"
```

Pendant la phase de bootstrap uniquement, configurer le DNS temporaire correspondant à l'environnement du lab.

Vérifier :

```powershell
Get-NetIPConfiguration
```

Puis tester pfSense :

```powershell
Test-NetConnection 10.10.10.1
```

---

# 7. Mettre Windows à jour avant le durcissement

Avant d'utiliser `MLA-MGMT01` pour une opération privilégiée :

1. installer les mises à jour Windows
2. vérifier qu'aucune mise à jour critique n'est en attente
3. installer uniquement les composants nécessaires

Pendant cette phase de bootstrap, un accès Internet temporaire peut être autorisé depuis le VLAN MGMT pour Windows Update et les dépôts Microsoft nécessaires, cet accès sera ensuite retiré.

>Un Secure Administrative Host ne doit pas devenir une machine de navigation Internet. Microsoft recommande explicitement de bloquer l'accès Internet et les usages bureautiques sur les hôtes d'administration sécurisés.

---

# 8. Appliquer une baseline avant même la jonction au domaine

## Comprendre OSConfig et les Security Baselines

Avant d'appliquer la configuration, il est utile de distinguer **OSConfig** de la **Security Baseline**.

### OSConfig

OSConfig est un mécanisme de configuration de sécurité intégré à l'écosystème Windows Server 2025.

Son rôle est d'appliquer une configuration souhaitée puis de vérifier que le serveur reste conforme à cet état dans le temps.

Autrement dit, OSConfig ne se contente pas d'appliquer une série de paramètres une seule fois. Il peut également détecter une dérive de configuration et contribuer à ramener le serveur vers l'état attendu.

Microsoft utilise pour cela des **scénarios**, c'est-à-dire des ensembles prédéfinis de paramètres correspondant à un usage particulier.

Dans notre lab, nous allons principalement utiliser les scénarios liés aux Security Baselines.

> Référence Microsoft :  
> [OSConfig security configuration for Windows Server](https://learn.microsoft.com/en-us/windows-server/security/osconfig/osconfig-overview)

---

### Security Baseline

Une Security Baseline est un ensemble cohérent de paramètres de sécurité recommandés par Microsoft.

L'objectif n'est pas de configurer manuellement des centaines de paramètres Windows, mais de partir d'un socle de sécurité cohérent et adapté au rôle réel du serveur.

Windows Server 2025 propose notamment trois baselines principales :

| Rôle | Scénario OSConfig |
|---|---|
| Serveur hors domaine | `SecurityBaseline/WindowsServer/2025/WorkgroupMember` |
| Serveur membre du domaine | `SecurityBaseline/WindowsServer/2025/MemberServer` |
| Contrôleur de domaine | `SecurityBaseline/WindowsServer/2025/DomainController` |

Dans le cas de `MLA-MGMT01`, nous commencerons avec :

```powershell
Set-OSConfigDesiredConfiguration `
  -Scenario SecurityBaseline/WindowsServer/2025/WorkgroupMember `
  -Default
````

Lorsque la machine sera ensuite intégrée au domaine `ad.monlabazure.fr`, son rôle changera.

Nous retirerons alors la baseline Workgroup et appliquerons celle prévue pour un serveur membre :

```text
WorkgroupMember
        ↓
MemberServer
```

Les Domain Controllers utiliseront de leur côté la baseline spécifique :

```text
SecurityBaseline/WindowsServer/2025/DomainController
```

> Référence Microsoft :
> [Configurer les bases de référence de sécurité pour Windows Server 2025](https://learn.microsoft.com/fr-fr/windows-server/security/osconfig/osconfig-how-to-configure-security-baselines)

---

### Pourquoi l'utiliser dans MonLabAzure ?

L'intérêt est double.

D'abord, nous disposons d'un socle Microsoft cohérent pour le durcissement des serveurs au lieu de modifier les paramètres de sécurité au cas par cas.

Ensuite, nous pouvons vérifier la conformité de la machine après application de la baseline :

```powershell
Get-OSConfigDesiredConfiguration `
  -Scenario SecurityBaseline/WindowsServer/2025/WorkgroupMember
```

OSConfig apporte donc une logique proche de la configuration souhaitée :

```text
Baseline Microsoft
        ↓
      OSConfig
        ↓
Windows Server
        ↓
Contrôle de conformité
        ↓
Détection de dérive
```

La baseline reste néanmoins un **point de départ de sécurité**.

Dans un environnement de production, elle doit être testée avec les applications et les contraintes de l'entreprise avant un déploiement généralisé.

>À retenir pour la suite : **OSConfig est le mécanisme qui applique et maintient la configuration ; la Security Baseline est le jeu de paramètres de sécurité que nous lui demandons d'appliquer.**

- [OSConfig security configuration for Windows Server](https://learn.microsoft.com/en-us/windows-server/security/osconfig/osconfig-overview)
- [Security Baselines Windows Server 2025](https://learn.microsoft.com/fr-fr/windows-server/security/osconfig/osconfig-how-to-configure-security-baselines)
---

## Installer OSConfig

Depuis PowerShell en administrateur :

```powershell
Install-Module -Name Microsoft.OSConfig -Scope AllUsers -Force
```

Vérifier :

```powershell
Get-Module -ListAvailable -Name Microsoft.OSConfig
```

Microsoft prévoit aussi une installation hors ligne du module, ce qui sera intéressant lorsque `MLA-MGMT01` n'aura plus d'accès Internet général.

---

## Appliquer la baseline Workgroup

```powershell
Set-OSConfigDesiredConfiguration `
  -Scenario SecurityBaseline/WindowsServer/2025/WorkgroupMember `
  -Default
```

Redémarrer :

```powershell
Restart-Computer
```

---

## Vérifier la baseline

```powershell
Get-OSConfigDesiredConfiguration `
  -Scenario SecurityBaseline/WindowsServer/2025/WorkgroupMember
```

Pour obtenir le détail :

```powershell
Get-OSConfigDesiredConfiguration `
  -Scenario SecurityBaseline/WindowsServer/2025/WorkgroupMember |
Format-Table Name,
@{Name="Status";Expression={$_.Compliance.Status}},
@{Name="Reason";Expression={$_.Compliance.Reason}} `
-AutoSize -Wrap
```

OSConfig permet également de contrôler la dérive de la configuration après application de la baseline.

[Microsoft Security Compliance Toolkit 1.0](https://www.microsoft.com/en-us/download/details.aspx?id=55319)


---

# 9. Durcir Microsoft Defender

Nous ne désactivons pas Microsoft Defender sur le serveur d'administration.

Appliquer le scénario Defender Windows Server 2025 :

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

Contrôler également l'état local :

```powershell
Get-MpComputerStatus
```

Les champs importants comprennent notamment :

```text
AntivirusEnabled
RealTimeProtectionEnabled
AntispywareEnabled
```

Ils doivent être cohérents avec la politique de sécurité définie.

---

# 10. Windows Firewall reste activé

Le firewall Windows ne doit pas être désactivé sous prétexte que pfSense filtre déjà le réseau.

Vérifier :

```powershell
Get-NetFirewallProfile |
Select-Object Name, Enabled, DefaultInboundAction, DefaultOutboundAction
```

Si nécessaire :

```powershell
Set-NetFirewallProfile `
  -Profile Domain,Private,Public `
  -Enabled True
```

Nous avons donc deux niveaux complémentaires :

```text
pfSense
     │
     │ segmentation réseau
     ▼
Windows Firewall
     │
     │ protection du serveur
     ▼
MLA-MGMT01
```

---

# 11. Credential Guard avant les premiers comptes de domaine

C'est un point important.

Credential Guard protège notamment les secrets d'authentification en utilisant Virtualization-Based Security (VBS).

Sur Windows Server 2025, Credential Guard peut être activé par défaut sur les serveurs membres du domaine répondant aux prérequis. Microsoft recommande toutefois de l'activer **avant la jonction au domaine ou avant la première connexion avec un utilisateur du domaine**, afin d'éviter que des secrets aient déjà été exposés sur une machine non protégée.

Dans MonLabAzure, nous allons donc le contrôler avant la première utilisation de comptes privilégiés.

---

## Vérifier l'état

```powershell
(Get-CimInstance `
  -ClassName Win32_DeviceGuard `
  -Namespace root\Microsoft\Windows\DeviceGuard).SecurityServicesRunning
```

Microsoft indique que la valeur `1` signale que Credential Guard est en cours d'exécution.

On peut également vérifier avec :

```text
msinfo32.exe
```

puis :

```text
Résumé système
→ Services de sécurité basés sur la virtualisation en cours d'exécution
```

---

## Si une activation explicite est nécessaire

Pour notre lab, nous privilégions une configuration sans verrouillage UEFI afin de conserver une capacité de rollback.

Exemple :

```powershell
New-Item `
  -Path "HKLM:\SYSTEM\CurrentControlSet\Control\DeviceGuard" `
  -Force | Out-Null

New-ItemProperty `
  -Path "HKLM:\SYSTEM\CurrentControlSet\Control\DeviceGuard" `
  -Name "EnableVirtualizationBasedSecurity" `
  -PropertyType DWord `
  -Value 1 `
  -Force

New-ItemProperty `
  -Path "HKLM:\SYSTEM\CurrentControlSet\Control\DeviceGuard" `
  -Name "RequirePlatformSecurityFeatures" `
  -PropertyType DWord `
  -Value 1 `
  -Force

New-Item `
  -Path "HKLM:\SYSTEM\CurrentControlSet\Control\Lsa" `
  -Force | Out-Null

New-ItemProperty `
  -Path "HKLM:\SYSTEM\CurrentControlSet\Control\Lsa" `
  -Name "LsaCfgFlags" `
  -PropertyType DWord `
  -Value 2 `
  -Force
```

Redémarrer :

```powershell
Restart-Computer
```

Cette configuration correspond aux paramètres documentés par Microsoft pour VBS et Credential Guard sans verrouillage UEFI.

---

# 12. BitLocker

Lorsque l'hyperviseur fournit un Virtual Trusted Platform Module (vTPM), le disque système de `MLA-MGMT01` doit être chiffré.

Le but n'est pas uniquement de protéger les fichiers.

Un hôte d'administration peut contenir :

* historiques PowerShell ;
* caches ;
* journaux ;
* certificats ;
* informations administratives ;
* traces d'identifiants.

Microsoft recommande BitLocker sur les Secure Administrative Hosts afin de protéger les données en cas d'accès au disque ou de vol du système.

L'activation dépendra du support vTPM/Secure Boot fourni par l'hyperviseur.

La clé de récupération devra être conservée **en dehors de la VM**.

Dans un vrai environnement, elle doit être stockée dans une solution de récupération administrée et protégée.

---

# 13. Ne pas installer les outils avant de savoir pourquoi

Un poste d'administration privilégié ne doit pas devenir une boîte à outils universelle.

Avant installation :

```powershell
Get-WindowsFeature -Name RSAT*
```

Nous allons commencer avec les outils nécessaires à Active Directory :

```powershell
Install-WindowsFeature `
  -Name RSAT-AD-Tools `
  -IncludeAllSubFeature
```

Les outils DNS seront également nécessaires :

```powershell
Install-WindowsFeature `
  -Name RSAT-DNS-Server
```

Pour la gestion des stratégies de groupe :

```powershell
Install-WindowsFeature `
  -Name GPMC
```

Vérifier ensuite :

```powershell
Get-WindowsFeature |
Where-Object InstallState -eq "Installed" |
Where-Object Name -Match "RSAT|GPMC"
```

Microsoft documente l'utilisation de RSAT sur Windows Server 2025, notamment pour AD DS et DNS.

---

# 14. Ce que nous n'installons pas

Sur `MLA-MGMT01`, nous n'installons pas :

```text
Chrome / Firefox
Microsoft Office
Teams
Outlook
Visual Studio
Terraform
Ansible
Azure CLI
outils personnels
applications de messagerie
```

Cela peut sembler restrictif, c'est volontaire.

>Plus une machine utilisée par un compte privilégié exécute de logiciels et accède à des sources externes, plus elle augmente les possibilités de compromission.

---

# 15. Bloquer l'accès Internet après le bootstrap

Une fois :

* Windows Server patché ;
* Defender à jour ;
* OSConfig installé ;
* la baseline appliquée ;
* RSAT installé ;

la règle Internet temporaire du réseau Management doit être retirée ou fortement limitée.

La situation cible devient :

```text
MLA-MGMT01
      │
      ├── DC01 / DC02
      ├── DNS
      ├── PKI
      ├── services d'administration autorisés
      │
      X
   Internet général
```

Les mises à jour devront continuer à être assurées, mais via une méthode contrôlée.

Dans le lab, nous pourrons temporairement réautoriser les endpoints nécessaires au patching.

En production, un mécanisme de mise à jour dédié aux systèmes sensibles serait préférable.

---

# 16. RDP : accès limité

Remote Desktop Protocol (RDP) peut être utilisé pour administrer `MLA-MGMT01`, mais il ne doit pas être ouvert à tout le réseau.

Dans le lab, deux approches sont possibles.

## Option la plus sûre

Utiliser la console de l'hyperviseur pour les premières étapes.

## RDP

Si RDP est utilisé :

* autoriser uniquement des sources d'administration explicitement définies ;
* bloquer les autres VLAN ;
* utiliser Network Level Authentication ;
* limiter les comptes autorisés ;
* ne pas exposer RDP sur Internet ;
* éviter les redirections inutiles.

>La baseline Windows Server 2025 applique d'ailleurs plusieurs protections RDP et peut désactiver certaines fonctions de redirection.

---

# 17. Les règles pfSense

Au départ, le réseau Management ne doit pas avoir un accès illimité à tout le SI.

Les premières règles seront volontairement limitées.

Pendant le bootstrap :

| Source       | Destination       | Usage            | État       |
| ------------ | ----------------- | ---------------- | ---------- |
| `MLA-MGMT01` | pfSense           | Administration   | Autorisé   |
| `MLA-MGMT01` | Internet contrôlé | Update/bootstrap | Temporaire |
| Autres VLAN  | `MLA-MGMT01`      | Administration   | Bloqué     |

Après déploiement de `DC01` :

| Source       | Destination   | Usage              |
| ------------ | ------------- | ------------------ |
| `MLA-MGMT01` | `MLA-DC01/02` | Administration AD  |
| `MLA-MGMT01` | DNS AD        | Résolution         |
| `MLA-MGMT01` | PKI           | Administration PKI |
| CLIENTS      | `MLA-MGMT01`  | Bloqué             |
| APP          | `MLA-MGMT01`  | Bloqué             |
| DB           | `MLA-MGMT01`  | Bloqué             |

Les ports précis seront définis avec chaque série afin d'éviter une règle :

```text
MGMT → ANY → ANY
```

qui serait simple, mais beaucoup trop permissive.

---

# 18. Deux phases de déploiement

![Parcours de sécurisation de MLA-MGMT01](/images/on-premises/administration/mla-mgmt-001-parcours-securisation.png)

*MLA-MGMT01 est d'abord préparé en workgroup, puis finalisé après la création d'Active Directory.*

Cette séquence est importante.

---

## Phase 1 - Avant Active Directory

Nous pouvons déjà :

* installer Windows Server 2025 ;
* configurer l'adresse IP ;
* patcher le serveur ;
* activer Defender ;
* appliquer la baseline WorkgroupMember ;
* vérifier Credential Guard ;
* configurer BitLocker ;
* installer RSAT ;
* sécuriser le firewall ;
* supprimer l'accès Internet général.

À ce stade :

```text
MLA-MGMT01
WORKGROUP
```

---

## Phase 2 - Après création de DC01

Une fois la forêt :

```text
ad.monlabazure.fr
```

créée, nous reviendrons sur `MLA-MGMT01`.

Nous remplacerons son DNS temporaire par :

```text
10.10.20.10
10.10.20.11
```

lorsque les deux DC seront disponibles.

Puis nous joindrons la machine au domaine.

---

# 19. Après la jonction au domaine

La baseline doit suivre le rôle réel du serveur.

`MLA-MGMT01` ne sera plus un Workgroup Member, mais un Member Server.

Nous retirerons donc la baseline précédente :

```powershell
Remove-OSConfigDesiredConfiguration `
  -Scenario SecurityBaseline/WindowsServer/2025/WorkgroupMember
```

Puis nous appliquerons :

```powershell
Set-OSConfigDesiredConfiguration `
  -Scenario SecurityBaseline/WindowsServer/2025/MemberServer `
  -Default
```

Redémarrer :

```powershell
Restart-Computer
```

Puis vérifier :

```powershell
Get-OSConfigDesiredConfiguration `
  -Scenario SecurityBaseline/WindowsServer/2025/MemberServer
```

Microsoft prévoit explicitement des scénarios différents pour les serveurs Workgroup, Member Server et Domain Controller.

---

# 20. Windows LAPS après la jonction au domaine

Une fois `MLA-MGMT01` membre du domaine, nous pourrons appliquer le scénario Windows Local Administrator Password Solution (Windows LAPS) prévu pour les Member Servers :

```powershell
Set-OSConfigDesiredConfiguration `
  -Scenario LAPS/WindowsServer/2025/MemberServer `
  -Default
```

Vérifier :

```powershell
Get-OSConfigDesiredConfiguration `
  -Scenario LAPS/WindowsServer/2025/MemberServer
```

La stratégie Active Directory qui définit le stockage, la rotation et les droits de lecture du mot de passe sera construite dans la série de sécurisation AD.

Microsoft indique que ce scénario OSConfig LAPS s'applique aux Member Servers.

---

# 21. Placer MLA-MGMT01 dans une OU dédiée

Après création de l'arborescence Active Directory, `MLA-MGMT01` sera déplacé dans une Organizational Unit (OU) réservée à l'administration privilégiée.

Architecture cible :

```text
ad.monlabazure.fr
│
└── Tier0
    │
    ├── Admin-Accounts
    │
    ├── Admin-Groups
    │
    └── Admin-Hosts
        │
        └── MLA-MGMT01
```

Cette OU permettra d'appliquer des stratégies spécifiques sans mélanger le poste d'administration avec les serveurs applicatifs.

---

# 22. Comptes autorisés

À terme, seuls les comptes explicitement prévus pour l'administration Tier 0 devront pouvoir ouvrir une session sur `MLA-MGMT01`.

Par exemple :

```text
adm-domain01
```

Un compte comme :

```text
user01
```

ne doit pas être utilisé sur cette machine.

L'inverse est également important :

```text
adm-domain01
```

ne doit pas ouvrir de session sur :

```text
MLA-W11-01
MLA-IIS01
MLA-SQL01
```

sauf justification exceptionnelle.

L'objectif est d'empêcher le dépôt inutile d'identifiants privilégiés sur des systèmes moins sûrs.

---

# 23. Pas de mot de passe sauvegardé dans les outils

Nous éviterons :

* `cmdkey` pour stocker des identifiants privilégiés ;
* les mots de passe dans les scripts ;
* les fichiers `.ps1` contenant des secrets ;
* les fichiers texte de mots de passe ;
* les tâches planifiées utilisant un Domain Admin permanent ;
* les connexions RDP enregistrant les identifiants.

Un hôte sécurisé ne compense pas une mauvaise gestion des secrets.

---

# 24. PowerShell

PowerShell sera l'un des principaux outils d'administration.

Vérifier la version :

```powershell
$PSVersionTable
```

Nous utiliserons ensuite les modules :

```text
ActiveDirectory
DnsServer
GroupPolicy
```

selon les rôles installés.

Exemple après création du domaine :

```powershell
Import-Module ActiveDirectory
```

Puis :

```powershell
Get-ADDomain
```

Ce test fera partie de la validation de `MLA-AD-001`.

---

# 25. Journalisation

`MLA-MGMT01` est un poste particulièrement intéressant pour l'audit.

Il concentre les opérations administratives.

Nous surveillerons progressivement :

* ouvertures de session ;
* échecs de connexion ;
* élévations de privilèges ;
* PowerShell ;
* utilisation de comptes administratifs ;
* modifications de configuration ;
* connexions RDP.

Ces événements seront ensuite envoyés vers :

```text
MLA-SIEM01
Wazuh
```

lorsque la plateforme de sécurité sera construite.

---

# 26. Snapshot et sauvegarde

Dans le lab, un snapshot peut être créé après :

```text
Windows Server installé
Patching terminé
Baseline appliquée
Credential Guard validé
RSAT installé
```

Nom possible :

```text
01-mgmt-secure-bootstrap
```

Attention toutefois : une fois que des comptes privilégiés ont été utilisés sur la machine, il faut éviter de conserver indéfiniment des snapshots contenant un ancien état du système et potentiellement des données sensibles.

Un snapshot reste un outil de lab.

Ce n'est pas une stratégie de sauvegarde d'entreprise.

---

# 27. Contrôles de validation

Avant d'utiliser `MLA-MGMT01` pour construire Active Directory :

* [ ] Windows Server 2025 est installé.
* [ ] Le serveur est nommé `MLA-MGMT01`.
* [ ] L'adresse IP est `10.10.10.10/24`.
* [ ] La passerelle est `10.10.10.1`.
* [ ] Windows est à jour.
* [ ] Microsoft Defender est actif.
* [ ] Windows Firewall est actif.
* [ ] La baseline `WorkgroupMember` est appliquée.
* [ ] Credential Guard est actif ou explicitement préparé.
* [ ] BitLocker est activé lorsque le vTPM le permet.
* [ ] Les outils RSAT nécessaires sont installés.
* [ ] Aucun logiciel bureautique n'est installé.
* [ ] Aucun navigateur n'est utilisé pour un usage général.
* [ ] Aucun secret n'est stocké dans les scripts.
* [ ] RDP n'est pas exposé largement.
* [ ] L'accès Internet général est retiré après le bootstrap.
* [ ] Une sauvegarde ou un snapshot propre du lab existe.

---

# Lab versus production

## Dans MonLabAzure

`MLA-MGMT01` est une machine virtuelle Windows Server 2025 dédiée à l'administration.

Elle permet de travailler sur les principes d'un Secure Administrative Host sans multiplier immédiatement les postes physiques.

## En production

Selon le niveau de maturité de l'entreprise, on pourrait aller plus loin :

* Privileged Access Workstations dédiés ;
* authentification multifacteur ;
* smart cards ou certificats ;
* application control ;
* poste physique dédié ;
* réseau d'administration isolé ;
* Remote Desktop Gateway ;
* bastion d'administration ;
* comptes à privilèges temporaires ;
* Privileged Identity Management pour les environnements Cloud ;
* mécanismes Just Enough Administration / Just In Time ;
* infrastructure de patching séparée.

La règle reste toutefois la même : **un compte très privilégié ne doit pas être utilisé depuis une machine moins sécurisée que la ressource administrée**.

---

# Références officielles

Les choix présentés dans cet article s'appuient principalement sur les documents Microsoft suivants :

* **Microsoft Learn - Implementing Secure Administrative Hosts** : principes de conception des postes et jump servers d'administration sécurisés.
* **Microsoft Learn - Best practices for securing Active Directory** : moindre privilège, administration sécurisée et protection d'Active Directory.
* **Microsoft Learn - Privileged access** : rôle des Privileged Access Workstations dans la protection du plan de contrôle.
* **Microsoft Learn - Configure Security Baselines for Windows Server 2025** : baselines Workgroup Member, Member Server, Defender, Secured-core et Windows LAPS.
* **Microsoft Learn - Install and Manage Remote Server Administration Tools** : installation et utilisation de RSAT sur Windows Server 2025.
* **Microsoft Learn - Configure Credential Guard** : configuration et validation de Credential Guard.

---

# Résultat obtenu

`MLA-MGMT01` n'est pas encore membre d'Active Directory, mais il est prêt pour la première opération sensible du lab.

Nous disposons désormais :

```text
pfSense
   │
   ▼
VLAN MGMT
10.10.10.0/24
   │
   ▼
MLA-MGMT01
10.10.10.10
   │
   ├── Windows Server 2025
   ├── Security Baseline
   ├── Defender
   ├── Credential Guard
   ├── Windows Firewall
   ├── BitLocker
   └── RSAT
```

Nous pouvons maintenant créer Active Directory sans utiliser un poste utilisateur standard ni administrer quotidiennement le domaine depuis le contrôleur de domaine lui-même.

La prochaine étape est :

**MLA-AD-001 - Déployer et sécuriser le premier contrôleur de domaine `MLA-DC01`.**
