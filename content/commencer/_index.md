---
title: "Commencer"
description: "Point de départ du parcours MonLabAzure : comprendre l'architecture du lab, préparer l'environnement et suivre la progression de l'on-premises vers Azure."
draft: false
weight: 10
---

# Commencer avec MonLabAzure

Bienvenue sur **MonLabAzure**.

Le projet part d'un scénario volontairement proche de ce qu'on retrouve encore dans beaucoup d'entreprises : une infrastructure on-premises avec Active Directory, DNS, serveurs applicatifs, bases de données, postes utilisateurs, supervision et contraintes de sécurité.

Cette infrastructure va être construite progressivement, puis connectée à Microsoft 365, Microsoft Entra ID et Microsoft Azure.

L'idée n'est pas de créer une nouvelle démonstration à chaque article. On construit un environnement, on le fait évoluer et on réutilise les mêmes briques dans les sujets suivants.

---

## Le parcours du lab

![Commencer avec MonLabAzure](/images/commencer/commencer-monlabazure.png)

*Vue générale du parcours : préparation du socle on-premises, extension vers Microsoft Cloud puis automatisation et modernisation.*

Le parcours suit plusieurs grandes étapes :

1. préparer l'environnement de travail ;
2. construire le réseau on-premises ;
3. déployer les services d'infrastructure ;
4. sécuriser et superviser l'environnement ;
5. introduire Microsoft 365 et Microsoft Entra ID ;
6. mettre en place l'identité et la connectivité hybrides ;
7. construire les fondations Azure ;
8. automatiser progressivement l'infrastructure ;
9. moderniser les services lorsque cela apporte une réelle valeur.

Chaque étape s'appuie sur ce qui a déjà été mis en place.

---

## Avant de créer les premières machines

Un lab devient rapidement difficile à maintenir si on commence directement par installer des serveurs sans définir l'architecture.

Nous commençons donc par fixer quelques éléments :

- le plan d'adressage ;
- les segments réseau ;
- les conventions de nommage ;
- le domaine Active Directory ;
- le domaine Internet ;
- les rôles des différentes machines ;
- les principes d'administration ;
- les règles de sécurité ;
- les ressources disponibles sur l'hyperviseur.

Ces choix sont détaillés dans :

**MLA-ARCH-000 — Architecture de référence MonLabAzure**

Ce document sert de référence aux différentes séries du site.

---

## Le socle on-premises

La première partie du parcours consiste à construire l'infrastructure locale.

Nous aurons notamment :

- OPNsense pour le routage et le filtrage ;
- Active Directory Domain Services (AD DS) ;
- Domain Name System (DNS) ;
- Network Time Protocol (NTP) ;
- Active Directory Certificate Services (AD CS) ;
- une Public Key Infrastructure (PKI) ;
- SQL Server ;
- SQL Server Always On ;
- Internet Information Services (IIS) ;
- Linux ;
- Windows 11 ;
- Zabbix ;
- Grafana ;
- Wazuh ;
- des mécanismes de sauvegarde et de restauration.

L'objectif n'est pas simplement d'installer ces produits.

Nous allons aussi travailler sur leurs dépendances, les flux réseau, les comptes de service, les certificats, la supervision et les scénarios de panne.

---

## Un lab segmenté dès le départ

Le réseau sera construit avant Active Directory.

Ce choix permet de ne pas déployer tous les serveurs sur un unique réseau plat puis de devoir tout reprendre plus tard.

Le lab prévoit plusieurs segments dédiés, notamment pour :

- l'administration ;
- l'identité ;
- les applications ;
- les bases de données ;
- les postes utilisateurs ;
- la supervision ;
- la PKI ;
- la future connectivité Azure.

OPNsense assurera le routage entre ces réseaux.

Le principe retenu sera simple :

> Un flux n'est pas autorisé parce que les machines doivent communiquer. Il est autorisé parce qu'un besoin précis a été identifié.

Cette règle nous obligera à comprendre réellement les dépendances des services que nous déployons.

---

## La sécurité fait partie de la construction

La sécurité ne sera pas ajoutée dans une dernière série une fois le lab terminé.

Elle sera intégrée progressivement.

Nous utiliserons notamment :

- des comptes administratifs séparés ;
- le principe du moindre privilège ;
- des Group Policy Objects (GPO) de sécurité ;
- Windows Local Administrator Password Solution (Windows LAPS) ;
- Microsoft Defender ;
- Credential Guard lorsque pertinent ;
- des Group Managed Service Accounts (gMSA) ;
- Transport Layer Security (TLS) ;
- une PKI interne ;
- la centralisation des événements ;
- des règles firewall explicites ;
- des sauvegardes et des tests de restauration.

Certaines configurations resteront simplifiées pour le lab. Les écarts importants avec une vraie production seront indiqués lorsqu'ils ont un impact technique ou de sécurité.

---

## De l'on-premises vers Microsoft Cloud

Une fois le socle local stabilisé, nous commencerons à introduire les services Microsoft Cloud.

Le parcours continuera avec :

![Évolution vers Microsoft Cloud](/images/commencer/evolution-vers-microsoft-cloud.png)

Cette progression permettra de travailler sur des sujets comme :

* le domaine personnalisé ;
* les identités Cloud et synchronisées ;
* Microsoft Entra Connect ;
* Microsoft Entra Cloud Sync ;
* le Single Sign-On (SSO) ;
* Microsoft Entra Hybrid Join ;
* les licences ;
* les rôles administratifs ;
* l'authentification forte ;
* Conditional Access.

---

## Puis vient Azure

Azure ne sera pas abordé comme un environnement complètement séparé.

Les fondations seront construites en tenant compte de l'existant.

Nous travaillerons notamment sur :

* Management Groups ;
* subscriptions ;
* Resource Groups ;
* Role-Based Access Control (RBAC) ;
* Azure Policy ;
* conventions de nommage ;
* tags ;
* budgets ;
* Landing Zone ;
* Hub & Spoke ;
* Azure Firewall ;
* DNS hybride ;
* Azure DNS Private Resolver ;
* Private Endpoint ;
* sauvegarde ;
* supervision ;
* résilience.

La connectivité avec l'on-premises sera ensuite mise en place avec un VPN Site-to-Site, puis étudiée dans des scénarios plus avancés.

---

## Comprendre avant d'automatiser

Terraform, Ansible ou un pipeline CI/CD ne doivent pas devenir une boîte noire.

Quand le sujet le justifie, nous commençons par comprendre et construire le service.

Ensuite seulement, nous l'automatisons.

La progression est la suivante :

![La méthode MonLabAzure](/images/commencer/la-methode-monlabazure-progression-technique.png)

Les principaux outils utilisés seront :

* Git ;
* PowerShell ;
* Azure CLI ;
* Terraform ;
* Ansible ;
* GitHub Actions ;
* Azure DevOps Pipelines.

---

## Deux façons de suivre MonLabAzure

### Suivre le parcours complet

C'est l'approche recommandée si vous voulez reconstruire le lab depuis zéro.

Les différentes séries sont suivies dans l'ordre et l'environnement évolue progressivement.

### Travailler sur un sujet précis

Vous pouvez également accéder directement à une technologie ou une série.

Dans ce cas, vérifiez les prérequis indiqués au début de l'article.

Un sujet SQL Server Always On pourra par exemple supposer qu'Active Directory, DNS et le réseau sont déjà disponibles.

---

## Comment sont construits les articles ?

Un article technique ne se limite pas à une suite de commandes.

Selon le sujet, vous trouverez :

* le contexte ;
* les objectifs ;
* les prérequis ;
* l'architecture concernée ;
* les choix techniques ;
* les étapes de mise en œuvre ;
* les commandes de contrôle ;
* les résultats attendus ;
* les points de sécurité ;
* les erreurs fréquentes ;
* les scénarios de dépannage ;
* les différences entre lab et production ;
* les opérations de nettoyage ou de restauration.

Le but est de pouvoir reproduire le scénario et surtout de comprendre ce qui a été construit.

---

## Par où commencer ?

La première lecture recommandée est :

[**MLA-ARCH-000 - Architecture de référence MonLabAzure**]({{< relref "commencer/mla-arch-000-architecture-reference-monlabazure.md" >}})

Vous y trouverez le plan réseau, les machines du lab, les rôles, les conventions et les principaux choix d'architecture.

Ensuite, la construction commence réellement avec :

[**MLA-NET-001 - Construire le réseau MonLabAzure et déployer pfsense**]({{< relref "on-premises/reseau/mla-net-001-reseau-pfsense.md" >}})

Nous commencerons donc par le réseau, avant de déployer le premier contrôleur de domaine.
