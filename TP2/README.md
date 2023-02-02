# Question 2.1 

Ce sont des bibliothèques Java qui vous permettent d'exécuter un ensemble de conteneurs Docker pendant les tests.

# Question 2.2 et 2.3

Décrit ci-dessous dans notre compte-rendu.

# Création du github
- Connexion sur mon compte github
- Création d'un nouveau repo
- Je le clone sur mon poste 
- Je met dedans tous mes fichiers que j'ai fait pendant le TP1 et tout le nécéssaire au bon déroulement du docker compose

# First step in CI
Le but étant d'avoir une vérification en continu du bon déroulement et de la bonne exécution afin de se rendre compte au plus vite des erreurs. C'est l'intégration continue. On utilise git action pour lancer des pipeline de build et de test à chaque push sur une branche.
- Ajout d'un fichier main.xml dans un dossier .gihut/workflows/ à la racine du repo
``` yml
name: CI devops 2023
on:
  #to begin you want to launch this job in main and develop
  push:
    branches: main
  pull_request:

jobs:
  test-backend: 
    runs-on: ubuntu-22.04
    steps:
     #checkout your github code using actions/checkout@v2.5.0
      - uses: actions/checkout@v2.5.0

     #do the same with another action (actions/setup-java@v3) that enable to setup jdk 17
      - name: Set up JDK 17
        uses: actions/setup-java@v3
        with:
          java-version: '17'
          distribution: 'adopt'

     #finally build your app with the latest command
      - name: Build and test with Maven
        run: mvn -f ./TP1/simple-api-student-main/pom.xml clean verify 
```
- Le fichier est lancé à chaque push sur la branche main
- On ajoute l'utilisation d'une action java 17
- Le fichier représente une pipeline qui exécute la commande `mvn clean verify` pour build notre app. Le fichier main.xml est exécuté dans github comme s'il était éxécuté à partir de la racine. Donc on met un path pour aller chercher notre .xml à partir de la racine.
![alt text](/TP2/img/test-job.PNG)

# First step in CD
Il est aussi important de faire en sorte que notre code se déploie continuellemnt à chaque push sur une branche. Notre but est de faire comme un docker compose afin d'avoir nos images à partir des Dockerfile. C'est notre déploiement continu, voici le job qu'on va rajouter à notre main.yml

```yml
# define job to build and publish docker image
  build-and-push-docker-image:
    needs: test-backend
    # run only when code is compiling and tests are passing
    runs-on: ubuntu-22.04

    # steps to perform in job
    steps:
      - name: Checkout code
        uses: actions/checkout@v2.5.0

      - name: Login to DockerHub
        run: docker login -u lvcascds -p dckr_pat_Wcop_UGAQ8T7XGrfDs0BqkvPBcQ

      - name: Build image and push backend
        uses: docker/build-push-action@v3
        with:
          # relative path to the place where source code with Dockerfile is located
          context: ./TP1/simple-api-student-main
          # Note: tags has to be all lower-case
          tags:  lvcascds/tp-devops:simple-api
          # build on feature branches, push only on main branch
          push: ${{ github.ref == 'refs/heads/main' }}

      - name: Build image and push database
        # DO the same for database
        uses: docker/build-push-action@v3
        with:
          # relative path to the place where source code with Dockerfile is located
          context: ./TP1/db
          # Note: tags has to be all lower-case
          tags:  lvcascds/tp-devops:database
          # build on feature branches, push only on main branch
          push: ${{ github.ref == 'refs/heads/main' }}

      - name: Build image and push httpd
        # DO the same for httpd
        uses: docker/build-push-action@v3
        with:
          # relative path to the place where source code with Dockerfile is located
          context: ./TP1/httpd
          # Note: tags has to be all lower-case
          tags:  lvcascds/tp-devops:httpd
          # build on feature branches, push only on main branch
          push: ${{ github.ref == 'refs/heads/main' }}
```
Pour que ça marche coté dockerhub :
- Création d'un compte dockerhub avec le username
- Génération dun token qu'on ajoute à l'endroit de la connexion
- On ajoute un repo `tp-devops` à notre docker hub puis on est prêt à faire tourner notre nouveau job dans la pipeline
Ce que fait notre nouveau job : 
- Se connecte à docker hub
- Pour les 3 étapes d'après : 
- - Se base le dossier ou se trouve les Dockerfile
- - Utilise les actions de build de docker
- - génère une image dans le repo `tp-devops` avec un tag prore à l'image
![alt text](/TP2/img/pipeline.PNG)
![alt text](/TP2/img/job.PNG)
![alt text](/TP2/img/dockerhub.PNG)

# Setup Quality Gate

- Création d'un compte sonarCloud à partir de github
- On lie le repo de notre github dans sonarCloud pour créer notre organisation
- Génération d'un token de connexion à partir de notre compte.
- On remplace la ligne de run de Maven par la ligne donnée dans l'énoncé en remplaçant certaines valeurs par nos données, ce qui nous donne la commande suivante : `mvn -B verify sonar:sonar -Dsonar.projectKey=LucasCoudrais_S8DevOps -Dsonar.organization=lucascoudrais -Dsonar.host.url=https://sonarcloud.io -Dsonar.login=a3e9c30740f5c476564a95cea2f8be433910e2b8  --file ./TP1/simple-api-student-main/pom.xml`
- On push pour lancer la nouvelle pipeline avec notre nouveau main.
- On voit maintenant notre analyse de qualité du code dans Sonarcloud (avec le code coverage et les duplications exprimés en pourcentage par exemple).
![alt text](/TP2/img/Sonar.PNG)
