FROM postgres:17.2-alpine

ENV POSTGRES_DB=db \
    POSTGRES_USER=usr \
    POSTGRES_PASSWORD=pwd

COPY CreateScheme.sql /docker-entrypoint-initdb.d/01-CreateScheme.sql
COPY InsertData.sql /docker-entrypoint-initdb.d/02-InsertData.sql
