# Publier une application .NET sur Docker Hub avec Azure DevOps

Récapitulatif des étapes pour construire une image Docker d'une application ASP.NET Core (.NET 10)
et la publier automatiquement sur Docker Hub via un pipeline Azure DevOps.

---

## Sommaire

1. [Le Dockerfile](#1-le-dockerfile)
2. [Tester en local](#2-tester-en-local)
3. [Créer un jeton Docker Hub](#3-créer-un-jeton-docker-hub-une-seule-fois)
4. [Créer la service connection](#4-créer-la-service-connection-une-seule-fois-par-projet)
5. [La tâche Docker dans le pipeline](#5-la-tâche-docker-dans-le-pipeline)
6. [Lancer et autoriser le pipeline](#6-lancer-et-autoriser-le-pipeline)
7. [Vérifier](#7-vérifier)
8. [Pièges rencontrés](#8-pièges-rencontrés)
9. [Rappels](#9-rappels)

---

## 1. Le Dockerfile

Emplacement : `devopswebapp.Mvc/Dockerfile`. Le **contexte de build** est la **racine** du dépôt.

```dockerfile
# ---------- BASE : image d'exécution légère ----------
FROM mcr.microsoft.com/dotnet/aspnet:10.0 AS base
WORKDIR /app
EXPOSE 5263
ENV ASPNETCORE_URLS=http://+:5263
# Utilisateur non-root (sécurité)
USER app

# ---------- BUILD : compilation avec le SDK ----------
FROM --platform=$BUILDPLATFORM mcr.microsoft.com/dotnet/sdk:10.0 AS build
ARG configuration=Release
WORKDIR /src
# Copier le .csproj seul d'abord => le restore reste en cache
COPY ["devopswebapp.Mvc/devopswebapp.csproj", "devopswebapp.Mvc/"]
RUN dotnet restore "devopswebapp.Mvc/devopswebapp.csproj"
COPY . .
WORKDIR "/src/devopswebapp.Mvc"
RUN dotnet build "devopswebapp.csproj" -c $configuration -o /app/build

# ---------- PUBLISH ----------
FROM build AS publish
ARG configuration=Release
RUN dotnet publish "devopswebapp.csproj" -c $configuration -o /app/publish /p:UseAppHost=false

# ---------- FINAL : uniquement le résultat publié ----------
FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "devopswebapp.dll"]
```

### Les instructions à retenir

| Instruction  | Rôle                                                        |
|--------------|-------------------------------------------------------------|
| `FROM`       | image de départ (`AS nom` pour nommer une étape)            |
| `WORKDIR`    | dossier de travail dans l'image (comme un `cd`)             |
| `COPY`       | copier depuis le PC, ou depuis une étape avec `--from=`     |
| `RUN`        | exécuter une commande **pendant le build**                  |
| `ENV` / `ARG`| variable d'environnement / paramètre de build               |
| `EXPOSE`     | documenter le port écouté                                   |
| `USER`       | utilisateur qui lance l'app                                 |
| `ENTRYPOINT` | commande lancée **au démarrage du conteneur**               |

**Multi-stage** : on compile dans l'image SDK (~800 Mo), puis on ne garde que le résultat dans l'image
runtime `aspnet` (~220 Mo). Plus léger, et le code source n'est pas dans l'image finale.

### `.dockerignore` (à côté du Dockerfile / à la racine)

```
**/bin
**/obj
**/publish
.vs
.vscode
.git
```

---

## 2. Tester en local

Prérequis : **Docker Desktop** lancé.

Depuis la **racine** du dépôt (le dossier qui contient `devopswebapp.Mvc/`) :

```powershell
docker build -f devopswebapp.Mvc/Dockerfile -t devopswebapp .
```

```powershell
docker run -p 5263:5263 devopswebapp
```

Ouvrir http://localhost:5263 — arrêter avec `Ctrl + C`.

- `-f` = où se trouve le Dockerfile
- `.` = le **contexte** (dossier envoyé à Docker)
- `-p PORT_PC:PORT_CONTENEUR`

> `build` crée l'image (prend le `.`), `run` démarre un conteneur (**pas** de `.`).

---

## 3. Créer un jeton Docker Hub (une seule fois)

On utilise un **Personal Access Token** plutôt que le mot de passe (révocable, plus sûr).

1. https://hub.docker.com → avatar → **Account settings** → **Personal access tokens**
2. **Generate new token**
   - Description : `azure-devops-pipeline`
   - Expiration : durée de la formation
   - Permissions : **Read & Write**
3. **Copier le jeton immédiatement** (il ne sera plus réaffiché).

> Ne jamais mettre ce jeton dans le YAML ni dans Git.

---

## 4. Créer la service connection (une seule fois par projet)

Azure DevOps → **Project settings** → **Service connections** → **New service connection** → **Docker Registry**

| Champ                        | Valeur                          |
|------------------------------|---------------------------------|
| Registry type                | **Docker Hub**                  |
| Docker ID                    | identifiant Docker Hub (pas l'e-mail) |
| Docker Password              | le **jeton** de l'étape 3       |
| Service connection name      | `dockerhub-connection`          |
| Grant access to all pipelines| **Non** (moindre privilège)     |

### Réutiliser la connexion dans un autre projet

Projet d'origine → **Service connections** → la connexion → `⋯` → **Security** → **Project permissions** → **+ Add** → choisir le projet.

> ⚠️ Une connexion partagée peut être **renommée** dans le projet cible
> (ex. `dockerhub-connection-DotNetAzureLab`). Utiliser le **nom exact** affiché dans le pipeline.

---

## 5. La tâche Docker dans le pipeline

### Pipeline YAML complet (`azure-pipelines.yml` à la racine)

```yaml
trigger:
  branches:
    include:
      - main

pool:
  vmImage: 'ubuntu-latest'

variables:
  buildConfiguration: 'Release'
  dockerRegistryServiceConnection: 'dockerhub-connection'   # nom EXACT de la connexion
  imageRepository: 'DOCKER_ID/devopswebapp'
  dockerfilePath: 'devopswebapp.Mvc/Dockerfile'
  tag: '$(Build.BuildId)'

stages:

# ---------- STAGE 1 : CI ----------
- stage: Build
  displayName: 'Build & Tests'
  jobs:
  - job: BuildAndTest
    steps:
    - task: UseDotNet@2
      inputs:
        version: '10.x'

    - script: dotnet restore
      displayName: 'Restore'

    - script: dotnet build --configuration $(buildConfiguration) --no-restore
      displayName: 'Build'

    - script: dotnet test --configuration $(buildConfiguration) --no-build --logger trx --results-directory $(Agent.TempDirectory)/TestResults
      displayName: 'Tests'

    - task: PublishTestResults@2
      condition: succeededOrFailed()
      inputs:
        testResultsFormat: 'VSTest'
        testResultsFiles: '$(Agent.TempDirectory)/TestResults/*.trx'
        failTaskOnMissingResultsFile: false

# ---------- STAGE 2 : Continuous Delivery ----------
- stage: Docker
  displayName: 'Docker Build & Push'
  dependsOn: Build
  condition: and(succeeded(), ne(variables['Build.Reason'], 'PullRequest'))
  jobs:
  - job: DockerBuildPush
    steps:
    - task: Docker@2
      inputs:
        containerRegistry: '$(dockerRegistryServiceConnection)'
        repository: '$(imageRepository)'
        command: 'buildAndPush'
        Dockerfile: '$(dockerfilePath)'
        buildContext: '.'
        tags: |
          $(tag)
          latest
```

Points clés :

- Si un test échoue, le stage Docker ne s'exécute pas → **aucune image cassée publiée**.
- `condition: ... ne(Build.Reason, 'PullRequest')` → pas de push d'image pendant une validation de PR.
- `buildContext: '.'` est indispensable car le Dockerfile est dans un sous-dossier.
- Tags : `$(Build.BuildId)` (version unique, traçable) + `latest`.

### Équivalent dans l'éditeur classique

`+` sur *Agent job 1* → tâche **Docker** :

| Champ                | Valeur                          |
|----------------------|---------------------------------|
| Container registry   | la connexion Docker Hub         |
| Container repository | `DOCKER_ID/devopswebapp`        |
| Command              | **buildAndPush**                |
| Dockerfile           | `devopswebapp.Mvc/Dockerfile`   |
| Build context        | `$(Build.SourcesDirectory)`     |
| Tags                 | `$(Build.BuildId)` et `latest`  |

> Astuce : dans l'éditeur YAML, **Show assistant** → chercher la tâche → remplir le formulaire → **Add** génère le YAML.

---

## 6. Lancer et autoriser le pipeline

1. **Run** (YAML) ou **Save & queue** (classique).
2. Premier lancement : bandeau *« This pipeline needs permission to access a resource »*
   → **View** → **Permit** → **Permit**.
   (Conséquence normale de « Grant access to all pipelines = Non ». À faire une fois par pipeline.)

---

## 7. Vérifier

- Azure DevOps : tous les stages **verts**.
- https://hub.docker.com → **Repositories** → `DOCKER_ID/devopswebapp` avec les tags `latest` et le n° de build.
- Tester l'image publiée :

```powershell
docker run -p 5263:5263 DOCKER_ID/devopswebapp:latest
```

---

## 8. Pièges rencontrés

| Erreur | Cause | Solution |
|--------|-------|----------|
| `Unable to find image 'devopswebapp:latest' locally` / `pull access denied` | `docker run` sans avoir fait `docker build` | faire `docker build` d'abord |
| `failed to read dockerfile: open Dockerfile: no such file or directory` | Dockerfile dans un sous-dossier | `docker build -f devopswebapp.Mvc/Dockerfile -t devopswebapp .` |
| `service connection ... could not be found` | nom différent (connexion partagée renommée) ou non partagée avec ce projet | vérifier **Project settings → Service connections**, utiliser le nom exact |
| Pipeline en pause « Permission needed » | accès non accordé à tous les pipelines | **View → Permit** |
| Tâches *Visual Studio build* qui échouent | modèle classique « ASP.NET Core (.NET Framework) » + agent macOS | utiliser le modèle **ASP.NET Core** (tâches `dotnet`) sur `ubuntu-latest` |

---

## 9. Rappels

**`git push` ≠ `docker push`**

| Commande      | Envoie            | Vers          |
|---------------|-------------------|---------------|
| `git push`    | le code source    | Azure Repos   |
| `docker push` | une image Docker  | Docker Hub    |

Avec le pipeline : **moi → `git push`** (via PR), **le pipeline → `docker push`**.

**CI / CD**

```
git push → [ Build & Tests ] → [ Docker Build & Push ] → [ Deploy sur Azure ]
                 CI               Continuous Delivery      Continuous Deployment
```

**Coûts**

- Docker Hub : gratuit (dépôts publics, 1 dépôt privé).
- Azure Container Registry : payant (~5 $/mois en Basic) → à éviter avec le crédit Azure for Students.
- Les agents Microsoft d'Azure DevOps ne consomment pas le crédit Azure.
