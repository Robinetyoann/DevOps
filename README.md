# TP1 

## Database
Ecriture du Dockerfile database
```console
docker build -t yoannrobinet/database . : build l'image 
```
**Creation d'un réseau pour connecter les containers**
```console
docker network create app-network 
```
**Récupération Docker Client SQL**
```console
docker pull adminer
```
**Lancement des containers**
Database
```console
docker run -d -e POSTGRES_PASSWORD=pwd -v /my/own/datadir:/var/lib/postgresql/data --network=app-network --name database yoannrobinet/database
Adminer
```console
docker run --network=app-network -p 8080:8080 --name clientSQL adminer
```
**Insertion des scripts sql dans le docker**
```console
COPY 01-CreateScheme.sql /docker-entrypoint-initdb.d/ 
COPY 02-InsertData.sql /docker-entrypoint-initdb.d/
```
**Suppression des containers**
```console
sudo docker rm -f clientSQL 
sudo docker rm -f database
```
**Suppression des images**
```console
sudo docker rmi database
```

## API
**Ecriture du Dockerfile**
```console
FROM openjdk:11
COPY Main.java .
RUN javac Main.java
CMD ["java", "Main"]
```

***Lauch Application**
```console
docker build -t yoannrobinet/api .
docker run -d --network=app-network -p 8081:8080 --name api yoannrobinet/api
```
***Dockerfile**
```console
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
```
Nous avons besoin d'un build multistage afin de configurer l'image Maven comme on le veut avant de la runer
**Lancement Simple Api**
```console
docker pull maven
docker build -t yoannrobinet/simple-api .
docker run -d --network=app-network -p 8081:8080 --name api yoannrobinet/simple-api
```

## HTTP
***Lancement Api**
```console
docker pull httpd
docker build -t yoannrobinet/httpd .
docker run -d --network=app-network -p 80:80 --name httpd yoannrobinet/httpd
```
**Reverse Proxy**
Récupération httpd.conf:
```console
docker cp httpd:/usr/local/apache2/conf/httpd.conf .
```
Dans httpd.conf :
```console
LoadModule proxy_module modules/mod_proxy.so
LoadModule proxy_http_module modules/mod_proxy_http.so

ServerName localhost
<VirtualHost *:80>
	ProxyPreserveHost On
	ProxyPass / http://api:8080/ 
	ProxyPassReverse / http://api:8080/
</VirtualHost>
```
**add au DockerFile**
```console
COPY ./httpd.conf /usr/local/apache2/conf/httpd.conf
COPY index.html /usr/local/apache2/htdocs
```


## Docker compose

**Ecriture du docker compose**
```console
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
```
**Lancement du docker compose**
```console
docker-compose up
```
