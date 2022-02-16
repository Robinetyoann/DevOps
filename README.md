# TP1 - Docker

## Database
**Ecriture du Dockerfile database**
```Dockerfile
FROM postgres:11.6-alpine
ENV POSTGRES_DB=db \
POSTGRES_USER=usr \
POSTGRES_PASSWORD=pwd
```
**Build de l'image**
```console
docker build -t yoannrobinet/database . 
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
- Database
```console
docker run -d -e POSTGRES_PASSWORD=pwd -v /my/own/datadir:/var/lib/postgresql/data --network=app-network --name database yoannrobinet/database
```
- Adminer
```console
docker run --network=app-network -p 8080:8080 --name clientSQL adminer
```
**Insertion des scripts sql dans le docker**
Ajout dans le Dockerfile:
```Dockerfile
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
```Dockerfile
FROM openjdk:11
COPY Main.java .
RUN javac Main.java
CMD ["java", "Main"]
```

**Lauch Application**
```console
docker build -t yoannrobinet/api .
docker run -d --network=app-network -p 8081:8080 --name api yoannrobinet/api
```
**Dockerfile**
```Dockerfile
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
**Lancement Api**
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
```Yml
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

## TP2 - CI/CD
Question : 2-1 What are testcontainers?
Testcontainers est une librairie Java de tests unitaires, les tests unitaires permette de vérifier le bon fonctionnement d'une application en testant plus précisément chaque fonctions/méthodes.
Dans workflows > .main.yml :
```Yml
name: CI devops 2022 CPE 
on:
  #to begin you want to launch this job in main and develop
  push:
    branches: 
      - main
      - develop
  pull_request:
jobs: 
  test-backend:
    runs-on: ubuntu-18.04 
    steps:
          #checkout your github code using actions/checkout@v2.3.3
      - uses: actions/checkout@v2.3.3
      - name: Set up JDK 11
        uses: actions/setup-java@v2
        with:
          distribution: 'zulu' # See 'Supported distributions' for available options
          java-version: '11'
          cache: maven
          #do the same with another action (actions/setup-java@v2) that enable to setup jdk 11
      
          #finally build your app with the latest command
      - name: Build and test with Maven 
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}  # Needed to get PR information, if any
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }} 
        run: mvn -B verify sonar:sonar -Dsonar.projectKey=Robinetyoann_DevOps -Dsonar.organization=robinetyoann
            -Dsonar.host.url=https://sonarcloud.io -Dsonar.login=${{ secrets.SONAR_TOKEN }} --file ./api/simple-api-main/simple-api/pom.xml 
        #  run: mvn clean verify --file ./api/simple-api-main/simple-api/pom.xml 
  # define job to build and publish docker image
  build-and-push-docker-image:
    needs: test-backend
    # run only when code is compiling and tests are passing
    runs-on: ubuntu-latest
    # steps to perform in job
    steps:
      - name: Login to DockerHub
        run: docker login -u ${{ secrets.DOCKERHUB_USERNAME }} -p ${{ secrets.DOCKERHUB_TOKEN }}
      - name: Checkout code
        uses: actions/checkout@v2
      - name: Build image and push api
        uses: docker/build-push-action@v2
        with:
        # relative path to the place where source code with Dockerfile is located
          context: ./api/simple-api-main/simple-api/
        # Note: tags has to be all lower-case
          tags: ${{secrets.DOCKERHUB_USERNAME}}/tp-devops-cpe-api
          # build on feature branches, push only on main branch
          push: ${{ github.ref == 'refs/heads/main' }}
      - name: Build image and push database
        uses: docker/build-push-action@v2
        with:
        # relative path to the place where source code with Dockerfile is located
          context: ./database
        # Note: tags has to be all lower-case
          tags: ${{secrets.DOCKERHUB_USERNAME}}/tp-devops-cpe-database
          # build on feature branches, push only on main branch
          push: ${{ github.ref == 'refs/heads/main' }}

      - name: Build image and push httpd
        uses: docker/build-push-action@v2
        with:
        # relative path to the place where source code with Dockerfile is located
          context: ./httpd
        # Note: tags has to be all lower-case
          tags: ${{secrets.DOCKERHUB_USERNAME}}/tp-devops-cpe-httpd
          # build on feature branches, push only on main branch
          push: ${{ github.ref == 'refs/heads/main' }}
```

**Descripton de la configuration**
- **on** : Permet de spécifier quand est ce qu'on exécute les actions (ici au push)
- **push** > **branches**: Défini sur quelles branches pour l'action 'push' on exécute les actions (ici: test-backend et build-and-push-docker-image)
- **jobs**: Permet de décrire les différentes actions à éxécuter et les nommer ( ici: test-backend et build-and-push-docker-image)
- **needs**: Indique si une étape doit être fini avant l'execution du jobs
- **runs-on**: Indique avec quel système d'exploitation exécuter le job
- **steps**: Précise les différentes étapes d'une action/job (un job peut avoir plusieurs steps)
- **name**: défini le nom d'un step 
- **run**: Permet de lancer une commande dans le terminal
- **uses**: défini la tâche du step à effectuer
- **with**: Permet d'ajouter des paramètres/informations à la tâche
  - **context**: Renseigne la path relative au Dockerfile correspondant
  - **tags**: path de l'image dans Dockerhub
  - **push**: Spécifie quand effectuer le step

# TP3 - Ansible
**Ecriture de l'inventory**
Dans setup.yml:
```Yml
all:
  vars:
    ansible_user: centos
    ansible_ssh_private_key_file: ../id_rsa
  children:
    prod:
      hosts: yoann.robinet.takima.cloud
```
- **vars**: pour renseigner les informations de connection
- **children**: renseigner les serveurs de prod, dev, etc ...   

Pour tester l'inventory:
```Console
ansible all -i inventories/setup.yml -m ping
```

**Premier Playbook**   
Dans playbook.yml:   
```Yml
- hosts: all
  gather_facts: false
  become: yes

  tasks:
    - name: Test connection
      ping:
```

Lancer le playbook :  
Remarque: l'option **'--syntax-check'** permet de vérifier la syntax 
```Console
ansible-playbook -i inventories/setup.yml playbook.yml
```

**Playbook avancé**
```Yml
- hosts: all
  gather_facts: false
  become: yes

  roles:
    - docker
    - network
    - database
    - api
    - proxy
```
- **hosts**: Réference vers les hosts du setup.yml
- **become**: yes pour avoir les droits admin
- **roles**: Permet de scinder les étapes du playbook en plusieurs partie (tâches ordonnées)

**Creation d'un role**
```Console
  ansible-galaxy init roles/database
```

**Ecriture du role**   
Dans le fichier tasks/main.yml:
```Yml
---
# tasks file for roles/database
- name: Lauch database
  docker_container:
    name: database
    image: yoannrobinet/tp-devops-cpe-database
    state: started
    env:
      POSTGRES_DB: db
      POSTGRES_USR: usr
      POSTGRES_PASSWORD: pwd
    networks:
      - name: backNet
```
- **name**: Renseigne le nom de la tâche   
- **image**: Nom de l'image sur dockerHub
- **state**: Defini l'état du container
- **env**: Setup les variables d'environnement 
- **networks**: Ajout d'un réseau au container