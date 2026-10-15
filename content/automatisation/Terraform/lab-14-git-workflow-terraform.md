---
title: "LAB-14 — Git et workflow professionnel pour Terraform"
date: 2026-10-15
draft: false
description: "Structurer un workflow Git professionnel pour Terraform avec branches courtes, commits lisibles, Pull Requests, revue du plan et protection des fichiers sensibles."
summary: "Passer d’un Terraform exécuté localement à un changement d’infrastructure traçable, relu et prêt pour l’automatisation CI/CD."
slug: "terraform-azure-lab-14-git-workflow-professionnel"
categories:
  - "Automatisation"
tags:
  - "Terraform"
  - "Git"
  - "GitHub"
  - "Pull Request"
  - "Infrastructure as Code"
series:
  - "Terraform Azure — De zéro à production"
weight: 15
ShowToc: true
TocOpen: true
---

# LAB-14 — Git et workflow professionnel pour Terraform

Jusqu’ici, Git nous servait surtout à versionner les Labs. À partir de maintenant, Git devient une partie du mécanisme de changement de l’infrastructure.

Le modèle cible n’est plus :

```text
Modifier Terraform
      │
      ▼
terraform apply
```

mais :

```text
Issue / besoin
      │
      ▼
Feature branch
      │
      ▼
Modification Terraform
      │
      ▼
Checks locaux
      │
      ▼
Pull Request
      │
      ▼
CI
      │
      ▼
Plan relu
      │
      ▼
Merge
      │
      ▼
CD
```

## 1. Objectifs

Nous allons :

- structurer le repository ;
- protéger state, plans et fichiers locaux ;
- utiliser des feature branches courtes ;
- écrire des commits exploitables ;
- séparer changement de code et changement d’infrastructure ;
- préparer une Pull Request ;
- définir une revue Terraform ;
- éviter les branches permanentes DEV/QUAL/PROD ;
- préparer GitHub Actions.

## 2. Repository cible

```text
terraform-azure-platform/
├── .github/
│   └── workflows/
├── modules/
│   ├── network/
│   ├── network-security/
│   ├── container-platform/
│   └── application-gateway/
├── environments/
│   ├── dev/
│   ├── qual/
│   └── prod/
├── .gitignore
├── .editorconfig
└── README.md
```

Pour les Labs, cette structure peut rester dans `lab-14/`. Pour une vraie plateforme, elle deviendra progressivement le repository principal.

## 3. `.gitignore`

Créer à la racine :

```gitignore
# Terraform working directories
**/.terraform/*

# Terraform state
*.tfstate
*.tfstate.*

# Terraform plan files
*.tfplan

# Local variable values
*.tfvars
*.tfvars.json

# Keep examples
!*.tfvars.example

# Backend local config
*.tfbackend

# Crash logs
crash.log
crash.*.log

# Local overrides
override.tf
override.tf.json
*_override.tf
*_override.tf.json

# Terraform CLI local configuration
.terraformrc
terraform.rc

# OS/editor files
.DS_Store
Thumbs.db
.vscode/*
!.vscode/extensions.json
```

Ne pas ignorer :

```text
.terraform.lock.hcl
```

Le Dependency Lock File fait partie des dépendances reproductibles du Root Module.

## 4. Vérifier ce qui est déjà suivi

Un `.gitignore` n’enlève pas automatiquement un fichier déjà commité.

Vérifier :

```powershell
git ls-files |
  Select-String `
    -Pattern "tfstate|tfplan|tfvars$|\.tfbackend$"
```

Si un state a été commité, il faut considérer son contenu comme potentiellement exposé et traiter l’incident, pas seulement supprimer le fichier du dernier commit.

## 5. État de départ

```powershell
git status
git branch --show-current
git log --oneline -5
```

La branche principale sera :

```text
main
```

Elle représente la version intégrée du code.

## 6. Ne pas utiliser une branche permanente par environnement

Nous évitons :

```text
dev
qual
prod
```

comme branches permanentes représentant les environnements.

Les environnements existent déjà sous forme de :

```text
environments/dev
environments/qual
environments/prod
```

avec states, identités et pipelines séparés.

Nous voulons promouvoir le même commit :

```text
commit abc123
   │
   ├── DEV
   ├── QUAL
   └── PROD
```

et non trois histoires Git qui divergent.

## 7. Créer une feature branch

Exemple :

```powershell
git switch main
git pull --ff-only
git switch -c "feature/key-vault-private-endpoint"
```

Une branche courte doit représenter un changement cohérent.

Bon exemple :

```text
feature/add-key-vault-private-endpoint
```

Moins bon :

```text
herve-test-2-final-new
```

## 8. Travailler par petit changement

Une Pull Request Terraform de 2 000 lignes qui ajoute :

- réseau ;
- Key Vault ;
- Container Apps ;
- WAF ;
- IAM ;
- monitoring ;

est difficile à relire correctement.

Préférer :

```text
PR 1 — module Private Endpoint
PR 2 — intégration Key Vault
PR 3 — DNS privé
PR 4 — monitoring
```

lorsque ces changements peuvent être isolés sans casser la plateforme.

## 9. Checks avant commit

À la racine :

```powershell
terraform fmt -recursive
```

Pour chaque Root Module :

```powershell
terraform `
  -chdir="environments/dev" `
  init `
  -backend=false

terraform `
  -chdir="environments/dev" `
  validate
```

Le `-backend=false` est utile lorsque nous voulons uniquement préparer la configuration pour la validation, sans contacter le Remote State.

## 10. Inspecter le diff

Avant `git add` :

```powershell
git diff
```

Vérifier :

- changement attendu ;
- absence de secret ;
- absence d’ID personnel inutile ;
- absence de state ;
- absence de plan binaire ;
- noms cohérents.

## 11. Ajouter sélectivement

Préférer :

```powershell
git add `
  ".\modules\key-vault-private-endpoint" `
  ".\environments\dev\private-link.tf"
```

à un :

```powershell
git add .
```

non relu.

`git add .` n’est pas mauvais en soi, mais il doit être suivi de contrôles.

## 12. Contrôler le staging area

```powershell
git diff --cached
```

Puis :

```powershell
git diff --cached --name-only
```

C’est le contenu exact du prochain commit.

## 13. Convention de commit

Nous utilisons une convention simple inspirée de Conventional Commits :

```text
feat(terraform): add key vault private endpoint
fix(terraform): correct container app private dns
refactor(terraform): extract network module
docs(terraform): document backend recovery
chore(terraform): update provider lock file
```

Le message doit expliquer ce que fait le changement.

## 14. Créer le commit

```powershell
git commit `
  -m "feat(terraform): add key vault private endpoint"
```

Puis :

```powershell
git show --stat
```

## 15. Plusieurs commits dans une PR

Une PR peut contenir plusieurs commits cohérents :

```text
feat(terraform): add private endpoint module
feat(terraform): link key vault private dns
docs(terraform): add troubleshooting notes
```

Éviter une suite comme :

```text
test
fix
fix2
final
final2
really-final
```

si ces commits ne présentent aucune valeur pour l’historique.

## 16. Synchroniser avec `main`

Avant d’ouvrir la PR :

```powershell
git fetch origin
git rebase origin/main
```

Cette stratégie garde un historique lisible.

Dans une équipe qui préfère merge commits, respecter la convention collective. Git n’impose pas une seule politique universelle.

## 17. Résoudre un conflit

En cas de conflit :

```powershell
git status
```

Corriger les fichiers puis :

```powershell
git add <fichier>
git rebase --continue
```

Ne pas résoudre automatiquement un conflit Terraform sans comprendre quelle configuration doit être conservée.

## 18. Pousser la branche

```powershell
git push `
  --set-upstream origin `
  "feature/key-vault-private-endpoint"
```

À partir de ce point, la Pull Request devient l’unité de revue.

## 19. Template de Pull Request

Créer :

```text
.github/pull_request_template.md
```

avec :

```markdown
## Objectif

Décrire le besoin et le résultat attendu.

## Périmètre

- [ ] DEV
- [ ] QUALIF
- [ ] PROD
- [ ] Module partagé
- [ ] Backend / IAM

## Terraform

- [ ] `terraform fmt -check -recursive`
- [ ] `terraform validate`
- [ ] Plan relu
- [ ] Aucun secret dans le code
- [ ] Aucun state / plan binaire versionné

## Impact

Créer / modifier / remplacer / détruire les ressources suivantes :

- ...

## Risques

- ...

## Rollback

- ...
```

## 20. Une PR Terraform doit expliquer l’impact

Le reviewer ne doit pas uniquement voir :

```text
+35 -12
```

Il doit comprendre :

```text
0 create
2 update
0 replace
0 destroy
```

ou :

```text
1 replacement
```

et savoir pourquoi.

Le plan Terraform sera donc intégré dans les Labs CI suivants.

## 21. Review checklist

Pour une revue Terraform, regarder notamment :

```text
Architecture
Naming
Tags
Scope RBAC
Private/Public exposure
State boundary
Dependencies
Provider versions
Resource replacements
Destroy actions
Secrets
Cost
Recovery
```

## 22. Attention aux changements `ForceNew`

Une modification qui semble petite dans le code peut forcer un remplacement.

Exemple conceptuel :

```text
change subnet
      │
      ▼
resource replacement
      │
      ▼
downtime potentiel
```

La revue du diff Git ne remplace jamais la revue du `terraform plan`.

## 23. Secret scanning

Même avec `.gitignore`, un secret peut être ajouté dans un fichier `.tf`, `.md` ou YAML.

La règle reste :

> ne jamais considérer Git comme un gestionnaire de secrets.

GitHub Secret Scanning ou des outils équivalents peuvent ajouter une barrière, mais ils ne remplacent pas les bonnes pratiques d’écriture.

## 24. Branch Protection

Sur GitHub, `main` devra progressivement exiger :

```text
Pull Request
Required reviews
Required status checks
No direct push
```

Quand la CI du LAB-15 sera en place, nous ajouterons les checks Terraform comme conditions de merge.

## 25. CODEOWNERS

Exemple :

```text
.github/CODEOWNERS
```

```text
/environments/prod/ @cloud-platform-team
/modules/           @cloud-platform-team
```

Adapter les owners réels à l’organisation.

Le principe est de rendre explicite qui doit relire les composants sensibles.

## 26. Tag et release

Un repository de plateforme peut évoluer en continu sur `main`.

Un module partagé dans son propre repository mérite souvent des releases :

```text
v1.0.0
v1.1.0
v2.0.0
```

Ne pas confondre :

```text
version du module
```

et :

```text
environnement déployé
```

## 27. Rollback Git vs rollback infrastructure

Faire :

```powershell
git revert <commit>
```

crée un changement de code inverse.

Cela ne modifie Azure qu’après un nouveau :

```text
terraform plan
terraform apply
```

Le rollback infrastructure reste donc un changement Terraform à part entière.

## 28. Workflow final

```text
main
 │
 └── feature branch
       │
       ├── fmt
       ├── validate
       ├── diff
       └── commit
             │
             ▼
         Pull Request
             │
             ▼
            CI
             │
             ▼
         Terraform Plan
             │
             ▼
            Review
             │
             ▼
            Merge
```

## 29. Validation

```text
[ ] .gitignore sécurisé
[ ] lock files versionnés
[ ] feature branch créée
[ ] terraform fmt exécuté
[ ] terraform validate exécuté
[ ] git diff relu
[ ] staging area relue
[ ] commit clair
[ ] PR template créé
[ ] branches d'environnement permanentes évitées
[ ] protection future de main comprise
[ ] rollback Git vs Terraform compris
```

## 30. Suite

Nous avons maintenant un processus humain correct.

Il faut automatiser les contrôles répétitifs.

**LAB-15 — Continuous Integration Terraform avec GitHub Actions**

## Références officielles

- Git documentation
- GitHub Docs — Pull Requests
- GitHub Docs — Protected Branches
- HashiCorp — Terraform Style Guide
- HashiCorp — `terraform fmt`
- HashiCorp — `terraform validate`
