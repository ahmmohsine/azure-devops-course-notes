# Aide-mémoire YAML – Azure Pipelines

Tout ce qu'il faut pour lire et écrire un pipeline `azure-pipelines.yml`.

## Sommaire

1. [La syntaxe YAML en 5 règles](#1-la-syntaxe-yaml-en-5-règles)
2. [La structure d'un pipeline](#2-la-structure-dun-pipeline)
3. [trigger – Quand ?](#3-trigger--quand-)
4. [pool – Où ?](#4-pool--où-)
5. [variables – Avec quelles valeurs ?](#5-variables--avec-quelles-valeurs-)
6. [stages – Les phases](#6-stages--les-phases)
7. [jobs – Les machines](#7-jobs--les-machines)
8. [steps – Les actions (script / task)](#8-steps--les-actions-script--task)
9. [Conditions et dépendances](#9-conditions-et-dépendances)
10. [Tâches les plus utilisées](#10-tâches-les-plus-utilisées)
11. [Modèles prêts à l'emploi](#11-modèles-prêts-à-lemploi)
12. [Erreurs fréquentes](#12-erreurs-fréquentes)
13. [Méthode de travail](#13-méthode-de-travail)

---

## 1. La syntaxe YAML en 5 règles

### Règle 1 – `clé: valeur` (toujours un espace après `:`)

```yaml
name: devopswebapp
version: 10
```

❌ `script:dotnet build` → ✅ `script: dotnet build`

### Règle 2 – L'indentation = la hiérarchie

```yaml
pool:
  vmImage: 'ubuntu-latest'     # appartient à pool
```

⚠️ **Espaces uniquement, jamais de tabulation.** 2 espaces par niveau.

### Règle 3 – Le tiret `-` = un élément de liste

```yaml
branches:
  - main
  - development
```

### Règle 4 – Liste d'objets : les lignes d'un même élément s'alignent sous le 1er mot après le tiret

```yaml
steps:
  - script: dotnet build       # élément 1
    displayName: 'Build'       # même élément (aligné sous "script")
  - script: dotnet test        # élément 2 (nouveau tiret)
    displayName: 'Tests'
```

❌ `displayName` décalé d'un espace de trop → erreur.

### Règle 5 – Texte multi-lignes

| Symbole | Retours à la ligne | Usage |
|---------|-------------------|-------|
| `\|` | **conservés** | scripts multi-lignes, listes de tags |
| `>` | remplacés par des espaces | longues phrases |

```yaml
- script: |
    dotnet restore
    dotnet build -c Release
    dotnet test
  displayName: 'Build et tests'
```

Le bloc s'arrête dès qu'une ligne revient au niveau de la clé (`displayName`).

### Autres éléments utiles

```yaml
# Ceci est un commentaire
nom: 'texte entre guillemets'     # conseillé si la valeur contient : # * { } [ ]
actif: true                       # booléen
liste_courte: [main, develop]     # liste sur une ligne
objet_court: { vmImage: 'ubuntu-latest' }
```

---

## 2. La structure d'un pipeline

```
pipeline           → le fichier entier
 └── stages        → les phases          (l'un après l'autre)
      └── jobs     → une machine chacun  (en parallèle par défaut)
           └── steps → les actions       (l'un après l'autre)
```

### Le squelette à retenir

```yaml
trigger:            # QUAND ?
  - main

pool:               # OÙ ?
  vmImage: 'ubuntu-latest'

variables:          # AVEC QUELLES VALEURS ?
  config: 'Release'

stages:             # QUOI ?
  - stage: Build
    jobs:
      - job: BuildJob
        steps:
          - script: dotnet build -c $(config)
```

### Forme courte (un seul job)

Sans `stages` ni `jobs` : Azure DevOps les crée automatiquement.

```yaml
trigger:
  - main
pool:
  vmImage: 'ubuntu-latest'
steps:
  - script: dotnet build
```

### Analogie du chantier

| Pipeline | Chantier |
|----------|----------|
| stage | une phase : fondations → murs → toit |
| job | un ouvrier avec ses propres outils |
| step | une tâche de l'ouvrier |

### Comment découper ?

- Nouvelle **phase** (qui pourrait demander une validation) ? → nouveau **stage**
- Autre **machine** ou exécution **en parallèle** ? → nouveau **job**
- Juste une **commande** de plus ? → nouveau **step**

---

## 3. `trigger` – Quand ?

```yaml
# Forme simple
trigger:
  - main
  - feature/*            # joker

# Forme complète
trigger:
  branches:
    include:
      - main
      - release/*
    exclude:
      - release/old-*
  paths:
    include:
      - devopswebapp.Mvc/*
    exclude:
      - '*.md'           # modifier la doc ne relance pas le pipeline

# Désactiver (lancement manuel uniquement)
trigger: none
```

⚠️ **Sans `trigger`**, le pipeline se lance sur **toutes les branches** à chaque push.

### Pull Requests

```yaml
pr:
  - main
```

⚠️ **Ignoré avec Azure Repos.** Utiliser **Repos → Branches → main → Branch policies → Build Validation**.
(`pr:` fonctionne avec GitHub et Bitbucket.)

### Planification (cron, en UTC)

```yaml
schedules:
  - cron: '0 2 * * 1-5'        # minute heure jour mois jour-semaine
    displayName: 'Build de nuit'
    branches:
      include:
        - main
    always: false              # seulement s'il y a eu des changements
```

| Cron | Signification |
|------|---------------|
| `0 2 * * 1-5` | 02:00 UTC, lundi → vendredi |
| `0 */6 * * *` | toutes les 6 heures |
| `30 8 * * 1` | lundi à 08:30 UTC |

### Après un autre pipeline

```yaml
resources:
  pipelines:
    - pipeline: buildPipeline
      source: 'NomDuPipelineSource'
      trigger: true
```

---

## 4. `pool` – Où ?

```yaml
# Agents Microsoft (machine neuve à chaque job)
pool:
  vmImage: 'ubuntu-latest'

# Agents auto-hébergés (ma machine / un serveur)
pool:
  name: 'MonPoolPerso'
```

| `vmImage` | Quand l'utiliser |
|-----------|------------------|
| `ubuntu-latest` | par défaut : .NET, Angular, Node, **Docker** |
| `windows-latest` | **WPF**, .NET Framework, tâches Visual Studio |
| `macos-latest` | iOS / macOS |

| | Microsoft-hosted | Self-hosted |
|---|---|---|
| Machine | fournie, neuve à chaque fois | la mienne |
| Entretien | aucun | à ma charge |
| Coût | minutes gratuites limitées | gratuit, illimité |
| Réseau interne | ❌ | ✅ |

Le `pool` peut être défini au niveau **pipeline**, **stage** ou **job** (le plus proche gagne).

---

## 5. `variables` – Avec quelles valeurs ?

```yaml
variables:
  buildConfiguration: 'Release'

steps:
  - script: dotnet build -c $(buildConfiguration)
```

### Portée

```yaml
variables:           # pipeline → partout
  config: 'Release'
stages:
  - stage: Build
    variables:       # stage → ce stage uniquement
      dossier: 'src'
    jobs:
      - job: BuildJob
        variables:   # job → ce job uniquement
          verbose: 'true'
```

### Avec un Variable Group (Pipelines → Library)

```yaml
variables:
  - group: 'mes-secrets'
  - name: buildConfiguration      # forme name/value obligatoire quand on utilise group
    value: 'Release'
```

### Variables prédéfinies utiles

| Variable | Contenu |
|----------|---------|
| `$(Build.BuildId)` | numéro unique du build (tag d'image) |
| `$(Build.Reason)` | `Manual`, `IndividualCI`, `PullRequest`, `Schedule`… |
| `$(Build.SourceBranchName)` | nom de la branche (`main`) |
| `$(Build.SourcesDirectory)` | dossier du code récupéré |
| `$(Build.ArtifactStagingDirectory)` | dossier pour préparer les artefacts |
| `$(Agent.TempDirectory)` | dossier temporaire |
| `$(Pipeline.Workspace)` | dossier de travail (artefacts téléchargés) |

### Les 3 syntaxes

| Syntaxe | Remplacée | Usage |
|---------|-----------|-------|
| **`$(var)`** | avant chaque step | **95 % des cas** |
| `${{ variables.var }}` | au chargement du YAML | conditions / templates |
| `$[variables.var]` | à l'exécution, dans une expression | conditions dynamiques |

### Secrets

- ❌ **Jamais** dans le YAML (il est dans Git).
- ✅ Pipeline → **Variables** → 🔒 *Keep this value secret*
- ✅ **Library → Variable group** (éventuellement lié à **Azure Key Vault**)
- Les secrets ne sont pas transmis automatiquement aux scripts :

```yaml
- script: dotnet run
  env:
    API_KEY: $(monSecret)
```

### Dans un script : variables d'environnement

Majuscules, `.` remplacé par `_` :

```yaml
- pwsh: Write-Host "Build n° $env:BUILD_BUILDID en $env:BUILDCONFIGURATION"
```

---

## 6. `stages` – Les phases

```yaml
stages:
  - stage: Build
    displayName: 'Build & Tests'
    jobs: [...]

  - stage: Docker
    displayName: 'Docker Build & Push'
    dependsOn: Build
    condition: and(succeeded(), ne(variables['Build.Reason'], 'PullRequest'))
    jobs: [...]

  - stage: DeployProd
    dependsOn: Docker
    jobs: [...]
```

- Exécution **l'un après l'autre** par défaut.
- Si un stage échoue → les suivants ne s'exécutent pas.
- Visibles comme des **boîtes** dans l'interface ; on peut relancer uniquement celui en échec.
- Points de contrôle : **conditions**, **approbations**, **environnements**.

---

## 7. `jobs` – Les machines

```yaml
jobs:
  - job: Linux
    pool:
      vmImage: 'ubuntu-latest'
    steps:
      - script: dotnet test

  - job: Windows
    dependsOn: Linux          # sans cette ligne : en parallèle
    pool:
      vmImage: 'windows-latest'
    steps:
      - script: dotnet test
```

- **1 job = 1 agent = 1 machine neuve.**
- Les steps d'un même job partagent les fichiers.
- ⚠️ Deux jobs **ne partagent pas** leurs fichiers → passer par un **artefact** :

```yaml
# Job A : publier
- task: PublishPipelineArtifact@1
  inputs:
    targetPath: '$(Build.ArtifactStagingDirectory)/app'
    artifact: 'webapp'

# Job B : récupérer
- download: current
  artifact: webapp            # disponible dans $(Pipeline.Workspace)/webapp
```

### Job de déploiement (avec environnement et approbation)

```yaml
jobs:
  - deployment: DeployWeb
    environment: 'production'      # Pipelines → Environments (approbations possibles)
    strategy:
      runOnce:
        deploy:
          steps:
            - script: echo "Déploiement"
```

---

## 8. `steps` – Les actions (script / task)

`steps` = **la liste**. Chaque `-` = **une étape**, qui peut être de plusieurs types :

| Type | Rôle | Exemple |
|------|------|---------|
| `script` | une commande que j'écris | `- script: dotnet test` |
| `pwsh` / `bash` | idem, en forçant PowerShell / Bash | `- pwsh: Get-ChildItem` |
| `task` | un outil tout fait d'Azure DevOps, réglé via `inputs` | `- task: Docker@2` |
| `checkout` | récupérer le code (automatique) | `- checkout: self` |
| `download` | récupérer un artefact | `- download: current` |

### Même chose, deux façons

```yaml
# script : j'écris la commande
- script: dotnet build -c Release
  displayName: 'Build'

# task : je règle les paramètres de l'outil
- task: DotNetCoreCLI@2
  displayName: 'Build'
  inputs:
    command: 'build'
    arguments: '-c Release'
```

- **`script`** → simple et lisible (dotnet, npm, git…).
- **`task`** → indispensable avec une **service connection** (Docker Hub, Azure) ou une action spécifique (résultats de tests, artefacts).
- `@2` = version de la tâche.

### Propriétés communes à tous les steps

```yaml
- script: dotnet test
  displayName: 'Tests'              # nom affiché
  condition: succeededOrFailed()    # quand l'exécuter
  continueOnError: true             # ne pas faire échouer le job
  timeoutInMinutes: 10
  workingDirectory: 'devopswebapp.Mvc'
  env:
    MA_VAR: 'valeur'
```

---

## 9. Conditions et dépendances

```yaml
dependsOn: Build                       # attendre un stage / job
dependsOn: [Build, Tests]              # attendre plusieurs
dependsOn: []                          # ne rien attendre (parallèle)
```

| Condition | S'exécute… |
|-----------|-----------|
| `succeeded()` | si tout ce qui précède a réussi (**défaut**) |
| `failed()` | seulement si quelque chose a échoué |
| `succeededOrFailed()` | dans tous les cas sauf annulation |
| `always()` | toujours, même si annulé |

```yaml
# Seulement sur main
condition: and(succeeded(), eq(variables['Build.SourceBranchName'], 'main'))

# Jamais pendant une Pull Request
condition: and(succeeded(), ne(variables['Build.Reason'], 'PullRequest'))
```

Opérateurs : `and`, `or`, `not`, `eq` (égal), `ne` (différent), `contains`, `startsWith`.

---

## 10. Tâches les plus utilisées

```yaml
# Installer une version de .NET
- task: UseDotNet@2
  inputs:
    version: '10.x'

# Commandes .NET
- task: DotNetCoreCLI@2
  inputs:
    command: 'test'                 # restore | build | test | publish
    projects: '**/*Tests.csproj'
    arguments: '-c Release'

# Publier les résultats de tests (onglet Tests)
- task: PublishTestResults@2
  condition: succeededOrFailed()
  inputs:
    testResultsFormat: 'VSTest'
    testResultsFiles: '**/*.trx'

# Publier / récupérer un artefact
- task: PublishPipelineArtifact@1
  inputs:
    targetPath: '$(Build.ArtifactStagingDirectory)'
    artifact: 'drop'

# Docker : build + push
- task: Docker@2
  inputs:
    containerRegistry: 'dockerhub-connection'
    repository: 'DOCKER_ID/devopswebapp'
    command: 'buildAndPush'
    Dockerfile: 'devopswebapp.Mvc/Dockerfile'
    buildContext: '.'
    tags: |
      $(Build.BuildId)
      latest

# Déployer sur Azure App Service (code)
- task: AzureWebApp@1
  inputs:
    azureSubscription: 'azure-students'
    appType: 'webAppLinux'
    appName: 'nom-de-la-webapp'
    package: '$(Pipeline.Workspace)/webapp'
    runtimeStack: 'DOTNETCORE|10.0'
```

> Astuce : éditeur de pipeline → **Show assistant** → chercher la tâche → remplir le formulaire → **Add** génère le YAML.

---

## 11. Modèles prêts à l'emploi

### A. .NET – build + tests

```yaml
trigger:
  - main

pool:
  vmImage: 'ubuntu-latest'

variables:
  buildConfiguration: 'Release'

steps:
  - task: UseDotNet@2
    inputs:
      version: '10.x'
  - script: |
      dotnet restore
      dotnet build -c $(buildConfiguration) --no-restore
      dotnet test -c $(buildConfiguration) --no-build
    displayName: 'Restore, build, tests'
```

### B. WPF – build + artefact (.exe)

```yaml
trigger:
  - main

pool:
  vmImage: 'windows-latest'          # WPF = Windows obligatoire

steps:
  - task: UseDotNet@2
    inputs:
      version: '10.x'
  - script: dotnet publish MonAppWpf/MonAppWpf.csproj -c Release -r win-x64 --self-contained true -p:PublishSingleFile=true -o $(Build.ArtifactStagingDirectory)/wpf
    displayName: 'Publish WPF'
  - task: PublishPipelineArtifact@1
    inputs:
      targetPath: '$(Build.ArtifactStagingDirectory)/wpf'
      artifact: 'wpf-app'
```

### C. Web App – build + déploiement Azure

```yaml
trigger:
  - main

pool:
  vmImage: 'ubuntu-latest'

variables:
  project: 'devopswebapp.Mvc/devopswebapp.csproj'

stages:
  - stage: Build
    jobs:
      - job: Build
        steps:
          - task: UseDotNet@2
            inputs:
              version: '10.x'
          - script: dotnet publish $(project) -c Release -o $(Build.ArtifactStagingDirectory)/app
            displayName: 'Publish'
          - task: PublishPipelineArtifact@1
            inputs:
              targetPath: '$(Build.ArtifactStagingDirectory)/app'
              artifact: 'webapp'

  - stage: Deploy
    dependsOn: Build
    jobs:
      - job: Deploy
        steps:
          - download: current
            artifact: webapp
          - task: AzureWebApp@1
            inputs:
              azureSubscription: 'azure-students'
              appType: 'webAppLinux'
              appName: 'nom-de-la-webapp'
              package: '$(Pipeline.Workspace)/webapp'
              runtimeStack: 'DOTNETCORE|10.0'
```

### D. Docker – voir `DOCKER.md`

---

## 12. Erreurs fréquentes

| Symptôme | Cause probable | Correction |
|----------|----------------|------------|
| `Mapping values are not allowed here` / `did not find expected key` | indentation incorrecte ou tabulation | espaces uniquement, aligner sous le 1er mot après `-` |
| La clé est lue comme du texte | pas d'espace après `:` | `script: dotnet build` |
| Le pipeline se lance sur toutes les branches | pas de `trigger` | ajouter `trigger: [main]` |
| Le pipeline ne se lance pas sur les PR | `pr:` ignoré avec Azure Repos | Branch policy → Build Validation |
| `service connection ... could not be found` | nom différent / non partagée | vérifier le nom exact dans Project settings |
| « Permission needed » | accès non accordé à ce pipeline | **View → Permit** |
| Fichier introuvable dans le 2ᵉ job | les jobs ne partagent pas leurs fichiers | `PublishPipelineArtifact` + `download` |
| Tâche Visual Studio qui échoue | agent Linux/macOS | `windows-latest` |
| Secret visible dans Git | secret écrit dans le YAML | variable secrète / Variable group / Key Vault |

---

## 13. Méthode de travail

1. **Ne jamais partir d'une page blanche** : reprendre un modèle qui fonctionne.
2. **Lire de haut en bas avec 3 questions** : **Quand ?** (`trigger`) → **Où ?** (`pool`) → **Quoi ?** (`stages` → `jobs` → `steps`).
3. **Utiliser l'assistant** (*Show assistant*) pour générer les tâches.
4. **Extension VS Code « Azure Pipelines »** : autocomplétion + erreurs soulignées.
5. **Avancer par petits pas** : une étape → Run → vert → étape suivante.

---

## Exercices

### Exercice 1 – Trouver les 3 erreurs

```yaml
trigger:
  - main
pool:
vmImage: 'ubuntu-latest'
steps:
  - script:dotnet build
  - script: dotnet test
     displayName: 'Tests'
```

<details>
<summary>Solution</summary>

1. `vmImage` doit être indenté sous `pool:`.
2. Il manque l'espace après `script:`.
3. `displayName` a un espace de trop : il doit être aligné sous `script`.

```yaml
trigger:
  - main
pool:
  vmImage: 'ubuntu-latest'
steps:
  - script: dotnet build
  - script: dotnet test
    displayName: 'Tests'
```
</details>

### Exercice 2 – Écrire un pipeline

Qui : se lance sur `main`, tourne sur `windows-latest`, définit une variable `config = Release`,
et fait `dotnet build` avec cette variable, affiché « Compilation ».

<details>
<summary>Solution</summary>

```yaml
trigger:
  - main

pool:
  vmImage: 'windows-latest'

variables:
  config: 'Release'

steps:
  - script: dotnet build -c $(config)
    displayName: 'Compilation'
```
</details>
