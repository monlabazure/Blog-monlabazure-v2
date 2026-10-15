---
title: "MLA-MGMT-002 — Intégrer MLA-MGMT01 au domaine et sécuriser l'administration Tier 0"
date: 2026-09-24
draft: false
description: "Intégration de MLA-MGMT01 au domaine ad.monlabazure.fr et mise en place d'un hôte d'administration privilégié Tier 0 avec séparation des comptes, OSConfig, Credential Guard et contrôles d'accès."
categories:
  - "On-Premises"
tags:
  - "MonLabAzure"
  - "Active Directory"
  - "Tier 0"
  - "Privileged Access Workstation"
  - "Administration"
  - "Windows Server 2025"
  - "OSConfig"
  - "RSAT"
  - "Credential Guard"
  - "Sécurité"
series:
  - "Active Directory"
weight: 40
---

# MLA-MGMT-002 — Intégrer MLA-MGMT01 au domaine et sécuriser l'administration Tier 0

`MLA-MGMT01` a été préparé avant la création d'Active Directory.

À ce stade, il dispose déjà de :

- Windows Server 2025 ;
- son adresse `10.10.10.10/24` ;
- Microsoft Defender ;
- Windows Firewall ;
- Credential Guard ;
- Remote Server Administration Tools (RSAT) ;
- la Security Baseline `WorkgroupMember` ;
- une configuration dédiée à l'administration.

Dans l'article précédent, nous avons créé :

```text
MLA-DC01
10.10.20.10

ad.monlabazure.fr
````

Nous pouvons maintenant terminer la transformation de `MLA-MGMT01` en véritable hôte d'administration privilégié.

L'objectif n'est pas simplement de joindre un serveur au domaine.

Nous allons établir une frontière de confiance entre :

```text
usage quotidien
```

et :

```text
administration Active Directory
```

---

## Prérequis

Cet article s'appuie sur :

[**MLA-MGMT-001 — Déployer et sécuriser MLA-MGMT01**]({{< relref "on-premises/administration/mla-mgmt-001-administration-securisee.md" >}})

et :

[**MLA-AD-001 — Déployer et sécuriser le premier contrôleur de domaine MLA-DC01**]({{< relref "on-premises/active-directory/mla-ad-001-dc01-securise.md" >}})

---

# Architecture cible

![Administration Tier 0 depuis MLA-MGMT01](/images/on-premises/administration/mla-mgmt-002-tier0-administration.png)

`MLA-MGMT01` devient une machine faisant partie du périmètre de confiance Active Directory.

Dans le modèle de niveaux AD DS, les Domain Controllers, les comptes privilégiés qui les administrent et les machines depuis lesquelles ces comptes sont utilisés appartiennent au même niveau de confiance.

Microsoft classe notamment dans le Tier 0 :

```text
Domain Controllers
AD CS
services d'identité
comptes Tier 0
groupes Tier 0
postes d'administration Tier 0
```

Un jump server ou un poste utilisé pour administrer un Domain Controller devient lui-même une ressource Tier 0.

C'est exactement le rôle que nous allons donner à :

```text
MLA-MGMT01
```

---

# 1. Ce que signifie Tier 0 dans MonLabAzure

Le Tier 0 représente le plan de contrôle de l'identité.

Une compromission de ce niveau peut permettre de prendre le contrôle de l'ensemble du domaine.

Dans MonLabAzure, nous allons progressivement y placer :

```text
MLA-DC01
MLA-DC02
MLA-MGMT01
AD CS
comptes privilégiés AD
groupes privilégiés AD
```

En revanche :

```text
MLA-IIS01
MLA-IIS02
MLA-SQL01
MLA-SQL02
MLA-W11-01
```

ne sont pas Tier 0.

Cette distinction est importante.

Il serait contre-productif de déclarer toute l'infrastructure Tier 0 : cela agrandirait inutilement la surface d'attaque du niveau le plus sensible.

---

# 2. Le principe fondamental : les credentials définissent le niveau

La segmentation réseau est importante, mais elle ne suffit pas.

Un compte Tier 0 utilisé sur un serveur applicatif transforme ce serveur en point potentiel de compromission du Tier 0.

Exemple à éviter :

```text
adm-domain01
      │
      ▼
MLA-IIS01
      │
      ▼
credentials Tier 0 exposés sur Tier 1
```

Même problème avec un poste utilisateur :

```text
adm-domain01
      │
      ▼
MLA-W11-01
      │
      ▼
poste avec navigateur / mail / usage quotidien
```

La règle MonLabAzure sera donc :

> Un compte Tier 0 n'est utilisé que depuis une machine Tier 0 vers une ressource Tier 0.

Microsoft insiste également sur le fait que le niveau de confiance commence au poste depuis lequel l'administrateur saisit ses identifiants.

---

# 3. Séquence retenue

![Séquence d'intégration de MLA-MGMT01](/images/on-premises/administration/mla-mgmt-002-sequence-integration-domaine.png)

Nous allons suivre cet ordre :

```text
DNS Active Directory
        ↓
Jonction au domaine
        ↓
Baseline Member Server
        ↓
OU Tier 0
        ↓
Compte privilégié dédié
        ↓
Restrictions de connexion
        ↓
Validation RSAT / Kerberos / DNS
```

Nous évitons volontairement de créer le compte d'administration avant que l'hôte d'administration soit prêt à le recevoir.

---

# 4. Vérifier MLA-DC01

Avant de joindre `MLA-MGMT01`, vérifier que le Domain Controller est sain.

Depuis `MLA-DC01` :

```powershell
dcdiag
```

Puis :

```powershell
dcdiag /test:dns
```

Vérifier également :

```powershell
Get-ADDomain
```

et :

```powershell
Get-ADForest
```

Le domaine attendu est :

```text
ad.monlabazure.fr
```

---

# 5. Configurer le DNS de MLA-MGMT01

C'est probablement le point le plus important avant la jonction au domaine.

`MLA-MGMT01` ne doit plus utiliser le DNS temporaire utilisé pendant le bootstrap.

Configurer :

```text
DNS préféré : 10.10.20.10
```

Identifier l'interface :

```powershell
Get-NetAdapter
```

Puis :

```powershell
Set-DnsClientServerAddress `
  -InterfaceAlias "Ethernet" `
  -ServerAddresses "10.10.20.10"
```

Vérifier :

```powershell
Get-DnsClientServerAddress `
  -AddressFamily IPv4
```

---

# 6. Tester la résolution Active Directory

Tester le Domain Controller :

```powershell
Resolve-DnsName MLA-DC01.ad.monlabazure.fr
```

Tester le domaine :

```powershell
Resolve-DnsName ad.monlabazure.fr
```

Mais surtout, vérifier les enregistrements SRV utilisés pour localiser les Domain Controllers :

```powershell
Resolve-DnsName `
  _ldap._tcp.dc._msdcs.ad.monlabazure.fr `
  -Type SRV
```

Le résultat doit référencer :

```text
MLA-DC01.ad.monlabazure.fr
```

Tester également Kerberos :

```powershell
Resolve-DnsName `
  _kerberos._tcp.ad.monlabazure.fr `
  -Type SRV
```

Si ces tests échouent, je préfère corriger DNS avant la jonction plutôt que poursuivre et diagnostiquer ensuite un domaine partiellement fonctionnel.

---

# 7. Vérifier la connectivité pfSense

`MLA-MGMT01` se trouve dans :

```text
10.10.10.0/24
```

et `MLA-DC01` dans :

```text
10.10.20.0/24
```

Le trafic traverse donc pfSense.

Tester :

```powershell
Test-NetConnection 10.10.20.10 -Port 53
```

Puis :

```powershell
Test-NetConnection 10.10.20.10 -Port 88
```

et :

```powershell
Test-NetConnection 10.10.20.10 -Port 389
```

Ces premiers tests permettent déjà de distinguer :

```text
problème réseau
```

de :

```text
problème Active Directory
```

---

# 8. Les flux nécessaires

L'administration Active Directory ne se limite pas au port 389.

Selon les outils utilisés, plusieurs protocoles entrent en jeu.

Parmi les principaux :

| Service             | Port            |
| ------------------- | --------------- |
| DNS                 | TCP/UDP 53      |
| Kerberos            | TCP/UDP 88      |
| NTP                 | UDP 123         |
| RPC Endpoint Mapper | TCP 135         |
| LDAP                | TCP/UDP 389     |
| SMB                 | TCP 445         |
| Kerberos Password   | TCP/UDP 464     |
| LDAPS               | TCP 636         |
| Global Catalog      | TCP 3268        |
| Global Catalog TLS  | TCP 3269        |
| RPC dynamique       | TCP 49152-65535 |

Dans pfSense, nous éviterons une règle générale :

```text
MGMT → IDENTITY → ANY
```

Nous créerons plutôt des alias dédiés.

Par exemple :

```text
SRV_DOMAIN_CONTROLLERS
```

avec :

```text
10.10.20.10
10.10.20.11
```

puis un alias de ports adapté aux besoins d'administration AD.

Lorsque `MLA-DC02` sera ajouté, il suffira de compléter l'alias.

---

# 9. Vérifier Credential Guard avant le domaine

Credential Guard doit déjà avoir été contrôlé dans `MLA-MGMT-001`.

Windows Server 2025 peut l'activer automatiquement lorsque les prérequis sont réunis.

Vérifier :

```powershell
(Get-CimInstance `
  -ClassName Win32_DeviceGuard `
  -Namespace root\Microsoft\Windows\DeviceGuard).SecurityServicesRunning
```

Il est préférable de vérifier Credential Guard avant la première utilisation d'identifiants du domaine sur la machine.

Une fois des credentials privilégiés utilisés sur un système, il est trop tard pour considérer qu'ils n'y ont jamais été exposés.

---

# 10. Joindre MLA-MGMT01 au domaine

Nous pouvons maintenant effectuer la jonction.

Depuis PowerShell en administrateur :

```powershell
Add-Computer `
  -DomainName "ad.monlabazure.fr" `
  -Credential (Get-Credential)
```

Utiliser temporairement un compte autorisé à joindre la machine au domaine.

Dans notre lab de bootstrap, cela peut être :

```text
MONLABAZURE\Administrator
```

puis redémarrer :

```powershell
Restart-Computer
```

En production, je préfère déléguer explicitement le droit de joindre les ordinateurs au domaine plutôt que d'utiliser un compte Domain Admin pour les opérations courantes.

---

# 11. Vérifier la jonction

Après redémarrage :

```powershell
Get-CimInstance Win32_ComputerSystem |
Select-Object Name, Domain, PartOfDomain
```

Résultat attendu :

```text
Name         MLA-MGMT01
Domain       ad.monlabazure.fr
PartOfDomain True
```

Vérifier également le canal sécurisé :

```powershell
Test-ComputerSecureChannel -Verbose
```

Résultat attendu :

```text
True
```

---

# 12. Vérifier le Domain Controller utilisé

Exécuter :

```cmd
nltest /dsgetdc:ad.monlabazure.fr
```

Puis :

```cmd
nltest /sc_verify:ad.monlabazure.fr
```

Nous devons retrouver :

```text
MLA-DC01
```

comme Domain Controller disponible.

---

# 13. Vérifier Kerberos

Ouvrir une session avec un compte de domaine autorisé puis :

```cmd
klist
```

Nous devons notamment retrouver un Ticket Granting Ticket (TGT) Kerberos.

La présence de Kerberos permet de vérifier que nous ne sommes pas simplement dans une situation où DNS fonctionne mais où l'authentification est dégradée.

---

# 14. Le rôle de MLA-MGMT01 a changé

Avant la jonction :

```text
Workgroup Member
```

Après la jonction :

```text
Member Server
```

La Security Baseline OSConfig doit donc correspondre au nouveau rôle.

Nous avions appliqué :

```text
SecurityBaseline/WindowsServer/2025/WorkgroupMember
```

Cette baseline n'est plus adaptée.

---

# 15. Retirer la baseline WorkgroupMember

Exécuter :

```powershell
Remove-OSConfigDesiredConfiguration `
  -Scenario SecurityBaseline/WindowsServer/2025/WorkgroupMember
```

Redémarrer si demandé :

```powershell
Restart-Computer
```

---

# 16. Appliquer la baseline MemberServer

Après redémarrage :

```powershell
Set-OSConfigDesiredConfiguration `
  -Scenario SecurityBaseline/WindowsServer/2025/MemberServer `
  -Default
```

Puis :

```powershell
Restart-Computer
```

Microsoft fournit explicitement des scénarios distincts pour :

```text
WorkgroupMember
MemberServer
DomainController
```

Il est donc préférable d'appliquer la baseline correspondant au rôle réel du serveur plutôt que de conserver une configuration devenue incohérente.

---

# 17. Vérifier la conformité OSConfig

Exécuter :

```powershell
Get-OSConfigDesiredConfiguration `
  -Scenario SecurityBaseline/WindowsServer/2025/MemberServer
```

Pour obtenir le détail :

```powershell
Get-OSConfigDesiredConfiguration `
  -Scenario SecurityBaseline/WindowsServer/2025/MemberServer |
Format-Table `
  Name,
  @{Name="Status";Expression={$_.Compliance.Status}},
  @{Name="Reason";Expression={$_.Compliance.Reason}} `
  -AutoSize -Wrap
```

Les éventuelles non-conformités doivent être comprises avant de poursuivre.

---

# 18. Vérifier Defender

La baseline spécifique Microsoft Defender peut rester appliquée :

```powershell
Get-OSConfigDesiredConfiguration `
  -Scenario Defender/Antivirus/WindowsServer/2025
```

Vérifier également :

```powershell
Get-MpComputerStatus
```

Nous recherchons notamment :

```text
AntivirusEnabled
RealTimeProtectionEnabled
```

---

# 19. Créer une structure Tier 0 minimale

La structure complète des Organizational Units (OU), groupes et délégations sera détaillée dans un article dédié.

Nous avons néanmoins besoin d'un minimum de structure pour isoler `MLA-MGMT01`.

Depuis une console PowerShell disposant du module Active Directory :

```powershell
Import-Module ActiveDirectory
```

Créer l'OU Tier 0 :

```powershell
New-ADOrganizationalUnit `
  -Name "Tier0" `
  -Path "DC=ad,DC=monlabazure,DC=fr" `
  -ProtectedFromAccidentalDeletion $true
```

Puis :

```powershell
New-ADOrganizationalUnit `
  -Name "Admin-Hosts" `
  -Path "OU=Tier0,DC=ad,DC=monlabazure,DC=fr" `
  -ProtectedFromAccidentalDeletion $true
```

Créer également :

```powershell
New-ADOrganizationalUnit `
  -Name "Admin-Accounts" `
  -Path "OU=Tier0,DC=ad,DC=monlabazure,DC=fr" `
  -ProtectedFromAccidentalDeletion $true
```

et :

```powershell
New-ADOrganizationalUnit `
  -Name "Admin-Groups" `
  -Path "OU=Tier0,DC=ad,DC=monlabazure,DC=fr" `
  -ProtectedFromAccidentalDeletion $true
```

Nous obtenons :

```text
ad.monlabazure.fr
│
└── Tier0
    ├── Admin-Hosts
    ├── Admin-Accounts
    └── Admin-Groups
```

Cette structure reste volontairement minimale.

---

# 20. Déplacer MLA-MGMT01

Rechercher l'objet :

```powershell
Get-ADComputer MLA-MGMT01
```

Puis :

```powershell
Get-ADComputer MLA-MGMT01 |
Move-ADObject `
  -TargetPath "OU=Admin-Hosts,OU=Tier0,DC=ad,DC=monlabazure,DC=fr"
```

Vérifier :

```powershell
Get-ADComputer MLA-MGMT01 |
Select-Object Name, DistinguishedName
```

Le Distinguished Name doit désormais contenir :

```text
OU=Admin-Hosts,OU=Tier0
```

---

# 21. Créer le premier compte d'administration dédié

Nous n'allons pas utiliser :

```text
Administrator
```

comme compte quotidien.

Créer par exemple :

```text
adm-domain01
```

Demander un mot de passe :

```powershell
$Password = Read-Host `
  "Mot de passe adm-domain01" `
  -AsSecureString
```

Créer le compte :

```powershell
New-ADUser `
  -Name "adm-domain01" `
  -SamAccountName "adm-domain01" `
  -UserPrincipalName "adm-domain01@ad.monlabazure.fr" `
  -Path "OU=Admin-Accounts,OU=Tier0,DC=ad,DC=monlabazure,DC=fr" `
  -AccountPassword $Password `
  -Enabled $true `
  -ChangePasswordAtLogon $true
```

À ce stade, le compte existe mais ne possède encore aucun privilège particulier.

C'est volontaire.

Créer un compte dans l'OU Tier 0 ne lui donne pas automatiquement des droits administratifs.

---

# 22. Attribuer les privilèges nécessaires

Pour le bootstrap du lab, ce compte devra pouvoir effectuer les opérations d'administration Active Directory prévues dans les prochains articles.

La solution la plus simple serait :

```powershell
Add-ADGroupMember `
  -Identity "Domain Admins" `
  -Members "adm-domain01"
```

Cela donne cependant des privilèges très élevés.

Dans MonLabAzure, nous utiliserons cette appartenance uniquement pour les opérations nécessitant réellement Domain Admin pendant la construction initiale.

L'objectif futur est de déléguer les tâches qui peuvent l'être plutôt que de transformer chaque compte d'administration en Domain Admin permanent.

Même dans le Tier 0 :

> moindre privilège reste la règle.

---

# 23. Pourquoi séparer user01 et adm-domain01 ?

Nous aurons donc :

```text
user01
```

pour les activités normales,

et :

```text
adm-domain01
```

pour l'administration Active Directory.

Le compte :

```text
adm-domain01
```

ne doit pas être utilisé pour :

* naviguer sur Internet ;
* consulter une messagerie ;
* ouvrir une session sur `MLA-W11-01` ;
* administrer IIS ;
* administrer SQL ;
* utiliser Git ;
* exécuter Terraform ;
* travailler quotidiennement.

Inversement :

```text
user01
```

ne doit pas administrer :

```text
MLA-DC01
MLA-DC02
AD CS
```

---

# 24. Une erreur fréquente : utiliser Domain Admin partout

L'appartenance à Domain Admins ne doit pas devenir une solution de facilité.

Exemple :

```text
problème d'accès
      ↓
ajouter Domain Admin
      ↓
ça fonctionne
```

Ce fonctionnement masque souvent :

* une délégation mal conçue ;
* un groupe absent ;
* une ACL incorrecte ;
* un problème de droits locaux ;
* un problème de configuration.

Dans les prochains articles, nous chercherons progressivement à remplacer les privilèges globaux par des délégations adaptées.

---

# 25. Limiter les connexions Tier 0

Le compte Tier 0 ne doit pas se connecter sur des machines de niveau inférieur.

À terme :

```text
adm-domain01
      │
      ├── MLA-MGMT01       AUTORISÉ
      ├── MLA-DC01         uniquement si nécessaire
      ├── MLA-DC02         uniquement si nécessaire
      │
      ├── MLA-W11-01       INTERDIT
      ├── MLA-IIS01        INTERDIT
      └── MLA-SQL01        INTERDIT
```

Microsoft recommande explicitement de ne jamais saisir des credentials de niveau supérieur sur un système de confiance inférieure.

Les restrictions seront progressivement imposées par Group Policy Objects (GPO) et par la structure Tier 0.

---

# 26. Restreindre les comptes autorisés sur MLA-MGMT01

`MLA-MGMT01` ne doit pas accepter les connexions de n'importe quel utilisateur du domaine.

Nous allons créer un groupe dédié :

```powershell
New-ADGroup `
  -Name "GG_T0_MGMT01_Logon" `
  -SamAccountName "GG_T0_MGMT01_Logon" `
  -GroupScope Global `
  -GroupCategory Security `
  -Path "OU=Admin-Groups,OU=Tier0,DC=ad,DC=monlabazure,DC=fr"
```

Ajouter :

```powershell
Add-ADGroupMember `
  -Identity "GG_T0_MGMT01_Logon" `
  -Members "adm-domain01"
```

Ce groupe sera ensuite utilisé dans une GPO pour contrôler les droits :

```text
Allow log on locally
Allow log on through Remote Desktop Services
```

La GPO sera appliquée à :

```text
OU=Admin-Hosts
```

Nous ne modifions pas manuellement ces droits sur chaque machine : nous voulons une configuration reproductible et auditable.

---

# 27. Créer la GPO Tier 0

Créer une GPO :

```text
T0 - Admin Hosts - Security
```

Elle sera liée à :

```text
OU=Admin-Hosts
```

Cette GPO pourra progressivement gérer :

```text
droits de logon
restrictions RDP
audit
firewall
durcissement supplémentaire
restrictions d'applications
```

La configuration détaillée de cette GPO sera traitée dans l'article consacré au durcissement Active Directory.

Pour le moment, nous établissons surtout la structure.

---

# 28. Ne pas utiliser MLA-MGMT01 comme serveur généraliste

Maintenant que des credentials Tier 0 vont être utilisés sur `MLA-MGMT01`, son statut change définitivement.

Nous ne devons plus y installer :

```text
Terraform
Ansible
Azure CLI
Git
outils de développement
applications métiers
messagerie
Office
navigateur à usage général
```

Ces outils seront placés sur une machine distincte.

Par exemple :

```text
MLA-MGMT02
```

pour :

```text
Azure
Terraform
Ansible
Git
automatisation
administration Tier 1
```

Cette séparation évite que des dépendances externes ou des outils de développement augmentent la surface d'attaque du Tier 0.

---

# 29. Bloquer Internet

Une fois l'intégration terminée, `MLA-MGMT01` ne doit plus disposer d'un accès Internet général.

Cible :

```text
MLA-MGMT01
      │
      ├── DC01 / DC02
      ├── DNS
      ├── PKI
      ├── mécanismes de mise à jour approuvés
      │
      X
  navigation Internet
```

Microsoft recommande que les hôtes d'administration sécurisés ne soient pas utilisés pour la navigation Internet, la messagerie ou les applications bureautiques.

---

# 30. Windows LAPS : préparer, mais ne pas improviser

`MLA-MGMT01` est désormais un serveur membre.

OSConfig fournit bien le scénario :

```text
LAPS/WindowsServer/2025/MemberServer
```

Mais l'utilisation de Windows Local Administrator Password Solution (Windows LAPS) avec Active Directory nécessite également une préparation côté domaine.

Elle comprend notamment :

```text
mise à jour du schéma AD
permissions sur les OU
droits de lecture du mot de passe
stratégie LAPS
chiffrement éventuel du mot de passe
```

Nous ne nous contenterons donc pas d'appliquer le scénario OSConfig et de considérer LAPS comme terminé.

La configuration complète sera réalisée dans la série de sécurisation Active Directory.

Microsoft demande notamment d'exécuter une fois par forêt :

```powershell
Update-LapsADSchema
```

avant d'utiliser le stockage Windows LAPS dans Active Directory.

---

# 31. Vérifier RSAT

Vérifier les outils installés :

```powershell
Get-WindowsFeature |
Where-Object InstallState -eq "Installed" |
Where-Object Name -Match "RSAT|GPMC"
```

Nous voulons au minimum disposer des composants permettant d'administrer :

```text
Active Directory
DNS
Group Policy
```

---

# 32. Tester Active Directory depuis MLA-MGMT01

Importer :

```powershell
Import-Module ActiveDirectory
```

Tester :

```powershell
Get-ADDomain
```

Puis :

```powershell
Get-ADForest
```

Tester le Domain Controller :

```powershell
Get-ADDomainController
```

Résultat attendu :

```text
MLA-DC01
```

---

# 33. Tester DNS depuis la console distante

Si les outils DNS sont installés :

```powershell
Get-DnsServerZone `
  -ComputerName MLA-DC01
```

Nous devons retrouver notamment :

```text
ad.monlabazure.fr
_msdcs.ad.monlabazure.fr
```

---

# 34. Tester Group Policy Management

Lancer :

```text
gpmc.msc
```

Nous devons pouvoir administrer :

```text
Forest: ad.monlabazure.fr
```

depuis `MLA-MGMT01`.

À partir de maintenant, la console locale du Domain Controller n'est plus notre poste d'administration principal.

---

# 35. Tester Active Directory Administrative Center

Lancer :

```text
dsac.exe
```

Vérifier l'accès au domaine :

```text
ad.monlabazure.fr
```

Nous pourrons utiliser cet outil en complément de PowerShell et de Active Directory Users and Computers.

---

# 36. Tester Active Directory Users and Computers

Lancer :

```text
dsa.msc
```

Vérifier :

```text
Tier0
├── Admin-Accounts
├── Admin-Groups
└── Admin-Hosts
```

et vérifier que :

```text
MLA-MGMT01
```

apparaît dans :

```text
Admin-Hosts
```

---

# 37. Vérifier les tickets Kerberos

Depuis la session `adm-domain01` :

```cmd
klist
```

Nous devons retrouver un TGT pour :

```text
ad.monlabazure.fr
```

Après l'utilisation de services AD, d'autres tickets apparaîtront.

Cette commande sera utile plus tard pour diagnostiquer les problèmes d'authentification Kerberos.

---

# 38. Vérifier la stratégie appliquée

Utiliser :

```powershell
gpupdate /force
```

Puis :

```powershell
gpresult /r
```

ou :

```powershell
gpresult /h C:\Temp\gpresult.html
```

Le rapport HTML sera particulièrement pratique lorsque plusieurs GPO Tier 0 seront présentes.

---

# 39. Journalisation

Un poste d'administration Tier 0 doit être surveillé avec le même sérieux que les Domain Controllers.

Nous voulons pouvoir retrouver :

```text
qui s'est connecté
quand
depuis où
avec quel compte
quelles commandes ont été utilisées
quelles modifications ont été apportées
```

Les événements seront plus tard centralisés vers :

```text
MLA-SIEM01
```

avec Wazuh.

La centralisation des logs ne devra cependant pas créer un chemin de compromission inverse depuis un système Tier 1 vers le Tier 0.

Ce point devra être pris en compte dans l'architecture de supervision.

---

# 40. Snapshot après intégration

Dans le lab, nous pouvons créer un nouveau point de restauration :

```text
02-mgmt-domain-tier0-ready
```

après validation de :

```text
Join domaine OK
DNS AD OK
Kerberos OK
Baseline MemberServer OK
RSAT OK
Credential Guard OK
Tier0 OU OK
```

Comme toujours, ce snapshot facilite le lab mais ne remplace pas une sauvegarde adaptée à un environnement de production.

---

# 41. Checklist finale

## Domaine

* [ ] `MLA-MGMT01` utilise `10.10.20.10` comme DNS.
* [ ] Les enregistrements SRV sont résolus.
* [ ] La machine appartient à `ad.monlabazure.fr`.
* [ ] `Test-ComputerSecureChannel` retourne `True`.
* [ ] `nltest /dsgetdc` localise correctement `MLA-DC01`.
* [ ] Kerberos fonctionne.

## Sécurité

* [ ] Credential Guard est actif.
* [ ] Windows Defender est actif.
* [ ] Windows Firewall est actif.
* [ ] La baseline `WorkgroupMember` a été retirée.
* [ ] La baseline `MemberServer` est appliquée.
* [ ] Internet général n'est plus utilisé.
* [ ] Aucun logiciel bureautique n'est installé.
* [ ] Aucun outil DevOps n'est installé sur `MLA-MGMT01`.

## Tier 0

* [ ] `OU=Tier0` existe.
* [ ] `Admin-Hosts` existe.
* [ ] `Admin-Accounts` existe.
* [ ] `Admin-Groups` existe.
* [ ] `MLA-MGMT01` se trouve dans `Admin-Hosts`.
* [ ] `adm-domain01` existe.
* [ ] `adm-domain01` n'est pas utilisé pour les tâches quotidiennes.
* [ ] Les connexions Tier 0 ne sont effectuées que depuis un hôte Tier 0.

## Administration

* [ ] `Get-ADDomain` fonctionne depuis `MLA-MGMT01`.
* [ ] `Get-ADForest` fonctionne.
* [ ] DNS Manager fonctionne.
* [ ] GPMC fonctionne.
* [ ] AD Administrative Center fonctionne.
* [ ] Active Directory Users and Computers fonctionne.

---

# Lab versus production

Dans MonLabAzure, `MLA-MGMT01` est une machine virtuelle Windows Server 2025 utilisée comme hôte d'administration Tier 0.

Cette architecture permet de travailler correctement les principes de séparation des privilèges.

Dans une grande entreprise, l'architecture pourrait aller plus loin :

```text
PAW physiques dédiés
MFA
smart cards
JIT
vault de credentials
application allowlisting
administration par rôle
plusieurs PAW par niveau
bastions Tier 0 dédiés
supervision spécifique
infrastructure de patching isolée
```

Mais le principe reste identique :

> Le chemin utilisé pour administrer une ressource sensible doit être au moins aussi fiable que la ressource administrée.

---

# Références officielles

Les principes utilisés dans cet article s'appuient notamment sur les documentations Microsoft suivantes :

* Microsoft Learn — **Tier model for Active Directory Domain Services**
* Microsoft Learn — **Implementing Secure Administrative Hosts**
* Microsoft Learn — **Join a computer to a domain**
* Microsoft Learn — **Configure Security Baselines for Windows Server 2025**
* Microsoft Learn — **Configure Credential Guard**
* Microsoft Learn — **Get started with Windows LAPS and Windows Server Active Directory**

---

# Résultat obtenu

L'administration Active Directory n'est maintenant plus réalisée depuis :

```text
MLA-DC01
```

ni depuis un poste utilisateur.

Nous disposons de :

```text
                   TIER 0
                     │
        ┌────────────┴────────────┐
        │                         │
  adm-domain01               MLA-MGMT01
                                  │
                            RSAT / PowerShell
                                  │
                                  ▼
                             MLA-DC01
                                  │
                             AD DS / DNS
```

La séparation commence donc réellement à prendre forme :

```text
user01
  │
  ▼
usage quotidien

adm-domain01
  │
  ▼
MLA-MGMT01
  │
  ▼
Tier 0
```

Le premier Domain Controller et son poste d'administration sont maintenant opérationnels.

La prochaine étape est d'éliminer le principal point de panne actuel de l'infrastructure d'identité.

## Prochaine étape

**MLA-AD-002 — Déployer MLA-DC02 et valider la réplication Active Directory et DNS.**

```

Les points sensibles de cette version ont été vérifiés dans la documentation Microsoft actuelle : Microsoft classe bien les DC, AD CS, comptes privilégiés et machines qui administrent ces ressources dans le périmètre Tier 0, et précise qu'un PAW Tier 0 doit être utilisé pour accéder aux actifs Tier 0.

La séparation de `MLA-MGMT01` des usages Web, mail et bureautiques correspond aussi aux recommandations Microsoft pour les Secure Administrative Hosts. OSConfig fournit bien une baseline `MemberServer` distincte de `WorkgroupMember`, ainsi qu'un scénario LAPS réservé aux serveurs membres.

[Microsoft Learn — AD DS Tier Model](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/tier-model?utm_source=chatgpt.com)  
[Microsoft Learn — Secure Administrative Hosts](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/plan/security-best-practices/implementing-secure-administrative-hosts?utm_source=chatgpt.com)  
[Microsoft Learn — Join a computer to a domain](https://learn.microsoft.com/fr-fr/windows-server/identity/ad-ds/manage/join-computer-to-domain?utm_source=chatgpt.com)  
[Microsoft Learn — Windows Server 2025 Security Baselines](https://learn.microsoft.com/fr-fr/windows-server/security/osconfig/osconfig-how-to-configure-security-baselines?utm_source=chatgpt.com)  
[Microsoft Learn — Windows LAPS with Active Directory](https://learn.microsoft.com/fr-fr/windows-server/identity/laps/laps-scenarios-windows-server-active-directory?utm_source=chatgpt.com)