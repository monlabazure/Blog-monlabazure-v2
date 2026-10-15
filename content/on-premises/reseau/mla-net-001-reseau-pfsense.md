---
title: "MLA-NET-001 - Construire le réseau MonLabAzure et déployer pfSense"
date: 2026-09-22
draft: false
description: "Construction du socle réseau MonLabAzure avec pfSense : segmentation, VLAN, adressage IP, routage, filtrage, NAT et préparation de la future connectivité Azure."
categories:
  - "On-Premises"
tags:
  - "MonLabAzure"
  - "Réseau"
  - "pfSense"
  - "Firewall"
  - "VLAN"
  - "Segmentation"
  - "Sécurité"
  - "Azure"
series:
  - "Infrastructure On-Premises"
weight: 10
---

# MLA-NET-001 - Construire le réseau MonLabAzure et déployer pfSense

Avant Active Directory, SQL Server ou IIS, il faut un réseau propre.

C'est le premier vrai composant de l'infrastructure MonLabAzure.

Nous pourrions placer toutes les machines virtuelles dans un seul sous-réseau et commencer immédiatement à installer les serveurs. Ce serait plus rapide, mais peu intéressant pour la suite : aucun filtrage entre les rôles, aucune séparation entre utilisateurs et bases de données, et pratiquement aucun travail réel sur les flux réseau.

Nous allons donc construire le réseau avant les services.

Le firewall/router retenu pour le lab est **pfSense**.

Il assurera :

- le routage entre les différents segments ;
- le filtrage des flux ;
- la sortie vers Internet ;
- le Network Address Translation (NAT) ;
- la journalisation ;
- plus tard, la connexion VPN vers Microsoft Azure.

---

## Architecture cible

![Topologie réseau MonLabAzure avec pfSense](/images/on-premises/reseau/mla-net-001-topologie-reseau-pfsense.png)

*Topologie réseau de référence du lab MonLabAzure.*

L'espace d'adressage on-premises est :

```text
10.10.0.0/16
````

Il est découpé selon le rôle des machines.

| VLAN | Réseau          | Usage                   |
| ---: | --------------- | ----------------------- |
|   10 | `10.10.10.0/24` | Administration          |
|   20 | `10.10.20.0/24` | Identité                |
|   30 | `10.10.30.0/24` | Applications            |
|   40 | `10.10.40.0/24` | Bases de données        |
|   50 | `10.10.50.0/24` | Postes utilisateurs     |
|   60 | `10.10.60.0/24` | DMZ                     |
|   70 | `10.10.70.0/24` | Supervision et sécurité |
|   80 | `10.10.80.0/24` | PKI                     |
|   90 | `10.10.90.0/24` | Transit / Azure         |

La passerelle de chaque réseau suivra la même convention :

```text
10.10.X.1
```

Par exemple :

```text
MGMT       10.10.10.1
IDENTITY   10.10.20.1
APP        10.10.30.1
DB         10.10.40.1
CLIENTS    10.10.50.1
```

Ces adresses seront portées par pfSense.

---

## Pourquoi segmenter le lab ?

Une application IIS qui utilise SQL Server n'implique pas qu'un poste utilisateur doive pouvoir se connecter directement à SQL Server.

Le flux attendu ressemble plutôt à ceci :

```text
POSTE UTILISATEUR
       │
       │ HTTPS
       ▼
   SERVEUR IIS
       │
       │ SQL
       ▼
 SQL LISTENER
       │
       ▼
SQL01 / SQL02
```

À côté de ce chemin applicatif, d'autres services interviennent :

```text
IDENTITY
├── DNS
├── Kerberos
├── LDAP
└── stratégies de domaine
```

et :

```text
MGMT
└── administration contrôlée des serveurs
```

La segmentation nous permettra donc de répondre à une question simple pour chaque flux :

> Cette communication est-elle réellement nécessaire ?

Si la réponse est non, elle reste bloquée.

---

# 1. Architecture virtuelle retenue

La topologie peut être reproduite de différentes manières selon l'hyperviseur.

## Option recommandée

Lorsque l'hyperviseur permet de gérer correctement les VLAN, pfSense peut disposer de :

```text
vNIC 1
WAN

vNIC 2
Management

vNIC 3
Trunk VLAN
```

Le trunk transporte alors les réseaux :

```text
VLAN 20
VLAN 30
VLAN 40
VLAN 50
VLAN 60
VLAN 70
VLAN 80
VLAN 90
```

Le réseau d'administration peut rester séparé sur une interface dédiée.

Cette solution évite de dépendre du trunk pour administrer le firewall pendant les premières étapes.

---

## Variante à deux interfaces

Un lab plus compact peut fonctionner avec :

```text
vNIC 1
WAN

vNIC 2
Trunk
```

Le VLAN 10 est alors également transporté sur le trunk.

Cette configuration fonctionne, mais demande davantage de prudence lors de la configuration initiale : il faut s'assurer que l'accès à l'interface Web pfSense est opérationnel sur le VLAN de management avant de supprimer l'accès temporaire utilisé pendant l'installation.

---

## Variante sans VLAN 802.1Q

Si l'hyperviseur ne facilite pas l'utilisation des trunks, plusieurs réseaux virtuels distincts peuvent être utilisés.

Par exemple :

```text
NET-MGMT
NET-IDENTITY
NET-APP
NET-DB
NET-CLIENTS
NET-MONSEC
NET-PKI
```

pfSense reçoit alors plusieurs cartes réseau virtuelles.

Le fonctionnement logique reste identique : pfSense assure le routage et le filtrage entre les différents segments.

---

# 2. Préparer la VM pfSense

La machine firewall sera nommée :

```text
MLA-FW01
```

Configuration adaptée au lab :

| Ressource  | Configuration           |
| ---------- | ----------------------- |
| vCPU       | 2                       |
| RAM        | 4 Go                    |
| Disque     | 20 à 40 Go              |
| WAN        | 1 interface             |
| Management | 1 interface recommandée |
| Trunk      | 1 interface recommandée |

La VM démarre sur l'image d'installation pfSense.

Le WAN pourra être raccordé à un réseau NAT ou externe fourni par l'hyperviseur.

L'objectif est que `MLA-FW01` dispose d'une connectivité Internet sans exposer directement l'administration du firewall.

---

# 3. Installer pfSense

Lancer l'installation depuis la console de la VM.

Après l'installation :

1. redémarrer `MLA-FW01` ;
2. retirer le média d'installation ;
3. identifier les interfaces réseau ;
4. affecter le WAN ;
5. affecter temporairement le LAN ou l'interface Management ;
6. noter les noms des interfaces détectées.

Selon la plateforme, les interfaces peuvent porter des noms comme :

```text
em0
em1
igb0
igb1
vtnet0
vtnet1
```

Le nom exact n'a pas d'importance.

Ce qui compte est de savoir quelle interface virtuelle correspond au WAN, au management et au trunk.

---

# 4. Première connexion

Après une installation standard, pfSense fournit une interface LAN permettant d'accéder à l'interface Web.

Depuis une machine connectée à ce réseau temporaire, ouvrir l'interface d'administration en HTTPS.

Avant de modifier la segmentation, vérifier :

* accès à la GUI ;
* accès Internet côté WAN ;
* correspondance correcte des interfaces ;
* heure système ;
* hostname.

Le firewall sera nommé :

```text
MLA-FW01
```

À ce stade, Active Directory n'existe pas encore.

pfSense doit pouvoir fonctionner indépendamment du domaine.

---

# 5. Configurer le réseau Management

Le réseau d'administration sera :

```text
10.10.10.0/24
```

La passerelle :

```text
10.10.10.1
```

Le poste d'administration :

```text
MLA-MGMT01
10.10.10.10
```

Configuration IPv4 :

```text
Adresse    10.10.10.10
Masque     255.255.255.0
Gateway    10.10.10.1
```

L'accès à l'interface Web pfSense devra être validé depuis `MLA-MGMT01` avant de poursuivre.

---

# 6. Créer les VLAN

Dans pfSense :

```text
Interfaces
→ Assignments
→ VLANs
```

Créer les VLAN nécessaires sur l'interface utilisée comme trunk.

| Tag | Description |
| --: | ----------- |
|  20 | IDENTITY    |
|  30 | APP         |
|  40 | DB          |
|  50 | CLIENTS     |
|  60 | DMZ         |
|  70 | MONSEC      |
|  80 | PKI         |
|  90 | TRANSIT     |

Si le VLAN 10 est également transporté sur le trunk, ajouter :

```text
VLAN 10
MGMT
```

Pour chaque VLAN :

1. choisir l'interface parent ;
2. utiliser un tag IEEE 802.1Q ;
3. saisir l'identifiant du VLAN ;
4. ajouter une description claire ;
5. sauvegarder.

---

# 7. Affecter les VLAN aux interfaces

La création d'un VLAN ne suffit pas.

Il faut ensuite l'affecter comme interface pfSense.

Menu :

```text
Interfaces
→ Assignments
```

Ajouter chaque VLAN puis renommer les interfaces.

Nous utiliserons :

```text
MGMT
IDENTITY
APP
DB
CLIENTS
DMZ
MONSEC
PKI
TRANSIT
```

Éviter de conserver uniquement des noms comme :

```text
OPT1
OPT2
OPT3
```

Une règle nommée ou affichée avec `APP` et `DB` sera beaucoup plus simple à comprendre six mois plus tard.

---

# 8. Configurer les interfaces

Chaque interface doit être :

* activée ;
* configurée en IPv4 statique ;
* associée à son adresse de passerelle locale.

## IDENTITY

```text
IPv4
10.10.20.1/24
```

## APP

```text
IPv4
10.10.30.1/24
```

## DB

```text
IPv4
10.10.40.1/24
```

## CLIENTS

```text
IPv4
10.10.50.1/24
```

## DMZ

```text
IPv4
10.10.60.1/24
```

## MONSEC

```text
IPv4
10.10.70.1/24
```

## PKI

```text
IPv4
10.10.80.1/24
```

## TRANSIT

```text
IPv4
10.10.90.1/24
```

Pour les interfaces internes, ne pas appliquer des options destinées à un WAN public qui bloqueraient les plages privées RFC 1918.

---

# 9. Vérifier le routage local

À partir du moment où les interfaces sont configurées, pfSense connaît directement les réseaux :

```text
10.10.10.0/24
10.10.20.0/24
10.10.30.0/24
10.10.40.0/24
10.10.50.0/24
10.10.60.0/24
10.10.70.0/24
10.10.80.0/24
10.10.90.0/24
```

Aucune route statique n'est nécessaire entre ces réseaux directement connectés.

Cela ne signifie pas pour autant que les machines peuvent communiquer.

Le routage et le filtrage sont deux sujets différents.

---

# 10. Le comportement du firewall

pfSense applique les règles sur l'interface où le trafic entre dans le firewall.

Pour un poste du VLAN CLIENTS qui souhaite joindre un serveur IIS :

```text
CLIENTS
10.10.50.0/24
      │
      │ trafic entrant dans pfSense
      ▼
   pfSense
      │
      ▼
APP
10.10.30.0/24
```

La règle doit donc être créée sur :

```text
Firewall
→ Rules
→ CLIENTS
```

et non sur l'interface APP.

Cette convention sera utilisée dans tout MonLabAzure.

---

# 11. Supprimer les règles trop permissives

Une installation pfSense standard peut fournir une règle LAN autorisant largement les sorties du réseau LAN.

Cette règle est pratique pour démarrer.

Elle ne correspond pas à l'objectif final de MonLabAzure.

Une fois l'administration sécurisée depuis MGMT, nous allons remplacer les autorisations générales par des règles ciblées.

Principe :

```text
DENY BY DEFAULT
```

Puis :

```text
ouvrir uniquement ce qui est nécessaire
```

---

# 12. Politique de filtrage

![Flux réseau de référence pfSense](/images/on-premises/reseau/mla-net-001-flux-securite-pfsense.png)

*Premiers principes de filtrage entre les segments MonLabAzure.*

Quelques exemples :

| Source  | Destination        | Besoin                 | Décision                         |
| ------- | ------------------ | ---------------------- | -------------------------------- |
| CLIENTS | Domain Controllers | DNS / authentification | À autoriser                      |
| CLIENTS | APP                | HTTPS                  | À autoriser                      |
| CLIENTS | DB                 | SQL                    | Bloqué                           |
| APP     | DB                 | SQL                    | À autoriser                      |
| MGMT    | Serveurs           | Administration         | À autoriser de manière contrôlée |
| DMZ     | Interne            | Accès général          | Bloqué                           |
| MONSEC  | Serveurs           | Supervision            | À autoriser selon besoin         |

Les ports Active Directory exacts seront traités dans la série Active Directory.

Il n'est pas utile d'ouvrir maintenant une grande liste de ports pour des services qui ne sont pas encore installés.

---

# 13. Créer les alias réseau

Avant de multiplier les règles, créer des alias.

Menu :

```text
Firewall
→ Aliases
```

Exemples :

```text
NET_MGMT
10.10.10.0/24
```

```text
NET_IDENTITY
10.10.20.0/24
```

```text
NET_APP
10.10.30.0/24
```

```text
NET_DB
10.10.40.0/24
```

```text
NET_CLIENTS
10.10.50.0/24
```

```text
NET_MONSEC
10.10.70.0/24
```

```text
NET_PKI
10.10.80.0/24
```

Les alias rendent les règles beaucoup plus lisibles.

Au lieu de :

```text
10.10.30.10
10.10.30.11
```

nous pourrons ensuite utiliser :

```text
SRV_IIS
```

---

# 14. Convention de nommage des règles

Éviter :

```text
test
rule1
allow
server
```

Préférer :

```text
ALLOW_MGMT_TO_SERVERS
ALLOW_CLIENTS_TO_IIS_HTTPS
ALLOW_IIS_TO_SQL
BLOCK_CLIENTS_TO_DB
```

Une règle doit pouvoir être comprise sans ouvrir son détail.

---

# 15. Première règle : administration de pfSense

L'interface Web du firewall doit être accessible uniquement depuis le réseau d'administration.

Objectif :

```text
MGMT → pfSense GUI
ALLOW
```

et :

```text
CLIENTS → pfSense GUI
BLOCK
```

```text
APP → pfSense GUI
BLOCK
```

```text
DB → pfSense GUI
BLOCK
```

pfSense est un composant sensible de l'infrastructure. Son interface d'administration ne doit pas être disponible depuis tous les réseaux.

---

# 16. Accès Internet

Les machines du lab auront progressivement besoin d'Internet pour :

* Windows Update ;
* dépôts Linux ;
* GitHub ;
* modules PowerShell ;
* paquets Terraform ;
* téléchargements Microsoft ;
* mises à jour des outils.

Pour la première phase, nous autoriserons les sorties strictement nécessaires.

Nous renforcerons ensuite les règles lorsque les services seront stabilisés.

---

# 17. NAT sortant

Les réseaux MonLabAzure utilisent des adresses privées :

```text
10.10.0.0/16
```

Pour accéder à Internet, pfSense réalise un NAT vers l'adresse WAN.

Dans une configuration simple, le mode Automatic Outbound NAT convient au lab.

Le flux devient :

```text
VM
10.10.X.X
   │
   ▼
pfSense
   │
   │ Source NAT
   ▼
WAN
   │
   ▼
Internet
```

Nous n'avons pour le moment aucun besoin de publication entrante.

Aucun port forwarding vers IIS, SQL ou les autres serveurs internes ne doit donc être créé à cette étape.

---

# 18. DNS pendant la phase de bootstrap

Active Directory et les serveurs DNS du domaine ne sont pas encore disponibles.

Pendant cette première phase, les quelques machines nécessaires au déploiement peuvent utiliser temporairement un résolveur externe ou le resolver disponible dans l'environnement de bootstrap.

Cette configuration est temporaire.

Lorsque :

```text
MLA-DC01
10.10.20.10
```

et :

```text
MLA-DC02
10.10.20.11
```

seront opérationnels, les machines membres du domaine utiliseront ces serveurs comme DNS.

La passerelle restera pfSense.

---

# 19. DHCP

Les serveurs du lab utiliseront des adresses statiques.

Exemples :

```text
MLA-DC01      10.10.20.10
MLA-DC02      10.10.20.11

MLA-IIS01     10.10.30.10
MLA-IIS02     10.10.30.11

MLA-SQL01     10.10.40.10
MLA-SQL02     10.10.40.11
```

Le DHCP n'est donc pas indispensable pour construire le socle.

Il pourra être introduit plus tard pour les réseaux utilisateurs lorsque nous déciderons précisément où héberger ce service.

---

# 20. Tester les passerelles

Depuis chaque réseau actif, vérifier d'abord la passerelle locale.

Exemple :

```powershell
ping 10.10.10.1
```

ou :

```powershell
Test-NetConnection 10.10.10.1
```

Sur le réseau CLIENTS :

```powershell
ping 10.10.50.1
```

Le premier contrôle consiste simplement à vérifier que la machine atteint pfSense sur son segment.

---

# 21. Tester Internet

Depuis une machine autorisée :

```powershell
Test-NetConnection 1.1.1.1 -Port 443
```

Sous Linux :

```bash
curl -I https://example.com
```

Si l'accès par IP fonctionne mais pas par nom, le problème est probablement lié à la résolution DNS plutôt qu'au routage.

---

# 22. Tester le filtrage

Il faut tester les deux situations :

* un flux autorisé ;
* un flux interdit.

Par exemple, lorsqu'un serveur IIS sera disponible :

```powershell
Test-NetConnection 10.10.30.10 -Port 443
```

Depuis CLIENTS, le résultat devra correspondre à la règle prévue.

Pour SQL :

```powershell
Test-NetConnection 10.10.40.10 -Port 1433
```

Depuis CLIENTS, le résultat attendu sera :

```text
TcpTestSucceeded : False
```

Le même test exécuté depuis le réseau APP pourra être autorisé lorsque l'architecture SQL sera construite.

---

# 23. Utiliser les logs pfSense

Lorsqu'une connexion ne fonctionne pas, ne pas commencer par modifier les règles au hasard.

La méthode sera :

1. lancer le test depuis le client ;
2. consulter les logs firewall ;
3. identifier l'adresse source ;
4. identifier la destination ;
5. vérifier le protocole et le port ;
6. identifier la règle qui bloque ou autorise le trafic.

Cette démarche sera réutilisée dans les séries Active Directory, SQL, IIS et Azure hybride.

---

# 24. Préparer les futurs flux Active Directory

Le VLAN IDENTITY accueillera :

```text
MLA-DC01
10.10.20.10
```

et :

```text
MLA-DC02
10.10.20.11
```

Nous n'ouvrons pas encore toutes les communications Active Directory.

La série suivante déterminera précisément les besoins liés à :

* DNS ;
* Kerberos ;
* LDAP ;
* SMB ;
* RPC ;
* Group Policy ;
* synchronisation du temps.

Cette approche évite de créer dès maintenant une règle :

```text
ANY → IDENTITY → ANY
```

qui annulerait une grande partie de l'intérêt de la segmentation.

---

# 25. Réserver la DMZ

Le réseau :

```text
10.10.60.0/24
```

est réservé à une future DMZ.

Il n'est pas nécessaire d'y déployer immédiatement des machines.

Cette zone servira plus tard pour des scénarios d'exposition contrôlée de services.

La règle initiale reste :

```text
DMZ → INTERNAL
BLOCK
```

Les exceptions seront définies service par service.

---

# 26. Réserver le réseau de transit Azure

Le réseau :

```text
10.10.90.0/24
```

prépare la future connectivité hybride.

À terme :

```text
ON-PREMISES
10.10.0.0/16
      │
      ▼
   pfSense
      │
      │ IPsec / IKE
      ▼
Azure VPN Gateway
      │
      ▼
   Azure Hub
    /     \
   /       \
Spoke     Spoke
```

Une connexion Site-to-Site Azure utilisera un tunnel IPsec/IKE entre le firewall on-premises et Azure VPN Gateway.

Cette connexion n'est pas déployée dans cet article.

Nous préparons simplement l'architecture pour ne pas devoir revoir le plan IP plus tard.

---

# 27. Sauvegarder pfSense

Une fois la configuration réseau validée :

```text
Diagnostics
→ Backup & Restore
```

Exporter la configuration pfSense.

Conserver au minimum une sauvegarde :

* après l'installation initiale ;
* après création des VLAN ;
* après validation des premières règles ;
* avant une modification importante.

La configuration pfSense est suffisamment critique pour ne pas dépendre d'une reconstruction manuelle en cas d'erreur.

---

# 28. Validation finale

Avant de passer à Active Directory, vérifier les points suivants.

* [ ] `MLA-FW01` est installé.
* [ ] Le WAN fonctionne.
* [ ] L'accès d'administration fonctionne depuis MGMT.
* [ ] L'administration de pfSense n'est pas ouverte aux autres segments.
* [ ] Les interfaces internes sont créées.
* [ ] Les VLAN sont correctement affectés.
* [ ] Chaque segment possède sa passerelle.
* [ ] Les alias réseau sont créés.
* [ ] Le filtrage inter-réseaux fonctionne.
* [ ] Un flux explicitement autorisé a été testé.
* [ ] Un flux interdit a été testé.
* [ ] Les logs firewall ont été contrôlés.
* [ ] L'accès Internet fonctionne pour les réseaux prévus.
* [ ] Aucun port entrant inutile n'est publié depuis Internet.
* [ ] Une sauvegarde pfSense a été exportée.

---

# Lab versus production

## Dans MonLabAzure

Nous utilisons une seule VM pfSense.

Le firewall constitue donc un point de panne unique.

Ce choix reste acceptable pour un environnement de laboratoire.

## En production

Selon les exigences de disponibilité, il faudrait étudier notamment :

* plusieurs firewalls ;
* haute disponibilité ;
* CARP ;
* synchronisation de configuration ;
* liens réseau redondants ;
* switches redondants ;
* supervision du firewall ;
* sauvegardes automatisées ;
* procédures de remplacement.

L'objectif de ce lab est de maîtriser la segmentation, le routage et le filtrage avant d'introduire ces mécanismes supplémentaires.

---

# Références officielles

Les points techniques utilisés dans cet article s'appuient sur les documentations officielles suivantes.

## Netgate / pfSense

* **pfSense Documentation - Perform the Installation**
  Installation initiale de pfSense.

* **pfSense Documentation - Assign Interfaces**
  Affectation des interfaces WAN, LAN et interfaces optionnelles.

* **pfSense Documentation - VLAN Configuration**
  Création des VLAN IEEE 802.1Q, définition de l'interface parent et affectation des VLAN comme interfaces.

* **pfSense Documentation - Interface Configuration**
  Configuration IPv4 des interfaces et options spécifiques aux réseaux internes et externes.

* **pfSense Documentation - Basic Firewall Configuration Example**
  Fonctionnement des règles sur les interfaces et construction d'un filtrage plus strict.

* **pfSense Documentation - Aliases**
  Utilisation d'alias d'hôtes, de réseaux et de ports pour simplifier les règles.

* **pfSense Documentation - Outbound NAT**
  Fonctionnement du NAT sortant automatique, hybride et manuel.

* **pfSense Documentation - Making Backups in the GUI**
  Export de la configuration depuis `Diagnostics > Backup & Restore`.

## Microsoft Azure

* **Microsoft Learn - VPN Gateway topology and design**
  Présentation des architectures VPN Site-to-Site, Point-to-Site, VNet-to-VNet et des modèles de haute disponibilité.

* **Microsoft Learn - Create a Site-to-Site VPN connection**
  Mise en place d'une connexion IPsec/IKE entre un réseau on-premises et Azure VPN Gateway.

---

# Résultat obtenu

Le socle réseau MonLabAzure est maintenant prêt.

Nous disposons de :

* plusieurs segments clairement identifiés ;
* pfSense comme point central de routage ;
* une politique de filtrage contrôlée ;
* des conventions de nommage ;
* un plan IP stable ;
* une sortie Internet ;
* une zone de management ;
* une DMZ réservée ;
* un réseau destiné à la supervision ;
* un réseau PKI ;
* un espace de transit pour Azure ;
* une première sauvegarde du firewall.

Nous pouvons maintenant installer les premiers services d'identité sans remettre en cause l'architecture réseau.


La prochaine étape consiste à déployer les premiers Domain Controllers et à construire le domaine :

**MLA-AD-001 - Déployer Active Directory Domain Services et le premier contrôleur de domaine.**


