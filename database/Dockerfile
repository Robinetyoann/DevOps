FROM postgres:11.6-alpine
COPY 01-CreateScheme.sql /docker-entrypoint-initdb.d
COPY 02-InsertData.sql /docker-entrypoint-initdb.d
ENV POSTGRES_DB=db \ 
    POSTGRES_USER=usr
   
