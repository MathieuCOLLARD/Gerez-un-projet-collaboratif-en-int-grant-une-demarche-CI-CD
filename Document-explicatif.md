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
      
**Étapes pour exécuter les tests**    
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
    
**Étapes pour l’analyse SonarCloud** 
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
    
### Backend CI - Test & SonarCloud Analysis Workflow
### 1. Introduction au workflow

Ce workflow exécute des actions automatisées pour vérifier la qualité du code backend. Il est divisé en trois sections :

- Tests Backend : Exécute les tests et génère un rapport de couverture de code.
- Analyse SonarCloud : Analyse la qualité du code via SonarCloud.
- Image Docker : Crée une image Docker pour le backend et la pousse vers Docker Hub.

### 2. En-tête du Workflow

```yaml
name: Backend CI - Test & Sonarcloud Analysis
```
- name : Nom du workflow, ici "Backend CI - Test & Sonarcloud Analysis" pour indiquer que ce workflow est dédié au backend et inclut des tests et une analyse SonarCloud.
  
```yaml
on:
  push:
    branches: [ "develop", "main" ]
    paths:
      - 'back/**'
  pull_request:
    types: [opened, synchronize, reopened]
    branches: [ "main" ]
    paths:
      - 'back/**'
```
- on : Définit les déclencheurs du workflow, c’est-à-dire les événements qui lancent le workflow.
   - push : Le workflow est lancé lorsqu’un code est poussé dans les branches develop ou main et que les modifications concernent uniquement le dossier back/.
   - pull_request : Il est également lancé lors de l’ouverture, de la synchronisation ou de la réouverture d’une pull request vers la branche main, avec des modifications dans back/.

### 3. Jobs - Section de tests pour le backend (tests-backend)

```yaml
jobs:
  tests-backend:
      name: Build & Test Backend
      runs-on: ubuntu-latest
```
- jobs : Liste des actions à exécuter. Chaque job est une unité de travail dans le workflow.
   - tests-backend : Le premier job gère les tests backend.
   - runs-on : Indique le système d’exploitation où exécuter le job. ubuntu-latest correspond à une version récente d'Ubuntu.

**Étapes pour exécuter les tests**
```yaml
defaults:
  run:
    working-directory: ${{ github.workspace }}/back
```
- defaults : Définit les paramètres par défaut pour toutes les étapes du job. Ici, working-directory: ${{ github.workspace }}/back indique que toutes les étapes de ce job s'exécutent dans le dossier back.
  
```yaml
steps:
   - name: Git Checkout Backend
    uses: actions/checkout@v2     # checkout the repo
```
- steps : Liste des actions dans le job.
   - Git Checkout Backend : Clone le code source depuis le dépôt GitHub pour que le job puisse l'utiliser. actions/checkout@v2 est une action GitHub standard qui permet de récupérer le dépôt dans l'environnement de travail.

```yaml
   - name: Install Backend Dependencies
    run: mvn clean install        # install packages
```
- Install Backend Dependencies : Installe les dépendances Java Maven nécessaires au projet backend. mvn clean install exécute Maven pour télécharger et installer les dépendances listées dans le fichier pom.xml.
  
```yaml
   - name: Execute Backend Tests
    run: mvn clean test
```
- Execute Backend Tests : Exécute les tests backend définis dans le projet Maven. mvn clean test nettoie l'environnement et exécute tous les tests présents dans le dossier src/test.
  
```yaml
   - name: Creation Backend Tests Report
    run: mvn jacoco:report
```
- Creation Backend Tests Report : Génère un rapport de couverture de code avec JaCoCo. mvn jacoco:report produit un rapport pour analyser quelles parties du code sont couvertes par les tests.
  
```yaml
   - name: Backend Code Coverage Report
    uses: actions/upload-artifact@v4
    with:
      name: Jacoco Code Coverage
      path: ./back/target/site/jacoco/
```
- Backend Code Coverage Report : Sauvegarde le rapport de couverture de code généré pour le rendre accessible. name: Jacoco Code Coverage nomme l'artefact, et path: ./back/target/site/jacoco/ indique où se trouve le rapport de couverture.

### 4. Job pour l’analyse SonarCloud (sonarcloud)

```yaml
  sonarcloud:
    name: Build and analyze
    runs-on: ubuntu-latest
```
- sonarcloud : Job qui analyse le code backend avec SonarCloud.
   - runs-on : Exécute le job sur ubuntu-latest.

**Étapes pour l’analyse SonarCloud**
```yaml
 steps:
   - uses: actions/checkout@v4
     with:
       fetch-depth: 0
```
- checkout : Clone le dépôt pour que SonarCloud puisse accéder au code complet. fetch-depth: 0 désactive le clonage en mode "shallow" pour garantir l'accès à l'historique complet des commits, ce qui améliore la précision de l'analyse SonarCloud.

```yaml
- name: Set up JDK 17
  uses: actions/setup-java@v4
  with:
    java-version: 17
    distribution: 'zulu'
```
- Set up JDK 17 : Installe Java Development Kit (JDK) version 17, nécessaire pour exécuter les outils Maven et SonarCloud. distribution: 'zulu' indique une distribution Java open-source alternative.

```yaml
- name: Cache SonarCloud packages
  uses: actions/cache@v4
  with:
    path: ~/.sonar/cache
    key: ${{ runner.os }}-sonar
    restore-keys: ${{ runner.os }}-sonar
```
- Cache SonarCloud packages : Met en cache les fichiers SonarCloud pour optimiser les futures exécutions. Cela évite de télécharger à chaque fois les mêmes fichiers et accélère le workflow.

```yaml
- name: Cache Maven packages
  uses: actions/cache@v4
  with:
    path: ~/.m2
    key: ${{ runner.os }}-m2-${{ hashFiles('**/pom.xml') }}
    restore-keys: ${{ runner.os }}-m2
```
- Cache Maven packages : Met en cache les dépendances Maven pour optimiser les futures exécutions. path: ~/.m2 est le répertoire où Maven stocke ses dépendances.

```yaml
- name: Build and analyze
  working-directory: back
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
    SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
  run: mvn -B verify org.sonarsource.scanner.maven:sonar-maven-plugin:sonar -Dsonar.projectKey=MathieuCOLLARD_Gerez-un- projet-collaboratif-en-int-grant-une-demarche-CI-CD -Dsonar.projectName=BobappBack
```
- Build and analyze : Exécute une analyse de code avec le plugin Sonar pour Maven.
   - GITHUB_TOKEN : Jeton GitHub pour accéder aux informations du dépôt.
   - SONAR_TOKEN : Jeton pour se connecter à SonarCloud.
   - -Dsonar.projectKey et -Dsonar.projectName sont les identifiants pour le projet sur SonarCloud.

### 5. Job pour la création d'une image Docker (docker)

```yaml
docker:
 name: Docker Image Back
 needs: sonarcloud
 runs-on: ubuntu-latest
```
- docker : Job pour créer et pousser une image Docker du backend.
   - needs: sonarcloud indique que ce job dépend du succès du job sonarcloud.
   - runs-on : Exécute le job sur ubuntu-latest.

**Étapes pour construire et publier l'image Docker**
```yaml
 steps:
    - name: checkout code
      uses: actions/checkout@v4
```
- checkout code : Clone le code pour préparer la création de l'image Docker.
  
```yaml
 - name: Log in to Docker Hub
   uses: docker/login-action@v2
   with:
    username: ${{ secrets.DOCKER_USERNAME }}
    password: ${{ secrets.DOCKER_PASSWORD }}
```
- Log in to Docker Hub : Se connecte à Docker Hub avec des identifiants sécurisés pour pouvoir pousser l'image Docker. username et password utilisent les identifiants stockés dans les secrets GitHub.

```yaml
 - name: Build Docker image
   working-directory: back
   run: docker build -t ${{ secrets.DOCKER_USERNAME }}/bobapp-back:latest .
```
- Build Docker image : Crée une image Docker à partir du Dockerfile dans le dossier back.
   - -t : Donne un nom et une balise latest à l'image Docker.
     
```yaml
- name: Push Docker image
  working-directory: back
  run: docker push ${{ secrets.DOCKER_USERNAME }}/bobapp-back:latest
```
- Push Docker image : Envoie l'image Docker créée vers Docker Hub. Cela permet à l'image d'être accessible pour le déploiement ou d'autres environnements. L’image bobapp-back:latest est taguée comme "latest" pour indiquer qu’elle est la version la plus récente de l’application backend.
  
## Indicateurs de Performance Clés (KPI)

Les KPIs suivants sont définis pour surveiller la performance et la qualité des workflows CI/CD et garantir la stabilité et la sécurité de l'application dans un contexte de production.

- Couverture de code minimale : Maintenir une couverture de code supérieure à 80 % pour s'assurer que la majorité du code est testé, ce qui limite les risques de régressions.
- Taux de réussite des builds : Objectif de 95 % pour garantir la fiabilité et la stabilité du code après chaque modification.
- Nouveaux "blocker issues" : Pas de "blocker issues" sur les nouveaux changements de code, afin d’assurer une qualité élevée et de minimiser les failles de sécurité et les erreurs critiques.
- Temps de build moyen : Suivre le temps de build pour identifier les builds trop longs et optimiser la rapidité des déploiements.

## Analyse des Métriques
### Couverture de code

**Frontend - Couverture de code (Image de l’index.html du rapport de couverture)**
![image](https://github.com/user-attachments/assets/519f586a-e824-4553-a4b6-9268d1061a45)

Commentaire : Pour cette partie du code, le coverage est satisfaisant mais il est toujours possible de l'améliorer. Le rapport de couverture permet également de repérer des fichiers ou des fonctions spécifiques qui nécessitent des tests supplémentaires.

**Frontend - Métriques SonarCloud**
![image](https://github.com/user-attachments/assets/90bd51ad-cf88-4276-93f1-82def1960bff)

Commentaires : Le rapport SonarCloud pour le frontend montre des indicateurs de qualité tels que la sécurité et la fiabilité. Voici les points essentiels qu'il est important de surveiller :

- Fiabilité : Une note de "A" pour la fiabilité signifie qu'aucun problème critique ou bug majeur n'a été détecté.
- Maintenabilité : Vérifier la colonne des "code smells" ou problèmes de maintenabilité pour s'assurer qu'il n'y a pas d’accumulation de problèmes techniques. Bien que ces problèmes ne soient pas bloquants, les surveiller est important pour garder un code de qualité.
- Sécurité : Une note "A" dans cette section indique que le code est exempt de vulnérabilités de sécurité.    

**Backend - Couverture de code (Image de l’index.html du rapport de couverture)**
![image](https://github.com/user-attachments/assets/ba11479a-f56d-4475-8c8f-2a1a3191d50a)

Commentaire : La couverture de code pour le backend est insuffisante car nettement inférieure à 80%. La faible couverture suggère que plusieurs parties du code ne sont pas testées et pourraient contenir des bugs non détectés. Il serait judicieux de cibler les fichiers ou les fonctions avec une faible couverture pour des tests unitaires supplémentaires.

**Backend - Métriques SonarCloud**

![image](https://github.com/user-attachments/assets/02c14b6b-0c52-47a5-8add-8fea9a6281b0)

Commentaires : Le rapport SonarCloud du backend fournit des informations importantes pour l’amélioration de la qualité :

- Sécurité : Une note de "A" indique que le code est sécurisé et sans vulnérabilités critiques.
- Fiabilité : Une note de "D" en fiabilité indique des problèmes potentiels qui peuvent entraîner des dysfonctionnements ou des bugs. Il est recommandé de corriger ces problèmes pour éviter des échecs en production.
- Maintenabilité : Les "code smells" (problèmes mineurs de maintenabilité) sont notés "A", ce qui est excellent et montre une bonne structure de code.
- Coverage : Seulement 38.8% du code est convert par les tests.
- Duplications de code : Une faible duplication, comme 0 %, est idéale et montre que le code n’a pas de sections répétées inutiles.

## Suggestions de KPIs additionnels
Pour garantir la qualité, il est utile de s’appuyer sur les Quality Gates par défaut de SonarCloud (Sonar Way), qui sont des seuils de qualité prédéfinis pour chaque métrique. Les KPIs suivants peuvent être établis :

1. Bugs : Pas de nouveaux bugs. Tout bug dans le code doit être corrigé avant d'approuver un build.
2. Fiabilité : Une note de A pour la fiabilité est requise, indiquant l'absence de problèmes critiques susceptibles d'affecter la stabilité.
3. Vulnérabilités : Pas de nouvelles vulnérabilités. La sécurité de l'application reste ainsi optimale.
4. Sécurité : Une note de A en sécurité est exigée, ce qui confirme que le code ne comporte pas de vulnérabilités importantes.
5. Dette technique : Le code doit maintenir une dette technique limitée. Ceci est évalué par une note A en maintenabilité, garantissant que les modifications n'introduisent pas de complexité excessive ou de maintenance supplémentaire.
6. Points chauds de sécurité : Tous les points chauds (ou security hotspots) doivent être examinés et, le cas échéant, traités pour limiter les risques de sécurité latents.
7. Couverture des tests : La couverture de code doit être supérieure ou égale à 80 %. Cela permet de vérifier que le code introduit est bien testé et limite les régressions.
8. Duplication de code : Le pourcentage de lignes de code dupliquées ne doit pas dépasser 3 %. Une faible duplication réduit la complexité et simplifie la maintenance.
   
Suivre ces Quality Gates pourrai permettre de rapidement évaluer la qualité du code. Un statut "passed" dans SonarCloud Quality Gate pour ces KPI indiquera que le code est en bonne santé, tandis qu’un statut "failed" devra déclencher une investigation et correction.
