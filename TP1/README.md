# Database 
On créé le Dockerfile suivant 
```
FROM postgres:14.1-alpine

ENV POSTGRES_DB=db \
   POSTGRES_USER=usr \
   POSTGRES_PASSWORD=pwd

COPY CreateScheme.sql /docker-entrypoint-initdb.d/
COPY InsertData.sql /docker-entrypoint-initdb.d/
```

## Création Image 

Puis on run la commande ` docker build -t lucas/database . ` </br>
Elle permet de : 
 - Récupérer l'image postgres et la pull si on l'a pas 
 - Créé une image avec une tag `lucas/database` à partir de notre Dockerfile avec les variables d'environnement qui configurent le nom de notre base de données, son login et son mot de passe
 - A la création de notre image docker va exécuter nos deux script de création de schéma et d'insertion des données (qu'on aura mis au meme endroit que le Dockerfile) qu'on a copié dans le dossier `/docker-entrypoint-initdb.d/` </br>

 Nous avons l'image de notre base de donnée, il nous faut maintenant un outil permettant d'intéragir avec, nous allons utiliser adminer. Son image est déjà existant avec docker. On va donc s'intéresser aux conteneurs. </br>

## Création réseau 
 Il va falloir que notre adminer puisse accéder à notre base de donnée, hors les conteneurs ne sont pas fait pour communiquer entre eux, ainsi il faudra mettre nos conteneurs dans un même réseau. Création d'un réseau `app-network` avec la ligne : `docker network create app-network` </br>

## Création Conteneur  
 Ensuite on lance notre conteneur en arrière plan à partir de l'image qu'on à fait en lui donnant notre réseau et un nom avec la ligne suivante : `docker run -d --name database --network app-network lucas/database` </br>
Puis on fait de même pour notre conteneur depuis l'image adminer en lui donnant une redirection de port pour qu'on puisse y accéder sur notre machine : ` docker run -d --network app-network --name adminer -p "8090:8080" adminer ` </br>

On a maintenant accès au adminer pour se donnecter a notre DB. En nom de serveur on pourra mettre l'ip du conteneur de la databse ou même son nom car docker gère tout seul une résolution de nom de domaine. Notre base de donnée est déjà fournie grace au DockerFile.

## Association volume

Afin de ne pas perdre les modification quand on perd notre conteneur, on doit associer un volume (une mémoire plus durable, ici celle du pote) afin que les changements se gardent. On supprimer donc le conteneur de la database avec cette ligne : `docker rm -f database `</br> Puis on le relance en lui associant un volume avec cette ligne : `docker run -d --name database --network app-network -v /my/own/datadir:/var/lib/postgresql/data lucas/database` </br>
On fait une modif en base puis on stoppe le container, on le relance et on a toujours la modif.

# BackEnd API
## HelloWorld
On créé notre fichier Main.java dans un dossier. Dans le meme dossier, on créé le Dockerfile suivant :
```
FROM openjdk:19-alpine

COPY Main.java Main.java
RUN javac Main.java

CMD ["java", "Main"]
```
 - On utilise le jdk 19-alpine
 - On met notre fichier java dans notre image avec le `COPY`
 - On exécute la commande `javac Main.java` à la construction de l'image qui va compliler notre ficher `.java` en un fichier `.class` exécutable.
 - CMD permet de jouer une commande qui va etre executé quand on run l'image généré par note Dockerfile. Donc quand on run notre image pour en faire un container, on va exéduter notre `Main.class` </br>

 Build de l'image à partir du docker en se placant dans son dossier : `docker build -t hello-world-java .` </br>
 Run de l'image préalablement créé : `docker run hello-world-java`. Contrairement aux lancements d'avant, on ne la lance pas en arrière plan, on ne lui donne pas de nom, inutile car le but n'est pas de conserver le container, l'image ne va pas perdurer, elle va juste éxécuter un fichier qui va print un ligne.

## API
 - Génération d'un projet spring boot avec les paramètres donnés
 - Ajout de la classe fournie `GreetingController` dans notre projet. On remarque qu'il est dans un package controller au début. On met donc notre class dans un dossier `controller` qui se trouve au même niveau que `SimpleapiApplication.java ` généré automatiquement
 ```
 # Build
FROM maven:3.8.6-amazoncorretto-17 AS myapp-build
ENV MYAPP_HOME /opt/myapp
WORKDIR $MYAPP_HOME
COPY pom.xml .
COPY src ./src
RUN mvn package -DskipTests

# Run
FROM amazoncorretto:17
ENV MYAPP_HOME /opt/myapp
WORKDIR $MYAPP_HOME
COPY --from=myapp-build $MYAPP_HOME/target/*.jar $MYAPP_HOME/myapp.jar

ENTRYPOINT java -jar myapp.jar
 ```
 - On met ce Dockerfile dans le dossier `simpleapi` au même niveau que `src`
 - Puis on build notre image avec la commande : `docker build -t lucas/java-api .` en étant dans `simpleapi`
 - Run de l'image avec `docker run -d --name java-api lucas/java-api`
 - Problème : Pour accéder à l'api à travers notre navigateur, il nous faut un port à mettre derrière notre `localhost`. On doit faire un redirection de port dans notre commande. Main sur quel port tourne notre api dans notre docker ? 
 - Solution : On peut voir les logs de l'éxécution de notre container avec la commande `docker logs java-api ` ainsi, on peut voir que l'api tourne acutellement sur le port `8080` donc on supprime le container puis on le relance avec la redirection de port : `docker run -d --name java-api -p "8081:8080" lucas/java-api` </br>
 ![alt text](/TP1/img/java-api.png)

 ## Multistage Build

Le multistage build Docker est une fonctionnalité de Docker qui permet de construire une image Docker en utilisant plusieurs étapes de construction distinctes dans un seul fichier Dockerfile. Chaque étape de construction peut utiliser une image différente en tant que base et les résultats de cette étape peuvent être copiés dans une nouvelle image. 

Cela permet de séparer les différentes phases de construction et de ne conserver que les composants nécessaires dans l'image finale, ce qui réduit la taille de l'image et améliore la sécurité. Le multistage build est très utile pour les applications qui nécessitent un environnement de build séparé pour la compilation et l'exécution, ainsi que pour les applications qui nécessitent des dépendances qui ne sont pas nécessaires en production.

 ## API avec DB
- On récupère le projet du github puis dans le dossier `simple-api-student-main` on rajoute le même Dockerfile que précédemment.
- On relance nos container database et adminer avec la commande `docker start <Nom_Container>`. On se retrouve avec les infos suivantes : 
```
CONTAINER ID   IMAGE               COMMAND                  CREATED         STATUS          PORTS                                       NAMES
ae93953f02b1   lucas/database      "docker-entrypoint.s…"   18 hours ago    Up 30 minutes   5432/tcp                                    database
759ec93f3c0e   adminer             "entrypoint.sh php -…"   19 hours ago    Up 30 minutes   0.0.0.0:8090->8080/tcp, :::8090->8080/tcp   adminer
```
- Adminer tourner déjà sur le port 8080 de notre dokcer, c'est le port que va prendre l'api de base, il faut faire en sorte que notre api tourne, par défaut sur un autre port par exemple le `8081`. On rajoute dans le fichier `application.yml`:
```
server:
  port : 8081
  ```
- Dans ce fichier, on renseigne également les infos de connexion à la BDD avec les ligne suivantes : 
```
spring:
  datasource:
    url: jdbc:postgresql://database/db
    username: usr
    password: pwd
```
- Puis on peut build l'image en étant dans `simple-api-student-main` : `docker build -t lucas/java-db-api .` 
- On run l'image comme la partie d'avant mais on rajoute bien le container dans le meme réseau que notre database : `docker run -d --name java-db-api -p "8091:8081" --network app-network lucas/java-db-api`
- On accède maintenant bien aux données de notre base de données à travers notre api : 

![alt text](/TP1/img/java-db-api-students.png)
![alt text](/TP1/img/java-db-api-departments.png)
![alt text](/TP1/img/java-db-api-custom.png)
</br>

## HTTP serve
- Récupération de la correction pour cette partie

## Docker compose
- On met en place une architecture semblable à celle de la correction avec nos fichiers avec du coup :
- - un dossier pour la db (le notre)
- - un dossier pour l'api (le notre)
- - un dossier pour l'httpd (correction)

- On prend le même docker compose que la correction
- On adapte certaine conf à notre contexte et notre projet notammenr 
- - `build: ./simple-api-student-main` dans le premier service
- - Et on change dans le conf du http les adresses de proxy car nous on les a forcé a 8081 donc elles doivent correspondre.
- Puis on supprime toutes nos images et nos container.
- Puis on run la commande `docker compose build` qui va créé toutes les images
- Puis la commande `docker compose up` qui elle va créé tous les conteneurs.