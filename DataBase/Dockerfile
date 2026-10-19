FROM postgres:17.2-alpine

COPY CreateScheme.sql /docker-entrypoint-initdb.d/01-CreateScheme.sql
COPY InsertData.sql /docker-entrypoint-initdb.d/02-InsertData.sql
