# TP1 

## Database
Ecriture du Dockerfile database
docker build -t yoannrobinet/database . : build l'image 

**Creation d'un réseau pour connecter les containers**
...
docker network create app-network 
...
**Récupération Docker Client SQL**
...
docker pull adminer
...
**Lancement des containers**
Database
...
docker run -d -e POSTGRES_PASSWORD=pwd -v /my/own/datadir:/var/lib/postgresql/data --network=app-network --name database yoannrobinet/database
Adminer
...
docker run --network=app-network -p 8080:8080 --name clientSQL adminer
...
**Insertion des scripts sql dans le docker**
...
COPY 01-CreateScheme.sql /docker-entrypoint-initdb.d/ 
COPY 02-InsertData.sql /docker-entrypoint-initdb.d/
...
**Suppression des containers**
...
sudo docker rm -f clientSQL 
sudo docker rm -f database
...
**Suppression des images**
...
sudo docker rmi database
...

## API
**Ecriture du Dockerfile**
...
FROM openjdk:11
COPY Main.java .
RUN javac Main.java
CMD ["java", "Main"]
...

***Lauch Application**
...
docker build -t yoannrobinet/api .
docker run -d --network=app-network -p 8081:8080 --name api yoannrobinet/api
...
***Dockerfile**
...
# Build
FROM maven:3.6.3-jdk-11 AS myapp-build 
ENV MYAPP_HOME /opt/myapp 
WORKDIR $MYAPP_HOME
COPY pom.xml .
COPY src ./src
RUN mvn package -DskipTests
# Run
FROM openjdk:11-jre
ENV MYAPP_HOME /opt/myapp
WORKDIR $MYAPP_HOME
COPY --from=myapp-build $MYAPP_HOME/target/*.jar $MYAPP_HOME/myapp.jar
ENTRYPOINT java -jar myapp.jar
...
Nous avons besoin d'un build multistage afin de configurer l'image Maven comme on le veut avant de la runer
**Lancement Simple Api**
...
docker pull maven
docker build -t yoannrobinet/simple-api .
docker run -d --network=app-network -p 8081:8080 --name api yoannrobinet/simple-api
...

## HTTP
***Lancement Api**
...
docker pull httpd
docker build -t yoannrobinet/httpd .
docker run -d --network=app-network -p 80:80 --name httpd yoannrobinet/httpd
...
**Reverse Proxy**
Récupération httpd.conf:
...
docker cp httpd:/usr/local/apache2/conf/httpd.conf .
...
Dans httpd.conf :
...
LoadModule proxy_module modules/mod_proxy.so
LoadModule proxy_http_module modules/mod_proxy_http.so

ServerName localhost
<VirtualHost *:80>
	ProxyPreserveHost On
	ProxyPass / http://api:8080/ 
	ProxyPassReverse / http://api:8080/
</VirtualHost>
...
**add au DockerFile**
...
COPY ./httpd.conf /usr/local/apache2/conf/httpd.conf
COPY index.html /usr/local/apache2/htdocs
...


## Docker compose

**Ecriture du docker compose**
...
version: '3.3'
services:
  api:
    build:
      ./api/simple-api-main/simple-api
    networks:
      - app-network
    depends_on:
      - database

  database:
    build:
      ./database
    networks:
      - app-network
    volumes:
      - /my/own/datadir:/var/lib/postgresql/data
  httpd:
    build:
      ./httpd
    ports:
      - 80:80
    networks:
      - app-network
    depends_on:
      - api

networks:
  app-network:
...
**Lancement du docker compose**
...
docker-compose up
...
