# Document explicatif sur les GitHub Actions mises en place
Introduction
Ce document décrit les GitHub Actions configurées pour le projet, présente les indicateurs de performance clés (KPIs), analyse les premières métriques recueillies et prend en compte les retours des utilisateurs pour identifier les améliorations nécessaires.

## Workflows GitHub Actions
### Workflow Frontend CI - Test & SonarCloud Analysis
### 1. Introduction au workflow
Ce workflow exécute des étapes automatisées pour vérifier la qualité du code frontend. Il est divisé en trois sections :
   - Tests de Frontend : Exécute les tests et génère un rapport de couverture de code.
   - Analyse SonarCloud : Évalue la qualité du code via SonarCloud.
   - Image Docker : Crée une image Docker du frontend et la pousse vers Docker Hub.

### 2. En-tête du Workflow
```yaml
name: Frontend CI - Test & Sonarcloud Analysis
```

- name : Nom du workflow. Ici, "Frontend CI - Test & Sonarcloud Analysis" indique que ce workflow s'applique au frontend et exécute des tests et une analyse SonarCloud.
  
```yaml
on:
  push:
    branches: [ "develop", "main" ]
    paths:
      - 'front/**'
  pull_request:
    types: [opened, synchronize, reopened]
    branches: [ "main" ]
    paths:
      - 'front/**'
```
- on : Définit les déclencheurs du workflow, c’est-à-dire les événements qui lancent le workflow.
  - push : Ce workflow s’exécute lorsqu'un code est poussé dans les branches develop ou main et concerne uniquement les modifications dans le dossier front/.
  - pull_request : Il s’exécute également lorsque des pull requests sont ouvertes, synchronisées (mises à jour) ou rouvertes vers la branche main, avec des modifications dans front/.

### 3. Jobs - Section de tests pour le frontend (tests-front)
```yaml
jobs:
  tests-front:
    runs-on: ubuntu-latest
```
- jobs : Les actions à exécuter. Chaque tâche est une unité de travail dans le workflow.
    - tests-front : Un premier job qui gère les tests frontend.
    - runs-on : Précise le système d’exploitation sur lequel exécuter le job. ubuntu-latest est une version récente d'Ubuntu
    
```yaml
steps:
  - name: checkout code
    uses: actions/checkout@v4
```

- steps : Liste des actions dans le job.
  - checkout code : Télécharge le code du projet depuis le dépôt GitHub. actions/checkout@v4 est une action standard pour cloner le dépôt dans l'environnement de travail.

```yaml
- name: setup node.js
  uses: actions/setup-node@v3
  with:
    node-version: 16
```
- setup node.js : Installe Node.js, version 16, nécessaire pour exécuter les tests et les scripts frontend. actions/setup-node@v3 est une action qui gère l'installation de Node.js dans la version spécifiée.
  
```yaml
- name: install dependencies
working-directory: front
run: npm install
```
- install dependencies : Installe les dépendances nécessaires à l’exécution du projet. working-directory: front indique que cette étape s'exécute dans le dossier front. npm install télécharge toutes les dépendances listées dans package.json.

```yaml
- name: run tests
  working-directory: front
  run: npm run test -- --code-coverage --browsers=ChromeHeadless --watch=false
```
- run tests : Exécute les tests du projet. Le drapeau --code-coverage génère un rapport de couverture de code. --browsers=ChromeHeadless utilise Chrome en mode sans interface graphique, et --watch=false exécute les tests une fois sans les surveiller pour des changements.

```yaml
- name: upload coverage report
  uses: actions/upload-artifact@v3
  with:
    name: coverage-report
    path: front/coverage
```
- upload coverage report : Sauvegarde le rapport de couverture pour qu’il soit accessible par d'autres étapes ou workflows. name: coverage-report attribue un nom à cet artefact, et path: front/coverage précise son emplacement.

### 4. Job pour l’analyse SonarCloud (sonarcloud)

```yaml
sonarcloud:
  name: SonarCloud
  needs: tests-front
  runs-on: ubuntu-latest
```
- sonarcloud : Job pour analyser le code avec SonarCloud.
  - needs: tests-front signifie que ce job dépend du succès du job tests-front.
  - runs-on : Exécute le job sur ubuntu-latest.
 
```yaml
steps:
  - uses: actions/checkout@v4
    with:
      fetch-depth: 0
```
- checkout : Cloner le dépôt. fetch-depth: 0 indique un clonage complet pour assurer que SonarCloud analyse l’ensemble des commits et du code.

```yaml
- name: download coverage report
  uses: actions/download-artifact@v3
  with:
    name: coverage-report
    path: front/coverage
```
- download coverage report : Récupère le rapport de couverture généré lors du job tests-front. path: front/coverage précise où télécharger le rapport.

```yaml
- name: SonarCloud Scan
  uses: SonarSource/sonarcloud-github-action@master
  with:
    projectBaseDir: front
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
    SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
```
- SonarCloud Scan : Analyse le code avec SonarCloud pour évaluer la qualité du code et détecter les éventuels problèmes.
  - projectBaseDir : Répertoire racine du projet frontend.
  - GITHUB_TOKEN : Jeton fourni par GitHub pour accéder au dépôt.
  - SONAR_TOKEN : Jeton d’authentification SonarCloud pour sécuriser l’accès.

### 5. Job pour la création d'une image Docker (docker)

```yaml
docker:
  name: Docker Image Front
  needs: sonarcloud
  runs-on: ubuntu-latest
```
- docker : Job pour créer et pousser une image Docker du frontend.
  - needs: sonarcloud signifie que ce job dépend du succès du job sonarcloud.
  - runs-on : Exécute ce job sur ubuntu-latest.
   
**Étapes pour construire et publier l'image Docker**

```yaml
steps:
   - name: checkout code
     uses: actions/checkout@v4
```
- checkout code : Clone le code pour préparer la création de l'image Docker.

```yaml
steps:
 - name: Log in to Docker Hub
   uses: docker/login-action@v2
   with:
    username: ${{ secrets.DOCKER_USERNAME }}
    password: ${{ secrets.DOCKER_PASSWORD }}
```
- Log in to Docker Hub : Se connecte à Docker Hub avec des identifiants sécurisés, nécessaires pour pousser une image.
  - username et password : Identifiants fournis via les secrets GitHub.
    
```yaml
steps:
 - name: Build Docker image
   working-directory: front
   run: docker build -t ${{ secrets.DOCKER_USERNAME }}/bobapp-front:latest .
```
- Build Docker image : Construit une image Docker à partir du Dockerfile dans le dossier front.
  - -t : Attribue un nom et une balise (latest) à l'image Docker.
    
```yaml
steps:
 - name: Push Docker image
   working-directory: front
   run: docker push ${{ secrets.DOCKER_USERNAME }}/bobapp-front:latest
```
- Push Docker image : Envoie l'image Docker vers Docker Hub.
    
### Workflow Backend CI - Test & SonarCloud Analysis

Description : Ce workflow assure la qualité du code backend en exécutant des tests et en réalisant une analyse de code avec SonarCloud. Il crée également une image Docker du backend.
Étapes impliquées :
  1. Déclencheurs :
     
    Sur push sur la branche develop et main pour les changements dans back/**.
    
    Sur pull_request vers main pour les changements dans back/**.
    
  2. Étapes de CI/CD :
     
    Clonage du code : Utilisation de actions/checkout@v2.
    
    Installation des dépendances : mvn clean install pour installer les dépendances backend.
    
    Exécution des tests : mvn clean test.
    
    Génération du rapport de couverture : mvn jacoco:report.
    
    Upload du rapport de couverture : actions/upload-artifact@v4 pour stocker le rapport Jacoco.

  3. Étapes d’analyse avec SonarCloud :
     
    Clonage du code : actions/checkout@v4.

    Configuration de JDK 17 : actions/setup-java@v4.
    
    Cache des packages SonarCloud et Maven : actions/cache@v4 pour stocker le cache.
    
    Analyse avec SonarCloud : Exécution du plugin Maven Sonar pour analyser la qualité du code.
    
  4. Étapes de création de l'image Docker :
     
    Connexion à Docker Hub : docker/login-action@v2.
    
    Build de l’image Docker : docker build -t ${{ secrets.DOCKER_USERNAME }}/bobapp-back:latest.
    
    Push de l’image Docker : docker push ${{ secrets.DOCKER_USERNAME }}/bobapp-back:latest.
    
### Indicateurs de Performance Clés (KPI)

Les KPIs suivants sont définis pour surveiller la performance et la qualité des builds CI/CD :

Couverture de code minimale : Maintenir une couverture de code de 80% pour garantir que la majorité du code est testé.
Taux de réussite des builds : Objectif de 95% pour garantir la fiabilité et la stabilité du code.

Analyse des Métriques
Les métriques initiales de qualité et de performance des workflows sont les suivantes :

Couverture de code :

Backend : 38.8%

Frontend : 83.3%

Temps moyen de build :

Backend : 2 minutes

Frontend : 3 minutes

Commentaires : Bien que la couverture de code soit excellente côté frontend, le backend pourrait bénéficier de tests supplémentaires pour se rapprocher de l'objectif de 80%.


### Notes de Qualité SonarCloud

1. Backend
   - Sécurité : Note A (0 vulnérabilités détectées)
   - Fiabilité : Note D (1 problème de fiabilité détecté)
   - Maintenabilité : Note A (8 problèmes mineurs de maintenabilité)
   - Note E (0% des points critiques de sécurité revus)
   - Duplications : 0.0%
   
2. Frontend
   - Note A (0 vulnérabilités détectées)
   - Fiabilité : Note A (aucun problème de fiabilité)
   - Note A (4 problèmes mineurs de maintenabilité)
   - Note A (100% des points critiques de sécurité revus)
   - Duplications : 0.0%
     
### Retours Utilisateurs

- Retours positifs :

    Couverture de code : Le frontend atteint une couverture de code de 83.3%, ce qui dépasse l’objectif de 80%.
  
    Fiabilité des workflows : Le retour est positif pour les tests stables et le temps de build rapide.
  
- Suggestions d’amélioration :
  
    Tests end-to-end : Ajouter des tests end-to-end pour mieux couvrir les scénarios d’utilisation.
  
    Documentation : Améliorer la documentation pour clarifier le fonctionnement de chaque étape CI/CD.
  
## Conclusion et Recommandations

Ce document a décrit les workflows CI/CD, les KPIs et l’analyse des métriques de qualité. Les retours d’utilisateurs indiquent des axes d’amélioration :

Renforcer la couverture de test backend.
Ajouter des tests end-to-end pour couvrir des cas d’utilisation plus complets.
Ces recommandations contribueront à garantir une qualité de code optimale et une meilleure satisfaction des utilisateurs.
