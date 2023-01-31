# Création du github
- Connexion sur mon compte github
- Création d'un nouveau repo
- Je le clone sur mon poste 
- Je met dedans tous mes fichiers que j'ai fait pendant le TP1 et tout le nécéssaire au bon déroulement du docker compose

# Ajout de la première pipeline
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