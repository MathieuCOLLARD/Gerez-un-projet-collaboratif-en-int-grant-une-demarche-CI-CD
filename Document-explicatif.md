# Document explicatif sur les GitHub Actions mises en place
Introduction
Ce document décrit les GitHub Actions configurées pour le projet, présente les indicateurs de performance clés (KPIs), analyse les premières métriques recueillies et prend en compte les retours des utilisateurs pour identifier les améliorations nécessaires.

## Workflows GitHub Actions
### Workflow Frontend CI - Test & SonarCloud Analysis
Description : Ce workflow assure la qualité du code du frontend en exécutant des tests et en effectuant une analyse de code avec SonarCloud.
Il crée également une image Docker du frontend.
Étapes impliquées :
  1. Déclencheurs :
  
    - Sur push sur les branches develop et main pour les changements dans front/**.
  
    - Sur pull_request vers main pour les changements dans front/**.
  2. Étapes de CI/CD :
     
    Clonage du code : Utilisation de actions/checkout@v4 pour cloner le dépôt.

    Installation de Node.js : Installation de Node.js 16 avec actions/setup-node@v3.
    
    Installation des dépendances : npm install pour installer les dépendances front.
    
    Exécution des tests : npm run test -- --code-coverage --browsers=ChromeHeadless --watch=false
    
    Upload du rapport de couverture : Utilisation de actions/upload-artifact@v3 pour stocker le rapport de couverture.

  3. Étapes d’analyse avec SonarCloud :
     
    Clonage du code : actions/checkout@v4.
    
    Téléchargement du rapport de couverture : Utilisation de actions/download-artifact@v3 pour récupérer le rapport de couverture (fichier lcov.info) généré lors des tests.
    
    Scan avec SonarCloud : Utilisation de SonarSource/sonarcloud-github-action@master.
    
  4. Étapes de création de l'image Docker :
     
    Connexion à Docker Hub : Utilisation de docker/login-action@v2.
    
    Build de l’image Docker : docker build -t ${{ secrets.DOCKER_USERNAME }}/bobapp-front:latest.
    
    Push de l’image Docker : docker push ${{ secrets.DOCKER_USERNAME }}/bobapp-front:latest.
    
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

Backend : 

Frontend : 

Temps moyen de build :

Backend :  minutes

Frontend :  minutes

Commentaires : Le backend est bien couvert, mais le frontend pourrait bénéficier de tests supplémentaires pour améliorer la couverture.

### Retours Utilisateurs

- Retours positifs :

    La couverture du code est jugée satisfaisante et les tests sont stables.
  
    Bonne performance des workflows avec un temps de build rapide.
  
- Suggestions d’amélioration :
  
    Ajouter des tests end-to-end pour mieux couvrir les scénarios d’utilisation.
  
    Améliorer la documentation pour expliquer le fonctionnement de chaque étape CI/CD.
  
## Conclusion et Recommandations
Ce document a décrit les workflows CI/CD, les KPIs et l’analyse des métriques de qualité. Les retours d’utilisateurs indiquent des axes d’amélioration :

Renforcer la couverture de test frontend.
Ajouter des tests end-to-end pour couvrir des cas d’utilisation plus complets.
Ces recommandations contribueront à garantir une qualité de code optimale et une meilleure satisfaction des utilisateurs.
