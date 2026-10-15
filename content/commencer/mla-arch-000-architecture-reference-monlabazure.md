---
title: "MLA-ARCH-000 - Architecture de référence MonLabAzure"
date: 2026-09-21
draft: false
description: "Architecture de référence du lab MonLabAzure : réseau segmenté, Active Directory, PKI, SQL Server Always On, IIS, Linux, supervision, sécurité et préparation de l'environnement hybride Azure."
categories:
  - "Commencer"
tags:
  - "MonLabAzure"
  - "Architecture"
  - "On-Premises"
  - "Active Directory"
  - "OPNsense"
  - "PKI"
  - "SQL Server"
  - "IIS"
  - "Azure"
  - "Architecture Hybride"
series:
  - "Fondations MonLabAzure"
weight: 20
---

# MLA-ARCH-000 — Architecture de référence MonLabAzure

Avant de déployer le premier contrôleur de domaine ou la première machine Azure, il faut savoir où l'on va.

MonLabAzure va couvrir progressivement le réseau, Active Directory, la Public Key Infrastructure (PKI), SQL Server, IIS, Linux, Microsoft 365, Microsoft Entra ID, Azure, Terraform, Ansible et les pipelines CI/CD.

Si chaque sujet est construit indépendamment, le lab devient rapidement une collection de machines sans véritable cohérence.

L'approche retenue est différente : **construire un seul environnement d'entreprise et le faire évoluer tout au long du parcours**.

Ce document définit cette architecture de référence.

---

## Architecture générale du lab

![Architecture de référence MonLabAzure](/images/commencer/architecture-reference-monlabazure.png)

*Architecture de référence utilisée comme socle pour les différentes séries MonLabAzure.*

Le lab part d'une infrastructure **on-premises** segmentée.

OPNsense assure le routage et le filtrage entre les différents réseaux. Les services sont répartis selon leur fonction : administration, identité, applications, bases de données, postes clients, supervision et PKI.

Un réseau de transit est réservé dès le départ pour préparer la future connexion avec Microsoft Azure.

L'objectif n'est pas de multiplier les VLAN pour rendre le lab artificiellement complexe. La segmentation doit nous permettre de travailler sur de vrais sujets : dépendances applicatives, filtrage, administration, résolution DNS, certificats et troubleshooting.

---

## Les principes retenus

Quelques règles structurent toute l'architecture :

- segmenter les principaux rôles ;
- appliquer un filtrage **deny-by-default** entre les réseaux ;
- séparer les comptes utilisateurs et les comptes d'administration ;
- éviter d'utiliser Domain Admin pour les opérations courantes ;
- utiliser Transport Layer Security (TLS) lorsque le service le permet ;
- disposer d'une PKI interne ;
- superviser les services importants ;
- centraliser les événements utiles à la sécurité ;
- sauvegarder les composants critiques ;
- tester réellement les restaurations ;
- documenter les flux et les décisions ;
- automatiser une infrastructure déjà comprise et validée.

Ces principes resteront valables lorsque le lab évoluera vers Azure.

---

# 1. Domaines et identité

Le domaine Internet du projet est :

```text
monlabazure.fr
````

Le domaine Active Directory Domain Services (AD DS) utilisé dans le lab sera :

```text
ad.monlabazure.fr
```

Le nom NetBIOS sera :

```text
MONLABAZURE
```

Pour les utilisateurs, nous utiliserons à terme un User Principal Name (UPN) basé sur le domaine public.

Exemple :

```text
Compte Active Directory
user01

UPN
user01@monlabazure.fr

Domaine AD DS
ad.monlabazure.fr
```

Ce choix prépare proprement la future intégration avec Microsoft Entra ID sans imposer aux utilisateurs un UPN en `@ad.monlabazure.fr`.

---

# 2. Plan d'adressage

Le bloc suivant est réservé au lab on-premises :

```text
10.10.0.0/16
```

Il est découpé en plusieurs réseaux.

| VLAN | Réseau          | Usage principal            |
| ---: | --------------- | -------------------------- |
|   10 | `10.10.10.0/24` | Administration             |
|   20 | `10.10.20.0/24` | Identité et infrastructure |
|   30 | `10.10.30.0/24` | Applications               |
|   40 | `10.10.40.0/24` | Bases de données           |
|   50 | `10.10.50.0/24` | Postes utilisateurs        |
|   60 | `10.10.60.0/24` | DMZ                        |
|   70 | `10.10.70.0/24` | Supervision et sécurité    |
|   80 | `10.10.80.0/24` | PKI                        |
|   90 | `10.10.90.0/24` | Transit et Azure           |

Tous ces segments ne seront pas exploités immédiatement.

Le but est surtout d'éviter de devoir revoir tout le plan réseau lorsqu'un nouveau besoin apparaît.

---

## Convention d'adressage

Chaque réseau suit la même logique :

| Plage          | Utilisation                    |
| -------------- | ------------------------------ |
| `.1`           | Passerelle                     |
| `.2` à `.9`    | Infrastructure réservée        |
| `.10` à `.49`  | Serveurs avec adresse statique |
| `.50` à `.99`  | Extensions futures             |
| `.100` et plus | DHCP lorsque nécessaire        |

Exemple pour le réseau Identity :

```text
Réseau          10.10.20.0/24
Gateway         10.10.20.1
MLA-DC01        10.10.20.10
MLA-DC02        10.10.20.11
```

Cette convention simple facilite les diagnostics et garde un adressage lisible.

---

# 3. Firewall et segmentation

Le firewall du lab sera :

```text
MLA-FW01
```

La solution retenue est :

```text
OPNsense
```

OPNsense assurera :

* le routage inter-réseaux ;
* le filtrage ;
* le Network Address Translation (NAT) ;
* l'accès Internet ;
* la journalisation réseau ;
* les futurs VPN ;
* la connexion avec Azure.

Le principe est de ne pas autoriser un réseau complet vers un autre réseau sans raison.

Quelques exemples :

| Source         | Destination    | Flux           | Politique         |
| -------------- | -------------- | -------------- | ----------------- |
| Clients        | DNS AD         | DNS            | Autorisé          |
| Clients        | IIS            | HTTPS          | Autorisé          |
| Clients        | SQL            | SQL            | Bloqué            |
| IIS            | SQL Listener   | SQL            | Autorisé          |
| Administration | Serveurs       | Administration | Contrôlé          |
| DMZ            | Réseau interne | Tous           | Bloqué par défaut |
| Serveurs       | Internet       | Selon besoin   | Contrôlé          |

La matrice complète des flux sera construite dans `MLA-NET-001`.

---

# 4. Inventaire des machines

Le socle de référence comprend les machines suivantes.

| Machine        |           Adresse IP | Fonction         |
| -------------- | -------------------: | ---------------- |
| `MLA-FW01`     | Plusieurs interfaces | OPNsense         |
| `MLA-MGMT01`   |        `10.10.10.10` | Administration   |
| `MLA-DC01`     |        `10.10.20.10` | AD DS / DNS      |
| `MLA-DC02`     |        `10.10.20.11` | AD DS / DNS      |
| `MLA-IIS01`    |        `10.10.30.10` | IIS              |
| `MLA-IIS02`    |        `10.10.30.11` | IIS              |
| `MLA-LNX01`    |        `10.10.30.20` | Linux            |
| `MLA-SQL01`    |        `10.10.40.10` | SQL Server       |
| `MLA-SQL02`    |        `10.10.40.11` | SQL Server       |
| `MLA-W11-01`   |        `10.10.50.10` | Poste Windows    |
| `MLA-MON01`    |        `10.10.70.10` | Zabbix / Grafana |
| `MLA-SIEM01`   |        `10.10.70.20` | Wazuh            |
| `MLA-CA-ISS01` |        `10.10.80.10` | Issuing CA       |
| `MLA-CA-ROOT`  |           Hors ligne | Root CA          |

Ces machines ne seront pas toutes démarrées simultanément.

Le lab doit rester exploitable sur une station de travail raisonnable : nous démarrerons les rôles nécessaires au scénario en cours.

---

# 5. Active Directory et DNS

Deux contrôleurs de domaine seront déployés :

```text
MLA-DC01
MLA-DC02
```

Ils hébergeront :

* Active Directory Domain Services ;
* Domain Name System (DNS) ;
* Global Catalog.

Les zones DNS internes seront intégrées à Active Directory afin de bénéficier de la réplication entre contrôleurs de domaine.

Le lab permettra ensuite de travailler sur :

* Organizational Units (OU) ;
* utilisateurs et groupes ;
* délégation ;
* Group Policy Objects (GPO) ;
* Password Settings Objects (PSO) ;
* Windows Local Administrator Password Solution (Windows LAPS) ;
* rôles Flexible Single Master Operations (FSMO) ;
* audit ;
* réplication ;
* sauvegarde du System State ;
* restauration.

La présence de deux contrôleurs de domaine est importante : elle permettra de tester la continuité du service au lieu de considérer AD DS comme une simple machine isolée.

---

# 6. Synchronisation du temps

La synchronisation du temps est une dépendance souvent ignorée dans les petits labs.

Elle devient pourtant rapidement critique avec Kerberos, les certificats et certains mécanismes d'authentification.

Le contrôleur détenant le rôle Primary Domain Controller Emulator (PDC Emulator) utilisera une source de temps fiable.

Les autres machines du domaine suivront ensuite la hiérarchie de temps Active Directory.

Nous éviterons donc de configurer manuellement une source NTP différente sur chaque serveur.

---

# 7. PKI et certificats

La Public Key Infrastructure sera construite sur deux niveaux.

| Serveur        | Type                  | État       |
| -------------- | --------------------- | ---------- |
| `MLA-CA-ROOT`  | Standalone Root CA    | Hors ligne |
| `MLA-CA-ISS01` | Enterprise Issuing CA | En ligne   |

Le principe est le suivant :

```text
Root CA hors ligne
        │
        └── signe l'Issuing CA
                    │
                    ├── certificats serveurs
                    ├── certificats clients
                    ├── IIS / HTTPS
                    ├── SQL Server / TLS
                    └── auto-enrollment
```

La Root Certification Authority (Root CA) ne restera pas connectée au réseau en fonctionnement normal.

Nous pourrons ainsi travailler sur :

* templates ;
* auto-enrollment ;
* renouvellement ;
* révocation ;
* Certificate Revocation List (CRL) ;
* Authority Information Access (AIA) ;
* certificats machines ;
* certificats utilisateurs ;
* certificats applicatifs.

La même PKI sera utilisée dans plusieurs séries du lab.

---

# 8. Chaîne applicative IIS et SQL Server

![Chaîne applicative MonLabAzure](/images/commencer/flux-application-monlabazure.png)

*Une application ne dépend pas uniquement d'IIS et de SQL Server : DNS, identité, certificats et filtrage réseau font partie de la chaîne.*

Deux serveurs IIS seront déployés :

```text
MLA-IIS01
MLA-IIS02
```

et deux instances SQL Server :

```text
MLA-SQL01
MLA-SQL02
```

L'objectif est de disposer d'une petite architecture applicative suffisamment réaliste pour tester plusieurs sujets avec le même environnement.

La partie SQL s'appuiera sur :

* Windows Server Failover Cluster (WSFC) ;
* SQL Server Always On Availability Groups ;
* Availability Group Listener ;
* quorum ;
* witness ;
* basculement.

Les serveurs IIS ne cibleront pas directement `MLA-SQL01`.

Ils utiliseront le listener de l'Availability Group.

Cela permet de conserver un point de connexion stable même lorsqu'un basculement SQL est effectué.

---

## Dépendances de l'application

La chaîne applicative fait intervenir plusieurs services.

| Composant         | Dépendance                          |
| ----------------- | ----------------------------------- |
| Poste utilisateur | DNS, identité, réseau               |
| IIS               | AD DS, PKI, DNS, SQL                |
| SQL Listener      | DNS, WSFC, réseau                   |
| SQL Server        | AD DS, comptes de service, stockage |
| HTTPS             | PKI, certificats, TLS               |
| Supervision       | Accès réseau et agents              |
| Sécurité          | Logs, événements, Wazuh             |

Ce type de dépendance sera utilisé dans les scénarios de troubleshooting.

Un problème d'accès à l'application ne signifie donc pas automatiquement qu'IIS est en panne.

---

# 9. Linux

Le serveur :

```text
MLA-LNX01
```

sera basé sur Ubuntu Server.

Il servira notamment pour :

* Secure Shell (SSH) ;
* Bash ;
* Git ;
* Ansible ;
* Terraform ;
* Azure CLI ;
* tests DNS ;
* tests réseau ;
* certificats ;
* automatisation.

Nous pourrons ensuite étudier son intégration à Active Directory lorsqu'elle apporte une valeur au scénario.

---

# 10. Poste utilisateur

Le poste :

```text
MLA-W11-01
```

représentera un poste utilisateur Windows membre du domaine.

Il permettra de valider :

* ouverture de session ;
* Kerberos ;
* application des GPO ;
* DNS ;
* certificats ;
* Windows LAPS ;
* accès HTTPS aux applications ;
* droits utilisateurs.

Il servira également plus tard pour les scénarios Microsoft Entra Hybrid Join et Microsoft Intune.

---

# 11. Administration

L'administration sera centralisée autour de :

```text
MLA-MGMT01
```

Cette machine pourra accueillir :

* Remote Server Administration Tools (RSAT) ;
* PowerShell ;
* Windows Admin Center ;
* outils SQL ;
* outils PKI ;
* Git ;
* SSH ;
* Terraform ;
* outils Azure.

Le but est d'éviter de transformer les contrôleurs de domaine en postes d'administration.

---

## Comptes d'administration

Les identités privilégiées seront séparées des comptes utilisateurs.

Exemple :

| Compte         | Usage                           |
| -------------- | ------------------------------- |
| `user01`       | Travail quotidien               |
| `adm-server01` | Administration des serveurs     |
| `adm-domain01` | Administration Active Directory |

Le modèle sera affiné lorsque nous traiterons la sécurisation des accès privilégiés.

---

# 12. Supervision et sécurité

Deux fonctions distinctes seront mises en place.

### Supervision

`MLA-MON01` hébergera principalement :

* Zabbix ;
* Grafana.

Zabbix assurera la collecte, les contrôles et l'alerting.

Grafana sera utilisé pour construire des dashboards plus lisibles.

Les éléments supervisés comprendront notamment :

* CPU ;
* mémoire ;
* stockage ;
* services Windows ;
* Linux ;
* AD DS ;
* DNS ;
* IIS ;
* SQL Server ;
* certificats ;
* disponibilité réseau.

### Sécurité et événements

`MLA-SIEM01` hébergera Wazuh.

Il servira à centraliser et analyser :

* événements Windows ;
* logs Linux ;
* changements de fichiers ;
* événements de sécurité ;
* alertes sur les endpoints.

Ces briques pourront plus tard être comparées avec Azure Monitor, Log Analytics, Microsoft Sentinel et Defender for Cloud.

---

# 13. Patching, sauvegarde et restauration

Une infrastructure correctement construite mais impossible à restaurer reste fragile.

Le lab intégrera donc trois sujets dès le parcours on-premises :

* patching ;
* sauvegarde ;
* restauration.

Nous couvrirons notamment :

* Active Directory System State ;
* bases SQL Server ;
* PKI ;
* configuration OPNsense ;
* données applicatives ;
* configurations importantes.

Des scénarios de panne seront volontairement provoqués :

| Incident                  | Objectif                            |
| ------------------------- | ----------------------------------- |
| Arrêt de `MLA-DC01`       | Vérifier la continuité AD / DNS     |
| Perte de `MLA-SQL01`      | Tester le basculement Always On     |
| Erreur DNS                | Diagnostiquer une panne applicative |
| Certificat expiré         | Identifier un problème TLS          |
| Règle firewall incorrecte | Analyser un flux bloqué             |
| Restauration de données   | Valider réellement la sauvegarde    |

Le troubleshooting fait donc partie du design du lab, pas uniquement des articles de dépannage.

---

# 14. Préparation de la connectivité Azure

Le réseau suivant est réservé aux futurs besoins de transit :

```text
10.10.90.0/24
```

L'architecture évoluera ensuite vers une connectivité hybride.

```text
ON-PREMISES
10.10.0.0/16
     │
     │
 OPNsense
     │
     │ VPN Site-to-Site
     │
Azure VPN Gateway
     │
     ▼
  Azure Hub
     │
 ┌───┴────────────┐
 │                │
Spoke            Spoke
Workloads        Services
```

Les plages Azure devront être choisies sans chevauchement avec `10.10.0.0/16`.

Cette décision est importante : corriger un chevauchement d'adresses après la mise en place d'une architecture hybride est beaucoup plus pénible que le prévoir dès le départ.

---

# 15. Microsoft 365 et Microsoft Entra ID

Une fois le socle local stabilisé, le parcours continuera vers Microsoft Cloud.

La progression sera :

| Étape              | Objectif                                |
| ------------------ | --------------------------------------- |
| Microsoft 365      | Tenant, domaine, licences et services   |
| Microsoft Entra ID | Identités et accès Cloud                |
| Synchronisation    | Relier AD DS et Entra ID                |
| Identité hybride   | SSO, Hybrid Join, gouvernance des accès |
| Azure              | Étendre l'infrastructure vers le Cloud  |

Nous travaillerons notamment sur :

* domaine personnalisé ;
* utilisateurs et groupes ;
* Microsoft Entra Connect ;
* Microsoft Entra Cloud Sync ;
* Password Hash Synchronization (PHS) ;
* Single Sign-On (SSO) ;
* Microsoft Entra Hybrid Join ;
* authentification multifacteur ;
* Conditional Access.

---

# 16. Azure comme extension du système d'information

Azure ne sera pas traité comme un deuxième lab complètement indépendant.

Il sera construit à partir des besoins et contraintes de l'environnement existant.

Les principales étapes couvriront :

* Management Groups ;
* subscriptions ;
* Resource Groups ;
* Role-Based Access Control (RBAC) ;
* Azure Policy ;
* Landing Zone ;
* Hub & Spoke ;
* Azure Firewall ;
* DNS hybride ;
* Azure Private DNS ;
* Azure DNS Private Resolver ;
* Private Endpoint ;
* supervision ;
* sauvegarde ;
* résilience.

Cette continuité permettra de travailler sur de vrais scénarios hybrides.

---

# 17. Automatisation

L'automatisation viendra progressivement après la compréhension et la validation des briques.

Les principaux outils seront :

* Git ;
* PowerShell ;
* Azure CLI ;
* Terraform ;
* Ansible ;
* GitHub Actions ;
* Azure DevOps Pipelines.

La logique suivie dans MonLabAzure reste :

```text
Comprendre
   ↓
Concevoir
   ↓
Construire
   ↓
Valider
   ↓
Sécuriser
   ↓
Automatiser
   ↓
Industrialiser
```

Terraform ou Ansible doivent permettre de rendre un déploiement reproductible, pas de masquer une architecture que l'on ne sait pas diagnostiquer.

---

# 18. Ressources du lab

Toutes les machines n'ont pas besoin de fonctionner simultanément.

| Mémoire hôte  | Usage                                                 |
| ------------- | ----------------------------------------------------- |
| 32 Go         | Possible en démarrant uniquement les VM nécessaires   |
| 64 Go         | Configuration recommandée                             |
| 96 Go ou plus | Confortable pour les scénarios de haute disponibilité |

Un stockage SSD ou NVMe est fortement recommandé.

Pour construire progressivement l'ensemble du lab, environ **1 To** de capacité disponible constitue une base raisonnable lorsque les disques virtuels sont provisionnés dynamiquement.

---

# 19. Hyperviseur

L'architecture est volontairement indépendante de l'hyperviseur.

Elle pourra être adaptée à :

* Hyper-V ;
* VMware ;
* VirtualBox ;
* Proxmox.

Le point important reste la capacité à représenter les différents segments réseau.

Si les VLAN IEEE 802.1Q ne sont pas facilement utilisables, plusieurs réseaux virtuels distincts peuvent reproduire le même fonctionnement.

---

# 20. Lab et production

MonLabAzure cherche à reproduire des **principes de production**, pas à prétendre qu'une seule machine physique peut reproduire toute la résilience d'un datacenter.

Le lab pourra donc simplifier certains éléments :

* hyperviseur unique ;
* stockage local ;
* witness mutualisé ;
* ressources limitées ;
* absence de redondance matérielle ;
* licences adaptées au lab.

Ces simplifications seront acceptées tant qu'elles ne masquent pas le fonctionnement du service étudié.

Lorsqu'un écart modifie réellement l'architecture ou la sécurité, il sera indiqué dans l'article concerné.

---

# Roadmap

`MLA-ARCH-000` fixe maintenant le cadre technique.

La construction suivra cette progression :

| Ordre | Série                               |
| ----: | ----------------------------------- |
|     1 | Réseau et OPNsense                  |
|     2 | Active Directory                    |
|     3 | DNS et synchronisation du temps     |
|     4 | Sécurité AD                         |
|     5 | PKI                                 |
|     6 | SQL Server Always On                |
|     7 | IIS                                 |
|     8 | Linux et postes Windows             |
|     9 | Supervision et sécurité             |
|    10 | Sauvegarde et restauration          |
|    11 | Microsoft 365 et Microsoft Entra ID |
|    12 | Identité hybride                    |
|    13 | Fondations Azure                    |
|    14 | Automatisation et DevOps            |

---

# Conclusion

L'architecture de référence est maintenant définie.

Nous avons un plan réseau, une séparation claire des rôles, une stratégie d'identité, une PKI, une chaîne applicative IIS/SQL, des outils de supervision et un espace réservé pour la future connexion Azure.

À partir de maintenant, chaque nouveau composant devra s'intégrer dans ce cadre ou justifier explicitement pourquoi l'architecture doit évoluer.

La prochaine étape est concrète :

[**MLA-NET-001 - Construire le réseau MonLabAzure et déployer PFsense**]({{< relref "on-premises/reseau/mla-net-001-reseau-pfsense.md" >}})



