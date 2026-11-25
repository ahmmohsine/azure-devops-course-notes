# CI/CD Pipelines using Azure DevOps — Notes de cours

> Support de cours personnel, basé sur le cours Udemy *CI/CD Pipelines using Azure DevOps*.
> Les termes techniques sont gardés en anglais, les explications sont en français.

---

## Sommaire

- [L'essentiel en 10 points](#lessentiel-en-10-points)
- [1. Qu'est-ce que DevOps ?](#1-quest-ce-que-devops-)
- [2. Le processus DevOps (la boucle ∞)](#2-le-processus-devops-la-boucle-)
- [3. Continuous Integration (CI)](#3-continuous-integration-ci)
- [4. Continuous Delivery vs Continuous Deployment (CD)](#4-continuous-delivery-vs-continuous-deployment-cd)
- [5. Compétences requises en DevOps](#5-compétences-requises-en-devops)
- [6. Outils utilisés dans le cours](#6-outils-utilisés-dans-le-cours)
- [7. Outils de test (Testing Tools)](#7-outils-de-test-testing-tools)
- [8. DevSecOps : la sécurité dans DevOps](#8-devsecops--la-sécurité-dans-devops)
- [9. La plateforme Azure DevOps](#9-la-plateforme-azure-devops)
- [10. Azure Repos](#10-azure-repos)
- [11. Architecture DevOps du cours](#11-architecture-devops-du-cours)
- [12. Créer un pipeline CI/CD : étapes et exemples](#12-créer-un-pipeline-cicd--étapes-et-exemples)
- [13. Concepts liés](#13-concepts-liés)
- [14. Stratégies de branches Git](#14-stratégies-de-branches-git)
- [15. Dockerfile pour une application .NET](#15-dockerfile-pour-une-application-net)
- [16. Variables et secrets dans les pipelines](#16-variables-et-secrets-dans-les-pipelines)
- [17. Infrastructure as Code (IaC)](#17-infrastructure-as-code-iac)
- [18. Monitoring : surveiller l'application en production](#18-monitoring--surveiller-lapplication-en-production)
- [19. GitHub Actions : l'équivalent sur GitHub](#19-github-actions--léquivalent-sur-github)
- [20. Dépannage : erreurs fréquentes et solutions](#20-dépannage--erreurs-fréquentes-et-solutions)
- [21. Glossaire](#21-glossaire)
- [22. Questions d'entretien fréquentes](#22-questions-dentretien-fréquentes)
- [23. Exercice de formation : application Palindrome avec CI](#23-exercice-de-formation--application-palindrome-avec-ci)
- [Ma progression dans le cours](#ma-progression-dans-le-cours)

---

## L'essentiel en 10 points

1. **DevOps** = culture qui réunit Dev et Ops pour livrer plus vite, plus souvent, avec moins de risques.
2. La boucle : **Plan → Build → Integration → Deploy → Operate → Feedback**, en continu.
3. **CI** : chaque push est compilé et testé automatiquement ; un test rouge bloque tout.
4. **CD** : *Delivery* = prêt à déployer avec validation humaine ; *Deployment* = déployé automatiquement.
5. **Tests** : beaucoup de tests unitaires (F.I.R.S.T., AAA), moins de tests d'intégration, peu de tests UI.
6. **DevSecOps** : la sécurité à chaque étape (*shift left*), secrets dans Key Vault, jamais dans le code.
7. **Azure DevOps** = Boards, Repos, Pipelines, Test Plans, Artifacts.
8. Chemin du code : **commit → push → pipeline → image Docker → ACR → App Service**.
9. Pipelines en **YAML multi-stage**, versionnés avec le code, protégés par des **approvals et gates**.
10. **Operate** : surveiller avec Application Insights, logs structurés et `/health`.

---

## 1. Qu'est-ce que DevOps ?

**DevOps** = **Dev**elopment + **Op**eration**s**.

C'est une **culture** et un ensemble de **pratiques** qui rapprochent deux équipes historiquement séparées :

| Avant (silos) | Avec DevOps |
|---|---|
| Les développeurs écrivent le code puis le « jettent par-dessus le mur » aux Ops | Dev et Ops travaillent ensemble du début à la fin |
| Déploiements rares, manuels et risqués | Déploiements fréquents, automatisés et petits |
| Les bugs sont découverts tard, en production | Les bugs sont détectés tôt grâce aux tests automatiques |
| « Ça marche sur ma machine » | Même environnement partout (conteneurs, configuration versionnée) |

**Objectif :** livrer de la valeur **plus vite**, **plus souvent** et **avec moins de risques**.

> DevOps n'est pas un outil ni un poste : c'est une façon de travailler. Les outils (Azure DevOps, Docker, etc.) servent à la mettre en pratique.

---

## 2. Le processus DevOps (la boucle ∞)

Le processus est représenté par un **symbole infini** : il ne s'arrête jamais, chaque livraison alimente la suivante.

```
        PLAN ──► BUILD ──► CONTINUOUS ──► INTEGRATION ──► DEPLOY ──► OPERATE
          ▲                                                              │
          └───────────────── CONTINUOUS FEEDBACK ◄──────────────────────┘
                   (le tout entouré de REAL-TIME COMMUNICATION)
```

### Les étapes en détail

| Étape | Ce qu'on y fait (d'après le cours) | Exemple concret (HotelListing.API) |
|---|---|---|
| **Plan** | Requirements and planning, Architecture and design | Rédiger la user story « En tant que client, je veux annuler une réservation » |
| **Build** | Implementation : coding, configuration, unit tests. Source control : code / config check-ins | Coder l'endpoint `DELETE /api/bookings/{id}`, écrire les tests, faire un commit |
| **Continuous (Integration)** | Integrated builds, automated testing | À chaque push, le pipeline lance `dotnet build` et `dotnet test` |
| **Deploy** | Configuration management, artifact management, deployment environments, package, deploy and smoke test | Le pipeline publie l'API sur Azure App Service (Dev → Test → Prod) |
| **Operate** | Run, operate and monitor | Surveiller les logs (Serilog), les performances, la disponibilité |
| **Continuous Feedback** | Reports, bug fix reporting, ticket management | Un bug trouvé dans les logs devient un ticket dans Azure Boards → retour à **Plan** |

### Real-time communication

L'anneau autour de la boucle rappelle que **la communication est permanente** entre tous les acteurs (devs, testeurs, ops, métier), et pas seulement à la fin d'une étape.

---

## 3. Continuous Integration (CI)

**Définition :** chaque modification de code est **intégrée** dans la branche partagée **plusieurs fois par jour**, et chaque intégration est **vérifiée automatiquement** (build + tests).

### Les étapes de la CI

```
Coding and check-in ──► Build & Package ──► Testing ──► Ready to publish ──► (Continuous Deployment)
```

| Étape | Explication |
|---|---|
| **Coding and check-in** | Le développeur écrit son code et le pousse (`git push`) dans le repository |
| **Build & Package** | Le serveur de CI compile le code (`dotnet build`) et crée un paquet (`dotnet publish`) |
| **Testing** | Les tests automatisés sont exécutés (`dotnet test`) |
| **Ready to publish** | Si tout est vert, on obtient un **artifact** prêt à être déployé |

### Pourquoi c'est important

- On détecte les erreurs **tout de suite**, quand elles sont faciles à corriger.
- On évite l'« enfer de l'intégration » (merger des semaines de travail d'un coup).
- La branche principale est **toujours dans un état qui compile et passe les tests**.

> **Règle d'or :** si le build est cassé (rouge), réparer le build devient la priorité de l'équipe.

---

## 4. Continuous Delivery vs Continuous Deployment (CD)

« CD » a **deux sens** qu'on confond souvent :

| | Continuous **Delivery** | Continuous **Deployment** |
|---|---|---|
| Principe | L'application est **toujours prête** à être déployée | L'application est **déployée automatiquement** |
| Mise en production | Un humain **valide** (clic sur un bouton, approbation) | **Aucune intervention** humaine |
| Quand l'utiliser | Contexte réglementé, besoin de validation métier | Équipe mature avec beaucoup de tests automatisés |

### Les étapes du Continuous Deployment

```
Package ──► Configuration ──► Deploy ──► Smoke Test ──► Production
```

| Étape | Explication |
|---|---|
| **Package** | On récupère l'artifact produit par la CI (ex. : un `.zip` de l'API publiée ou une image Docker) |
| **Configuration** | On applique la configuration propre à l'environnement (chaînes de connexion, variables d'environnement, secrets) |
| **Deploy** | On installe l'application sur l'environnement cible (ex. : Azure App Service) |
| **Smoke Test** | Test rapide pour vérifier que l'application **démarre et répond** (ex. : appeler `/health` ou `GET /api/hotels`) |
| **Production** | L'application est en ligne pour les utilisateurs |

> **Smoke test** : le nom vient de l'électronique : on branche l'appareil et on vérifie qu'il ne fume pas. Ce n'est pas un test complet, juste une vérification que « ça tourne ».

### Environnements de déploiement

On déploie généralement par étapes :

```
Dev ──► Test / QA ──► Staging ──► Production
```

Chaque environnement a sa propre configuration. On « promeut » le même artifact d'un environnement à l'autre, sans recompiler.

---

## 5. Compétences requises en DevOps

| Compétence | Exemples | Où je la travaille |
|---|---|---|
| Scripting Linux / Unix / Shell | PowerShell, Bash, Python | Commandes dans les pipelines |
| CI/CD | Jenkins, **Azure DevOps**, GitHub Actions | Ce cours |
| Tests de performance | JMeter, Gatling | — |
| Operations readiness & tests automatisés | Smoke tests, SonarQube, Selenium | Cours TDD / xUnit |
| Conteneurs | **Docker**, Kubernetes | Cours Cloud-Native |
| Outils de monitoring | Azure Monitor, Application Insights, Seq | Cours Cloud-Native (Serilog / Seq) |
| Services cloud | AWS, **Azure** | Cours Azure for .NET Developers |
| Bases de réseau | DNS, ports, HTTP/HTTPS, proxy | Bachelier réseaux & télécoms |

> En tant que **développeuse .NET**, on n'attend pas de moi d'être experte DevOps, mais de savoir **créer et comprendre un pipeline** pour mon application.

---

## 6. Outils utilisés dans le cours

- **Azure DevOps Platform** : la plateforme centrale
- **Visual Studio Code** : environnement de développement
- **Git** : gestion de versions
- **Microsoft Azure Portal** : création des ressources cloud (App Service, Container Registry…)
- **Docker** : conteneurisation de l'application
- **Azure Repos** : hébergement du code
- **Testing tools** : outils de tests automatisés

---

## 7. Outils de test (Testing Tools)

Les tests automatisés sont au cœur de la CI : sans eux, le pipeline compile, mais ne **vérifie** rien.

### Les 6 types de tests automatisés vus dans le cours

| Type | Question à laquelle il répond | Détail |
|---|---|---|
| **Unit Testing** | Chaque méthode fait-elle ce qu'elle doit ? | 7.1 |
| **User Interface Testing** | L'écran réagit-il correctement aux actions de l'utilisateur ? | 7.2 (Selenium) |
| **Code Quality Checks** | Le code est-il propre et maintenable ? | 7.3 |
| **Security Vulnerability Testing** | Y a-t-il des failles dans le code ou les paquets ? | 7.3 et section 8 |
| **Performance Testing** | L'appli tient-elle la charge ? | 7.3 |
| **Integration Testing** | Les composants fonctionnent-ils ensemble ? | 7.3 |

### La pyramide des tests

```
            /\
           /  \        Tests UI / end-to-end (Selenium)  → peu nombreux, lents, fragiles
          /----\
         /      \      Tests d'intégration               → quantité moyenne
        /--------\
       /          \    Tests unitaires                   → très nombreux, rapides
      /____________\
```

Plus on monte, plus les tests sont **lents et coûteux** à maintenir. On en écrit donc **beaucoup en bas** et **peu en haut**.

### 7.1 Tests unitaires (Unit Testing)

Un **test unitaire** vérifie une **petite unité de code** isolée, en général **une méthode** d'une classe, sans base de données, sans réseau, sans fichier.

#### Les principes F.I.R.S.T.

Un bon test unitaire est :

| Lettre | Principe | Explication |
|---|---|---|
| **F** | **Fast** (rapide) | Il s'exécute en quelques millisecondes ; on peut en lancer des milliers à chaque build |
| **I** | **Isolated** (isolé) | Il ne dépend ni des autres tests, ni d'une base de données, ni d'un service externe (on utilise des *mocks*) |
| **R** | **Repeatable** (répétable) | Il donne **toujours le même résultat**, sur n'importe quelle machine, à n'importe quel moment |
| **S** | **Self-checking** (auto-vérifiant) | Il dit lui-même s'il réussit ou échoue (assertions) ; pas besoin de lire une sortie à la main |
| **T** | **Timely** (écrit au bon moment) | Il est écrit **en même temps** que le code (ou avant, en TDD), pas des semaines après |

#### Les frameworks de test en .NET

| Framework | Attribut d'un test | Particularités |
|---|---|---|
| **xUnit** | `[Fact]`, `[Theory]` | Le plus utilisé dans les projets .NET modernes ; utilisé par Microsoft pour .NET lui-même ; une nouvelle instance de la classe par test (meilleure isolation) |
| **NUnit** | `[Test]`, `[TestCase]` | Très répandu, ancien, riche en fonctionnalités |
| **MSTest** | `[TestMethod]`, `[DataRow]` | Framework de Microsoft, intégré à Visual Studio depuis longtemps |

> Les trois font le même travail. **xUnit** est le choix par défaut aujourd'hui, et c'est celui du cours.

#### Le modèle AAA : Arrange, Act, Assert

Chaque test suit trois étapes :

1. **Arrange** : préparer les données et les objets
2. **Act** : appeler la méthode testée
3. **Assert** : vérifier le résultat

#### Exemple avec xUnit

```csharp
// Code to test
public class BookingPriceCalculator
{
    public decimal CalculateTotal(decimal pricePerNight, int nights)
    {
        if (nights <= 0)
            throw new ArgumentException("Nights must be greater than zero.", nameof(nights));

        return pricePerNight * nights;
    }
}
```

```csharp
// Test project: HotelListing.API.Tests
public class BookingPriceCalculatorTests
{
    [Fact]
    public void CalculateTotal_ValidNights_ReturnsPriceTimesNights()
    {
        // Arrange
        var calculator = new BookingPriceCalculator();

        // Act
        var total = calculator.CalculateTotal(pricePerNight: 100m, nights: 3);

        // Assert
        Assert.Equal(300m, total);
    }

    [Theory]
    [InlineData(0)]
    [InlineData(-1)]
    public void CalculateTotal_InvalidNights_ThrowsArgumentException(int nights)
    {
        var calculator = new BookingPriceCalculator();

        Assert.Throws<ArgumentException>(() => calculator.CalculateTotal(100m, nights));
    }
}
```

- `[Fact]` : un test avec des données fixes.
- `[Theory]` + `[InlineData]` : le même test exécuté avec plusieurs jeux de données.
- Convention de nommage : `MethodName_Scenario_ExpectedResult`.

#### Commandes utiles

```bash
dotnet new xunit -n HotelListing.API.Tests            # create a test project
dotnet add HotelListing.API.Tests reference HotelListing.API
dotnet test                                           # run all tests
dotnet test --collect:"XPlat Code Coverage"           # with code coverage
```

### 7.2 Selenium : tests de pages web

**Selenium** pilote un **vrai navigateur** (Chrome, Firefox, Edge) pour simuler un utilisateur : cliquer, remplir un formulaire, vérifier qu'un texte s'affiche.

- Utile pour les **tests end-to-end** de l'interface (ex. : le front Angular).
- Plus **lent** et plus **fragile** que les tests unitaires (un bouton qui change de place casse le test).
- Alternative moderne souvent citée : **Playwright** (Microsoft).

### 7.3 Autres types de tests

| Type | Ce qu'on vérifie | Outils |
|---|---|---|
| **Code Quality Tests** | Code dupliqué, complexité, mauvaises pratiques, couverture de tests | **SonarQube / SonarCloud**, analyseurs Roslyn intégrés à .NET |
| **Security Vulnerability Checks** | Failles dans le code et dans les paquets NuGet utilisés | `dotnet list package --vulnerable`, Dependabot, outils SAST (lien avec la section DevSecOps) |
| **Integration Testing** | Plusieurs composants ensemble : API + base de données, contrôleur + service | `WebApplicationFactory` (ASP.NET Core), base de test ou conteneur Docker SQL Server |
| **Performance Testing** | Temps de réponse et comportement sous charge (beaucoup d'utilisateurs en même temps) | **JMeter**, **Gatling**, k6, NBomber |

#### Test unitaire vs test d'intégration

| | Test unitaire | Test d'intégration |
|---|---|---|
| Portée | Une méthode, isolée | Plusieurs couches ensemble |
| Dépendances | Remplacées par des *mocks* | Réelles (ou proches du réel) |
| Vitesse | Millisecondes | Secondes |
| Exemple | `CalculateTotal` renvoie 300 | `POST /api/bookings` crée bien une ligne en base |

### 7.4 Les tests dans le pipeline

Dans le pipeline CI, l'étape de test bloque la suite si un test échoue.

> ⚠️ **Piège vécu :** `- script: dotnet test` **exécute** bien les tests (le pipeline devient rouge si un test échoue), mais **ne publie pas** les résultats : l'onglet **Tests** reste vide (« Get started »). Les résultats ne sont visibles que dans le log texte.

**La bonne pratique :** utiliser la tâche `DotNetCoreCLI@2`, qui exécute les tests **et** publie automatiquement les résultats :

```yaml
- task: DotNetCoreCLI@2
  displayName: 'Run tests'
  inputs:
    command: 'test'
    projects: '**/*Tests.csproj'
    arguments: '--configuration $(buildConfiguration) --no-build --collect:"XPlat Code Coverage"'
```

- Test rouge → pipeline rouge → **pas de déploiement**.
- L'onglet **Tests** du pipeline affiche la liste des tests, ceux qui échouent et le message d'erreur exact.
- `--collect:"XPlat Code Coverage"` génère en plus les données de couverture de code.

---

## 8. DevSecOps : la sécurité dans DevOps

### Qu'est-ce que DevSecOps ?

```
Dev (Developers)  +  Sec (Security)  +  Ops (Operations)  =  DevSecOps
```

**Idée clé :** la sécurité n'est plus vérifiée **à la fin**, juste avant la mise en production, par une équipe séparée. Elle est **intégrée à chaque étape** de la boucle DevOps et devient la responsabilité de **tout le monde**.

On parle aussi de **« shift left »** : déplacer la sécurité vers la **gauche** de la boucle, c'est-à-dire le plus **tôt** possible (dès le Plan et le Build). Une faille trouvée pendant le développement coûte beaucoup moins cher qu'une faille trouvée en production.

### Les types de sécurité

| Type | Explication | Exemple concret |
|---|---|---|
| **Cloud environment securing you** | Le fournisseur cloud sécurise l'infrastructure (datacenters, matériel, réseau de base) | Microsoft gère la sécurité physique d'Azure ; moi, je reste responsable de mon code et de ma configuration (*modèle de responsabilité partagée*) |
| **Securing your data** | Protéger les données **au repos** (*at rest* : stockées en base, sur disque) et **en transit** (*in transit* : qui circulent sur le réseau) | Au repos : chiffrement d'Azure SQL. En transit : HTTPS / TLS partout |
| **Securing Azure DevOps access** | Contrôler qui peut voir le code, lancer un pipeline, déployer | Groupes d'utilisateurs, permissions, authentification multifacteur |
| **Security vulnerability checks** | Analyser automatiquement le code pour trouver des failles et des mauvaises pratiques | **SonarCloud** (renommé *SonarQube Cloud*) branché dans le pipeline |
| **Use secret keys, not passwords** | Ne jamais mettre de mot de passe en clair dans le code ou le pipeline | Variables secrètes du pipeline, **Azure Key Vault**, *service connections* |
| **Azure Active Directory** | Service d'identité de Microsoft : qui est l'utilisateur, à quoi a-t-il droit | Aujourd'hui renommé **Microsoft Entra ID**. Sert aussi à sécuriser une API (OAuth 2.0 / JWT) |

> **Data at rest vs data in transit :** une question classique en entretien. *At rest* = la donnée stockée ; *in transit* = la donnée qui voyage. Les deux doivent être chiffrées.

### Sécuriser les pipelines CI/CD

> **« Le pipeline CI/CD est l'environnement de production du DevOps Engineer. »**
> Si quelqu'un prend le contrôle du pipeline, il peut déployer **n'importe quoi** en production. Il faut donc le protéger autant que l'application elle-même.

| Mécanisme | Rôle | Exemple |
|---|---|---|
| **User groups (Allow / Deny)** | Autoriser ou interdire des actions selon le groupe | Les développeurs lancent les builds ; seuls les responsables peuvent déployer en production |
| **Pre-deployment conditions** | Conditions **avant** un déploiement, notamment une **approbation manuelle** | Le chef de projet doit cliquer sur « Approve » avant le déploiement en Production |
| **Post-deployment conditions** | Conditions **après** un déploiement, avant de passer à l'étape suivante | Approbation du testeur après le déploiement en QA avant de promouvoir vers Production |
| **Pre-deployment gates** | Vérifications **automatiques** avant le déploiement | Aucun bug critique ouvert dans Azure Boards ; analyse SonarCloud réussie |
| **Post-deployment gates** | Vérifications **automatiques** après le déploiement | Aucune alerte dans Azure Monitor pendant 15 minutes ; le smoke test passe |

**Condition vs gate :** une *condition* fait souvent intervenir un **humain** (approbation) ; une *gate* est une **vérification automatique**.

```
Build ──► [Pre-deployment: approbation + gates] ──► Deploy QA ──► [Post-deployment: gates] ──► Deploy Prod
```

> Dans les pipelines YAML récents, ces contrôles se configurent sur les **Environments** d'Azure DevOps (*Approvals and checks*).

---

## 9. La plateforme Azure DevOps

**Azure DevOps** est un **SaaS** (Software as a Service) de Microsoft qui fournit tout le nécessaire pour des **pipelines CI/CD de bout en bout**. Il s'intègre avec les outils DevOps du marché (GitHub, Docker, Jenkins, Slack…).

URL : `https://dev.azure.com/<organisation>`

### Organisation et projets

```
Organisation (ex. : ahlame-dev)
 └── Projet (ex. : HotelListing)
      ├── Boards
      ├── Repos
      ├── Pipelines
      ├── Test Plans
      └── Artifacts
```

### Les 5 services clés

| Service | Rôle | Équivalent / comparaison |
|---|---|---|
| **Azure Boards** | Gestion de projet Agile : user stories, tâches, bugs, sprints, tableaux Kanban | Jira, Trello |
| **Azure Repos** | Dépôts Git privés hébergés dans le cloud | GitHub, GitLab |
| **Azure Pipelines** | Automatisation du build, des tests et des déploiements (CI/CD) | GitHub Actions, Jenkins |
| **Azure Test Plans** | Gestion des tests manuels et exploratoires, campagnes de test | TestRail |
| **Azure Artifacts** | Hébergement de paquets (NuGet, npm, Maven…) privés pour l'équipe | NuGet.org privé |

> On peut utiliser **un seul** de ces services ou **tous**. Par exemple : code sur GitHub + pipeline dans Azure Pipelines.

---

## 10. Azure Repos

### Ce que c'est

- Un **repository Git hébergé dans le cloud** sur Azure DevOps.
- On s'y **connecte depuis son Git local**.
- Un push peut **déclencher automatiquement un pipeline**.
- On travaille sur **les mêmes branches** en local et dans le cloud grâce au **clonage**.
- **Visual Studio Code** (et Visual Studio) s'intègre avec Git et Azure Repos.

### Commandes Git essentielles

```bash
# Cloner un repo Azure Repos existant
git clone https://dev.azure.com/<organisation>/<projet>/_git/<repo>

# OU connecter un projet local existant à Azure Repos
git init
git remote add origin https://dev.azure.com/<organisation>/<projet>/_git/<repo>
git push -u origin main

# Travail quotidien
git status                     # voir les fichiers modifiés
git add .                      # préparer les modifications
git commit -m "Add booking cancellation endpoint"
git push                       # envoyer vers Azure Repos → déclenche le pipeline

# Branches
git checkout -b feature/cancel-booking   # créer une branche et s'y placer
git push -u origin feature/cancel-booking
```

### Bonnes pratiques

- Ne jamais travailler directement sur `main` : créer une **branche de feature**.
- Fusionner via une **Pull Request** (relecture de code par un collègue).
- Protéger `main` avec des **branch policies** : PR obligatoire, build réussi obligatoire.
- **Ne jamais committer de secrets** (mots de passe, chaînes de connexion) : utiliser User Secrets en local et Azure Key Vault en production.

---

## 11. Architecture DevOps du cours

Voici le chemin complet que suit le code, du PC du développeur jusqu'à l'application en ligne :

```
[ Windows PC ]
   VS Code ──commit──► Git
                        │ push
                        ▼
[ Azure DevOps ]
   Azure Repos ──trigger──► Azure Pipeline
                                │ push image
                                ▼
[ Azure Portal ]
   Azure Container Registry (Docker image)
        │ webhook                ▲
        ▼                        │ docker pull
   Azure App Service (Web App Container)
```

### Les étapes, une par une

| # | Étape | Où | Ce qui se passe |
|---|---|---|---|
| 1 | **Commit** | PC (VS Code + Git) | Le développeur enregistre ses modifications en local |
| 2 | **Push** | PC → Azure Repos | Le code est envoyé dans le repository cloud |
| 3 | **Trigger** | Azure Repos → Azure Pipelines | Le push **déclenche automatiquement** le pipeline |
| 4 | **Build + Push image** | Azure Pipelines → Container Registry | Le pipeline compile, teste, construit une **image Docker** et l'envoie dans **Azure Container Registry (ACR)** |
| 5 | **Webhook** | Container Registry → App Service | ACR **prévient** App Service qu'une nouvelle image est disponible |
| 6 | **Docker pull** | App Service ← Container Registry | App Service **télécharge** la nouvelle image et redémarre le conteneur |
| 7 | **En ligne** | Azure App Service (Web App for Containers) | La nouvelle version de l'application est accessible |

### Les composants

| Composant | Rôle |
|---|---|
| **Visual Studio Code** | Éditeur de code |
| **Git** | Gestion de versions locale |
| **Azure Repos** | Repository Git dans le cloud |
| **Azure Pipelines** | Automatisation : build, tests, création de l'image Docker |
| **Docker image** | Application + toutes ses dépendances emballées ensemble |
| **Azure Container Registry (ACR)** | « Entrepôt » privé d'images Docker (équivalent privé de Docker Hub) |
| **Webhook** | Notification automatique HTTP envoyée quand un événement se produit (ici : nouvelle image) |
| **Azure App Service – Web App for Containers** | Service qui exécute le conteneur et expose l'application sur Internet |

> **Lien avec mon parcours :** c'est exactement le chemin à reproduire pour HotelListing.API : Dockerfile (cours Cloud-Native) → pipeline (ce cours) → ACR → App Service (cours Azure for .NET Developers).

### Les services Azure utilisés

| Service | Rôle | À retenir |
|---|---|---|
| **Azure Container Registry (ACR)** | Registre **privé** d'images Docker | Stocke lui-même les images : pas besoin de storage account. Le tier *Basic* suffit pour apprendre |
| **App Service Web App** (*Web App for Containers*) | Exécute le conteneur et expose l'application sur Internet | A besoin d'un **App Service Plan** (le « serveur » et son prix) |
| **Docker image** | Application + dépendances emballées | Nommée `<registre>.azurecr.io/<nom>:<tag>` (ex. : `monacr.azurecr.io/hotellisting-api:v1`) |
| **Webhooks** | Configurent le **continuous deployment** | Activer *Continuous deployment* dans le **Deployment Center** de l'App Service crée le webhook dans ACR |

### Ordre de création des ressources

```
1. Resource Group            ex. rg-hotellisting-dev
2. Azure Container Registry  stocke les images
3. App Service Plan          le serveur / le tier de prix
4. Web App for Containers    exécute l'image venant d'ACR
5. Webhook (Deployment Center → Continuous deployment)
```

- Le **Resource Group** est le « dossier » du projet : le supprimer supprime **tout** ce qu'il contient, d'un coup.
- Convention de nommage utile : `rg-` (resource group), `acr` (registry, lettres et chiffres uniquement), `app-` (web app), `asp-` (app service plan).

### Commandes Docker vers ACR (pour tester à la main)

```bash
az login
az acr login --name monacr

docker build -t monacr.azurecr.io/hotellisting-api:v1 .
docker push monacr.azurecr.io/hotellisting-api:v1
```

Dans le pipeline, la même chose se fait avec la tâche `Docker@2` :

```yaml
- task: Docker@2
  displayName: 'Build and push image to ACR'
  inputs:
    containerRegistry: 'acr-service-connection'   # service connection created in Azure DevOps
    repository: 'hotellisting-api'
    command: 'buildAndPush'
    Dockerfile: '**/Dockerfile'
    tags: |
      $(Build.BuildId)
      latest
```

> **Tag des images :** utiliser `$(Build.BuildId)` (un numéro unique par build) en plus de `latest`. On sait ainsi exactement quelle version tourne, et on peut revenir à une version précédente.

### Le storage account : nécessaire ou pas ?

- **Pas nécessaire** pour ACR ni App Service.
- Azure le demande à la première ouverture de **Cloud Shell** (`>_` dans le portail) pour garder tes fichiers entre deux sessions. On peut aussi choisir une session **sans stockage** (éphémère).
- Utile ailleurs : logs d'App Service, fichiers uploadés (Blob Storage), état de Terraform.

### ⚠️ Points d'attention

1. **Compte Azure de formation = compte temporaire.** Il sera probablement désactivé à la fin de la formation (décembre 2026). **Ne pas y déployer mon application de facturation en « vrai »** : elle disparaîtrait avec le compte. Demander au formateur jusqu'à quand le compte reste actif et s'il y a des règles (crédit, régions, nettoyage).
2. **Sur le compte de formation :** créer, tester, puis **supprimer le Resource Group** à la fin de l'exercice.
3. **Pour le déploiement réel :** mon propre abonnement Azure plus tard, avec une **alerte de budget** dans *Cost Management*.
4. **Tout garder dans Git** (Dockerfile, `azure-pipelines.yml`) : pour redéployer sur un autre compte, il suffit de changer les noms (ACR, App Service, service connection).
5. **Aucun secret dans le code ni dans le YAML** : variables secrètes du pipeline ou Azure Key Vault.
6. **Commencer par les tiers les plus bas** (ACR *Basic*, petit App Service Plan) et monter en gamme seulement si nécessaire.

### Ma checklist pour déployer mon projet (après avoir fini le front)

- [ ] Application qui fonctionne en local (API + front)
- [ ] Tests unitaires qui passent (`dotnet test`)
- [ ] Dockerfile + `docker-compose` testés en local
- [ ] Aucun secret dans le dépôt Git (vérifier aussi l'historique)
- [ ] Pipeline CI : restore → build → test → build & push de l'image
- [ ] Resource Group + ACR + App Service Plan + Web App for Containers
- [ ] Continuous deployment (webhook) activé
- [ ] Chaîne de connexion et secrets configurés dans l'App Service / Key Vault
- [ ] Smoke test après déploiement (l'API répond)
- [ ] README du projet : schéma d'architecture + badge du pipeline

---

## 12. Créer un pipeline CI/CD : étapes et exemples

### 12.1 Exemple : pipeline CI pour une API .NET

Les pipelines modernes s'écrivent en **YAML** dans un fichier `azure-pipelines.yml` à la racine du repo. Le pipeline est donc **versionné avec le code** (*pipeline as code*).

```yaml
# azure-pipelines.yml
trigger:
  branches:
    include:
      - main            # le pipeline se lance à chaque push sur main

pool:
  vmImage: 'ubuntu-latest'   # machine (agent) fournie par Microsoft

variables:
  buildConfiguration: 'Release'

steps:
  - task: UseDotNet@2
    displayName: 'Install .NET SDK'
    inputs:
      packageType: 'sdk'
      version: '8.x'

  - script: dotnet restore
    displayName: 'Restore NuGet packages'

  - script: dotnet build --configuration $(buildConfiguration) --no-restore
    displayName: 'Build'

  - script: dotnet test --configuration $(buildConfiguration) --no-build
    displayName: 'Run tests'

  - script: dotnet publish --configuration $(buildConfiguration) --output $(Build.ArtifactStagingDirectory)
    displayName: 'Publish'

  - task: PublishBuildArtifacts@1
    displayName: 'Publish artifact'
    inputs:
      PathtoPublish: '$(Build.ArtifactStagingDirectory)'
      ArtifactName: 'drop'
```

### 12.2 Vocabulaire du pipeline

| Terme | Signification |
|---|---|
| **Trigger** | Événement qui lance le pipeline (push, pull request, horaire) |
| **Agent / Pool** | Machine qui exécute le pipeline (hébergée par Microsoft ou *self-hosted*) |
| **Stage** | Grande phase (ex. : Build, Deploy Dev, Deploy Prod) |
| **Job** | Ensemble d'étapes exécutées sur un même agent |
| **Step / Task** | Une action unique (une commande, une tâche prédéfinie) |
| **Artifact** | Résultat du build, réutilisé pour le déploiement |
| **Variable** | Valeur réutilisable ; peut être secrète |
| **Service connection** | Connexion sécurisée entre Azure DevOps et Azure (pour déployer) |
| **Classic editor** | Ancienne façon de créer un pipeline par interface graphique (moins utilisée aujourd'hui) |

### 12.3 Prérequis avant de créer un pipeline

| # | Prérequis | Où |
|---|---|---|
| 1 | Une **organisation** et un **projet** Azure DevOps | `https://dev.azure.com` |
| 2 | Le **code** dans Azure Repos ou GitHub, avec un projet qui compile et des tests | Repos |
| 3 | Un **abonnement Azure** et les ressources cibles : Resource Group, ACR, App Service | Portail Azure |
| 4 | Des **service connections** : la « clé » qui permet à Azure DevOps d'agir sur Azure | *Project settings → Service connections* |

**Créer une service connection :**

1. *Project settings* (en bas à gauche) → **Service connections** → **New service connection**.
2. Pour déployer sur Azure : **Azure Resource Manager** → choisir l'abonnement et le Resource Group → lui donner un nom (ex. `azure-service-connection`).
3. Pour pousser des images : **Docker Registry** → **Azure Container Registry** → choisir l'ACR (ex. `acr-service-connection`).

> La service connection évite de mettre un mot de passe Azure dans le pipeline : c'est l'application du principe « secret keys, not passwords » (section 8).

### 12.4 Étapes pour créer le pipeline CI

| # | Étape | Détail |
|---|---|---|
| 1 | **Pipelines → New pipeline** | Menu de gauche du projet |
| 2 | **Where is your code?** | Choisir *Azure Repos Git* ou *GitHub* |
| 3 | **Select a repository** | Choisir le dépôt de l'application |
| 4 | **Configure your pipeline** | Choisir un modèle : *ASP.NET Core*, *Docker* (build and push to ACR), ou *Starter pipeline* (vide) |
| 5 | **Review your pipeline YAML** | Azure DevOps propose un fichier `azure-pipelines.yml` : l'adapter (version .NET, chemins, nom de l'image) |
| 6 | **Save and run** | Le fichier YAML est **commité dans le dépôt** et le pipeline se lance |
| 7 | **Vérifier l'exécution** | Logs de chaque étape, onglet *Tests* (résultats, couverture), artifact publié |
| 8 | **Vérifier le trigger** | Faire un petit commit sur `main` : le pipeline doit se relancer tout seul |
| 9 | **(Recommandé) Protéger `main`** | *Repos → Branches → … → Branch policies* : PR obligatoire + *Build validation* avec ce pipeline |

**Ce que fait le pipeline CI :** `restore → build → test → publish` (ou `build & push` de l'image Docker vers ACR). Exemple complet : section 12.1 et section 11 (tâche `Docker@2`).

### 12.5 Étapes pour créer le CD : trois façons de faire

#### Option A — Pipeline YAML multi-stage (méthode moderne, recommandée)

Le CI et le CD sont dans **le même fichier YAML**, découpé en **stages**.

| # | Étape | Détail |
|---|---|---|
| 1 | **Créer un Environment** | *Pipelines → Environments → New environment* (ex. `dev`, `production`) |
| 2 | **Ajouter des contrôles** | Sur l'environment : *Approvals and checks* → approbation manuelle, horaires autorisés, etc. (= les *pre-deployment conditions* de la section 8) |
| 3 | **Ajouter un stage Deploy** dans le YAML | Il dépend du stage Build et cible l'environment |
| 4 | **Commit + push** | Le pipeline exécute Build, puis attend l'approbation, puis déploie |
| 5 | **Smoke test** | Appeler l'URL de l'application pour vérifier qu'elle répond |

```yaml
trigger:
  branches:
    include:
      - main

variables:
  imageName: 'hotellisting-api'
  acrLoginServer: 'monacr.azurecr.io'

stages:
  # ---------- CI ----------
  - stage: Build
    displayName: 'Build, test and push image'
    jobs:
      - job: Build
        pool:
          vmImage: 'ubuntu-latest'
        steps:
          - task: UseDotNet@2
            inputs:
              packageType: 'sdk'
              version: '8.x'

          - script: dotnet test --configuration Release
            displayName: 'Run tests'

          - task: Docker@2
            displayName: 'Build and push image to ACR'
            inputs:
              containerRegistry: 'acr-service-connection'
              repository: '$(imageName)'
              command: 'buildAndPush'
              Dockerfile: '**/Dockerfile'
              tags: |
                $(Build.BuildId)

  # ---------- CD ----------
  - stage: Deploy
    displayName: 'Deploy to App Service'
    dependsOn: Build
    condition: succeeded()          # only if Build succeeded
    jobs:
      - deployment: DeployWebApp
        environment: 'production'   # approvals and checks apply here
        pool:
          vmImage: 'ubuntu-latest'
        strategy:
          runOnce:
            deploy:
              steps:
                - task: AzureWebAppContainer@1
                  displayName: 'Deploy container to App Service'
                  inputs:
                    azureSubscription: 'azure-service-connection'
                    appName: 'app-hotellisting-dev'
                    containers: '$(acrLoginServer)/$(imageName):$(Build.BuildId)'
```

- `stages` : Build (CI) puis Deploy (CD).
- `dependsOn` + `condition: succeeded()` : on ne déploie **que si** le build et les tests ont réussi.
- `deployment` + `environment` : job de déploiement soumis aux approbations de l'environment.
- On ajoute facilement d'autres stages : `Deploy_Dev` → `Deploy_QA` → `Deploy_Prod`.

#### Option B — Release pipeline classique (interface graphique, celle montrée dans le cours)

| # | Étape | Détail |
|---|---|---|
| 1 | **Pipelines → Releases → New pipeline** | |
| 2 | **Choisir un modèle** | *Azure App Service deployment* |
| 3 | **Add an artifact** | Source = le pipeline CI (son artifact ou l'image dans ACR) |
| 4 | **Activer le continuous deployment trigger** | Icône ⚡ sur l'artifact : une release est créée à chaque nouveau build |
| 5 | **Configurer le stage** | Choisir la service connection Azure, l'App Service, l'image |
| 6 | **Ajouter des stages** | Dev → QA → Prod (cloner un stage) |
| 7 | **Pre/Post-deployment conditions et gates** | Icônes à gauche et à droite de chaque stage (approbations, gates : section 8) |
| 8 | **Save → Create release** | Lancer une première release manuellement pour tester |

> Les *Releases* classiques fonctionnent toujours et sont encore présentes dans beaucoup d'entreprises, mais Microsoft recommande aujourd'hui le **YAML multi-stage** (option A). Il est bon de connaître les deux.

#### Option C — Webhook ACR → App Service (le plus simple)

C'est l'architecture de la section 11 :

1. Le pipeline CI pousse l'image dans ACR (toujours avec le même tag, ex. `latest`).
2. Dans le portail Azure : *App Service → Deployment Center* → source **Azure Container Registry** → choisir l'image et le tag → activer **Continuous deployment**.
3. Azure crée le **webhook** : à chaque nouvelle image, App Service fait un `docker pull` et redémarre.

> Simple, mais **sans approbation** ni étapes Dev/QA/Prod : convient pour un environnement de développement ou un projet personnel.

### 12.6 Récapitulatif : CI vs CD dans Azure DevOps

| | CI | CD |
|---|---|---|
| Déclenché par | Un push / une Pull Request | La réussite du CI (ou une nouvelle image dans ACR) |
| Fait | restore, build, test, publish / build & push image | déploie l'artifact ou l'image sur un environnement |
| Résultat | Un **artifact** ou une **image Docker** | L'application **en ligne** |
| Sécurité | Branch policies, build validation | Environments, approvals, gates |
| Où dans Azure DevOps | *Pipelines* (YAML) | Stage Deploy en YAML (*Environments*) ou *Releases* (classique) |

---

## 13. Concepts liés

### Reverse proxy

Serveur placé **devant** une ou plusieurs applications. Il reçoit toutes les requêtes et les redirige vers la bonne application.

```
Client ──► Reverse proxy (Nginx) ──► API HotelListing (Kestrel)
                                 └──► Front Angular
```

**Utilités :** point d'entrée unique, gestion du HTTPS (terminaison SSL), répartition de charge, sécurité, cache.

**Piège en ASP.NET Core :** ajouter `app.UseForwardedHeaders(...)` pour récupérer la vraie IP et le vrai schéma (http/https) du client.

Sur Azure : App Service (front-end intégré), Application Gateway, Front Door. En .NET : la librairie **YARP**.

### Middleware (ASP.NET Core)

Composant placé sur le chemin de chaque requête. Les middlewares forment le **pipeline HTTP** dans `Program.cs`.

```
Requête  → [ExceptionHandler] → [HTTPS] → [CORS] → [Authentication] → [Authorization] → Controller
Réponse  ←          ←            ←          ←              ←                  ←
```

- Chaque `app.UseXxx()` ajoute un middleware.
- **L'ordre est crucial** (ex. : `UseAuthentication()` **avant** `UseAuthorization()`).
- `await _next(context)` passe la main au middleware suivant ; ne pas l'appeler arrête la requête (*short-circuit*).

> Ne pas confondre le **pipeline HTTP** (middlewares dans l'application) et le **pipeline CI/CD** (automatisation du build et du déploiement).

---

## 14. Stratégies de branches Git

Une **stratégie de branches** définit comment l'équipe crée, nomme et fusionne ses branches. Elle a un impact direct sur le CI/CD : quelles branches déclenchent un build, lesquelles déclenchent un déploiement.

### Les trois stratégies principales

#### GitFlow

```
main      ●───────────────●────────────────●        (versions en production)
           \             / \              /
release     \       ●───●   \            /
             \     /         \          /
develop   ●───●───●───●───●───●───●───●──         (intégration)
               \     /     \     /
feature/x       ●───●       \   /
feature/y                    ●─●
hotfix/z  (part de main, revient dans main ET develop)
```

| Branche | Rôle |
|---|---|
| `main` | Code en production, uniquement des versions stables |
| `develop` | Branche d'intégration des nouvelles fonctionnalités |
| `feature/*` | Une branche par fonctionnalité, créée depuis `develop` |
| `release/*` | Préparation d'une version (corrections finales, numéro de version) |
| `hotfix/*` | Correction urgente en production, créée depuis `main` |

**Avantages :** très structuré, adapté aux versions planifiées.
**Inconvénients :** lourd, beaucoup de merges, branches qui vivent longtemps.

#### GitHub Flow

```
main        ●────●────●────●────●      (toujours déployable)
             \  /      \  /
feature/x     ●●        ●●
```

- Une seule branche permanente : `main`.
- Chaque changement passe par une **branche courte** + une **Pull Request** + le build de validation.
- Après le merge, `main` est déployée.

**Avantages :** simple, parfait pour le déploiement continu et les petites équipes.

#### Trunk-based development

- Tout le monde intègre dans `main` (le *trunk*) **au moins une fois par jour**, avec des branches qui vivent quelques heures.
- Les fonctionnalités non terminées sont cachées derrière des **feature flags** (interrupteurs dans la configuration).
- Demande une **bonne couverture de tests** et une CI rapide.

**Avantages :** c'est la stratégie qui colle le mieux à l'esprit DevOps (intégration vraiment continue).

### Comparaison

| | GitFlow | GitHub Flow | Trunk-based |
|---|---|---|---|
| Branches permanentes | `main` + `develop` | `main` | `main` |
| Durée de vie d'une branche | Jours / semaines | Heures / jours | Heures |
| Complexité | Élevée | Faible | Faible (mais exige des tests solides) |
| Idéal pour | Versions planifiées, logiciels installés | Applications web, petites équipes | Équipes matures, déploiements très fréquents |

> **Pour un projet personnel ou une petite équipe : GitHub Flow.** C'est aussi le plus simple à expliquer en entretien.

### Conventions de nommage et de commits

```
feature/cancel-booking
bugfix/invoice-total-rounding
hotfix/login-500-error
```

**Conventional Commits** (format de message de commit très répandu) :

```
feat: add booking cancellation endpoint
fix: correct invoice total rounding
test: add unit tests for BookingPriceCalculator
docs: update README with deployment steps
refactor: extract pricing logic into service
ci: add Docker build stage to pipeline
```

### Pull Requests : bonnes pratiques

- Une PR = **un seul sujet**, petite et facile à relire.
- Description claire : quoi, pourquoi, comment tester.
- Le **build de validation** doit être vert avant le merge.
- Au moins **un relecteur** (*reviewer*).
- Choix du type de merge : *merge commit*, *squash* (tous les commits de la branche deviennent un seul) ou *rebase*.

---

## 15. Dockerfile pour une application .NET

### Rappel : image vs conteneur

| Terme | Analogie | Explication |
|---|---|---|
| **Dockerfile** | La recette | Fichier texte qui décrit comment construire l'image |
| **Image** | Le plat préparé et congelé | Paquet figé : application + runtime + dépendances |
| **Conteneur** | Le plat réchauffé et servi | Une image **en cours d'exécution** |
| **Registry** | Le congélateur partagé | Endroit où on stocke les images (Docker Hub, ACR) |

### Dockerfile multi-stage pour une API ASP.NET Core

```dockerfile
# ---------- Stage 1: build ----------
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src

# Copy the project file first and restore (better layer caching)
COPY ["HotelListing.API/HotelListing.API.csproj", "HotelListing.API/"]
RUN dotnet restore "HotelListing.API/HotelListing.API.csproj"

# Copy the rest of the source code and publish
COPY . .
WORKDIR /src/HotelListing.API
RUN dotnet publish -c Release -o /app/publish /p:UseAppHost=false

# ---------- Stage 2: runtime ----------
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS final
WORKDIR /app
EXPOSE 8080
COPY --from=build /app/publish .
ENTRYPOINT ["dotnet", "HotelListing.API.dll"]
```

### Pourquoi « multi-stage » ?

| Stage | Image de base | Contenu | Taille |
|---|---|---|---|
| `build` | `sdk` | Tout le SDK .NET pour compiler | Grande |
| `final` | `aspnet` | Uniquement le runtime + l'application publiée | Beaucoup plus petite |

- L'image finale est **plus légère** (déploiement plus rapide).
- Elle est **plus sûre** : pas de compilateur ni de code source dedans.
- Copier le `.csproj` et faire `dotnet restore` **avant** de copier le reste permet à Docker de **réutiliser le cache** tant que les dépendances ne changent pas.

### Le fichier `.dockerignore`

Comme `.gitignore`, il évite d'envoyer des fichiers inutiles (ou sensibles) dans l'image :

```
**/bin/
**/obj/
**/.vs/
**/.vscode/
**/*.user
**/node_modules/
**/appsettings.Development.json
**/secrets.json
.git/
```

### Commandes Docker essentielles

```bash
docker build -t hotellisting-api:dev .          # build the image
docker images                                   # list images
docker run -d -p 5000:8080 --name api hotellisting-api:dev   # run a container
docker ps                                       # list running containers
docker logs api                                 # see the logs
docker stop api && docker rm api                # stop and remove
```

> **Port :** depuis .NET 8, les images ASP.NET écoutent par défaut sur le port **8080** (et plus 80). Sur Azure App Service, si le conteneur ne répond pas, ajouter le paramètre d'application `WEBSITES_PORT=8080`.

### docker-compose : API + SQL Server en local

```yaml
services:
  api:
    build: .
    ports:
      - "5000:8080"
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
      - ConnectionStrings__DefaultConnection=Server=db;Database=HotelListing;User Id=sa;Password=${SA_PASSWORD};TrustServerCertificate=True
    depends_on:
      - db

  db:
    image: mcr.microsoft.com/mssql/server:2022-latest
    environment:
      - ACCEPT_EULA=Y
      - MSSQL_SA_PASSWORD=${SA_PASSWORD}
    ports:
      - "1433:1433"
```

- `${SA_PASSWORD}` est lu dans un fichier `.env` **non commité** (à ajouter au `.gitignore`).
- `ConnectionStrings__DefaultConnection` : le double underscore `__` remplace le `:` de `appsettings.json` dans les variables d'environnement.
- Dans le réseau Docker, l'API joint la base par le **nom du service** (`Server=db`), pas par `localhost`.

```bash
docker compose up -d --build     # start everything
docker compose down              # stop everything
```

---

## 16. Variables et secrets dans les pipelines

### Les niveaux de variables

| Où | Usage | Exemple |
|---|---|---|
| Dans le YAML (`variables:`) | Valeurs **non sensibles**, versionnées avec le code | `buildConfiguration: 'Release'` |
| Variables du pipeline (interface) | Valeurs modifiables sans commit ; peuvent être **secrètes** (cadenas) | Mot de passe d'un outil externe |
| **Variable groups** (*Pipelines → Library*) | Variables **partagées** entre plusieurs pipelines | `hotellisting-secrets` |
| Variable group **lié à Azure Key Vault** | Les secrets restent dans Key Vault, le pipeline les lit au moment de l'exécution | Chaîne de connexion de production |

### Utiliser un variable group

```yaml
variables:
  - group: hotellisting-secrets          # variable group from Library
  - name: buildConfiguration
    value: 'Release'

steps:
  - script: echo "Deploying with configuration $(buildConfiguration)"
```

### Lire des secrets depuis Azure Key Vault

```yaml
- task: AzureKeyVault@2
  displayName: 'Get secrets from Key Vault'
  inputs:
    azureSubscription: 'azure-service-connection'
    KeyVaultName: 'kv-hotellisting-dev'
    SecretsFilter: '*'          # or a comma-separated list of secret names
```

Après cette tâche, chaque secret est disponible comme variable (`$(NomDuSecret)`) et **masqué** (`***`) dans les logs.

### Secrets de l'application elle-même (App Service)

Deux façons propres de donner ses secrets à l'application en production :

1. **App Service → Configuration (Environment variables)** : paramètres d'application et chaînes de connexion, qui remplacent les valeurs de `appsettings.json`.
2. **Références Key Vault** : dans un paramètre d'application, au lieu de la valeur :

```
@Microsoft.KeyVault(SecretUri=https://kv-hotellisting-dev.vault.azure.net/secrets/DbConnection/)
```

L'App Service (avec une **managed identity** autorisée sur le Key Vault) lit le secret lui-même. Personne ne voit la valeur.

### Où mettre quoi ?

| Environnement | Outil pour les secrets |
|---|---|
| Développement local | **User Secrets** (`dotnet user-secrets set "ConnectionStrings:DefaultConnection" "..."`) ou fichier `.env` non commité |
| Pipeline | Variables secrètes, variable groups, **Azure Key Vault** |
| Production (App Service) | Paramètres d'application, **références Key Vault** |
| Jamais | `appsettings.json` commité, YAML, Dockerfile, code source |

> **Managed identity :** identité gérée par Azure et attribuée à une ressource (ex. : l'App Service). Elle permet à la ressource d'accéder à d'autres services (Key Vault, ACR, SQL) **sans aucun mot de passe** à stocker.

---

## 17. Infrastructure as Code (IaC)

### Le principe

Au lieu de créer les ressources à la main dans le portail Azure (*ClickOps*), on les **décrit dans des fichiers** versionnés dans Git. Le pipeline peut ensuite créer ou mettre à jour l'infrastructure automatiquement.

| Sans IaC (portail) | Avec IaC |
|---|---|
| Clics manuels, difficile à reproduire | Fichier texte, reproductible à l'identique |
| Pas d'historique | Historique Git, relecture en Pull Request |
| Recréer un environnement = refaire tous les clics | Recréer un environnement = relancer une commande |
| Risque d'oubli ou d'erreur | Même résultat à chaque fois |

### Les outils

| Outil | Éditeur | Particularité |
|---|---|---|
| **Bicep** | Microsoft | Langage simple, **spécifique à Azure**, recommandé pour débuter sur Azure |
| **ARM templates** | Microsoft | JSON verbeux ; Bicep est compilé en ARM |
| **Terraform** | HashiCorp | **Multi-cloud** (Azure, AWS, GCP), très demandé sur le marché ; garde un fichier d'**état** (*state*) |
| Ansible, Chef | — | Plutôt pour **configurer** des serveurs que pour créer des ressources cloud |

### Exemple Bicep : ACR + App Service Plan + Web App

```bicep
param location string = resourceGroup().location
param acrName string = 'acrhotellistingdev'
param appName string = 'app-hotellisting-dev'

resource acr 'Microsoft.ContainerRegistry/registries@2023-07-01' = {
  name: acrName
  location: location
  sku: {
    name: 'Basic'
  }
}

resource plan 'Microsoft.Web/serverfarms@2023-01-01' = {
  name: 'asp-hotellisting-dev'
  location: location
  kind: 'linux'
  sku: {
    name: 'B1'
  }
  properties: {
    reserved: true   // required for Linux plans
  }
}

resource webApp 'Microsoft.Web/sites@2023-01-01' = {
  name: appName
  location: location
  properties: {
    serverFarmId: plan.id
    siteConfig: {
      linuxFxVersion: 'DOCKER|${acr.properties.loginServer}/hotellisting-api:latest'
    }
  }
}
```

```bash
az group create --name rg-hotellisting-dev --location westeurope
az deployment group create --resource-group rg-hotellisting-dev --template-file main.bicep
```

> Dans un vrai projet, on ajoute une **managed identity** à la Web App et le rôle **AcrPull** sur l'ACR, pour que l'App Service puisse tirer l'image sans mot de passe.

### Dans le pipeline

```yaml
- task: AzureCLI@2
  displayName: 'Deploy infrastructure (Bicep)'
  inputs:
    azureSubscription: 'azure-service-connection'
    scriptType: 'bash'
    scriptLocation: 'inlineScript'
    inlineScript: |
      az deployment group create \
        --resource-group rg-hotellisting-dev \
        --template-file infra/main.bicep
```

---

## 18. Monitoring : surveiller l'application en production

C'est l'étape **Operate** de la boucle DevOps : une application déployée doit être **surveillée**, sinon on découvre les problèmes par les utilisateurs.

### Les trois piliers de l'observabilité

| Pilier | Question | Exemple |
|---|---|---|
| **Logs** | Que s'est-il passé ? | « Booking 42 created », « Exception: invoice not found » |
| **Metrics** | Combien ? À quelle vitesse ? | Requêtes par seconde, temps de réponse moyen, % d'erreurs, CPU |
| **Traces** | Par où est passée la requête ? | API → service → base de données, avec la durée de chaque étape |

### Les outils Azure

| Outil | Rôle |
|---|---|
| **Azure Monitor** | Plateforme globale de surveillance d'Azure (métriques, logs, alertes) |
| **Application Insights** | Surveillance **de l'application** : requêtes, erreurs, temps de réponse, dépendances (SQL, HTTP) |
| **Log Analytics** | Stockage et requêtes des logs avec le langage **KQL** |
| **Log stream (App Service)** | Voir les logs du conteneur **en direct** : très utile quand l'application ne démarre pas |
| **Alerts** | Envoyer un e-mail ou une notification quand un seuil est dépassé |

### Ajouter Application Insights à une API ASP.NET Core

Méthode recommandée aujourd'hui (OpenTelemetry) :

```bash
dotnet add package Azure.Monitor.OpenTelemetry.AspNetCore
```

```csharp
// Program.cs
builder.Services.AddOpenTelemetry().UseAzureMonitor();
```

Méthode classique (encore très répandue) :

```bash
dotnet add package Microsoft.ApplicationInsights.AspNetCore
```

```csharp
builder.Services.AddApplicationInsightsTelemetry();
```

Dans les deux cas, la **connection string** d'Application Insights est fournie par la variable d'environnement `APPLICATIONINSIGHTS_CONNECTION_STRING` (paramètre de l'App Service), jamais en dur dans le code.

### Health checks : un point d'entrée pour vérifier que l'app est vivante

```csharp
// Program.cs
builder.Services.AddHealthChecks()
    .AddDbContextCheck<HotelListingDbContext>();   // package: Microsoft.Extensions.Diagnostics.HealthChecks.EntityFrameworkCore

app.MapHealthChecks("/health");
```

- Le **smoke test** du pipeline peut simplement appeler `/health`.
- App Service peut utiliser `/health` (*Health check* dans les paramètres) pour redémarrer une instance malade.

### Exemple de requête KQL (Log Analytics / Application Insights)

```kusto
requests
| where timestamp > ago(1h)
| summarize count(), avg(duration) by resultCode
| order by count_ desc
```

### Logs structurés avec Serilog

```csharp
_logger.LogInformation("Booking {BookingId} created for hotel {HotelId}", booking.Id, booking.HotelId);
```

Les valeurs entre accolades deviennent des **champs** qu'on peut filtrer (dans Seq, Application Insights…), au lieu d'un simple texte.

---

## 19. GitHub Actions : l'équivalent sur GitHub

Mêmes principes qu'Azure Pipelines, avec un vocabulaire un peu différent.

| Azure Pipelines | GitHub Actions |
|---|---|
| `azure-pipelines.yml` à la racine | `.github/workflows/*.yml` |
| `trigger` | `on` |
| `pool: vmImage` | `runs-on` |
| `stages` / `jobs` | `jobs` (avec `needs` pour l'ordre) |
| `task: Xxx@2` | `uses: action@vX` |
| Service connection | **Secrets** du dépôt + `azure/login` |
| Environments (approvals) | Environments (*required reviewers*) |

### Exemple : build, test, image Docker, déploiement

```yaml
name: CI/CD

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '8.0.x'

      - run: dotnet restore
      - run: dotnet build --no-restore --configuration Release
      - run: dotnet test --no-build --configuration Release

  deploy:
    needs: build                       # only after build succeeded
    if: github.ref == 'refs/heads/main'  # never deploy from a pull request
    runs-on: ubuntu-latest
    environment: production            # approvals can be required here
    steps:
      - uses: actions/checkout@v4

      - uses: azure/login@v2
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}

      - run: |
          az acr login --name monacr
          docker build -t monacr.azurecr.io/hotellisting-api:${{ github.sha }} .
          docker push monacr.azurecr.io/hotellisting-api:${{ github.sha }}

      - uses: azure/webapps-deploy@v3
        with:
          app-name: app-hotellisting-dev
          images: monacr.azurecr.io/hotellisting-api:${{ github.sha }}
```

> `${{ secrets.AZURE_CREDENTIALS }}` : secret stocké dans *Settings → Secrets and variables → Actions* du dépôt GitHub. Pour aller plus loin : la connexion **OIDC** (*federated credentials*) évite même de stocker ce secret.

---

## 20. Dépannage : erreurs fréquentes et solutions

| Symptôme | Cause probable | Solution |
|---|---|---|
| *No hosted parallelism has been purchased or granted* | Les nouvelles organisations Azure DevOps n'ont pas d'agent Microsoft gratuit par défaut | Demander la gratuité via le formulaire de Microsoft (délai de quelques jours), ou utiliser un **self-hosted agent** sur son PC |
| *This pipeline needs permission to access a resource* | Première utilisation d'une service connection, d'un environment ou d'un variable group | Ouvrir l'exécution et cliquer sur **Permit** |
| Erreur de version .NET (*SDK not found*, *NETSDK1045*) | L'agent n'a pas la bonne version du SDK | Ajouter la tâche `UseDotNet@2` avec la bonne version (ou un `global.json`) |
| *COPY failed: file not found* pendant `docker build` | Mauvais **contexte de build** ou mauvais chemin dans le Dockerfile | Vérifier depuis quel dossier le build est lancé ; ajuster `buildContext` dans `Docker@2` |
| Le YAML ne se lance pas ou erreur de syntaxe | **Indentation** incorrecte (espaces vs tabulations) | Uniquement des espaces ; utiliser l'éditeur YAML d'Azure DevOps (validation) |
| Les tests échouent et le pipeline s'arrête | Comportement **normal** : c'est le rôle de la CI | Lancer `dotnet test` en local, corriger, recommiter |
| App Service : *Application Error* / conteneur qui ne démarre pas | Mauvais port, variable manquante, exception au démarrage | Lire le **Log stream** ; vérifier `WEBSITES_PORT=8080` et les paramètres d'application |
| App Service n'arrive pas à tirer l'image (*unauthorized*) | L'App Service n'a pas le droit de lire l'ACR | Managed identity + rôle **AcrPull** (ou *admin user* d'ACR pour un test rapide) |
| L'application démarre mais ne trouve pas la base | Chaîne de connexion absente ou pare-feu Azure SQL | Ajouter la chaîne dans la configuration de l'App Service ; autoriser les services Azure dans le pare-feu SQL |
| Le pipeline ne se déclenche pas après un push | `trigger` sur une autre branche, ou dépôt GitHub non autorisé | Vérifier la section `trigger` et la connexion GitHub |
| Un secret a été commité par erreur | — | **Le considérer comme compromis** : le changer immédiatement, puis le retirer de l'historique Git |
| Coûts qui augmentent | Ressources oubliées | Supprimer les Resource Groups de test, configurer une alerte de budget |

> **Réflexe de dépannage :** toujours commencer par **lire le log de l'étape en erreur** dans le pipeline (ou le Log stream de l'App Service). Le message exact se trouve presque toujours dans les dernières lignes.

---

## 21. Glossaire

| Terme | Définition |
|---|---|
| **ACR** | Azure Container Registry : registre privé d'images Docker |
| **Approval** | Validation manuelle demandée avant ou après un déploiement |
| **Artifact** | Paquet produit par un build (fichiers compilés, image Docker…) |
| **Bicep** | Langage d'Infrastructure as Code de Microsoft pour Azure |
| **Branch policy** | Règle qui protège une branche (PR obligatoire, build obligatoire) |
| **Build** | Compilation du code source en application exécutable |
| **CD** | Continuous Delivery (prêt à déployer) ou Continuous Deployment (déployé automatiquement) |
| **Check-in / Commit** | Enregistrement d'une modification dans le gestionnaire de versions |
| **CI** | Continuous Integration : intégration et vérification automatiques du code |
| **Code coverage** | Pourcentage du code exécuté par les tests |
| **Configuration management** | Gestion des paramètres propres à chaque environnement |
| **Container** | Application emballée avec toutes ses dépendances (Docker) |
| **Data at rest / in transit** | Donnée stockée / donnée qui circule sur le réseau ; les deux doivent être chiffrées |
| **DevSecOps** | DevOps avec la sécurité intégrée à chaque étape du pipeline |
| **Environment** | Lieu de déploiement : Dev, Test, Staging, Production |
| **F.I.R.S.T.** | Fast, Isolated, Repeatable, Self-checking, Timely : qualités d'un bon test unitaire |
| **Feature flag** | Interrupteur de configuration qui active ou cache une fonctionnalité sans redéployer |
| **Gate** | Vérification automatique avant ou après un déploiement |
| **GitFlow / GitHub Flow / Trunk-based** | Stratégies de branches Git (section 14) |
| **Health check** | Endpoint (ex. `/health`) qui indique si l'application fonctionne |
| **IaC** | Infrastructure as Code : décrire les ressources cloud dans des fichiers (Terraform, Bicep, ARM) |
| **Integration test** | Test de plusieurs composants ensemble (ex. : API + base de données) |
| **KQL** | Kusto Query Language : langage de requête des logs dans Azure |
| **Managed identity** | Identité Azure attribuée à une ressource, pour accéder à d'autres services sans mot de passe |
| **Microsoft Entra ID** | Nouveau nom d'Azure Active Directory : gestion des identités et des accès |
| **Mock** | Faux objet qui remplace une dépendance réelle pendant un test unitaire |
| **Monitoring** | Surveillance de l'application en production (logs, métriques, alertes) |
| **Multi-stage build** | Dockerfile en plusieurs étapes : compiler avec le SDK, exécuter avec le runtime seul |
| **Pipeline** | Suite d'étapes automatisées (build, test, déploiement) |
| **Pull Request (PR)** | Demande de fusion d'une branche, avec relecture de code |
| **SaaS** | Software as a Service : logiciel utilisé en ligne, sans installation |
| **Self-hosted agent** | Agent de pipeline installé sur sa propre machine plutôt que fourni par Microsoft |
| **Shift left** | Intégrer la sécurité et les tests le plus tôt possible dans le cycle |
| **Smoke test** | Test rapide qui vérifie que l'application démarre et répond |
| **SonarCloud** | Service d'analyse de qualité et de sécurité du code (renommé SonarQube Cloud) |
| **Terraform** | Outil d'Infrastructure as Code multi-cloud (HashiCorp) |
| **Unit test** | Test automatisé d'une petite unité de code (une méthode, une classe) |
| **Variable group** | Ensemble de variables partagé entre pipelines, éventuellement lié à Key Vault |
| **Webhook** | Notification HTTP automatique envoyée lors d'un événement |
| **YAML** | Format de fichier texte utilisé pour décrire les pipelines |

---

## 22. Questions d'entretien fréquentes

1. **C'est quoi la différence entre CI et CD ?**
   La CI compile et teste automatiquement chaque modification. Le CD prend le résultat et le livre : prêt à déployer (Delivery) ou déployé automatiquement (Deployment).

2. **Continuous Delivery ou Continuous Deployment ?**
   Delivery : une validation humaine avant la production. Deployment : aucune intervention.

3. **Qu'est-ce qu'un artifact ?**
   Le résultat du build, stocké et réutilisé pour déployer **le même paquet** sur chaque environnement.

4. **Pourquoi écrire les pipelines en YAML plutôt qu'avec l'éditeur graphique ?**
   Le YAML est **versionné avec le code**, relu en Pull Request et réutilisable.

5. **Comment gérer les secrets dans un pipeline ?**
   Variables secrètes du pipeline, Variable Groups liés à **Azure Key Vault**, jamais dans le code ni dans le YAML.

6. **Qu'est-ce qu'un smoke test ?**
   Un test rapide après déploiement pour vérifier que l'application démarre et répond.

7. **Quelles sont les qualités d'un bon test unitaire ?**
   F.I.R.S.T. : rapide, isolé, répétable, auto-vérifiant, écrit au bon moment. Structure AAA : Arrange, Act, Assert.

8. **Différence entre test unitaire et test d'intégration ?**
   L'unitaire teste une méthode isolée avec des mocks ; l'intégration teste plusieurs couches réelles ensemble (ex. : API + base).

9. **Qu'est-ce que DevSecOps ?**
   Intégrer la sécurité à chaque étape du cycle DevOps (*shift left*) au lieu de la vérifier seulement à la fin.

10. **Comment protéger un pipeline de déploiement ?**
    Permissions par groupes, approbations manuelles (pre/post-deployment conditions), vérifications automatiques (gates), secrets dans Key Vault.

11. **Décris le chemin de ton code jusqu'à la production.**
    Commit → push vers le repo → le pipeline se déclenche, compile, teste et construit une image Docker → l'image est poussée dans Azure Container Registry → App Service récupère la nouvelle image et redémarre.

12. **Quelles sont les étapes pour créer un pipeline CI/CD dans Azure DevOps ?**
    Créer les service connections (Azure + ACR), créer le pipeline CI depuis le dépôt (YAML : build, test, push de l'image), puis ajouter un stage Deploy lié à un Environment avec approbations, ou une Release classique.

13. **Quelle stratégie de branches utilises-tu ?**
    GitHub Flow pour un petit projet : `main` toujours déployable, branches courtes, Pull Request avec build de validation. GitFlow quand il y a des versions planifiées.

14. **Pourquoi un Dockerfile multi-stage ?**
    Compiler avec l'image SDK et exécuter avec l'image runtime seule : image finale plus légère et plus sûre.

15. **Qu'est-ce que l'Infrastructure as Code ?**
    Décrire les ressources cloud dans des fichiers versionnés (Bicep, Terraform) pour les créer de façon reproductible, au lieu de cliquer dans le portail.

16. **Comment sais-tu que ton application va bien en production ?**
    Application Insights (requêtes, erreurs, temps de réponse), logs structurés, endpoint `/health`, alertes Azure Monitor.

17. **Azure DevOps ou GitHub Actions ?**
   Mêmes principes (YAML, triggers, jobs, steps). Azure DevOps est très présent en entreprise Microsoft ; GitHub Actions est intégré à GitHub et de plus en plus utilisé.

---

## 23. Exercice de formation : application Palindrome avec CI

### L'énoncé

Réaliser une application .NET qui permet de tester si un mot ou une phrase est un **palindrome** (se lit de la même façon dans les deux sens).

- Créer une **application Desktop** avec un **service** qui permet de tester un mot et de tester une phrase.
- Créer un **projet de test xUnit** pour tester le service.
- Utiliser **Azure Repos** comme repository Git.
- Mettre en place un **pipeline CI** qui permet d'exécuter les tests et de lancer l'application.

| Entrée | Résultat attendu |
|---|---|
| `kayak` | `true` |
| `aa` | `true` |
| `pomme` | `false` |
| `Un roc si biscornu.` | `true` |
| `Et la marine va, papa, venir à Malte.` | `true` |

> Le code du service est à écrire soi-même : cette section donne les **étapes**, la **structure** et les **pièges à éviter**, pas la solution.

### Étape 0 : bien comprendre le problème avant de coder

Pour les **phrases**, on ne peut pas comparer le texte tel quel. Il faut d'abord le **normaliser** :

| Difficulté | Exemple | À faire |
|---|---|---|
| Majuscules | `Un` / `un` | Tout mettre en minuscules |
| Espaces | `Un roc si` | Les ignorer |
| Ponctuation | `,` `.` | L'ignorer (garder uniquement les lettres et les chiffres) |
| **Accents** | `à` dans « venir **à** Malte » | Considérer `à` comme `a`, sinon le test échoue |

Questions à se poser (et à transformer en tests) :
- Que renvoie le service pour une chaîne **vide** ? Pour `null` ? Pour une seule lettre ?
- Faut-il **deux méthodes** (`IsWordPalindrome`, `IsSentencePalindrome`) ou une seule ? L'énoncé parle de deux cas : deux méthodes rendent l'intention plus claire.

> **Indice pour les accents :** chercher `string.Normalize(NormalizationForm.FormD)` et `CharUnicodeInfo.GetUnicodeCategory(...) == UnicodeCategory.NonSpacingMark`. La forme FormD sépare la lettre de son accent ; il suffit ensuite d'ignorer les accents.
>
> **Indice pour garder uniquement les lettres :** `char.IsLetterOrDigit(c)`.

### Étape 1 : la structure de la solution

```
PalindromeChecker/
├── PalindromeChecker.sln
├── azure-pipelines.yml
├── README.md
├── src/
│   ├── PalindromeChecker.Core/        ← class library : le service (logique métier)
│   │   ├── IPalindromeService.cs
│   │   └── PalindromeService.cs
│   └── PalindromeChecker.Desktop/     ← WinForms ou WPF : l'interface
└── tests/
    └── PalindromeChecker.Tests/       ← xUnit : teste le Core
```

**Pourquoi séparer le service dans un projet `Core` ?**

1. **Séparation des responsabilités :** l'interface graphique affiche, le service calcule. Le formulaire ne contient aucune logique de palindrome.
2. **Testabilité :** le projet de test référence `Core` uniquement, pas l'interface graphique.
3. **Compatibilité CI :** `Core` et `Tests` ciblent `net8.0` (multiplateforme) ; seul `Desktop` cible `net8.0-windows`.
4. **Réutilisable :** le même service pourrait servir demain à une API ou une appli web.

```bash
mkdir PalindromeChecker && cd PalindromeChecker
dotnet new sln -n PalindromeChecker

dotnet new classlib -n PalindromeChecker.Core -o src/PalindromeChecker.Core
dotnet new winforms -n PalindromeChecker.Desktop -o src/PalindromeChecker.Desktop   # or: dotnet new wpf
dotnet new xunit    -n PalindromeChecker.Tests -o tests/PalindromeChecker.Tests

dotnet sln add src/PalindromeChecker.Core src/PalindromeChecker.Desktop tests/PalindromeChecker.Tests

dotnet add src/PalindromeChecker.Desktop reference src/PalindromeChecker.Core
dotnet add tests/PalindromeChecker.Tests reference src/PalindromeChecker.Core

dotnet new gitignore
```

### Étape 2 : le service (Core)

Commencer par l'**interface** : c'est le contrat que l'interface graphique et les tests vont utiliser.

```csharp
namespace PalindromeChecker.Core;

public interface IPalindromeService
{
    bool IsWordPalindrome(string word);
    bool IsSentencePalindrome(string sentence);
}
```

Puis écrire `PalindromeService : IPalindromeService`. Pistes :
- Une méthode privée `Normalize(string input)` pour les phrases (minuscules, sans accents, uniquement lettres et chiffres).
- Une méthode privée qui compare le texte à son inverse, ou qui compare les caractères deux à deux depuis les deux extrémités (plus efficace : pas de nouvelle chaîne créée).
- Réutiliser cette comparaison dans les deux méthodes publiques.
- Décider du comportement pour `null` (exception `ArgumentNullException` ?) et l'écrire dans un test.

### Étape 3 : les tests xUnit (idéalement AVANT le service : TDD)

Appliquer la structure **AAA** et les principes **F.I.R.S.T.** (section 7.1).

Squelette pour démarrer :

```csharp
using PalindromeChecker.Core;

namespace PalindromeChecker.Tests;

public class PalindromeServiceTests
{
    private readonly IPalindromeService _service = new PalindromeService();

    [Theory]
    [InlineData("kayak")]
    [InlineData("aa")]
    public void IsWordPalindrome_Palindrome_ReturnsTrue(string word)
    {
        // Act
        var result = _service.IsWordPalindrome(word);

        // Assert
        Assert.True(result);
    }

    // TODO: IsWordPalindrome_NotPalindrome_ReturnsFalse ("pomme")
    // TODO: IsSentencePalindrome_Palindrome_ReturnsTrue (the two sentences)
    // TODO: IsSentencePalindrome_NotPalindrome_ReturnsFalse (choose a sentence)
    // TODO: edge cases: empty string, single letter, uppercase word ("Kayak"), null
}
```

```bash
dotnet test
```

> **Bonne pratique :** lancer `dotnet test` en local et voir tous les tests **verts** avant de pousser. Le pipeline ne doit pas servir à découvrir des erreurs qu'on pouvait voir en 5 secondes.

### Étape 4 : l'interface Desktop

- Un champ de saisie, deux boutons (« Tester le mot », « Tester la phrase ») ou un choix, et un label pour le résultat.
- Le formulaire **appelle le service** et affiche le résultat ; aucune logique de palindrome dans le formulaire.
- Bonus : afficher un message clair si le champ est vide.

### Étape 5 : Azure Repos

1. Sur `https://dev.azure.com` : créer un **projet** (ex. `PalindromeChecker`).
2. *Repos* → copier l'URL du dépôt.
3. En local :

```bash
git init
git add .
git commit -m "feat: palindrome service, desktop UI and xUnit tests"
git branch -M main
git remote add origin https://dev.azure.com/<organisation>/PalindromeChecker/_git/PalindromeChecker
git push -u origin main
```

4. Vérifier que `.gitignore` exclut bien `bin/`, `obj/` et `.vs/`.

### Étape 6 : le pipeline CI

#### ⚠️ Piège n° 1 : l'agent doit être Windows

Une application **WinForms ou WPF** cible `net8.0-windows` : elle **ne compile pas** sur un agent Linux (`ubuntu-latest`). Il faut :

```yaml
pool:
  vmImage: 'windows-latest'
```

#### ⚠️ Piège n° 2 : que veut dire « lancer l'application » dans un pipeline ?

Un agent de pipeline **n'a pas d'écran** et personne ne peut cliquer. Si le pipeline démarre une fenêtre WinForms/WPF, il **reste bloqué** jusqu'au timeout. Interprétations possibles (à **confirmer avec le formateur**) :

| Interprétation | Comment faire |
|---|---|
| **Construire et publier** l'application exécutable | `dotnet publish` du projet Desktop + publier l'**artifact** (le `.exe` téléchargeable depuis le pipeline) |
| **Exécuter** une version console du service | Ajouter un petit projet console qui appelle le service avec quelques valeurs et affiche les résultats dans le log |
| Lancer l'application **puis l'arrêter** (smoke test) | Possible en PowerShell (démarrer le processus, attendre quelques secondes, vérifier qu'il tourne, l'arrêter), mais fragile |

La première interprétation est la plus courante dans un pipeline CI réel.

#### Les étapes du pipeline

| # | Étape | Commande / tâche |
|---|---|---|
| 1 | Déclencher sur `main` | `trigger` |
| 2 | Agent Windows | `pool: vmImage: 'windows-latest'` |
| 3 | Installer le SDK .NET | `UseDotNet@2` |
| 4 | Restaurer les paquets | `dotnet restore` |
| 5 | Compiler | `dotnet build --configuration Release --no-restore` |
| 6 | **Exécuter les tests** | Tâche `DotNetCoreCLI@2` avec `command: 'test'` (publie les résultats dans l'onglet Tests) |
| 7 | **Publier l'application** | `dotnet publish` du projet Desktop |
| 8 | Rendre l'exécutable téléchargeable | `PublishPipelineArtifact@1` |

Squelette à compléter (s'inspirer de la section 12.1) :

```yaml
trigger:
  branches:
    include:
      - main

pool:
  vmImage: 'windows-latest'

variables:
  buildConfiguration: 'Release'

steps:
  - task: UseDotNet@2
    displayName: 'Install .NET SDK'
    inputs:
      packageType: 'sdk'
      version: '8.x'

  # TODO: dotnet restore
  # TODO: dotnet build (Release, --no-restore)
  # TODO: tests with the DotNetCoreCLI@2 task (command: 'test') so results appear in the Tests tab
  # TODO: dotnet publish src/PalindromeChecker.Desktop --output $(Build.ArtifactStagingDirectory)

  - task: PublishPipelineArtifact@1
    displayName: 'Publish desktop app'
    inputs:
      targetPath: '$(Build.ArtifactStagingDirectory)'
      artifact: 'PalindromeChecker.Desktop'
```

Créer le pipeline : *Pipelines → New pipeline → Azure Repos Git → PalindromeChecker → Existing Azure Pipelines YAML file* (ou *Starter pipeline* et coller le YAML) → **Save and run**.

### Étape 7 : vérifier que la CI fait son travail

1. Le pipeline est **vert** et l'onglet **Tests** affiche tous les tests réussis.
2. L'artifact `PalindromeChecker.Desktop` est téléchargeable dans le résumé de l'exécution.
3. **Test volontaire :** casser le service (ex. retirer la gestion des accents), pousser, et vérifier que le pipeline devient **rouge** à l'étape de test. Puis corriger et repousser.
4. Bonus : protéger `main` avec une **branch policy** et un *build validation* (section 12.4, étape 9), puis passer par une Pull Request.

### Ce que j'ai appris en réalisant l'exercice

| Situation | Explication | Solution |
|---|---|---|
| Projet en **.NET 11** | La version du SDK dans le pipeline doit correspondre au `TargetFramework` | `UseDotNet@2` avec `version: '11.x'` |
| Indentation YAML refusée | `displayName` doit être aligné **sous** `script` (4 espaces), sinon YAML le lit comme un nouvel élément | Uniquement des espaces ; vérifier l'alignement |
| Les lignes `hint:` décommentées | Un commentaire `#` retiré devient du YAML, donc une erreur | Remplacer l'indice par une vraie étape `- script:` |
| Onglet **Tests** vide | `script: dotnet test` ne publie pas les résultats | Tâche `DotNetCoreCLI@2` (section 7.4) |
| Pipeline **en attente** (horloge) | 1 seul agent gratuit (1 job parallèle) pour toute l'organisation | Patienter ; ne pas lancer de run manuel en doublon d'un run de PR |
| `TF402455: Pushes to this branch are not permitted` | La **branch policy** protège `main` : tout passe par une Pull Request | `git checkout -b ma-branche` → push de la branche → PR → `git branch -f main origin/main` |
| La PR est créée où ? | La PR n'existe pas dans Git : c'est une fonctionnalité d'Azure Repos | Push de la branche sur le PC, PR dans Azure DevOps |
| **Auto-complete** | La PR est fusionnée automatiquement dès que le pipeline est vert et la PR approuvée | Renommer la PR **avant** d'approuver |
| Message de commit par défaut de l'éditeur web | « Update azure-pipelines.yml… » ne dit rien | Écrire un message clair : `ci: publish test results to Tests tab` |
| Test de la CI | Casser volontairement le code : pipeline rouge sur `main`, puis PR **bloquée** par la Build validation | La CI et la branch policy protègent `main` |

### Checklist de l'exercice

- [ ] Solution avec 3 projets : Core, Desktop, Tests
- [ ] `IPalindromeService` + `PalindromeService` (mot + phrase)
- [ ] Gestion des majuscules, espaces, ponctuation et **accents**
- [ ] Tests xUnit : les 5 exemples de l'énoncé + cas limites
- [ ] `dotnet test` vert en local
- [ ] Interface Desktop qui appelle le service
- [ ] Code poussé sur Azure Repos (sans `bin/` ni `obj/`)
- [ ] Pipeline CI sur agent **Windows** : restore, build, test, publish
- [ ] Artifact de l'application disponible
- [ ] Test volontaire d'un pipeline rouge, puis correction
- [ ] Branch policy sur `main` : reviewer minimum + Build validation
- [ ] Résultats des tests visibles dans l'onglet **Tests** (`DotNetCoreCLI@2`)

---

## Ma progression dans le cours

- [ ] Section 1-2 : Introduction to DevOps
- [ ] Section 3 : DevOps Platform and Tools
- [ ] Section 4 : Setting up Development Environment
- [ ] Section 5 : Branching and Source Control
- [ ] Section 6 : Automated Testing
- [ ] Section 7 : DevOps Security (DevSecOps)
- [ ] Section 8 : Microsoft Azure Platforms for Deployments
- [ ] Sections suivantes : application web et pipelines complets
- [ ] **Pratique :** pipeline CI/CD pour HotelListing.API
