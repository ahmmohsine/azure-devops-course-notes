# CD, Infrastructure as Code et Ansible

> Complément aux notes du [README](README.md) et à l'[aide-mémoire YAML](YAML-AZURE-PIPELINES.md).
> Les termes techniques sont gardés en anglais, les explications sont en français.
> Exemple fil rouge : **EventApp** (ASP.NET Core MVC, .NET 10, EF Core + SQL Server, architecture Domaine / Application / Infrastructure / Mvc).

---

## Sommaire

- [L'essentiel en 10 points](#lessentiel-en-10-points)
- [1. Azure Compute Services](#1-azure-compute-services)
- [2. Stages, jobs, steps : qui s'exécute quand ?](#2-stages-jobs-steps--qui-sexécute-quand-)
- [3. L'artefact : build once, deploy many](#3-lartefact--build-once-deploy-many)
- [4. Classic Releases vs pipelines YAML](#4-classic-releases-vs-pipelines-yaml)
- [5. Gates vs Checks](#5-gates-vs-checks)
- [6. Déployer une app .NET + Azure SQL (checklist EventApp)](#6-déployer-une-app-net--azure-sql-checklist-eventapp)
- [7. Infrastructure as Code : les principes](#7-infrastructure-as-code--les-principes)
- [8. Les outils IaC : deux familles](#8-les-outils-iac--deux-familles)
- [9. Ansible](#9-ansible)
- [10. Dev ou DevOps : quoi prioriser ?](#10-dev-ou-devops--quoi-prioriser-)
- [Auto-évaluation](#auto-évaluation)

---

## L'essentiel en 10 points

1. **Compute** = les services qui font tourner le code. Plus on va de **IaaS** vers **Serverless**, moins on gère, mais moins on contrôle.
2. **Stages** s'enchaînent, **jobs** d'un stage tournent en parallèle (une machine chacun), **steps** s'enchaînent sur la même machine.
3. **Artefact** = le résultat emballé du build. On le **teste une fois** et on **déploie exactement le même** partout.
4. **Classic Releases** = l'ancienne CD à la souris. Toujours là, mais Microsoft recommande le **YAML**.
5. **Gate** (Classic) = vérification automatique sur le **stage**, avant et après déploiement.
6. **Check** (YAML) = vérification sur la **ressource** (environment, service connection), impossible à contourner depuis le YAML.
7. **IaC** = infrastructure décrite dans des fichiers versionnés, **déclarative** et **idempotente**.
8. **Idempotence** = lancer 1 fois ou 10 fois donne le même résultat.
9. Deux familles d'outils : **provisioning** (créer : ARM/Bicep, Terraform, CloudFormation) et **configuration** (configurer : Ansible, Chef, Puppet, Salt).
10. **Ansible** = sans agent, YAML, idempotent. **Inventaire** = sur quelles machines ; **playbook** = quoi faire.

---

## 1. Azure Compute Services

La famille des services Azure qui **exécutent ton code** (CPU + mémoire).

### Qui gère quoi ?

```
Tu gères TOUT                                          Azure gère TOUT
◄─────────────────────────────────────────────────────────────►
Machine virtuelle    Conteneurs    App Service    Azure Functions
     (IaaS)                          (PaaS)        (Serverless)
```

| Modèle | Tu gères | Azure gère |
|---|---|---|
| **IaaS** (Infrastructure as a Service) | OS, mises à jour, runtime .NET, appli | Matériel, réseau |
| **PaaS** (Platform as a Service) | **Seulement l'appli** | OS, mises à jour, serveur web, scalabilité |
| **Serverless** | Seulement des **fonctions** | Tout ; paiement à l'exécution |

**Image de la pizza :** IaaS = on te prête la cuisine ; PaaS = pâte et four prêts, tu choisis la garniture ; Serverless = tu achètes une part quand tu as faim.

### Les services

| Service | En une phrase | Quand l'utiliser |
|---|---|---|
| **Virtual Machines** | Un ordinateur complet dans le cloud | Contrôle total, vieux logiciels |
| **VM Scale Sets** | Groupe de VM identiques qui grandit / rétrécit | Trafic variable sur des VM |
| **App Service** | Héberge sites web et API sans gérer de serveur | Appli web classique (EventApp) |
| **Azure Functions** | Code déclenché par un événement (HTTP, timer, message) | Petites tâches, jobs planifiés |
| **Container Instances (ACI)** | Lancer un conteneur vite, sans orchestration | Tests, tâches ponctuelles |
| **Container Apps** | Conteneurs gérés avec scalabilité auto | Microservices sans Kubernetes |
| **AKS** | Kubernetes géré | Grosses architectures conteneurs |
| **Azure Batch** | Calcul massif en parallèle | Rendu, simulations |

### Pourquoi App Service pour EventApp ?

- pas d'OS ni de serveur web à gérer (≠ VM)
- pas besoin de Docker (≠ Container Apps / AKS)
- pas besoin de découper en fonctions (≠ Functions)
- tâche `AzureWebApp@1` dans le pipeline, plan **Free F1**

---

## 2. Stages, jobs, steps : qui s'exécute quand ?

```
Pipeline
 └─ Stage (1 ou plusieurs)
     └─ Job (1 ou plusieurs)
         └─ Step (1 ou plusieurs)
```

| Niveau | Exécution par défaut | À retenir |
|---|---|---|
| **Stages** | L'un **après** l'autre | Grandes phases : Build, Deploy_Test, Deploy_Prod |
| **Jobs** d'un stage | En **parallèle** | **Une machine par job** : ils ne partagent pas leurs fichiers |
| **Steps** d'un job | L'un **après** l'autre | Même machine, mêmes fichiers |

- Un YAML n'a **pas toujours 2 stages** : 1, 2, 3 ou plus. Les noms (`Build`, `Deploy`) sont **libres**.
- Avec seulement `steps:`, Azure crée **un stage et un job invisibles**.
- `dependsOn:` force l'ordre, entre stages comme entre jobs.
- Plan gratuit : souvent **1 seul agent à la fois** → les jobs « parallèles » passent l'un après l'autre.

```yaml
- stage: Build
  jobs:
    - job: CompilerEtTester          # machine 1
      pool: { vmImage: 'ubuntu-latest' }
      steps:
        - script: dotnet build
        - script: dotnet test

    - job: VerifierFormatage         # machine 2, en même temps
      pool: { vmImage: 'ubuntu-latest' }
      steps:
        - script: dotnet format --verify-no-changes
```

---

## 3. L'artefact : build once, deploy many

**Artefact** = le résultat de la compilation, **emballé et sauvegardé** (le carton en sortie d'usine).

```
Code ──► Pipeline CI ──► ARTEFACT ──► Déploiement (CD)
         (usine)         (carton)      (livraison)
```

Pour EventApp, c'est le dossier produit par `dotnet publish` : les `.dll` des 4 projets, les packages NuGet, `appsettings.json`, `wwwroot/`. Plus de code source.

### Pourquoi ?

1. **L'agent est détruit** à la fin du job : sans artefact, tout est perdu.
2. **On compile une fois** : le stage Deploy prend **exactement** ce qui a été testé. Recompiler = déployer quelque chose de non testé.
3. **Traçabilité** : un artefact par run, lié à un commit. Bug en prod ? On **redéploie l'artefact d'hier**.

| Tâche | Où | Rôle |
|---|---|---|
| `PublishPipelineArtifact@1` | Stage Build | Ranger le carton (préférer à l'ancien `PublishBuildArtifacts@1`) |
| (automatique) | Job `deployment` | Télécharger dans `$(Pipeline.Workspace)/<nom>` |
| `AzureWebApp@1` | Stage Deploy | Livrer sur l'App Service |

Un artefact peut aussi être un paquet NuGet, une image Docker, un `.zip`, un `.apk`...

---

## 4. Classic Releases vs pipelines YAML

**Classic Releases** (menu *Pipelines → Releases*) = l'ancienne façon de faire la CD, **à la souris**, séparée du pipeline de build.

```
Pipeline de BUILD (CI)  ──artefact──►  RELEASE (CD) : Dev ──► Test ──► Prod
```

| Classic Release | YAML multi-stage |
|---|---|
| Artefact d'un pipeline de build | `PublishPipelineArtifact` dans le stage Build |
| Une « étape » Prod | `stage: Deploy` |
| Pre-deployment approvals | Approvals sur l'**environment** |
| Trigger ⚡ après build | `dependsOn` + `condition` |
| Bouton **Create release** | Bouton **Run pipeline** |
| **Redeploy** | **Rerun** du stage Deploy |

| | Classic | YAML |
|---|---|---|
| Fonctionne encore | Oui | Oui |
| Nouvelles fonctionnalités | **Non** | Oui |
| Nouvelle organisation | **Désactivé par défaut** (Organization settings → Pipelines → Settings) | Activé |
| Versionné dans git | Non | **Oui** |
| En entreprise | Anciens projets | Nouveaux projets |

**Montage hybride possible :** CI en YAML, CD en Classic Release (visuelle) pour bien comprendre, puis passer au tout-YAML.

**Astuce :** dans l'éditeur YAML (bouton *Edit*), le panneau **Tasks** génère le YAML à partir d'un formulaire.

---

## 5. Gates vs Checks

### Gates (Classic Releases)

Vérification **automatique** avant (*pre-deployment*) ou après (*post-deployment*) le déploiement d'un stage. Réévaluée à chaque *sampling interval* jusqu'à ce que **toutes** passent, ou jusqu'au **timeout** (→ échec).

```
   [Approvals] [Gates] ──► DEPLOY ──► [Gates] [Approvals]
```

| Gate | Vérifie |
|---|---|
| Query Azure Monitor alerts | Aucune alerte active |
| Invoke REST API | Un endpoint répond OK (health check) |
| Invoke Azure Function | Logique personnalisée |
| Query work items | Aucun bug bloquant ouvert |
| Check Azure Policy compliance | Respect des policies |

**Approbation** = un humain clique. **Gate** = automatique. Souvent combinées.

### Checks (YAML)

Même idée, mais configurés sur une **ressource** : *Pipelines → Environments → production → Approvals and checks*.

| | Gates | Checks |
|---|---|---|
| Système | Classic Releases | YAML multi-stage |
| Attaché à | Un **stage** de la release | Une **ressource** (environment, service connection, agent pool...) |
| Moment | Avant **et** après | **Avant** seulement |
| Approbation manuelle | À côté des gates | Un type de check (*Approvals*) |
| Contournable ? | Une autre release vers la prod n'a pas la gate | **Non** : tout pipeline qui utilise la ressource passe le check |

```
GATES                                   CHECKS
Release A ─► [gate] ─► Prod             Pipeline A ─┐
Release B ─────────────► Prod  (oups)   Pipeline B ─┼─► [check sur Prod] ─► Prod
                                        Pipeline C ─┘
```

Checks propres au YAML : **Business hours**, **Branch control** (seulement `main`), **Exclusive lock** (un déploiement à la fois), **Required template**.

Pas de check post-déploiement : on ajoute des **steps de vérification** en fin de stage Deploy, ou des checks sur le **stage suivant**.

> **Gates** = Classic, sur le **stage**, avant et après.
> **Checks** = YAML, sur la **ressource**, avant, impossibles à contourner depuis le YAML.

---

## 6. Déployer une app .NET + Azure SQL (checklist EventApp)

### A. Code dans Azure Repos

```bash
dotnet new gitignore        # pas de bin/ obj/ .vs/ dans git
git init -b main
git add .
git commit -m "Premier commit"
git remote add origin <url Azure Repos>
git push -u origin main
```

### B. Ressources Azure

- [ ] **App Service** : Code, .NET 10, **Linux**, plan **Free F1**
- [ ] **Azure SQL** : bouton **« Commencer gratuitement »** (Azure SQL hub) → vérifier « Offre gratuite appliquée » et coût = 0
  - limite gratuite atteinte → **auto-pause jusqu'au mois suivant**
  - authentification SQL (noter login / mot de passe)
  - réseau : point de terminaison public, **Autoriser les services Azure = Oui**, **Ajouter l'IP du client = Oui**
- [ ] Chaîne de connexion dans l'App Service : *Variables d'environnement → Chaînes de connexion*, nom **`LivreDor`** (même nom que dans `appsettings.json`), type SQLAzure

**Pourquoi pas dans `appsettings.json` ?** Pas de mot de passe dans git, et la valeur d'Azure **remplace** celle du fichier : LocalDB en local, Azure SQL en ligne, sans changer le code.

### C. Service connection

*Project settings → Service connections → Azure Resource Manager → Workload identity federation (automatic)*. Nom : `azure-eventapp`.

### D. Pipeline CI + CD

```yaml
trigger:
  branches:
    include: [main]
pr:
  branches:
    include: [main]

variables:
  buildConfiguration: 'Release'
  solution: 'EventApp.slnx'

stages:
  - stage: Build
    displayName: 'CI - Compilation et tests'
    jobs:
      - job: Build
        pool:
          vmImage: 'ubuntu-latest'
        steps:
          - task: UseDotNet@2
            inputs:
              packageType: 'sdk'
              version: '10.0.x'
          - script: dotnet restore $(solution)
          - script: dotnet build $(solution) --configuration $(buildConfiguration) --no-restore
          - task: DotNetCoreCLI@2
            inputs:
              command: 'test'
              projects: '**/*Tests.csproj'
              arguments: '--configuration $(buildConfiguration) --no-build'
              publishTestResults: true
          - script: dotnet publish Event.Mvc/Event.Mvc.csproj --configuration $(buildConfiguration) --no-build --output $(Build.ArtifactStagingDirectory)/mvc
          - task: PublishPipelineArtifact@1
            inputs:
              targetPath: '$(Build.ArtifactStagingDirectory)/mvc'
              artifact: 'eventapp-mvc'

  - stage: Deploy
    displayName: 'CD - Déploiement sur Azure'
    dependsOn: Build
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
    jobs:
      - deployment: DeployWeb
        pool:
          vmImage: 'ubuntu-latest'
        environment: 'production'
        strategy:
          runOnce:
            deploy:
              steps:
                - task: AzureWebApp@1
                  inputs:
                    azureSubscription: 'azure-eventapp'
                    appType: 'webAppLinux'
                    appName: 'eventapp-<prenom>'
                    package: '$(Pipeline.Workspace)/eventapp-mvc'
```

### E. Base de données

```bash
dotnet ef database update --project Event.Infrastructure --startup-project Event.Mvc --connection "<chaîne Azure SQL>"
```

### ⚠️ Pièges rencontrés

| Problème | Cause / solution |
|---|---|
| `dotnet test --no-build` ne trouve rien dans le pipeline | Le projet de test n'est **pas dans le `.slnx`** → `dotnet sln EventApp.slnx add Event.Tests/Event.Tests.csproj` |
| Pipeline rouge dès le premier run | Un test échoue : c'est la CI qui fait son travail |
| Run bloqué « en attente » | Cliquer **Permit** (service connection / environment), ou agents gratuits pas encore accordés |
| Premier accès au site très lent ou timeout | Azure SQL serverless se **réveille** (~1 min) : recharger |
| Bouton *Releases → New* absent | Classic désactivé sur l'organisation |
| Erreur YAML | Indentation : **2 espaces**, jamais de tabulation |

---

## 7. Infrastructure as Code : les principes

> Bases et exemple Bicep : voir [README §17](README.md#17-infrastructure-as-code-iac).

| Principe (slide du cours) | Signification |
|---|---|
| **Management of Infrastructure** | L'infra est gérée comme du code : git, historique, pull requests |
| **Descriptive Model** | **Déclaratif** : on décrit le résultat voulu, l'outil calcule les étapes |
| **Generate same Environment Every Time** | Même fichier → même résultat, aucun oubli |
| **Consistently Reproduce Deployment Environments** | Un fichier + des **paramètres** → Test et Prod identiques |
| **Idempotence** | 1 fois ou 10 fois → même résultat |
| **Test Production-Like Environments** | Recréer une copie de la prod pour tester, puis la supprimer |

### Impératif vs déclaratif

| Impératif (les étapes) | Déclaratif (le résultat) |
|---|---|
| « Crée un plan, puis un site, puis relie-les » | « Je veux un plan F1 et un site .NET 10 relié » |
| Je gère l'ordre et les cas particuliers | L'outil compare et corrige la différence |

Image : commander « une margherita » (déclaratif) vs donner la recette au cuisinier (impératif).

### Idempotence

| Lancement | Résultat |
|---|---|
| 1ᵉʳ | La ressource n'existe pas → **création** |
| 2ᵉ, rien changé | Déjà identique → **rien** |
| 3ᵉ, F1 → B1 dans le fichier | **Modification de la différence uniquement** |

Un script impératif « crée un App Service » **plante** ou crée un **doublon** au 2ᵉ lancement.
Image : appuyer 10 fois sur « 3 » dans l'ascenseur → 3ᵉ étage, pas 30ᵉ.

C'est ce qui permet un stage **Infra** à chaque push :

```
Build (CI) ──► Infra (IaC) ──► Deploy (CD)
```

---

## 8. Les outils IaC : deux familles

```
1. PROVISIONING                      2. CONFIGURATION MANAGEMENT
   Créer l'infrastructure              Configurer l'intérieur des serveurs
   (construire la maison)              (aménager la maison)
```

### Provisioning

| Outil | Cloud | Langage | À retenir |
|---|---|---|---|
| **AWS CloudFormation** | AWS | JSON / YAML | Natif AWS |
| **Azure Resource Manager** | Azure | JSON (ou **Bicep**) | Natif Azure ; Bicep est compilé en ARM |
| **Google Cloud Deployment Manager** | Google Cloud | YAML + Jinja/Python | Remplacé par *Infrastructure Manager* (basé sur Terraform) |
| **Terraform** | **Multi-cloud** | HCL | Le plus demandé en entreprise ; garde un *state* |

### Configuration management

| Outil | Langage | Agent ? | Mode |
|---|---|---|---|
| **Chef** | Ruby (recettes) | Oui | Pull |
| **Puppet** | Langage propre | Oui | Pull |
| **SaltStack** | YAML | Oui (*minions*), sans agent possible | Push / pull |
| **Ansible** | YAML (playbooks) | **Non** (SSH / WinRM) | Push |

- **Agent** = programme installé sur chaque serveur pour appliquer la config.
- **Push** = on envoie la config ; **pull** = le serveur vient la chercher.
- Les familles indiquent la **spécialité**, pas une limite : Ansible sait aussi provisionner.
- En **PaaS** (App Service), pas de serveur à configurer → surtout besoin de la famille 1.

---

## 9. Ansible

### Vocabulaire

| Mot | Signification |
|---|---|
| **Inventory** | La liste des serveurs, rangés en groupes |
| **Playbook** | Fichier YAML qui décrit ce qu'il faut faire |
| **Task** | Une action du playbook |
| **Module** | La brique qui agit (`apt`, `service`, `copy`, `azure_rm_*`...) |

```yaml
- name: Préparer les serveurs web
  hosts: webservers
  become: true
  tasks:
    - name: Installer le runtime ASP.NET Core 8
      apt:
        name: aspnetcore-runtime-8.0
        state: present          # déclaratif + idempotent

    - name: Démarrer le service EventApp
      service:
        name: eventapp
        state: started
```

```bash
ansible-playbook -i inventory.ini site.yml
```

### Ce qu'Ansible gère (6 domaines)

| Domaine | Exemple |
|---|---|
| **Configuration Management** | Installer .NET et Nginx sur 50 serveurs |
| **App Deployments** | Copier l'appli publiée et redémarrer le service |
| **Provisioning of Infrastructure** | Créer un resource group / une VM via `azure.azcollection` (Terraform/Bicep restent préférés) |
| **Continuous Delivery** | Appelé depuis le pipeline après le build |
| **Security & Compliance** | Correctifs, pare-feu, interdire root en SSH ; relancer = remise en conformité |
| **Orchestration** | Rolling update : un serveur à la fois, retiré du load balancer, mis à jour, vérifié, remis |

### Inventaires

**Fichier statique (INI) :**

```ini
[web]
my-web[1:50].domain.com            ; range : my-web1 ... my-web50

[db]
dbase-[a:c].mydomain.com            ; dbase-a, dbase-b, dbase-c
dbase1.mydomain.com  ansible_user=admin  ansible_port=2222

[web:vars]
http_port=80

[production:children]
web
db
```

| Élément | Rôle |
|---|---|
| `[groupe]` | Un groupe de serveurs (un serveur peut être dans plusieurs) |
| `all`, `ungrouped` | Groupes toujours présents automatiquement |
| `[1:50]`, `[a:c]` | Ranges (plages) |
| `ansible_user`, `ansible_port` | Variables de connexion par serveur |
| `[groupe:vars]` | Variables pour tout un groupe |
| `[groupe:children]` | Groupe de groupes |

⚠️ Un `---` en tête de fichier est un marqueur **YAML** : pas dans un inventaire INI.

**Équivalent YAML :**

```yaml
all:
  children:
    web:
      hosts:
        my-web1.domain.com:
        my-web2.domain.com:
    db:
      hosts:
        dbase1.mydomain.com:
```

**Inventaire dynamique (cloud) :** les VM apparaissent et disparaissent → Ansible interroge le cloud à chaque exécution. Fichier dont le nom finit par **`.azure_rm.yml`** :

```yaml
plugin: azure.azcollection.azure_rm
include_vm_resource_groups:
  - rg-eventapp
keyed_groups:
  - prefix: tag
    key: tags          # tag Azure role=web → groupe tag_role_web
```

Équivalents : `aws_ec2` (AWS), `gcp_compute` (Google Cloud).

```bash
ansible-inventory -i inventory.ini --graph     # voir les groupes sans rien exécuter
```

> **Inventaire** = sur quelles machines. **Playbook** = quoi faire dessus.

---

## 10. Dev ou DevOps : quoi prioriser ?

DevOps est d'abord une **culture** (Dev et Ops travaillent ensemble), devenue aussi un métier.

| Sujet | Niveau attendu pour un dev .NET |
|---|---|
| git, branches, pull requests | **Maîtriser** |
| Tests (xUnit, Moq) | **Maîtriser** |
| Pipeline CI/CD YAML | **Écrire et corriger le sien** |
| Configuration par environnement, secrets hors git | **Maîtriser** |
| Déployer sur Azure (App Service, Azure SQL) | **Savoir faire** |
| Docker | **Les bases** |
| Bicep / Terraform | **Lire** et faire de petites modifications |
| Ansible, Chef, Puppet, Salt | **Savoir ce que c'est** |
| Kubernetes | **Savoir ce que c'est** |

---

## Auto-évaluation

<details>
<summary>IaaS, PaaS, Serverless : un exemple Azure de chaque ?</summary>

Virtual Machines, App Service, Azure Functions.
</details>

<details>
<summary>Les jobs d'un même stage tournent comment ? Partagent-ils leurs fichiers ?</summary>

En parallèle, chacun sur sa machine. Non : il faut un artefact.
</details>

<details>
<summary>Pourquoi le stage Deploy ne recompile-t-il pas ?</summary>

Pour déployer exactement l'artefact testé (*build once, deploy many*).
</details>

<details>
<summary>À quoi sert une gate ?</summary>

À contrôler automatiquement si un déploiement peut continuer, selon des signaux de santé ou de qualité (alertes, API, work items...).
</details>

<details>
<summary>Différence gate / check ?</summary>

Gate : Classic, sur le stage, avant et après. Check : YAML, sur la ressource, avant, non contournable depuis le YAML.
</details>

<details>
<summary>C'est quoi l'idempotence ?</summary>

Lancer une opération 1 ou 10 fois donne le même résultat : l'outil ne corrige que la différence entre l'état décrit et l'état réel.
</details>

<details>
<summary>Provisioning ou configuration : où ranger Terraform, ARM, Ansible, Puppet ?</summary>

Provisioning : Terraform, ARM. Configuration : Ansible, Puppet (Ansible sait aussi provisionner).
</details>

<details>
<summary>Quel outil de configuration n'a pas besoin d'agent ?</summary>

Ansible (SSH / WinRM).
</details>

<details>
<summary>Pourquoi un inventaire dynamique dans le cloud ?</summary>

Les serveurs sont créés / supprimés automatiquement et changent d'IP : Ansible interroge le cloud pour avoir la liste à jour.
</details>

<details>
<summary>Où mettre la chaîne de connexion de prod d'EventApp ?</summary>

Dans l'App Service (Chaînes de connexion, nom `LivreDor`), jamais dans git.
</details>
