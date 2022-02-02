#TP1 

##Database

1-1.
docker build -t yoannrobinet/database . --> in folder tp01
docker pull adminer
docker network create app-network --> crait le réseau app-network pour connecter les containers 
docker run -d -e POSTGRES_PASSWORD=pwd -v /my/own/datadir:/var/lib/postgresql/data --network=app-network --name database yoannrobinet/database
docker run --network=app-network -p 8080:8080 --name clientSQL adminer

##API

docker pull openjdk

docker build -t yoannrobinet/api .
docker run -d --network=app-network -p 8081:8080 --name api yoannrobinet/api

docker pull maven
docker build -t yoannrobinet/simple-api .
docker run -d --network=app-network -p 8081:8080 --name api yoannrobinet/simple-api


#HTPP
docker pull httpd
docker build -t yoannrobinet/httpd .
docker run -d --network=app-network -p 80:80 --name httpd yoannrobinet/httpd
docker cp httpd:/usr/local/apache2/conf/httpd.conf .

LoadModule proxy_module modules/mod_proxy.so
LoadModule proxy_http_module modules/mod_proxy_http.so

ServerName localhost
<VirtualHost *:80>
	ProxyPreserveHost On
	ProxyPass / http://api:8080/ 
	ProxyPassReverse / http://api:8080/
</VirtualHost>
add to DockerFile : 
    COPY ./httpd.conf /usr/local/apache2/conf/httpd.conf
    COPY index.html /usr/local/apache2/htdocs
rebuilt and rerun

#Docker compose
docker-compose up